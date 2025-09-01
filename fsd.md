# zenv - Functional Specification Document (Comprehensive)

## 1. Introduction

This document provides the functional specifications for the `zenv` command-line utility, corresponding to the **Architecture Design Document**. `zenv` acts as a wrapper for system package managers (e.g., DNF, APT, Pip) to maintain a persistent SQLite ledger of package operations and command history, providing a detailed audit trail. It tracks rich metadata including version, manager, origin, installation reason, location, checksums, signatures, license, and size. **User-defined tags are stored in a normalized structure for efficient querying and management.** This document details the expected behavior of each component and command, centered around a new **Service Layer**.

## 2. Core Data Structures (`internal/ledger/models.go`)

This section defines the Go structs that directly map to the columns in the SQLite database tables. These structs are used for transferring data between the `ledger.Store` layer and the `service` layer.

*   **Nullable Field Handling:** Pointers (`*string`, `*int64`, `*time.Time`) are used to represent nullable database columns, distinguishing between zero values and actual `NULL` values. The `ledger.Store` layer handles conversions between these pointers and `sql.Null*` types.
*   **Struct Tags:** `db:"..."` struct tags are included for potential use with libraries like `sqlx`.

```go
package ledger

import (
	"database/sql" // Only needed if using sql.Null* directly in models
	"time"
)

// Represents an entry in the command_history table
// Corresponds to target database schema.
type CommandHistory struct {
	CommandID       int64      `db:"command_id"`        // Primary Key: Auto-incrementing ID for the command invocation.
	StartTimestamp  time.Time  `db:"start_timestamp"`   // Time command execution started (UTC): Recorded when the service layer method begins.
	EndTimestamp    *time.Time `db:"end_timestamp"`     // Time command execution finished (UTC), Nullable: Recorded on completion or error. NULL indicates interruption.
	ZenvVersion     string     `db:"zenv_version"`      // Version of zenv binary used: Obtained via build flags.
	CommandString   string     `db:"command_string"`    // Full command line invocation of zenv: Includes all args/flags.
	PMCommandString *string    `db:"pm_command_string"` // Exact underlying PM command executed, Nullable: Set if a PM command was run.
	ExitCode        *int       `db:"exit_code"`         // Final exit code of the zenv process, Nullable: 0 for success, non-zero for failure.
	ErrorMessage    *string    `db:"error_message"`     // User-facing error message if command failed, Nullable.
	Details         *string    `db:"details"`           // Additional context (e.g., JSON sync report), Nullable.
}

// Represents an entry in the current_packages table
// Corresponds to target database schema (normalized tags).
type CurrentPackage struct {
	PackageName      string    `db:"package_name"`        // Name of the package (Part of PK): Canonical name.
	Manager          string    `db:"manager"`             // Manager responsible (Part of PK): e.g., "dnf", "pip", "manual".
	Origin           *string   `db:"origin"`              // Source detail (repo, URL, path), Nullable: Best-effort based on manager.
	PackageVersion   *string   `db:"package_version"`     // Installed version string, Nullable.
	InstallReason    *string   `db:"install_reason"`      // "user", "dependency", "discovered", Nullable: Best-effort based on manager/context.
	Location         *string   `db:"location"`            // Filesystem path for manual entries, Nullable.
	Comment          *string   `db:"comment"`             // User comment, Nullable.
	InstallTimestamp *time.Time `db:"install_timestamp"`   // Original installation time (UTC), Nullable: Best-effort from PM.
	LastUpdatedTimestamp time.Time `db:"last_updated_timestamp"` // Time this state record was last updated by zenv (UTC): Set on create/update by zenv.
	Checksum         *string   `db:"checksum"`            // Package checksum (e.g., "sha256:<hash>"), Nullable: Best-effort based on manager.
	Signature        *string   `db:"signature"`           // Signature info (e.g., GPG Key ID), Nullable: Best-effort based on manager.
	License          *string   `db:"license"`             // License identifier (e.g., "MIT"), Nullable: Best-effort based on manager.
	Size             *int64    `db:"size"`                // Installed size in bytes, Nullable: Best-effort based on manager.
}

// Represents an entry in the package_log table
// Corresponds to target database schema (normalized tags).
type LogEntry struct {
	ID             int64     `db:"id"`                  // Primary Key: Auto-incrementing ID for the log event.
	CommandID      *int64    `db:"command_id"`          // FK to command_history, Nullable: Links to the initiating zenv command.
	Timestamp      time.Time `db:"timestamp"`           // Time of the specific package event (UTC): When the action was logged by zenv.
	Action         string    `db:"action"`              // Type of event: e.g., "installed", "removed", "tags_updated". See constants below.
	PackageName    string    `db:"package_name"`        // Name of the package affected.
	Manager        string    `db:"manager"`             // Manager associated at the time of the event.
	Origin         *string   `db:"origin"`              // Origin at the time of the event, Nullable.
	PackageVersion *string   `db:"package_version"`     // Version relevant to the event, Nullable.
	InstallReason  *string   `db:"install_reason"`      // Install reason at the time of the event, Nullable.
	Location       *string   `db:"location"`            // Location at the time of the event, Nullable.
	Comment        *string   `db:"comment"`             // Comment relevant to the event (e.g., from -c flag), Nullable.
	Checksum       *string   `db:"checksum"`            // Checksum at the time of the event, Nullable.
	Signature      *string   `db:"signature"`           // Signature info at the time of the event, Nullable.
	License        *string   `db:"license"`             // License at the time of the event, Nullable.
	Size           *int64    `db:"size"`                // Size at the time of the event, Nullable.
}

// Represents an entry in the tags table (New for normalized tags)
type Tag struct {
	TagID int64  `db:"tag_id"` // Primary Key: Auto-incrementing ID for the tag.
	Name  string `db:"name"`   // Unique tag name (case-insensitive): The actual tag string.
}

// Represents an entry in the package_tags linking table (New for normalized tags)
type PackageTag struct {
	PackageName string `db:"package_name"` // Part of PK & FK: References current_packages.package_name.
	Manager     string `db:"manager"`      // Part of PK & FK: References current_packages.manager.
	TagID       int64  `db:"tag_id"`       // Part of PK & FK: References tags.tag_id.
}

// Constants for Action field values in LogEntry
const (
	ActionInit           = "init"             // Database initialized
	ActionInstalled      = "installed"        // Package installed via zenv install
	ActionRemoved        = "removed"          // Package removed via zenv remove
	ActionCommentChanged = "comment_changed"  // Comment updated via zenv comment
	ActionTagsUpdated    = "tags_updated"     // Tags updated via zenv tag (applied/removed)
	ActionManualAdd      = "manual_add"       // Package manually added via zenv add-manual
	ActionManualRemoveLog= "manual_remove_log"// Entry manually removed via zenv remove-manual-log
	ActionSyncAdd        = "sync_add"         // Package added to ledger by zenv sync --apply
	ActionSyncRemove     = "sync_remove"      // Package removed from ledger by zenv sync --apply
	ActionSyncUpdate     = "sync_update"      // Package details updated by zenv sync --apply
)

// Constants for common Manager field values (Examples - not exhaustive)
const (
	ManagerDNF        = "dnf"
	ManagerAPT        = "apt"
	ManagerPacman     = "pacman"
	ManagerPip        = "pip"
	ManagerNPM        = "npm"
	ManagerCargo      = "cargo"
	ManagerGem        = "gem"
	ManagerBrew       = "brew"
	ManagerManual     = "manual"       // User-added, non-PM managed item
	ManagerAppImage   = "appimage"     // Specific type of manual item
	ManagerBinary     = "binary"       // Specific type of manual item
	ManagerDiscovered = "discovered"   // Found by initial discover/sync, manager unknown/irrelevant (*Note: Clarify usage*)
)

// Constants for InstallReason field values
const (
	ReasonUser       = "user"       // Directly requested by the user for install
	ReasonDependency = "dependency" // Pulled in as a dependency of a user-requested package
	ReasonDiscovered = "discovered" // Found via sync/discover, original reason unknown
)

```

## 3. Database Management (`internal/ledger/db.go`)

This component is responsible for establishing the connection to the SQLite database file and ensuring the schema is correctly initialized and migrated to the version expected by the application code using `golang-migrate/migrate`.

*   **Database File:**
    *   **Location Determination:** `util/fs.GetDBPath()` determines the path using the XDG Base Directory Specification: `$XDG_DATA_HOME/zenv/ledger.db` is preferred, falling back to `$HOME/.local/share/zenv/ledger.db` if `$XDG_DATA_HOME` is unset or empty. The function returns the determined absolute path.
    *   **Directory Creation:** `util/fs.EnsureDir()` is called with the directory containing the database path before opening. It creates the directory (`.../zenv/`) and any parent directories if they do not exist, using appropriate permissions (e.g., `0700` or `0755`).
*   **Connection:**
    *   The standard Go `database/sql` package is used for database interactions.
    *   The `github.com/mattn/go-sqlite3` driver provides the necessary SQLite implementation.
    *   `func OpenDB(dbPath string) (*sql.DB, error)`: This function opens the SQLite database file specified by `dbPath`. It configures the connection pool settings, typically setting `SetMaxOpenConns(1)` as recommended for SQLite write safety to avoid potential locking issues. It returns the `*sql.DB` connection pool handle or an error if the database cannot be opened.
    *   `func CloseDB(db *sql.DB)`: This function safely closes the database connection pool using `db.Close()`. It should be called during graceful application shutdown (e.g., using `defer` in `main.go`).
*   **Initialization & Schema Migration (`func InitializeDB(db *sql.DB) error`):**
    *   **Purpose:** To ensure the database schema is at the version required by the current version of the `zenv` application. This involves creating the initial schema if the database is new or applying sequential migrations if the database exists but is at an older schema version. This process is handled by the `golang-migrate/migrate` library.
    *   **Invocation:** This function must be called early in the application lifecycle, typically by the `zenv init` command's service logic. Other commands might also call it at startup to ensure the database is usable and perform any necessary migrations automatically before proceeding.
    *   **Migration Source:** The SQL migration files (e.g., `001_initial_schema.up.sql`, `001_initial_schema.down.sql`, `002_...`) defining the schema changes between versions are located in the `internal/ledger/migrations/` directory. These files will be embedded into the compiled `zenv` binary using Go's `//go:embed` directive.
    *   **Steps:**
        1.  **Enable Foreign Keys:** Execute `PRAGMA foreign_keys = ON;` on the provided `db` connection. This is essential for maintaining relational integrity between tables like `package_log` and `command_history`.
        2.  **Instantiate Migration Source:** Create a migration source driver compatible with `golang-migrate` using the embedded filesystem (`embed.FS`) containing the migration `.sql` files. This is typically done using `migrate/source/iofs`.
        3.  **Instantiate DB Driver:** Create a database driver instance compatible with `golang-migrate` specifically for SQLite, using the existing `*sql.DB` connection. This is typically done using `migrate/database/sqlite3`.
        4.  **Instantiate `migrate`:** Create a `migrate.Migrate` instance, providing the source driver and the database driver.
        5.  **Apply Migrations:** Call the `migrate.Up()` method on the `migrate.Migrate` instance. This core function performs the following:
            *   Checks the database for the existence and state of the `schema_migrations` table (which the library manages internally).
            *   Determines the current schema version based on the `schema_migrations` table.
            *   Compares the current version to the versions defined by the embedded `*.up.sql` files.
            *   Applies any pending `*.up.sql` files in sequential order, updating the `schema_migrations` table after each successful step.
        6.  **Error Handling:** Check the error returned by `migrate.Up()`. If the error is `migrate.ErrNoChange`, it means the schema is already up-to-date, which is not an actual error condition. Any other error indicates a failed migration. The error must be logged with details (it often contains information about which migration failed and why). A migration failure is critical and should likely prevent the application from proceeding with operations that depend on the correct schema.
        7.  **Logging:** Log informational messages indicating whether migrations were applied or if the schema was already current.
    *   **Backup Requirement:** Documentation must strongly emphasize the user requirement to back up their `ledger.db` file before upgrading `zenv` versions. Automated migrations are powerful but can be complex to roll back perfectly depending on the changes made. A backup is the safest recovery mechanism.

*   **Target Schema Definition:** (The functional specification relies on the detailed schema definition provided in the Architecture Design Document, Section 4.1, which includes `command_history`, `current_packages` (with `install_timestamp` and `last_updated_timestamp`), `package_log`, `tags`, `package_tags` tables, their columns, types, constraints, and indexes reflecting the normalized tag structure).

## 4. Ledger Store (`internal/ledger/store.go`)

This component provides the Data Access Object (DAO) layer, abstracting the SQL interactions needed to manage the `zenv` ledger database. It ensures that other parts of the application (primarily the service layer) interact with the database through a well-defined Go API, rather than embedding raw SQL.

*   **`Store` Struct:** Holds the `*sql.DB` connection pool.
    ```go
    type Store struct {
        db *sql.DB
    }
    ```
*   **Constructor:** `func NewStore(db *sql.DB) *Store`: Simple constructor to initialize the store with the database connection pool.
*   **`Querier` Interface:** Defined to allow methods to accept either a `*sql.DB` (for read-only operations outside explicit transactions) or a `*sql.Tx` (for operations within a service-managed transaction).
    ```go
    type Querier interface {
        QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error)
        QueryRowContext(ctx context.Context, query string, args ...interface{}) *sql.Row
        ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error)
    }
    ```
*   **General Principles:**
    *   **SQL Encapsulation:** All SQL query strings reside within this package. Prepared statements should be considered for frequently executed queries.
    *   **Context Propagation:** All query/exec methods accept and use `context.Context` for cancellation and deadline propagation.
    *   **Transaction Handling:** Write operations (`INSERT`, `UPDATE`, `DELETE`, `REPLACE`) must accept a `*sql.Tx` argument and execute using `tx.ExecContext` or `tx.QueryRowContext`. They do not manage the transaction lifecycle (Begin, Commit, Rollback); this is the responsibility of the calling Service layer.
    *   **Read Operations:** Read operations accept the `Querier` interface, allowing them to run within an existing transaction (`*sql.Tx`) or directly on the connection pool (`*sql.DB`).
    *   **Normalized Tag Handling:** Specific methods handle interactions with the `tags` and `package_tags` tables. Queries that need to filter by or display tags require appropriate `JOIN` clauses.
    *   **Nullable Field Handling:** Uses pointer types (`*string`, `*int64`, `*time.Time`) in Go structs (`models.go`). Methods are responsible for correctly handling the conversion between these pointers and `sql.Null*` types when scanning results or binding parameters for inserts/updates.
    *   **Error Handling:** Methods return standard Go errors. Underlying `database/sql` errors (e.g., `sql.ErrNoRows`, constraint violations, connection errors) must be wrapped with context (using `fmt.Errorf("...: %w", err)`). Specific custom errors (like `util.ErrPackageNotFound`) are returned when appropriate (e.g., `sql.ErrNoRows` on single-row lookups like `GetCurrentPackage`). Specific errors indicating database corruption should be returned from `CheckDatabaseIntegrity`.
*   **Filter Struct (`QueryFilters`):** Used by query methods (`GetCurrentPackages`, `GetHistory`, report queries) to dynamically build `WHERE` clauses.
    ```go
    type QueryFilters struct {
        Reasons      []string   // Filter by install_reason (e.g., ["user", "dependency"])
        Managers     []string   // Filter by manager (e.g., ["dnf", "pip"])
        Origins      []string   // Filter by origin (likely exact match)
        Tags         []string   // Filter by packages having ALL specified tags (requires JOINs and potentially GROUP BY/HAVING)
        Since        *time.Time // Filter for events/updates after or at this time (compares against last_updated_timestamp)
        Before       *time.Time // Filter for events/updates before or at this time (compares against last_updated_timestamp)
        PackageNames []string   // Filter by specific package names
        // Add other potential filters as needed: License, Action, etc.
    }
    // Associated helper methods within store.go build the WHERE clause string
    // and the corresponding argument slice safely to prevent SQL injection.
    ```
*   **Specific Method Implementations (Details):**
    *   **Command History Methods:**
        *   `func (s *Store) AddCommandStart(tx *sql.Tx, startTime time.Time, zenvVersion, cmdString string) (int64, error)`: Executes `INSERT INTO command_history (start_timestamp, zenv_version, command_string) VALUES (?, ?, ?)`. Returns the generated `command_id`.
        *   `func (s *Store) UpdateCommandEnd(tx *sql.Tx, commandID int64, endTime time.Time, exitCode int, errMsg *string, details *string) error`: Executes `UPDATE command_history SET end_timestamp = ?, exit_code = ?, error_message = ?, details = ? WHERE command_id = ?`. Uses `sql.Null*` for nullable fields.
        *   `func (s *Store) UpdateCommandPMString(tx *sql.Tx, commandID int64, pmCmd string) error`: Executes `UPDATE command_history SET pm_command_string = ? WHERE command_id = ?`.
        *   `func (s *Store) GetCommandHistory(db Querier, filters QueryFilters) ([]CommandHistory, error)`: Executes `SELECT * FROM command_history`. Applies filters dynamically. Orders results `ORDER BY start_timestamp DESC`. Scans rows into `[]CommandHistory`.

    *   **Package Log Write Methods:**
        *   `func (s *Store) AddLogEntry(tx *sql.Tx, commandID *int64, entry ledger.LogEntry) error`: Executes `INSERT INTO package_log (command_id, timestamp, action, package_name, ...) VALUES (?, ?, ?, ?, ...)`. Maps fields from the `LogEntry` struct (handling pointers/nullables) to parameters.

    *   **Current Packages Write Methods:**
        *   `func (s *Store) ReplaceCurrentPackage(tx *sql.Tx, pkg ledger.CurrentPackage) error`: Executes `INSERT OR REPLACE INTO current_packages (package_name, manager, origin, ..., install_timestamp, last_updated_timestamp) VALUES (?, ?, ?, ..., ?, ?)`. Maps fields from `CurrentPackage` struct, including both timestamps.
        *   `func (s *Store) DeleteCurrentPackage(tx *sql.Tx, packageName, manager string) error`: Executes `DELETE FROM current_packages WHERE package_name = ? AND manager = ?`. Returns error if `RowsAffected` is 0.
        *   `func (s *Store) UpdateCurrentPackageComment(tx *sql.Tx, packageName, manager string, comment *string) error`: Executes `UPDATE current_packages SET comment = ?, last_updated_timestamp = ? WHERE package_name = ? AND manager = ?`. **Must also update `last_updated_timestamp`**. Returns error if `RowsAffected` is 0.

    *   **Tag Methods:**
        *   `func (s *Store) FindOrCreateTag(tx *sql.Tx, tagName string) (int64, error)`: Attempts `SELECT tag_id FROM tags WHERE name = ? COLLATE NOCASE`. If `sql.ErrNoRows`, executes `INSERT INTO tags (name) VALUES (?)` and returns `LastInsertId`. Otherwise, returns the found `tag_id`.
        *   `func (s *Store) AddTagToPackage(tx *sql.Tx, packageName, manager string, tagID int64) error`: Executes `INSERT INTO package_tags (package_name, manager, tag_id) VALUES (?, ?, ?) ON CONFLICT(package_name, manager, tag_id) DO NOTHING`. **Must also update `last_updated_timestamp` on the corresponding `current_packages` row.**
        *   `func (s *Store) RemoveTagFromPackage(tx *sql.Tx, packageName, manager string, tagID int64) error`: Executes `DELETE FROM package_tags WHERE package_name = ? AND manager = ? AND tag_id = ?`. **Must also update `last_updated_timestamp` on the corresponding `current_packages` row if the delete was successful.** Optionally check `RowsAffected`.
        *   `func (s *Store) GetTagsForPackage(db Querier, packageName, manager string) ([]Tag, error)`: Executes `SELECT t.tag_id, t.name FROM tags t JOIN package_tags pt ON t.tag_id = pt.tag_id WHERE pt.package_name = ? AND pt.manager = ? ORDER BY t.name`. Scans results into `[]Tag`.

    *   **Read Methods:**
        *   `func (s *Store) GetCurrentPackage(db Querier, packageName, manager string) (*ledger.CurrentPackage, error)`: Executes `SELECT * FROM current_packages WHERE package_name = ? AND manager = ? LIMIT 1`. Uses `QueryRowContext`. Handles `sql.ErrNoRows` by returning `util.ErrPackageNotFound`. Scans results into struct (handling pointers/nullables for `install_timestamp` and others).
        *   `func (s *Store) GetCurrentPackages(db Querier, filters QueryFilters) ([]ledger.CurrentPackage, error)`: Executes `SELECT cp.* FROM current_packages cp`. Dynamically builds `WHERE` clause based on `filters`. If `filters.Tags` is populated, adds necessary `JOIN`s and conditions (e.g., `WHERE t.name IN (...) GROUP BY ... HAVING COUNT(...) = ?`). Appends `ORDER BY ...`. Executes `QueryContext`, iterates `rows.Scan` into `[]CurrentPackage`.
        *   `func (s *Store) GetHistory(db Querier, filters QueryFilters) ([]ledger.LogEntry, error)`: Executes `SELECT * FROM package_log`. Dynamically builds `WHERE` clause. Appends `ORDER BY timestamp DESC`. Executes `QueryContext`, scans rows into `[]LogEntry`.
        *   **(Report Support)** Specific methods (e.g., `GetPackageCountsByManager`) execute tailored SQL queries using appropriate aggregates (`COUNT`, `SUM`), grouping, ordering, and limiting, scanning results into dedicated report structs.

    *   **Database Integrity Check:**
        *   `func (s *Store) CheckDatabaseIntegrity(ctx context.Context, db Querier) (string, error)`: Executes `PRAGMA integrity_check;` using `QueryRowContext`. Returns the result string (e.g., "ok" or error description) or wraps any execution error.

## 5. Result Formatting (`internal/ledger/format.go`)

This component transforms structured data retrieved from the `Ledger Store` (slices of `CurrentPackage`, `LogEntry`, `CommandHistory`, or custom report structs) into user-readable output formats (`table`, `json`, `csv`) requested via the CLI.

*   **Primary Function:** `func FormatOutput(data interface{}, formatType string, viewType string) (string, error)` handles the logic.
    *   `data interface{}`: Accepts the slice of data (e.g., `[]CurrentPackage`, `[]LogEntry`, `[]CommandHistory`, `[]SizeReportEntry`). Type assertion or reflection determines the concrete type based on `viewType`.
    *   `formatType string`: Specifies the desired output format (`table`, `json`, `csv`).
    *   `viewType string`: Specifies the context or type of data (e.g., `installed`, `removed`, `history`, `command_history`, `report-disk-usage`) to select appropriate columns/headers for table/CSV and potentially influence JSON structure.
    *   Returns the formatted string or an error.
*   **Table Format (`formatType="table"`):
    *   **Implementation:** Uses the standard library's `text/tabwriter` for aligned columns.
    *   **Headers:** Defines header strings based on `viewType`. Examples:
        *   `installed`: `PACKAGE | MANAGER | ORIGIN | VERSION | REASON | TAGS | LOCATION | COMMENT | ORIG_INSTALL | LAST_UPDATED | ...` (Headers clearly distinguish timestamps)
        *   `history`: `TIMESTAMP | ACTION | PACKAGE | MANAGER | ORIGIN | VERSION | REASON | ... | CMD_ID`
        *   `command_history`: `CMD_ID | START_TIME | END_TIME | ZENV_VER | COMMAND | PM_COMMAND | EXIT_CODE | ERROR`
        *   `report-disk-usage`: `PACKAGE | MANAGER | SIZE | ...`
    *   **Data Formatting:** Iterates through the input slice. For each struct:
        *   Formats its fields into tab-separated values matching the header order.
        *   Handles `nil` pointers appropriately (e.g., display `original_install_timestamp` as "-" or empty string if NULL).
        *   Formats both `original_install_timestamp` (nullable) and `last_updated_timestamp` (non-null) consistently (e.g., ISO 8601).
        *   **Tag Handling:** For views like `installed`, the `Store` must provide the associated tag data. This layer formats the tag names into a single display string (e.g., comma-separated, sorted alphabetically: `dev,go,web`).
    *   **Output:** Writes headers and formatted data rows to the `tabwriter`, calls `Flush()`.
*   **JSON Format (`formatType="json"`):
    *   **Implementation:** Uses `encoding/json`.
    *   **Logic:** Marshals the input data slice using `json.MarshalIndent`. The Go structs passed here should include both `OriginalInstallTimestamp` (nullable) and `LastUpdatedTimestamp`, along with associated `Tags` (e.g., as `[]string`) for relevant views like `installed`.
    *   **Output:** Returns the JSON string or marshalling error.
*   **CSV Format (`formatType="csv"`):
    *   **Implementation:** Uses `encoding/csv` writing to a `bytes.Buffer`.
    *   **Logic:**
        1.  Writes the header row based on `viewType`, including distinct headers for both timestamps.
        2.  Iterates through the input slice. For each struct, creates a slice of strings for its field values, handling `nil` timestamp pointers as empty strings.
        3.  **Tag Handling:** For views like `installed`, formats associated tag names into a single string field (e.g., comma-separated) for the CSV row.
        4.  Writes each data row using `csvWriter.Write()`.
        5.  Calls `csvWriter.Flush()`.
    *   **Output:** Returns the buffer's content as a string or any CSV writing error.

## 6. Package Manager Abstraction (`internal/pkgmgr/`)

Defines interface/adapters for interacting with package managers. It isolates the service layer from the details of how package information is retrieved or how install/remove operations are performed.

*   **Interface (`pkgmgr.go`):
    *   Defines the `PackageManager` interface that all adapters must implement.
    *   **`PackageInfo` Struct:** Defines the rich, structured data returned by query methods. All potentially unavailable fields must be pointers (`*string`, `*int64`, `*time.Time`) to represent `NULL`.
        ```go
        type PackageInfo struct {
            Name           string
            Version        *string
            Manager        string // Should match the adapter's Name()
            Origin         *string
            InstallReason  *string
            InstallTime    *time.Time
            Checksum       *string
            Signature      *string
            License        *string
            Size           *int64
        }
        ```
    *   **`Capability` Constants:** Defines string constants for capability keys (e.g., `CapProvidesReason`, `CapProvidesOrigin`, `CapProvidesChecksum`, etc.) used in the `Capabilities()` map.
    *   **`PackageManager` Interface Methods:**
        *   `Install(packages []string, pmOptions []string) (cmdExecuted string, err error)`:
            *   Constructs and executes the appropriate install command for the underlying manager (e.g., `sudo dnf install -y ...`), including any pass-through `pmOptions`.
            *   Must accept `io.Writer` instances (passed via context or functional options) for streaming stdout and stderr directly from the `os/exec` command.
            *   Returns the exact command string executed (`cmdExecuted`) for logging in `command_history`.
            *   Returns `nil` on success.
            *   Returns a specific error (e.g., wrapping `util.ErrPkgMgrFailed`) if the command fails (non-zero exit), potentially wrapping context from stderr.
        *   `Remove(packages []string, pmOptions []string) (cmdExecuted string, err error)`:
            *   Constructs and executes the appropriate removal command.
            *   Streams stdout/stderr.
            *   Returns the executed command string (`cmdExecuted`).
            *   Returns `nil` on success or an error (e.g., wrapping `util.ErrPkgMgrFailed`) on failure.
        *   `GetAllInstalledPackages() ([]PackageInfo, error)`:
            *   Queries the system (e.g., `rpm -qa`, `dpkg -l`, `pip list`) to get a list of all packages managed by this specific tool.
            *   For each package, gathers as much metadata as possible (version, origin, reason, checksum, license, size, install time, etc.) using relevant commands or file parsing. This is **best-effort**.
            *   Populates and returns a slice of `PackageInfo` structs.
            *   Returns an error if the underlying query fails fundamentally.
        *   `GetPackageDetails(packageNames []string) ([]PackageInfo, error)`:
            *   Potentially a more targeted version of `GetAllInstalledPackages` if querying all is excessively slow for a specific manager. Queries details only for the requested package names.
            *   Returns a slice of `PackageInfo` (may have fewer elements than requested if some packages aren't found).
            *   Returns an error if the query fails.
        *   `Name() string`: Returns the canonical, lowercase string identifier for this package manager (e.g., "dnf", "pip", "apt"). Used as the value in the `manager` column in the database.
        *   `Capabilities() map[string]bool`: Returns a map where keys are `CapProvides*` constants and values are `true` if the adapter implements logic to *attempt* fetching that piece of metadata, `false` otherwise. Used by service/UI layers to understand data availability.
        *   `CheckAvailability() error`: Checks if the underlying package manager command is available and potentially executable on the system (e.g., using `exec.LookPath`). Returns `nil` if available, or an error (e.g., wrapping `util.ErrPkgMgrNotFound`) if not available.

*   **DNF Implementation (`dnf/dnf.go`):
    *   Implements the `PackageManager` interface.
    *   `Name()` returns `"dnf"`.
    *   `Install`/`Remove` use `os/exec` to run `sudo dnf install/remove -y ...`, connecting the provided `io.Writer`s for stdout/stderr streaming. They capture the exact command string executed. **Before execution, they detect and log the DNF/RPM version to the application log (`applog`).** They check the command's exit code and return `nil` or a wrapped `util.ErrPkgMgrFailed`.
    *   `GetAllInstalledPackages` likely uses `rpm -qa --qf '{...}'` with a comprehensive query format string to extract name, version, install time, vendor (for origin), license, size. May use heuristics or other commands (`dnf repoquery`) to infer `install_reason`. Checksum/signature might require `rpm -V` or individual package queries. Populates `PackageInfo` on a best-effort basis, setting fields to `nil` if data is unavailable or parsing fails for an optional field.
    *   `Capabilities()` returns `true` for fields it attempts to gather (e.g., version, origin(vendor), size, license, install_time) and `false` for others it doesn't reliably provide (e.g., checksum, signature might be false initially).
    *   `CheckAvailability()` executes `exec.LookPath("dnf")`. Returns `nil` if found, otherwise an error.

*   **Other Implementations (APT, Pip, etc.):**
    *   Follow the same pattern, implementing the `PackageManager` interface.
    *   Commands and parsing logic are specific to the manager (e.g., `apt-cache`, `dpkg-query`, `pip show`, `npm ls`).
    *   Each implementation must log its respective PM's version to `applog` during operations.
    *   Metadata extraction (`GetAllInstalledPackages`, `GetPackageDetails`) quality will vary.
    *   `Capabilities()` must accurately reflect the implemented metadata fetching attempts.
    *   `CheckAvailability()` must check for the correct executable.

## 7. Error Handling and Usability

Details the strategy for handling errors consistently across the application and ensuring a usable command-line experience.

*   **Layered Error Handling:** Errors are handled and potentially wrapped at each layer to provide context while ensuring user-facing messages are clear.
    *   **`pkgmgr` Layer:**
        *   Detects non-zero exit codes from external package manager commands.
        *   Parses stderr for known critical error patterns if necessary.
        *   Returns specific, wrapped errors (e.g., `fmt.Errorf("%w: dnf command failed: %v", util.ErrPkgMgrFailed, stderr)`). Use `util.ErrPkgMgrNotFound` from `CheckAvailability`.
        *   Returns `util.ErrNotFound` if a query indicates a package doesn't exist (distinct from command failure).
    *   **`ledger.Store` Layer:**
        *   Wraps generic `database/sql` errors with context (e.g., `fmt.Errorf("failed to query current packages: %w", err)`).
        *   Returns `util.ErrPackageNotFound` specifically when `sql.ErrNoRows` occurs on single-row lookups (e.g., `GetCurrentPackage`).
        *   Returns specific errors for known constraint violations if needed (e.g., `util.ErrDuplicateTag`).
        *   Returns `util.ErrDBNeedsBackupRestore` or similar if `CheckDatabaseIntegrity` detects corruption.
    *   **`Service` Layer:**
        *   Acts as the central coordinator.
        *   **Catches** errors from `pkgmgr` and `store`.
        *   **Logs** detailed internal errors to `applog` at `ERROR` level (including wrapped error details, relevant arguments like package names, command ID).
        *   **Manages Transactions:** Ensures `tx.Rollback()` is called via `defer` if any error occurs within a transaction block. Handles `tx.Commit()` errors (which also trigger rollback).
        *   **Handles Migration Errors:** Catches errors returned by `golang-migrate/migrate` during `InitializeDB`, logs them, and prevents the application from proceeding if the schema is not usable.
        *   **Command History Update:** Ensures the final status (`end_timestamp`, `exit_code`, `error_message`) of a command is recorded in `command_history`, even if the main operation failed (using a separate, short, final transaction is robust).
        *   **No State Change on Failure:** If `pkgmgr.Install` or `pkgmgr.Remove` returns an error, the service **must not** call `store` methods to persist changes to `current_packages` or add `installed`/`removed` entries to `package_log` for that operation.
        *   **Translates Errors:** Converts internal errors into simpler errors or specific result types intended for the CLI layer. Avoids leaking raw SQL or complex internal details.
    *   **`CLI` Layer:**
        *   Receives errors from the Service layer.
        *   Uses `errors.Is` or `errors.As` to check for specific error types (`util.ErrPackageNotFound`, `util.ErrPkgMgrFailed`, `util.ErrDBNeedsBackupRestore`, `util.ErrMigrationFailed`, `util.ErrInvalidInput`).
        *   Prints concise, helpful messages to `stderr` based on the error type (e.g., "Error: Package 'foo' (manager 'dnf') not found in ledger.", "Error: DNF command failed. See output above and log file for details.", "Error: Database integrity check failed. Restore from backup advised.").
        *   If a generic or unexpected error is received, prints a standard failure message and directs the user to the application log file.
        *   Exits with a non-zero status code (typically `1`) upon any error.
*   **Structured Application Logging (`internal/applog`):**
    *   Utilizes Go's standard `log/slog` library configured in `internal/applog/log.go`.
    *   Outputs structured logs (JSON format preferred) to `$XDG_STATE_HOME/zenv/zenv.log` (with fallback).
    *   Uses log levels consistently (DEBUG for flow, INFO for actions, WARN for recoverable issues, ERROR for failures).
    *   Logs include context attributes (e.g., `command_id`, `package_name`, `manager`, function name).
    *   **Logs detected package manager versions** during `pkgmgr` interactions.
    *   **Logs database migration activity** (start, success, failure, no-change).
*   **Custom Error Types (`internal/util/errors.go`):**
    *   Defines exported sentinel errors for specific conditions:
        *   `var ErrPackageNotFound = errors.New("package not found in ledger")`
        *   `var ErrAmbiguousPackage = errors.New("multiple managers found for package; use --manager flag")`
        *   `var ErrPkgMgrFailed = errors.New("package manager command failed")`
        *   `var ErrPkgMgrNotFound = errors.New("package manager executable not found")`
        *   `var ErrDBError = errors.New("database operation failed")`
        *   `var ErrMigrationFailed = errors.New("database schema migration failed")`
        *   `var ErrInvalidInput = errors.New("invalid user input")`
        *   `var ErrDBNeedsBackupRestore = errors.New("database integrity check failed; backup restore recommended")`
    *   Errors are wrapped using `fmt.Errorf("...: %w", ErrCustomType)` to preserve chain.
*   **Input Validation and Sanitization:**
    *   Cobra handles basic flag type/count validation.
    *   Service layer performs semantic validation (e.g., required fields, logical checks).
    *   Prepared statements in `store.go` mitigate SQL injection risks.
*   **Package Manager Error Handling Specifics:**
    *   Service layer checks error returned by `pkgmgr.Install`/`Remove`.
    *   On failure, the service logs the failure details (including PM command string) to the `command_history` entry and rolls back any associated transaction.
    *   No changes are persisted to `current_packages` or `package_log` for the failed operation.
    *   CLI informs user, referencing PM output and application log.
*   **Usability Enhancements:**
    *   **Direct PM Output Streaming:** `os/exec.Cmd` Stdout/Stderr are connected to writers provided by CLI for `install`/`remove`.
    *   **Status Messages:** CLI prints clear messages before (`zenv: Running dnf install...`) and after (`zenv: Install successful. Updating ledger...`, `zenv: Command failed.`) service calls.
    *   **Confirmation Prompts:** CLI presents interactive `y/N` prompts for `discover --clear-existing` and `sync --apply` before calling the service with confirmation flags.
    *   **Shell Completion:** Implemented via `zenv completion [shell]` using Cobra's built-in support. `Makefile` target provided. Setup documented.
    *   **Help Text:** Comprehensive Cobra help (`-h`, `--help`) for all commands/flags, including examples.
    *   **Documentation:** Clear `README.md` and man page covering concepts (normalized tags, `check-health`, `report`, config), backup procedures (emphasizing migrations), usage examples.
    *   **`sudo` Handling:** `pkgmgr` adapters prepend `sudo` internally where necessary. `zenv` itself does not require root. Password prompting handled by `sudo`.
    *   **Warnings:** Logged via `applog` for non-fatal issues (e.g., optional metadata fetch failure, stale manual paths if checked).
    *   **Configuration File Feedback:** CLI reports errors clearly if the config file cannot be parsed or contains invalid keys/values.

## 8. Testing Strategy

A multi-layered testing strategy ensures confidence in the correctness, robustness, and stability of `zenv` across its different components and interactions.

*   **Unit Tests (`_test.go` files):**
    *   **Goal:** Verify the logic of individual functions, methods, and small components in complete isolation from external dependencies (database, filesystem, package managers).
    *   **Scope & Techniques:**
        *   **`internal/ledger/store`:** Mock the `database/sql` interface using libraries like `github.com/DATA-DOG/go-sqlmock`. Define expected SQL queries (as regexps or exact strings), parameters, and simulated results/errors. Verify that store methods generate correct SQL for all operations, including those involving **normalized tags (JOINs, inserts/deletes on `package_tags`, `tags`)**. Test parameter handling, result scanning (including pointer/nullable conversions), and error management (e.g., correctly returning `util.ErrPackageNotFound` on `sql.ErrNoRows`, handling constraint violations). Test filter logic (`QueryFilters`) thoroughly, including tag combinations. Test `CheckDatabaseIntegrity` method execution.
        *   **`internal/ledger/db`:** Test the `InitializeDB` function. Focus on verifying that it correctly configures and invokes the `golang-migrate/migrate` library API. This may involve using test doubles for the migration library or inspecting logs/mock outputs to ensure `migrate.Up()` is called appropriately. Testing the SQL migrations themselves happens primarily through integration/E2E tests verifying the final schema state.
        *   **`internal/ledger/format`:** Provide sample slices of `CurrentPackage` (including associated `Tag` data), `LogEntry`, `CommandHistory`, and custom report structs. Assert that the output strings for `table`, `json`, and `csv` formats match expected results, paying attention to the formatting of nullable fields and the **comma-separated display string for normalized tags** in table/CSV outputs.
        *   **`internal/pkgmgr` Adapters (e.g., `dnf`):** Mock the `os/exec` interface (or use helper functions injecting command runners). Test command construction logic (correct arguments, `sudo` prepending). Test parsing functions for `GetAllInstalledPackages` and `GetPackageDetails` by providing sample command output strings and verifying correct `PackageInfo` generation (including handling of optional/nullable metadata). Test error handling by simulating non-zero exit codes or specific stderr patterns. Verify the `Capabilities()` method returns the expected map. Test the `CheckAvailability()` method's logic (e.g., interaction with `exec.LookPath`).
        *   **`internal/service`:** Define mocks/fakes for `ledger.Store` and `pkgmgr.PackageManager` interfaces (e.g., using `gomock`). Inject mocks into the `Service` struct. For each service method (including `Install`, `Remove`, `Sync`, `Discover`, `Tag`, `Comment`, `AddManual`, `GetLedgerData`, **`CheckHealth`, `CheckDatabaseIntegrity`, `GetReportData`, `RunReportAlias`**):
            *   Verify the correct sequence of calls to mocked store and pkgmgr methods with expected arguments (context, transaction objects, filters, data structs).
            *   Assert correct database transaction management (Begin, Commit on success, **Rollback on any error**).
            *   Verify **immediate logging of command start** (`store.AddCommandStart`) and **reliable logging of command end status** (`store.UpdateCommandEnd`), including correct error/exit code details, using separate transactions where appropriate.
            *   Verify correct handling of **normalized tags** (e.g., calls to `FindOrCreateTag`, `AddTagToPackage`, `RemoveTagFromPackage`).
            *   Test logic for **`CheckAvailability` coordination** and handling of unavailable PMs.
            *   Test **configuration application** (e.g., passing flags/settings to store or pkgmgr methods).
            *   Test error propagation and translation (including handling migration errors from `InitializeDB`).
            *   Test concurrency logic (e.g., in `Discover`/`Sync`).
        *   **`internal/util`:** Test all helper functions (filesystem path generation including `GetConfigPath`, time utilities, error definitions) with various inputs and edge cases.
        *   **`internal/cli`:** Unit test helper functions used for parsing or formatting within the CLI layer, if any exist separate from Cobra's functionality.
*   **Integration Tests (`*_test.go` or `tests/integration/`):
    *   **Goal:** Verify interactions *between* implemented components, particularly Service-Store-DB and CLI-Service communication, using a real database instance but mocking external processes.
    *   **Scope & Techniques:**
        *   **Service <-> Store <-> Database:** Test service layer methods using a *real* `ledger.Store` connected to a temporary SQLite database file (`os.CreateTemp`). Ensure the database schema is initialized/migrated using the actual `InitializeDB` function. **Mock only the `pkgmgr.PackageManager` interface** to simulate PM interactions without executing external commands. Verify service method calls result in correct data modifications and retrieval from the temporary database (query state before/after). Test transaction atomicity across service calls (e.g., ensure rollback occurs if a store method fails after a mocked PM success). Test **normalized tag operations** end-to-end through the service and store layers.
        *   **CLI -> Service -> Store:** Test CLI command execution end-to-end *within the application*, mocking only external PM execution. Use `cobra.Command.SetArgs()` and `Execute()` or helper functions. Inject a real service instance connected to a real (temporary) store (with migrations applied) and mocked `pkgmgr` adapters. Capture stdout/stderr and verify output messages, status updates, error messages, and exit codes. Check the state of the temporary database after command execution (`current_packages`, `package_log`, `command_history`, `tags`, `package_tags`). Verify filters (including `--tag`), output formats, and **new commands (`check-health`, `report`, `check-db`)**. Test interactions with **configuration file settings** (e.g., loading aliases, changing default formats).
*   **End-to-End (E2E) Tests (`docs/TESTING.md`, `scripts/`):
    *   **Goal:** Validate the complete, compiled application behaves correctly in realistic user environments, interacting with *actual* package managers.
    *   **Method:** Primarily manual testing based on detailed scenarios in `docs/TESTING.md`, potentially augmented with shell scripts for automation within controlled environments (Docker containers strongly preferred).
    *   **Environment Setup:** Requires setting up containers/VMs with specific OS versions (e.g., Fedora latest, Ubuntu LTS) and installing relevant package managers (`dnf`, `apt`, `pip`, etc.). Include setups with **multiple repositories** configured.
    *   **Test Scenarios:** Cover all major user workflows, including:
        *   `init` (including automatic migration application on subsequent runs).
        *   `install`, `remove` (checking streamed output, ledger state, history, origin capture with multiple repos).
        *   `add-manual`, `comment`, `remove-manual-log`.
        *   **`tag`**: Adding, removing, listing tags; verifying persistence and filtering in `ledger`.
        *   `discover`, `sync` (verifying detection of external changes, `--apply` functionality, context reporting for incomplete commands).
        *   `ledger` (all subcommands, comprehensive filter tests including tags, date ranges, multiple criteria; verify all output formats).
        *   **`report`**: Testing built-in reports (`disk-usage`, `licenses`, `activity`, `summary`) and `run <alias>` with aliases defined in a test config file.
        *   **`check-health`, `check-db`**: Verifying correct status reporting in various conditions (clean state, incomplete commands present, simulated DB corruption if feasible).
        *   **Configuration File:** Testing behavior changes based on different config settings (log level, default format, metadata fetching toggles, report aliases).
        *   **Error Handling:** Testing user feedback for invalid commands/flags, PM failures (permissions, package not found), DB errors (corruption), unavailable PMs.
        *   **PM Update Compatibility:** **Regularly run E2E tests against environments using the latest available OS/PM versions** within the CI/release process to catch regressions caused by changes in PM command output or behavior.
        *   **Recovery:** Test `sync` effectiveness after simulating interruptions (e.g., killing `zenv` during `install`, verifying incomplete command in history, then running `sync`).
*   **Automation (CI):**
    *   Use GitHub Actions (or similar) defined in `.github/workflows/test.yml`.
    *   Trigger on pushes/pull requests.
    *   **Linting:** Run `make lint` (`golangci-lint`). Fail build on errors.
    *   **Unit & Integration Tests:** Run `make test` (`go test -race ./...`). Fail build on errors. Report coverage.
    *   **Build:** Compile binaries for target platforms.
    *   **(Optional but Recommended) E2E Tests:** Integrate automated E2E tests using Docker if feasible. May run separately or on a different schedule due to setup/runtime complexity.