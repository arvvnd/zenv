# zenv - Architecture Design Document (Comprehensive)

**Date:** 11-04-2025

## 1. Introduction

'zenv' is envisioned as a comprehensive command-line utility for Linux systems, designed to enhance the user's control and understanding of their installed software environment. It functions primarily as an **auditing and tracking wrapper** around existing system and language package managers (initially focusing on DNF, but extensible to APT, Pip, NPM, etc.).

The core motivation is to address the limitations of default package manager logs, which often lack persistent, user-centric context and detailed historical information accessible through a unified interface. `zenv` aims to solve this by maintaining its own persistent ledger, stored in a local SQLite database. This ledger meticulously records package operations (installations, removals) performed *through* `zenv`, enriching this data with critical metadata such as:

*   Precise package **version**.
*   The **manager** responsible (e.g., `dnf`, `pip`, `manual`).
*   The **origin** or source (e.g., repository ID, URL, filepath).
*   The inferred installation **reason** (`user`-requested vs. `dependency`).
*   User-defined **tags** for categorization.
*   Filesystem **location** for manually tracked items.
*   Package integrity information like **checksums** and **signatures** (best-effort).
*   Other metadata like **license** and **size** (best-effort).
*   User-provided **comments**.
*   Event **timestamps**.

Crucially, `zenv` also maintains a separate **command history log**, recording every invocation of `zenv` itself, including the exact command string, start/end times, the `zenv` version used, exit status, and any specific package manager command executed. Package events are linked back to the originating `zenv` command, providing a robust **audit trail** connecting user actions to system changes.

To ensure reliability and avoid brittleness associated with parsing live command output, `zenv` relies on **post-operation checks** against the package manager's database or query tools after an action completes successfully.

Furthermore, `zenv` allows users to manually add entries for software installed outside standard package managers (e.g., AppImages, binaries) and provides `discover` and `sync` commands to reconcile its tracked state with the actual system state, accounting for external changes.

Architecturally, `zenv` utilizes a **service layer** to decouple the core application logic (transaction management, package manager interaction, database updates) from the command-line interface. This modular design facilitates easier testing and future development, such as the creation of alternative frontends (like a TUI). User experience is prioritized through clear command structures, informative output, direct streaming of underlying package manager messages during operations, and shell completion support.

This document details the comprehensive architecture and functional specifications for `zenv`.

## 2. Goals

The primary goals for `zenv` are:

*   **Comprehensive Audit Trail:** Provide a reliable and queryable history of software management actions performed via `zenv` and their effects on the system.
    *   Log every `zenv` command execution (`command_history`), including exact arguments, timing, version, status, and errors.
    *   Log every significant package state change (`package_log`) resulting from `zenv` operations (install, remove, sync updates, tag/comment changes, manual actions).
    *   Link `package_log` events directly to the corresponding `command_history` entry using a `command_id`.
*   **Rich Metadata Storage:** Capture and persist detailed information about tracked packages beyond basic names.
    *   Store package `name`, `version`, installation `manager`, `origin` (source repository/URL/path), installation `reason` (user, dependency, discovered), filesystem `location` (for manual items), user `comment`, event `timestamp`.
    *   Store the **original installation timestamp** (best-effort from PM) and the **last updated timestamp** (when zenv last updated the record).
    *   Store package integrity/metadata where available via the package manager: `checksum`, `signature` info, `license`, installed `size`. Store NULL if unavailable.
    *   Support user-defined **tags** for categorization, stored in a normalized structure.
*   **Unified Tracking Interface:** Act as a consistent wrapper around multiple underlying package managers (distro-native, language-specific, manual entries), providing a single point of interaction and logging.
*   **State Reconciliation & Health Checks:**
    *   Provide mechanisms (`discover`, `sync`) to align `zenv`'s tracked state with the actual system state.
    *   Provide health checks (`check-health`, `check-db`) to verify internal consistency and database integrity.
*   **Extensibility:** Design for easy addition of support for new package managers.
    *   Utilize a `PackageManager` interface (`internal/pkgmgr`) defining required operations.
    *   Include a `Capabilities()` method in the interface for managers to report supported optional metadata, allowing `zenv` to adapt gracefully.
    *   Detect available package managers on the system.
*   **User Control & Querying/Reporting:** Empower users to understand and manage their environment.
    *   Allow manual addition/removal of log entries for software not managed by supported PMs.
    *   Allow tagging and commenting on packages for custom organization (using normalized tags).
    *   Provide robust `ledger` querying with specific filter flags (`--reason`, `--manager`, `--origin`, `--since`, `--before`, `--tag`).
    *   Offer multiple output formats (`table`, `json`, `csv`) for `ledger` queries.
    *   Provide built-in `report` commands for common analytical queries (e.g., disk usage, license audit, activity summary).
    *   Allow users to define and run custom reports via aliases in a configuration file.
*   **Robustness & Reliability:**
    *   Prioritize reliable state determination using post-operation checks rather than fragile live output parsing.
    *   Ensure atomicity of logical operations via database transactions managed in the service layer.
    *   Handle errors gracefully, log detailed diagnostics (including PM versions when interacting), provide clear user feedback, and prevent ledger corruption due to failed operations.
    *   Implement a robust **schema migration strategy using `golang-migrate/migrate`**, emphasizing the need for backups.
    *   Log command initiation immediately to ensure incomplete operations are identifiable.
*   **Maintainability & Reusability:**
    *   Decouple core logic into a `service` layer, separating it from the `cli` presentation layer to facilitate testing and future interfaces (e.g., TUI).
    *   Maintain clear boundaries between components (`cli`, `service`, `pkgmgr`, `ledger`, `util`).
*   **User Experience:**
    *   Provide a clean, intuitive command-line interface using Cobra.
    *   Offer comprehensive help text.
    *   Support shell completion for discoverability.
    *   Stream underlying package manager output directly during relevant operations.
    *   Provide clear status messages and require confirmation for destructive actions.
*   **Scope Definition:** Explicitly **exclude** a dedicated `zenv upgrade` command wrapper (rely on `sync` for external updates and `install` for `zenv`-initiated upgrades). Explicitly **exclude** specific tracking of the running kernel version (`uname -r`), relying only on tracking kernel *packages*.

## 3. Core Components

This section details the primary logical and structural components of the `zenv` application.

*   **CLI Interface (Cobra) (`internal/cli`):**
    *   **Purpose:** Provides the user-facing command-line interface. It is responsible for defining commands, subcommands, flags, and arguments using the `github.com/spf13/cobra` library.
    *   **Responsibilities:**
        *   Parsing user input (commands, flags, arguments).
        *   Performing basic syntactic validation (e.g., required flags, argument counts).
        *   Displaying help messages and usage information.
        *   Generating shell completion scripts (`zenv completion ...`).
        *   **Acting as a thin client:** Constructing appropriate request objects or parameters based on user input.
        *   Invoking the corresponding methods on the `Service` layer.
        *   Receiving results or errors back from the `Service` layer.
        *   Handling the presentation of results to the user (e.g., calling `internal/ledger/format` functions to format data).
        *   Printing status messages (e.g., "Running dnf install...") and user-facing errors to `stdout`/`stderr`.
        *   Exiting with appropriate status codes.
    *   **Decoupling:** This layer should contain minimal business logic. Its primary role is user interaction and translation between user input and service layer calls. It does not directly interact with the database (`ledger.Store`) or execute package manager commands (`pkgmgr`).
    *   **Streaming:** For commands like `install` and `remove`, it provides `io.Writer` instances (typically `os.Stdout`, `os.Stderr`) to the service layer method call, allowing the service layer to stream the output from the underlying package manager directly to the user's terminal.

*   **Service Layer (`internal/service`):**
    *   **Purpose:** Contains the core application logic and orchestration. It acts as an intermediary between the presentation layer (CLI, future TUI) and the data/external interaction layers (`ledger`, `pkgmgr`).
    *   **Responsibilities:**
        *   Exposing high-level methods corresponding to user actions (e.g., `InstallPackages`, `RemovePackages`, `SyncState`, `DiscoverPackages`, `GetLedgerData`, `AddManualEntry`, `UpdateTag`, `UpdateComment`).
        *   **Transaction Management:** Starting, committing, and rolling back SQLite database transactions (`sql.Tx`) for all operations that modify the ledger state, ensuring atomicity. *(See Section 5 for specific notes on `remove` pre-fetch transaction strategy and Section 7 for command history logging transactions).*.
        *   **Command History Logging:** Interacting with `ledger.Store` to create an entry in `command_history` *immediately* when a service method starts (in a separate, committed transaction), recording the `zenv` version and full command string. It updates this entry with the executed `pm_command_string`, end timestamp, exit code, and error details upon completion or failure. *(Implementation Note: The final update for command end status should likely occur in a separate, short transaction after the main operation's transaction attempt to ensure the outcome is recorded even on main commit failure. An entry with a `NULL` `end_timestamp` indicates an interrupted operation.)*
        *   **Package Manager Interaction:** Selecting the appropriate `pkgmgr.PackageManager` instance based on context or request parameters. Calling methods like `Install`, `Remove`, `GetAllInstalledPackages`. Handling errors returned by the package manager. Logging the detected package manager version to the application log during interaction.
        *   **Ledger Updates:** Calling methods on `ledger.Store` to persist changes (`ReplaceCurrentPackage`, `AddLogEntry`, `DeleteCurrentPackage`, `AddTag`, `RemoveTag`, etc.), passing the active transaction (`sql.Tx`) and the relevant `command_id` to link package log entries back to the originating command.
        *   **Business Logic:** Implementing logic like comparing system state vs. DB state in `Sync`, determining `install_reason`, handling concurrency for `discover`/`sync` package manager queries, performing simple post-operation checks (e.g., warning if `add-manual --location` path doesn't exist), running health checks.
        *   **Configuration Handling:** Reading and applying settings from the user's configuration file (e.g., default flags, report aliases, metadata fetching toggles).
        *   **Error Handling:** Catching errors from `pkgmgr` and `ledger.Store`, logging detailed internal errors, potentially wrapping errors, and returning meaningful errors or results to the calling layer (CLI). Ensures failed PM operations do not leave the ledger in an inconsistent state relative to the command history.
    *   **Decoupling:** Provides a stable internal API, isolating the core workflows from the specifics of the user interface or the database implementation details. Facilitates easier unit testing of the core logic by mocking store and pkgmgr dependencies.

*   **Package Manager Abstraction (`internal/pkgmgr`):**
    *   **Purpose:** Provides a consistent way to interact with different underlying system and language package managers.
    *   **Interface (`pkgmgr.go`):** Defines the `PackageManager` interface with methods required by the service layer: `Install`, `Remove`, `GetAllInstalledPackages`, `GetPackageDetails`, `Name`, and `Capabilities`. `PackageInfo` struct defines the rich metadata to be returned (including checksum, signature, license, size - nullable).
    *   **Implementations (`dnf/`, `apt/`, `pip/` etc.):** Concrete types implementing the `PackageManager` interface. They contain the specific logic for:
        *   Constructing the correct command-line arguments for the underlying tool (including `sudo` where necessary).
        *   Executing the command using `os/exec`.
        *   Streaming stdout/stderr to provided writers.
        *   Checking exit codes and parsing stderr for common error conditions.
        *   Performing **post-operation checks** (e.g., querying `rpm`, `dpkg`, `pip show`, metadata files) to gather the required `PackageInfo` details after successful operations. This includes best-effort fetching of optional metadata like checksums, licenses, etc.
        *   Implementing the `Capabilities()` method accurately.

*   **Ledger Management (SQLite) (`internal/ledger`):**
    *   **Purpose:** Handles all aspects of persistence using the SQLite database.
    *   **Storage:** Manages a single SQLite database file (`ledger.db`) in the appropriate XDG data directory.
    *   **Schema (`db.go`, `models.go`, `migrations/`):
        *   `models.go`: Defines Go structs (`CommandHistory`, `CurrentPackage`, `LogEntry`, `Tag`, `PackageTag`) that map directly to the database tables. Includes struct tags (`db:"..."`). Uses pointers for nullable fields. **`CurrentPackage` and `LogEntry` no longer contain a `tags` field.**
        *   `db.go`: Contains the `InitializeDB` function responsible for database connection and initial setup. **Migration logic is now handled by `golang-migrate/migrate`.** This function ensures the migration library runs and applies necessary migrations.
        *   `migrations/`: New directory containing `.sql` files for schema migrations (e.g., `001_initial_schema.up.sql`, `001_initial_schema.down.sql`, `002_add_feature_x.up.sql`, etc.) managed by `golang-migrate/migrate`.
    *   **Store (`store.go`):
        *   Defines the `Store` struct holding the `*sql.DB` connection pool.
        *   Provides the primary Data Access Object (DAO) methods (e.g., `AddCommandStart`, `ReplaceCurrentPackage`, `AddLogEntry`, `GetCurrentPackages`, `GetHistory`, **`AddTagToPackage`, `RemoveTagFromPackage`, `GetTagsForPackage`, `FindOrCreateTag`**).
        *   Encapsulates all SQL query logic (using prepared statements where appropriate). Queries involving tags will now use JOINs with `package_tags` and `tags` tables.
        *   Methods accept `context.Context`, `sql.Tx` (for writes) or `Querier` (for reads), and necessary parameters (structs, IDs, filters).
        *   Handles mapping between database rows/results and Go structs defined in `models.go`.
        *   Returns specific errors (e.g., `ErrPackageNotFound`).
    *   **Formatting (`format.go`):
        *   Provides functions (e.g., `FormatOutput`) to convert slices of ledger data (`[]CurrentPackage`, `[]LogEntry`, potentially `[]CommandHistory`) into different string representations (`table`, `json`, `csv`) for display by the CLI. Decouples presentation format from data retrieval.

*   **Application Logging (`internal/applog`):**
    *   **Purpose:** Provides a consistent, leveled, structured logging facility for debugging and diagnostics.
    *   **Implementation:** Uses Go's standard `log/slog` library. Configured in `log.go` to write to a file (`$XDG_STATE_HOME/zenv/zenv.log` or fallback) typically in JSON format.
    *   **Usage:** A global logger instance can be made available or passed where needed (e.g., to the service layer) to log internal operations, errors, and debug information with appropriate levels and key-value context.

*   **Utilities (`internal/util`):**
    *   **Purpose:** Contains small, independent, reusable helper functions that don't belong to a specific core component.
    *   **Examples:**
        *   `fs.go`: Functions for finding XDG base directories (`GetDBPath`, `GetLogPath`, `GetConfigPath`), ensuring directories exist (`EnsureDir`).
        *   `timeutil.go`: Helpers for parsing/formatting timestamps (e.g., consistent ISO 8601 usage).
        *   **`tags.go`:** (Removed or significantly reduced scope) Functions for parsing tag *input* strings might still exist, but formatting tags for storage is no longer needed. Logic for handling the normalized tag structure resides in `store.go`.
        *   `errors.go`: Definitions for custom sentinel errors used across the application (e.g., `ErrPackageNotFound`, `ErrPkgMgrFailed`).

## 4. Ledger Database Schema

Stored in a single SQLite file at `$XDG_DATA_HOME/zenv/ledger.db` (or fallback).

### 4.1. Schema Target Definition

This section defines the target structure of the database tables and indexes that the application code expects. The migration process ensures the database reaches this state.

**Tables:**

1.  **`command_history`:** Records metadata about each execution of the `zenv` command itself, providing an audit trail of user actions. (Unchanged from previous spec).
2.  **`current_packages`:** A snapshot representing the last known state of each package tracked by `zenv`. Rows are added, updated (via `REPLACE`), or deleted based on `zenv` operations. **The `tags` column is removed.**
3.  **`package_log`:** An append-only log detailing specific package events (install, remove, update via sync, tag change, etc.), linked back to the `zenv` command that caused the event via `command_id`. **The `tags` column is removed.**
4.  **`tags`:** (New Table) Stores unique tag names.
5.  **`package_tags`:** (New Table) Linking table connecting packages in `current_packages` to tags in `tags`.

**Detailed Columns, Types, Constraints, and Indexes:**

*   **`command_history` Table:**
    *   `command_id` (INTEGER PRIMARY KEY AUTOINCREMENT): Automatically generated unique identifier for the command invocation.
    *   `start_timestamp` (TEXT NOT NULL): ISO 8601 formatted timestamp (`YYYY-MM-DDTHH:MM:SSZ`) recorded when the service layer method begins execution.
    *   `end_timestamp` (TEXT NULL): ISO 8601 formatted timestamp recorded when the service layer method finishes (successfully or with error). NULL if the command is interrupted or crashes before completion logging.
    *   `zenv_version` (TEXT NOT NULL): The version string of the `zenv` binary that executed the command (e.g., obtained via build flags).
    *   `command_string` (TEXT NOT NULL): The complete, original command-line invocation string including all arguments and flags as passed to the `zenv` executable.
    *   `pm_command_string` (TEXT NULL): If the `zenv` command resulted in executing an underlying package manager command (like `dnf install ...`), this field stores the exact command string that was executed. NULL for commands that don't invoke a PM.
    *   `exit_code` (INTEGER NULL): The final exit code returned by the `zenv` process for this invocation (0 for success, non-zero for failure). NULL if not recorded before termination.
    *   `error_message` (TEXT NULL): If the command failed, this stores the primary user-facing error message returned by the service layer.
    *   `details` (TEXT NULL): Optional field for storing additional structured context, such as a JSON summary of changes detected and applied during a `sync` operation.
    *   **Indexes:** `idx_command_history_id` (PK), `idx_command_history_start_timestamp`

*   **`current_packages` Table:**
    *   `package_name` (TEXT NOT NULL): The canonical name of the package.
    *   `manager` (TEXT NOT NULL): The identifier for the package manager or mechanism tracking this entry (e.g., `dnf`, `pip`, `manual`). Part of the primary key.
    *   `origin` (TEXT NULL): Information about the source of the package (e.g., DNF repository ID like `fedora`, PyPI registry `pypi.org`, URL, local filepath). Depends on the manager's ability to provide it.
    *   `package_version` (TEXT NULL): The specific version string of the currently tracked package.
    *   `install_reason` (TEXT NULL): The inferred reason for installation (`user`, `dependency`, `discovered`). Best-effort, depends on manager capabilities.
    *   `location` (TEXT NULL): Filesystem path associated with the package, particularly relevant for `manual`, `appimage`, `binary` managers.
    *   `comment` (TEXT NULL): A user-provided annotation for this package entry.
    *   `install_timestamp` (TEXT NULL): ISO 8601 timestamp indicating the original installation time of the package, if available from the underlying package manager (best-effort).
    *   `last_updated_timestamp` (TEXT NOT NULL): ISO 8601 timestamp indicating when this specific row/state was last written or verified by `zenv` (via `install`, `add-manual`, `discover`, `sync`, etc.).
    *   `checksum` (TEXT NULL): A cryptographic hash (e.g., SHA256) of the package file or a representative value, if obtainable from the manager. Format needs definition (e.g., `sha256:<hash>`).
    *   `signature` (TEXT NULL): Information about the package's digital signature (e.g., GPG key ID or fingerprint), if obtainable.
    *   `license` (TEXT NULL): The software license identifier (e.g., SPDX code like `MIT`, `GPL-3.0-or-later`) as detected by the package manager.
    *   `size` (INTEGER NULL): The installed size of the package in bytes, if provided by the manager.
    *   **Primary Key:** `PRIMARY KEY (package_name, manager)`

*   **`package_log` Table:**
    *   `id` (INTEGER PRIMARY KEY AUTOINCREMENT): Unique identifier for this log event.
    *   `command_id` (INTEGER NULL, REFERENCES `command_history`(`command_id`)): Foreign key referencing `command_history.command_id`. Links this package event to the `zenv` command invocation that caused it. NULL for events not directly tied to a command (e.g., the initial `init` log).
    *   `timestamp` (TEXT NOT NULL): ISO 8601 timestamp when this specific package event (e.g., installation finished, removal logged) occurred, as recorded by `zenv`.
    *   `action` (TEXT NOT NULL): A string code indicating the type of event (e.g., `installed`, `removed`, `sync_update`, `tags_updated`). See constants in `models.go`.
    *   `package_name` (TEXT NOT NULL): Name of the package affected by this event.
    *   `manager` (TEXT NOT NULL): Manager associated with the package at the time of the event.
    *   `origin` (TEXT NULL): Origin of the package at the time of the event.
    *   `package_version` (TEXT NULL): Version of the package relevant to this event (e.g., version installed, version removed, version before update).
    *   `install_reason` (TEXT NULL): Install reason at the time of the event.
    *   `location` (TEXT NULL): Location associated with the package at the time of the event.
    *   `comment` (TEXT NULL): Comment relevant to this specific event (e.g., comment provided with `-c` flag during install/remove, generated summary for `sync` actions).
    *   `checksum` (TEXT NULL): Checksum relevant to this event state.
    *   `signature` (TEXT NULL): Signature info relevant to this event state.
    *   `license` (TEXT NULL): License relevant to this event state.
    *   `size` (INTEGER NULL): Size relevant to this event state.
    *   **Constraints:** `FOREIGN KEY (command_id) REFERENCES command_history(command_id)`
    *   **Indexes:** `idx_package_log_timestamp`, `idx_package_log_package_name_manager`, `idx_package_log_action`, `idx_package_log_command_id`

*   **`tags` Table (New):**
    *   `tag_id` (INTEGER PRIMARY KEY AUTOINCREMENT): Internal unique identifier for the tag.
    *   `name` (TEXT NOT NULL UNIQUE COLLATE NOCASE): The user-defined tag name. `UNIQUE` constraint ensures no duplicate tag names (case-insensitive).
    *   **Indexes:** `idx_tags_name` (implicit via UNIQUE)

*   **`package_tags` Table (New - Linking Table):**
    *   `package_name` (TEXT NOT NULL): Part of the composite foreign key referencing `current_packages`.
    *   `manager` (TEXT NOT NULL): Part of the composite foreign key referencing `current_packages`.
    *   `tag_id` (INTEGER NOT NULL REFERENCES `tags`(`tag_id`) ON DELETE CASCADE): Foreign key referencing the `tags` table. `ON DELETE CASCADE` ensures that if a tag is deleted, all associations to packages are also removed.
    *   **Primary Key:** `PRIMARY KEY (package_name, manager, tag_id)`: Ensures a package cannot have the same tag applied multiple times.
    *   **Foreign Keys:** `FOREIGN KEY (package_name, manager) REFERENCES current_packages(package_name, manager) ON DELETE CASCADE`: Links to the specific package entry in `current_packages`. `ON DELETE CASCADE` ensures that if a package is removed from `current_packages`, its tag associations are also removed.
    *   **Indexes:**
        *   `CREATE INDEX idx_package_tags_tag_id ON package_tags (tag_id);` (Efficiently find packages by tag)
        *   `CREATE INDEX idx_package_tags_package ON package_tags (package_name, manager);` (Efficiently find tags for a package)

### 4.2. Schema Migration Logic (`internal/ledger/db.go`, `internal/ledger/migrations/`)

The `InitializeDB` function within `internal/ledger/db.go` is responsible for ensuring the database schema matches the application's expectations using the `golang-migrate/migrate` library.

*   **Process:**
    1.  Open DB connection.
    2.  Execute `PRAGMA foreign_keys = ON;`.
    3.  Embed or load migration `.sql` files from the `internal/ledger/migrations/` directory (using `migrate/source/iofs` or similar).
    4.  Instantiate the `migrate` library using the embedded source driver and the SQLite database driver (`migrate/database/sqlite3`).
    5.  Call the `migrate.Up()` method. This automatically checks the current database version (managed internally by the library in a `schema_migrations` table) and applies all pending `*.up.sql` files sequentially until the database reaches the latest version defined by the available migration files.
    6.  Handle errors returned by `migrate.Up()`. If migration fails, log the error and prevent the application from starting normally. Log success messages.
*   **Migration Files (`migrations/*.sql`):**
    *   Contain standard SQL DDL statements.
    *   Named with sequential numbers and descriptive names (e.g., `001_initial_schema.up.sql`, `001_initial_schema.down.sql`, `002_add_indices.up.sql`, `002_add_indices.down.sql`).
    *   The first migration (`001_initial_schema.up.sql`) will contain the `CREATE TABLE` statements for `command_history`, `current_packages`, `package_log`, `tags`, and `package_tags`, along with necessary `CREATE INDEX` statements.
*   **Error Handling & Backups:** The migration library handles transactional application of each migration file where possible with the driver. However, robust error handling within `zenv` for migration failures is still needed. Documentation must strongly advise users to back up `ledger.db` before upgrading `zenv`.

## 5. Command Specification (`zenv <command> [options]`)

This section details the expected functional behavior of each user-facing `zenv` command. The implementation involves the CLI layer (`internal/cli`) parsing arguments/flags and calling the corresponding method on the **`Service` layer (`internal/service`)**, which then orchestrates the operation.

*   **General Principles:**
    *   **Service Layer Interaction:** CLI commands act as thin clients to the service layer.
    *   **Transaction Management:** The service layer manages database transactions for state-modifying operations.
    *   **Command History:** The service layer logs the start and end of every command execution (including arguments, version, timing, status, errors, and executed PM command) to the `command_history` table. `NULL` `end_timestamp` indicates interruption.
    *   **Package Log Linking:** All package events recorded in `package_log` as a direct result of a command are linked to the corresponding `command_history` entry via the `command_id`.
    *   **PM Output Streaming:** For `install` and `remove`, the service layer receives `io.Writer` instances from the CLI and ensures the underlying package manager's stdout/stderr is streamed directly to the user. `zenv` status messages are printed before and after the PM execution.
    *   **Post-Operation Checks:** Package details (version, origin, reason, metadata) are determined by the `pkgmgr` adapter via checks performed *after* a successful PM operation, not by parsing live output.
    *   **Error Handling:** Failures in PM execution or critical DB operations within the service layer prevent ledger modification for that operation and are logged in `command_history`. The CLI presents user-friendly error messages.

*   **`zenv init`**
    *   **Goal:** Initialize the `zenv` environment, including directories and the database schema. Should be safe to run multiple times (idempotent).
    *   **CLI Action:** Parses the command, calls `service.Initialize()`.
    *   **Service Action (`Initialize`):**
        1.  Determines required directories (e.g., `$XDG_DATA_HOME/zenv`, `$XDG_STATE_HOME/zenv`) using `util/fs`.
        2.  Ensures these directories exist using `util/fs.EnsureDir`.
        3.  Configures application logging (`internal/applog`).
        4.  Opens the database connection (`ledger.OpenDB`).
        5.  Calls `ledger.InitializeDB()` to create tables and run migrations up to the target schema version. Logs migration steps. Returns error if migration fails.
        6.  If the database was newly created or migrated from version 0, starts a new transaction, logs the `init` action to `package_log` via `store.AddLogEntry` (without a specific `command_id`), and commits.
        7.  Closes the initial DB connection (main connection managed elsewhere).
    *   **CLI Output:** Reports success/failure, confirms DB and log file paths. Suggests running `zenv discover` if initialized for the first time.

*   **`zenv install <pkg...>` [flags...] [`--` `<pm_opts...>`]**
    *   **Flags:** `[--manager <mgr>]` (optional override), `[-c|--comment <comment>]`, `[--tag <tag>]...` (repeatable).
    *   **Args:** One or more package names. Trailing arguments after `--` are passed to the underlying PM.
    *   **CLI Action:** Parses flags/args, constructs `service.InstallArgs` struct, calls `service.Install(ctx, args, os.Stdout, os.Stderr)`. Prints final status/error based on service response.
    *   **Service Action (`Install`):**
        1.  Logs command start to `command_history` (Tx1), gets `command_id`.
        2.  Determines appropriate `pkgmgr.PackageManager` instance (auto-detect preferred, use `--manager` if provided). Logs PM version to applog.
        3.  Calls `pkgmgr.Install(packages, pm_opts)`, passing CLI's stdout/stderr writers for streaming. Gets back `pmCmdString` and `err`.
        4.  Starts main DB transaction (`tx`). Defer rollback.
        5.  Logs `pmCmdString` to `command_history` via `store.UpdateCommandPMString(tx, commandID, pmCmdString)`.
        6.  If `err` is not nil (PM failed): Log error details to `command_history` (via `UpdateCommandEnd`), rollback transaction, return error to CLI.
        7.  If PM succeeded:
            *   Perform post-operation checks: Call relevant `pkgmgr` methods to get `PackageInfo` for all *actually* installed/affected packages.
            *   Determine `install_reason` (`user` for explicitly requested, `dependency` otherwise).
            *   For each affected package: Create `ledger.CurrentPackage` struct. Call `store.ReplaceCurrentPackage(tx, currentPkg)`. Call `store.AddLogEntry(tx, commandID, logEntry)` with `action=installed`. Handle store errors (rollback).
            *   Handle tags: Use `store.FindOrCreateTag` and `store.AddTagToPackage` within the transaction for any tags provided via `--tag` flags, associating them with the newly added/updated `current_packages` entry.
        8.  Commit transaction. Handle commit error (log to command history).
        9.  Log command end details (success status) to `command_history` via `UpdateCommandEnd` (in separate transaction).
        10. Return success result to CLI.

*   **`zenv remove <pkg...>` [flags...] [`--` `<pm_opts...>`]**
    *   **Flags:** `[-c|--comment <comment>]`.
    *   **Args:** One or more package names. Trailing arguments after `--` are passed to the underlying PM.
    *   **CLI Action:** Parses flags/args, constructs `service.RemoveArgs`, calls `service.Remove(ctx, args, os.Stdout, os.Stderr)`. Prints final status/error.
    *   **Service Action (`Remove`):**
        1.  Logs command start to `command_history`, gets `command_id`.
        2.  Determine appropriate `pkgmgr.PackageManager` instance.
        3.  **Transaction Strategy Note:** Decide whether to fetch prior package details (step 4) *before* starting the main transaction (Option A) or *within* the main transaction (Option B). Option B is safer for consistency but holds the transaction longer. This choice impacts step 4 & 5 ordering.
        4.  **Fetch Prior Details (Assuming Option B):** Start DB transaction (`tx`). Defer rollback. Query `ledger.Store` (`GetCurrentPackage`) for requested packages. Error if ambiguous across managers unless `--manager` specified. Store prior details. Handle `ErrPackageNotFound`.
        5.  Calls `pkgmgr.Remove(packages, pm_opts)`, passing CLI's stdout/stderr writers. Gets back `pmCmdString` and `err`.
        6.  Logs `pmCmdString` to `command_history` (within `tx`).
        7.  If `err` is not nil: Log error, rollback, return error.
        8.  If PM succeeded:
            *   Perform post-operation checks to confirm which packages were *actually* removed.
            *   For each confirmed removed package: Use the *prior details* fetched earlier. Call `store.DeleteCurrentPackage(tx, name, manager)`. Call `store.AddLogEntry(tx, commandID, logEntry)` using prior details, `action=removed`, and the event comment. Handle store errors (rollback).
        9.  Commit transaction. Handle commit error.
        10. Log command end details to `command_history` *(Separate transaction preferred)*.
        11. Return success result.

*   **`zenv add-manual <pkg>` `--manager <mgr>` [flags...]**
    *   **Flags:** `[--origin <origin_info>]`, `[--version <ver>]`, `[--tag <tag>]...`, `[--location <path>]`, `[-c|--comment <comment>]`.
    *   **Args:** Exactly one package name. `--manager` is required and should typically be `manual` or another non-PM value (e.g., `appimage`, `binary`).
    *   **CLI Action:** Parses flags/args, constructs `service.AddManualArgs`, calls `service.AddManual(ctx, args)`. Prints status/error.
    *   **Service Action (`AddManual`):**
        1.  Logs command start, gets `command_id`.
        2.  Validate input (e.g., warn if manager looks like a standard PM). Sanitize inputs.
        3.  Start DB transaction. Defer rollback.
        4.  Create `ledger.CurrentPackage` struct from args. **Set `install_timestamp` to NULL (as it's manual). Set `last_updated_timestamp` to the current time.**
        5.  Call `store.ReplaceCurrentPackage(tx, currentPkg)`. Handle error (rollback).
        6.  Call `store.AddLogEntry(tx, commandID, logEntry)` with `action=manual_add` (log entry reflects NULL `install_timestamp`). Handle error (rollback).
        7.  **Post-Add Check:** If `location` was provided, attempt `os.Stat(location)`. If it fails (e.g., file not found), log a warning to the application log (but do not fail the transaction).
        8.  Commit transaction. Handle commit error.
        9.  Log command end. *(Separate transaction preferred)*.
        10. Return success result.

*   **`zenv remove-manual-log <pkg>` `--manager <mgr>` [`-c <comment>`]**
    *   **Flags:** `--manager <mgr>` (**required**), `[-c|--comment <comment>]`.
    *   **Args:** Exactly one package name.
    *   **CLI Action:** Parses, constructs `service.RemoveManualLogArgs`, calls `service.RemoveManualLog(ctx, args)`. Prints status/error.
    *   **Service Action (`RemoveManualLog`):**
        1.  Logs command start, gets `command_id`.
        2.  Start DB transaction. Defer rollback.
        3.  Call `store.GetCurrentPackage(tx, name, manager)` to fetch prior details. Handle `ErrPackageNotFound` (rollback, return error).
        4.  Call `store.DeleteCurrentPackage(tx, name, manager)`. Handle error (rollback).
        5.  Call `store.AddLogEntry(tx, commandID, logEntry)` using prior details, `action=manual_remove_log`, and event comment. Handle error (rollback).
        6.  Commit transaction. Handle commit error.
        7.  Log command end. *(Separate transaction preferred)*.
        8.  Return success result.

*   **`zenv comment <pkg>` `--manager <mgr>` `-c <comment>`**
    *   **Flags:** `--manager <mgr>` (**required**), `-c|--comment <comment>` (**required**).
    *   **Args:** Exactly one package name.
    *   **CLI Action:** Parses, constructs `service.CommentArgs`, calls `service.Comment(ctx, args)`. Prints status/error.
    *   **Service Action (`Comment`):**
        1.  Logs command start, gets `command_id`.
        2.  Start DB transaction. Defer rollback.
        3.  Call `store.GetCurrentPackage(tx, name, manager)` to get existing details (needed for log). Handle `ErrPackageNotFound` (rollback, return error).
        4.  Call `store.UpdateCurrentPackageComment(tx, name, manager, newComment)`. Handle error (rollback).
        5.  Call `store.AddLogEntry(tx, commandID, logEntry)` using existing details, `action=comment_changed`, and the new comment. Handle error (rollback).
        6.  Commit transaction. Handle commit error.
        7.  Log command end. *(Separate transaction preferred)*.
        8.  Return success result.

*   **`zenv tag <pkg>` `--manager <mgr>` [flags...]**
    *   **Flags:** `--manager <mgr>` (**required**), `[--add-tag <tag>]...`, `[--remove-tag <tag>]...`. Requires at least one `--add-tag` or `--remove-tag`.
    *   **Args:** Exactly one package name.
    *   **CLI Action:** Parses, constructs `service.TagArgs`, calls `service.Tag(ctx, args)`. Prints status/error.
    *   **Service Action (`Tag`):**
        1.  Logs command start, gets `command_id`.
        2.  Validate input (at least one add/remove flag).
        3.  Start DB transaction. Defer rollback.
        4.  Verify package exists using `store.GetCurrentPackage(tx, name, manager)`. Handle `ErrPackageNotFound` appropriately (rollback, return error).
        5.  Process `--add-tag`: For each tag string provided:
            *   Call `store.FindOrCreateTag(tx, tagName)` to get or create the tag entry in the `tags` table, retrieving its `tag_id`. Handle potential errors during find/create.
            *   Call `store.AddTagToPackage(tx, name, manager, tagID)`. Handle potential errors during association (though `ON CONFLICT DO NOTHING` should make this safe for duplicates).
        6.  Process `--remove-tag`: For each tag string provided:
            *   Attempt to find the tag ID using `store.FindTagByName(tx, tagName)` (or similar). If the tag doesn't exist, this step can be skipped or logged.
            *   If tag ID is found, call `store.RemoveTagFromPackage(tx, name, manager, tagID)`. Handle potential errors during disassociation.
        7.  Create a `package_log` entry with `action=tags_updated`, referencing the `command_id` and summarizing the changes (e.g., added 'foo', removed 'bar') in the log entry's `comment` field for explicit auditability.
        8.  Commit transaction. Handle commit error (log to command history, return error).
        9.  Log command end. *(Separate transaction preferred)*.
        10. Return success result.

*   **`zenv ledger` [`installed`|`removed`|`history`] [flags...]**
    *   **Flags:** `[--tag <t>]...`, `[--reason <r>]`, `[--manager <m>]`, `[--origin <o>]`, `[--since <d>]`, `[--before <d>]`, `[--output <fmt>]` (`r`: `user`|`dependency`|`discovered`, `fmt`: `table`|`json`|`csv`).
    *   **Args:** At most one subcommand (`installed` default).
    *   **CLI Action:** Determines subcommand, parses filter flags, constructs `service.LedgerArgs`, calls `service.GetLedgerData(ctx, args)`. On success, calls `ledger.FormatOutput` with the returned data and format type. Prints formatted output or error message.
    *   **Service Action (`GetLedgerData`):**
        1.  Logs command start, gets `command_id`. (Read-only).
        2.  Parse `LedgerArgs` into `ledger.QueryFilters`.
        3.  Based on subcommand, call the appropriate `store.Get*` method (using `db` Querier), passing filters. **Queries involving `--tag` filters will now require JOINs with `package_tags` and potentially `tags` tables.** Handle store errors.
        4.  Log command end (success/failure) *(Separate transaction preferred)*.
        5.  Return fetched data slice and view type to CLI.

*   **`zenv discover` [`--clear-existing`]**
    *   **Flags:** `--clear-existing` (bool, default: false).
    *   **CLI Action:** Parses flags. If `--clear-existing`, prompts user for confirmation. If confirmed, calls `service.Discover(ctx, args)`. Prints status/error.
    *   **Service Action (`Discover`):**
        1.  Logs command start, gets `commandID`.
        2.  Identify relevant `PackageManager` instances. Check availability. Log PM versions.
        3.  Launch goroutines for `pkgmgr.GetAllInstalledPackages()` concurrently (getting **original install_timestamp**). Collect results (`systemState []PackageInfo`, including best-effort **original install_timestamp**) and errors.
        4.  Aggregate results. Handle errors.
        5.  Start DB transaction. Defer rollback.
        6.  If `clearExisting`: Call `store.DeleteAllCurrentPackages(tx)`. Handle error.
        7.  Get current timestamp for `last_updated_timestamp`.
        8.  For each `pkgInfo` in aggregated results: Create `ledger.CurrentPackage` (populating both timestamps). Call `store.ReplaceCurrentPackage(tx, currentPkg)`.
        9.  Commit transaction. Handle commit error.
        10. Log command end.
        11. Return success result.

*   **`zenv sync` [`--apply`]**
    *   **Flags:** `--apply` (bool, default: false).
    *   **CLI Action:** Parses flags. If `--apply`, prompts user for confirmation. If confirmed (or if `--apply` is false), calls `service.Sync(ctx, args)`. Prints report or status/error.
    *   **Service Action (`Sync`):**
        1.  Logs command start, gets `commandID`.
        2.  Identify relevant `PackageManager` instances. Check availability. Log PM versions.
        3.  Launch goroutines for `pkgmgr.GetAllInstalledPackages()` concurrently (getting **original install_timestamp**). Collect results (`systemState []PackageInfo`).
        4.  Call `store.GetCurrentPackages(db, allFilters)` to get `dbState []CurrentPackage`.
        5.  Compare states: Identify adds, removals, updates (version, origin, etc.).
        6.  Perform optional location checks.
        7.  Generate detailed report (`SyncReport`). Correlate with incomplete commands.
        8.  If `apply` arg is false: Log command end, return report.
        9.  If `apply` arg is true:
            *   Start DB transaction. Defer rollback.
            *   Get current time for `last_updated_timestamp`.
            *   Apply Adds: Create `CurrentPackage` from `systemState` (incl. both timestamps). Call `store.ReplaceCurrentPackage(tx, pkg)`. Call `store.AddLogEntry(tx, commandID, ...)` action=`sync_add`.
            *   Apply Removals: Call `store.GetCurrentPackage(tx, ...)` for prior details. Call `store.DeleteCurrentPackage(tx, ...)`. Call `store.AddLogEntry(tx, commandID, ...)` action=`sync_remove`.
            *   Apply Updates: Call `store.GetCurrentPackage(tx, ...)` for prior details. Create updated `CurrentPackage` from `systemState` (setting new `last_updated_timestamp`, keeping original `install_timestamp`). Call `store.ReplaceCurrentPackage(tx, updatedPkg)`. Call `store.AddLogEntry(tx, commandID, ...)` action=`sync_update`.
            *   Handle errors during store operations (rollback).
            *   Commit transaction. Handle commit error.
            *   Log command end.
            *   Return success/error result.

*   **`zenv completion [bash|zsh|fish|powershell]`**
    *   **CLI Action:** Implemented directly using Cobra's built-in `completion` command functionality. No service layer interaction needed.

*   **`zenv report <subcommand>` [flags...]** (New Command Family)
    *   **Goal:** Provide built-in reports for common analytical queries.
    *   **Subcommands:**
        *   `disk-usage [--manager <mgr>] [--limit N] [--output <fmt>]`: Shows packages sorted by size (requires reliable `size` metadata from adapters).
        *   `licenses [--license <pattern>] [--group-by-package|--group-by-license] [--output <fmt>]`: Reports on licenses (requires reliable `license` metadata).
        *   `activity [--timespan <days|week|month>] [--action <action>] [--output <fmt>]`: Shows recent `package_log` events.
        *   `summary [--group-by manager|reason] [--output <fmt>]`: Shows counts of packages grouped by manager or install reason.
        *   `run <alias_name>`: Executes a predefined report alias from the configuration file.
    *   **CLI Action:** Parses subcommand and flags, constructs `service.ReportArgs`, calls appropriate `service.GetReportData` method (or `service.RunReportAlias`). Formats output using `ledger.FormatOutput`.
    *   **Service Action (`GetReportData`, `RunReportAlias`):**
        *   Logs command start.
        *   Validates args/alias name.
        *   Calls specific `store` methods with appropriate queries (using `GROUP BY`, `ORDER BY`, `LIMIT`, etc.) to gather data. Relies on JOINs where necessary (e.g., for tags if filtering aliases).
        *   For `run <alias>`, reads alias definition from parsed config, translates it into `ledger.QueryFilters`, calls `store.Get*` method.
        *   Logs command end.
        *   Returns data slice to CLI.

*   **`zenv check-health`** (New Command)
    *   **Goal:** Perform internal consistency and health checks.
    *   **CLI Action:** Calls `service.CheckHealth(ctx)`. Prints status report.
    *   **Service Action (`CheckHealth`):**
        *   Logs command start.
        *   Queries `command_history` for entries where `end_timestamp IS NULL`.
        *   Calls `store.CheckDatabaseIntegrity()` (which runs `PRAGMA integrity_check;`).
        *   (Optional) Iterates through configured/detected PMs and calls `pkgmgr.CheckAvailability()`.
        *   Constructs a health report summarizing findings (e.g., "DB Integrity: OK", "Incomplete Commands Found: 2", "DNF Manager: Available").
        *   Logs command end.
        *   Returns report/status to CLI. Recommends `sync` or backup restoration if issues found.

*   **`zenv check-db`** (New Command)
    *   **Goal:** Specifically check database file integrity.
    *   **CLI Action:** Calls `service.CheckDatabaseIntegrity(ctx)`. Prints result.
    *   **Service Action (`CheckDatabaseIntegrity`):** Calls store method to run `PRAGMA integrity_check;` and returns result string.

## 6. Modularity and Extensibility

The architecture is designed with modularity in mind to facilitate maintenance, testing, and future extensions. Key aspects include:

*   **Service Layer (`internal/service`):** This acts as the central hub for core logic, decoupling the command-line interface (and any future interfaces like a TUI) from the underlying data storage and package manager interactions. Changes to CLI presentation or the addition of new frontends should primarily interact with the service layer's defined methods, minimizing impact on core operations.
*   **Package Manager Abstraction (`internal/pkgmgr`):**
    *   **Interface (`pkgmgr.go`):** The `PackageManager` interface provides a strong contract for supporting different package management systems (DNF, APT, Pip, NPM, manual tracking, etc.). Adding support for a new manager primarily involves creating a new struct that implements this interface.
    *   **Capabilities (`Capabilities()` method):** The interface includes a `Capabilities()` method. This allows each package manager implementation to declare which optional metadata fields (like `origin`, `install_reason`, `checksum`, `signature`, `license`, `size`) it can reliably provide. This enables `zenv` (potentially in the service or CLI/TUI layer) to gracefully handle variations in information availability and potentially adapt its behavior or display accordingly.
    *   **Isolation:** Each package manager implementation is self-contained within its own subdirectory (e.g., `internal/pkgmgr/dnf/`), reducing the chance of interference between different manager logics.
*   **Ledger Management (`internal/ledger`):** This package encapsulates all direct database interactions.
    *   **Store (`store.go`):** Provides a clear API for data access, hiding the specifics of SQL queries and database interactions from the service layer. Changes to underlying SQL queries or indexing strategies should be contained within the store.
    *   **Database (`db.go`):** Isolates database connection setup and the schema migration logic.
    *   **Models (`models.go`):** Centralizes the data structure definitions used across the application for database records.
    *   **Formatting (`format.go`):** Separates the concern of how ledger data is presented (table, JSON, CSV) from how it is retrieved or stored.
*   **Utilities (`internal/util`):** Common, reusable functions (like XDG path handling, time utilities, tag parsing) are kept separate, preventing code duplication and promoting reuse.

This separation of concerns aims to make the codebase easier to understand, test (e.g., mocking interfaces between layers), and extend with new features or package manager support in the future.

## 7. Error Handling and Usability

A robust and user-friendly experience relies on clear communication and predictable error handling.

*   **Layered Error Handling:**
    *   **`pkgmgr` Layer:** Implementations must reliably detect errors from underlying package manager commands (non-zero exit codes, specific error messages on stderr). They should return distinct Go errors (e.g., `pkgmgr.ErrNotFound`, `pkgmgr.ErrExecutionFailed`) wrapping underlying context where possible. Avoid returning raw exit codes directly.
    *   **`ledger.Store` Layer:** Methods should return specific errors for database constraints (e.g., `ledger.ErrNotFound`, `ledger.ErrConstraintViolation`) or wrap underlying `database/sql` errors for context.
    *   **`Service` Layer:** This layer is central to handling errors from `pkgmgr` and `store`. It translates lower-level errors into meaningful states or potentially higher-level service errors. It logs detailed internal errors (including stack traces if helpful) to the application log (`internal/applog`). Crucially, it ensures that **failed package manager operations do not result in inconsistent ledger state** (e.g., by rolling back transactions). It updates the `command_history` table with the final exit status and error message.
    *   **`CLI` Layer:** Receives errors from the `Service` layer. It translates these service-level errors into clear, concise, user-facing messages printed to `stderr`. Avoid exposing raw stack traces or overly technical details to the end-user unless a verbose/debug flag is enabled. Exit with appropriate non-zero status codes (`os.Exit(1)` or more specific codes if desired).
*   **Structured Application Logging (`internal/applog`):**
    *   Use Go's standard `log/slog` for structured JSON or key-value logging to `$XDG_STATE_HOME/zenv/zenv.log` (or fallback).
    *   Log levels (DEBUG, INFO, WARN, ERROR) should be used appropriately. DEBUG for verbose execution flow, INFO for major actions (command start/end, sync summary), WARN for unexpected but recoverable situations, ERROR for failures preventing command completion.
    *   Log internal errors with sufficient context (function names, relevant variables) to aid debugging.
*   **Custom Error Types (`internal/util/errors.go`):** Define common custom error types (e.g., `ErrPackageNotFound`, `ErrAmbiguousPackage`, `ErrPkgMgrFailed`, `ErrDBError`) to allow for programmatic checking (`errors.Is`, `errors.As`) in the service and CLI layers.
*   **Input Validation and Sanitization:**
    *   CLI (Cobra) handles basic flag/argument type validation.
    *   Service layer should perform semantic validation (e.g., ensuring required fields are present, checking tag/comment/location/origin formats if necessary) before interacting with the database or package managers.
    *   Sanitize potentially problematic characters in user-provided strings (tags, comments, locations, origins) before database insertion to prevent SQL injection or formatting issues (though prepared statements largely mitigate SQL injection).
*   **Package Manager Error Handling:**
    *   If a `pkgmgr.Install` or `pkgmgr.Remove` call fails, the Service layer **must not** proceed with ledger updates for that operation.
    *   The error (including exit code and potentially captured stderr from the PM) should be logged clearly in the `command_history` entry.
    *   The CLI should inform the user clearly that the operation failed and the ledger was not modified, advising them to check the streamed PM output and the application log file.
*   **Usability Enhancements:**
    *   **Direct PM Output Streaming:** `install` and `remove` commands stream the underlying package manager's raw stdout/stderr directly, allowing users to see familiar progress and error messages. `zenv` adds status messages before and after.
    *   **Clear Status Messages:** Provide informative messages at the start and end of significant operations (`Initializing...`, `Running dnf install...`, `Install successful. Updating ledger...`, `Sync complete. X packages added, Y removed...`).
    *   **Confirmation Prompts:** Use interactive prompts (`y/N`) before executing potentially destructive or irreversible actions, specifically:
        *   `zenv discover --clear-existing`
        *   `zenv sync --apply`
    *   **Shell Completion:** Provide completion scripts for popular shells (Bash, Zsh, Fish) via `zenv completion` command and Makefile target, significantly improving discoverability of commands and flags. Document setup clearly.
    *   **Help Text:** Comprehensive help text for all commands and flags via Cobra (`-h`, `--help`).
    *   **Documentation:** Clear `README.md` and potentially man pages explaining concepts, commands, workflows, and especially the **critical importance of backing up the database** before upgrading `zenv` due to the embedded migration approach. Define `manager` and `origin` field expectations clearly.
    *   **Handling `sudo`:** `pkgmgr` implementations needing root privileges (like `dnf`, `apt`) must prepend `sudo` to their commands. `zenv` itself should generally not require root. Detection/prompting for `sudo` password is handled by `sudo` itself when invoked by the `pkgmgr` adapter.
    *   **Warnings:** Provide warnings for potentially confusing actions, like using `add-manual` with a manager name typically handled by a PM adapter (e.g., `zenv add-manual --manager dnf ...`).

## 8. Testing Strategy

A multi-layered testing approach is essential to ensure correctness, robustness, and maintainability.

*   **Unit Tests:**
    *   **Goal:** Verify individual functions and components in isolation, independent of external systems or databases.
    *   **Scope:**
        *   **`internal/ledger/store`:** Test all data access methods (CRUD, queries). Use mocking libraries like `sqlmock` to simulate database interactions and verify correct SQL generation and parameter handling *without* needing a real database. Test edge cases (not found, constraint violations). Test filter logic thoroughly.
        *   **`internal/ledger/db`:** Test the `InitializeDB` migration logic rigorously. For each migration step, create a temporary SQLite database in the *previous* schema state, run `InitializeDB`, and verify the schema was correctly updated (using `PRAGMA table_info`, `PRAGMA index_list`, `PRAGMA user_version`). Test initialization from an empty state.
        *   **`internal/ledger/format`:** Test output formatting functions for `table`, `json`, `csv` with various inputs, ensuring correctness and handling of edge cases (empty slices, nil values).
        *   **`internal/pkgmgr` Implementations (e.g., `dnf`):** Test functions that parse output or construct commands. Mock external command execution (`os/exec`) to provide controlled input/output/error conditions and verify the adapter's logic (e.g., correctly parsing `rpm -qa` output into `PackageInfo`, correctly extracting metadata, correctly constructing `dnf install` commands). Test the `Capabilities()` method returns accurate information.
        *   **`internal/service`:** Test core service methods (`Install`, `Remove`, `Sync`, `Discover`, etc.). **Mock** the `ledger.Store` and `pkgmgr.PackageManager` interfaces entirely. Verify that the service methods call the correct mocked methods with the expected arguments, handle transactions correctly (begin, commit, rollback scenarios), manage `command_history` logging appropriately, handle errors from dependencies, and implement concurrency logic correctly (for discover/sync).
        *   **`internal/util`:** Test all helper functions (filesystem, time, tags, errors) for correctness and edge cases.
    *   **Location:** `*_test.go` files alongside the code being tested.
    *   **Execution:** Run frequently during development (`go test ./...`).

*   **Integration Tests:**
    *   **Goal:** Verify the interactions *between* different components, particularly the CLI layer, Service layer, and the real Ledger Store (using a temporary database).
    *   **Scope:**
        *   **CLI -> Service -> Store Interaction:** Test CLI command execution (`cobra.Command.Execute()`). Instead of mocking the service layer, inject a *real* service instance. The service instance uses a *real* `ledger.Store` connected to a temporary, ephemeral SQLite database created for the test. **Mock only the `pkgmgr.PackageManager` methods that execute external commands** (`Install`, `Remove`, `GetAllInstalledPackages`, `GetPackageDetails`).
        *   Verify that running a CLI command (e.g., `zenv install ...`) correctly invokes the service, which interacts with the real store, leading to the expected state changes in the temporary database (check `current_packages`, `package_log`, `command_history`) and that the expected `pm_command_string` is logged.
        *   Test database state after sequences of commands.
        *   Verify filter flag functionality (`zenv ledger --manager ... --reason ...`) by checking the output against a pre-populated test database.
        *   Verify different output formats (`--output json`, etc.).
        *   Test `command_history` logging and the link via `command_id` in `package_log`.
    *   **Location:** Can be within `internal/cli/*_test.go` files or a dedicated `tests/integration` directory.
    *   **Execution:** Run as part of the main test suite (`go test ./...`) but potentially tagged separately if they become slow.

*   **End-to-End (E2E) Tests:**
    *   **Goal:** Verify the full application workflow in an environment closely resembling production, including interaction with *real* package managers.
    *   **Method:** Typically involves scripted or manual execution of `zenv` commands within a controlled environment (Docker container, VM) where specific package managers (`dnf`, `pip` etc.) are installed.
    *   **Scope:**
        *   Test `zenv init`.
        *   Test `zenv install <pkg>`: Observe successful installation via the package manager, verify the **streamed PM output**, check `current_packages` and `package_log` entries (including reason, origin, best-effort metadata), verify `command_history`.
        *   Test `zenv remove <pkg>`: Observe successful removal, verify streamed output, check `current_packages` deletion and `package_log` entry, verify `command_history`.
        *   Test `add-manual` with location/origin.
        *   Test `discover` and `sync`: Run PM commands *outside* of `zenv`, then run `zenv sync` and `zenv sync --apply` to verify detection and correction of discrepancies. Test `--clear-existing`.
        *   Test `ledger` with various filters and output formats against a populated ledger.
        *   Test `tag`, `comment`, `remove-manual-log`.
        *   Test shell completion script generation and basic functionality in relevant shells.
        *   Test error handling for invalid commands, flags, PM failures.
    *   **Location:** Documented test cases in `docs/TESTING.md`, potentially with helper scripts in `scripts/`.
    *   **Execution:** Run less frequently, typically before releases or major changes, due to setup and execution time.

*   **Automation (CI):**
    *   Configure GitHub Actions (or similar CI system) in `.github/workflows/test.yml`.
    *   Workflow should trigger on pushes and pull requests.
    *   Run `make lint` (using `golangci-lint`).
    *   Run `make test` (executes unit and integration tests).
    *   Optionally build binaries for different platforms.
    *   (E2E tests are often run manually or in separate, more complex CI jobs due to environment setup).

## 9. Build and Documentation

Comprehensive build processes and documentation are crucial for usability and maintainability.

*   **Dependency Management:** Use Go modules (`go.mod`, `go.sum`) to manage project dependencies. Core external dependencies include `github.com/spf13/cobra` (for CLI) and `github.com/mattn/go-sqlite3` (for SQLite driver).
*   **Build Process (`Makefile`):** Provide a `Makefile` defining standard targets:
    *   `build`: Compiles the `zenv` binary (e.g., `go build -o zenv ./cmd/zenv`). Should include version information embedding (e.g., via `-ldflags`).
    *   `test`: Runs all unit and integration tests (`go test ./... -race` recommended).
    *   `lint`: Runs static analysis checks (e.g., `golangci-lint run ./...`).
    *   `completion`: Generates shell completion scripts for various shells into the `completion/` directory (e.g., using `zenv completion bash > completion/zenv.bash`).
    *   `man`: Generates the man page from the source file (e.g., using `pandoc` or similar tool on `docs/man/zenv.1.md`).
    *   `install`: Installs the compiled binary, the generated man page, and potentially the shell completion scripts to standard system locations (e.g., `/usr/local/bin`, `/usr/local/share/man/man1`, shell-specific completion directories). Requires careful handling of permissions and destinations.
    *   `clean`: Removes build artifacts and generated files.
    *   **(New/Modify) `migrate-create NAME=<migration_name>`:** Convenience target to invoke `golang-migrate create` for new migration files.
    *   **(Modify) `db-migrate-up`, `db-migrate-down`:** Targets to apply/revert migrations using `golang-migrate` CLI against a specified DB file (for development/testing).
*   **User Documentation:**
    *   **`README.md`:** Cover purpose, installation, completion setup. Explain `manager`/`origin`. Explain **normalized tags** and the `tag` command. Explain `command_history` and audit value. Explain metadata fields. Explain `check-health`, `report` commands. Explain **configuration file usage** (`$XDG_CONFIG_HOME/zenv/config.toml`) for defaults and report aliases. **Prominently warn about backing up `ledger.db`** (mentioning migration process). Mention `applog` location.
    *   **Man Page (`docs/man/zenv.1.md`):** Update with new commands and config file details.
    *   **Command Help Text (`-h`, `--help`):** Comprehensive help generated by Cobra.
*   **Developer Documentation:**
    *   **`CONTRIBUTING.md`:** Guidelines. Explain how to add migrations using `golang-migrate`.
    *   **Architecture Document (This file):** Keep updated.
    *   **Code Comments:** Explain non-obvious logic.
*   **Testing Documentation (`docs/TESTING.md`):** Update E2E scenarios for new commands, tag handling, config file variations, and multi-repo testing.

## 10. Future Considerations

This section outlines potential enhancements and features beyond the core specification.

*   **Dependency Tracking Integration:** Integrate the separately designed dependency tracking system (potentially named `depplib`) as a library. This would involve calling its discovery mechanism and providing commands like `zenv depends <pkg>` via its API.
*   **TUI Frontend:** Develop an interactive Terminal User Interface (TUI) using a framework like `bubbletea`.
    *   The TUI would act as an alternative frontend, interacting directly with the `internal/service` layer.
    *   It could provide sortable/filterable table views of the ledger (`installed`, `history`, `commands`), potentially visual representations of timelines or package relationships (if dependency tracking is integrated), and interactive prompts for executing `zenv` commands.
*   **Additional Package Manager Support:** Implement more adapters satisfying the `PackageManager` interface.
    *   Target managers include `apt` (Debian/Ubuntu), `pacman` (Arch), `brew` (macOS), `pip` (Python), `npm` (Node.js), `cargo` (Rust), `gem` (Ruby), etc.
    *   Each implementation would need to handle command execution, output streaming, post-operation checks for package details (including best-effort fetching of checksums, signatures, license, size), and accurately report its capabilities via the `Capabilities()` method.
*   **Sophisticated Ledger Querying & Reporting Presets:** Enhance querying beyond the current flags.
    *   Implement more advanced filters within `internal/ledger/store` (e.g., regular expression matching on names/origins/comments, more complex date/time range logic).
    *   Expose these filters via new CLI flags on `zenv ledger`.
    *   Introduce dedicated reporting commands (e.g., `zenv report orphans` [requires `depplib` info], `zenv report disk-usage --manager <mgr>`, `zenv report license-audit`) that perform common analytical queries. (NOTE: Moved core reports out of future considerations).
*   **Configuration File:** Implement reading settings from a configuration file (e.g., `$XDG_CONFIG_HOME/zenv/config.toml` or YAML). (NOTE: Moved out of future considerations).
*   **Shell Integration & User Hooks:** Explore deeper integration with the user's shell environment.
    *   Potentially hook into shell history (e.g., `zsh` precmd/preexec hooks) to automatically suggest or log commands.
    *   Implement a user script hook system (e.g., placing executable scripts in `~/.config/zenv/hooks/post-install.d/`) that `zenv` executes after specific successful actions, passing event details via environment variables or stdin.
*   **Advanced Management Features:**
    *   **Manual Reason Change:** Add a command (e.g., `zenv modify <pkg> --manager <mgr> --set-reason <user|dependency>`) to allow manual correction of the `install_reason` field for specific packages.
    *   **Background Sync:** Implement an optional daemon mode or systemd service/timer to run `zenv sync` periodically in the background.
    *   **Backup Helper:** Add a simple `zenv backup <destination_path>` command to conveniently copy the SQLite database file, while still documenting that this is just a file copy and restoration is manual.
*   **Database Migration Library:** If the embedded migration logic in `internal/ledger/db.go` becomes too complex or difficult to manage due to frequent or intricate schema changes, adopt a dedicated library like `github.com/golang-migrate/migrate`.
    *   This would involve managing migrations as separate `.sql` files (up/down) and using the library's CLI or Go API to apply them.
*   **Dynamic UI based on Capabilities:** Enhance the CLI and potential TUI to query the `PackageManager.Capabilities()` method for relevant managers.
    *   Dynamically hide or disable filter flags (`--reason`, `--origin`, etc.) on the `ledger` command if the selected manager(s) do not support providing that data.
    *   Dynamically hide columns in table output if the data is known to be unavailable for the displayed managers.
*   **Snapshot Command:** Implement `zenv snapshot [--output <file>] [--format <fmt>] [filter-flags...]`.
    *   This command would reuse the filtering logic from `zenv ledger` (`service.GetLedgerData`) to query the current state (`current_packages`) or history (`package_log`) based on filters.
    *   It would format the output (table, JSON, CSV) and write it to the specified file or stdout.
    *   The execution of the snapshot command itself would be logged in `command_history`.

## 11. Proposed Directory Structure

```
zenv/
├── .github/                     # GitHub specific files
│   └── workflows/             # GitHub Actions CI/CD workflow definitions
│       └── test.yml
├── cmd/                         # Main application(s) entry points
│   └── zenv/                  # Entry point for the zenv CLI application
│       ├── main.go             # Sets up dependencies (DB, PkgMgrs, Service), wires up Cobra commands, executes root command
│       └── main_test.go        # Basic integration tests for main setup (optional)
├── completion/                  # Generated shell completion scripts (output location for `make completion`)
│   ├── zenv.bash
│   ├── zenv.zsh
│   ├── zenv.fish
│   └── zenv.ps1
├── docs/                        # Project documentation
│   ├── man/                   # Man page source files
│   │   └── zenv.1.md
│   └── TESTING.md              # Manual E2E testing procedures and scenarios
├── internal/                    # Private application code (not intended for import by other projects)
│   ├── applog/                 # Application logging setup
│   │   └── log.go              # Configures the global slog logger
│   ├── cli/                    # Cobra command definitions & the thin CLI interaction layer
│   │   ├── root.go             # Root Cobra command setup, global flags, completion command definition
│   │   ├── init.go             # Implementation for the `zenv init` command
│   │   ├── install.go          # Implementation for `zenv install` (parses flags, calls service.Install)
│   │   ├── remove.go           # Implementation for `zenv remove` (parses flags, calls service.Remove)
│   │   ├── ledger_cmd.go       # Implementation for `zenv ledger` (parses filters/flags, calls service, formats)
│   │   ├── comment.go          # Implementation for `zenv comment`
│   │   ├── add_manual.go       # Implementation for `zenv add-manual`
│   │   ├── remove_manual_log.go # Implementation for `zenv remove-manual-log`
│   │   ├── discover.go         # Implementation for `zenv discover`
│   │   ├── sync.go             # Implementation for `zenv sync`
│   │   ├── tag.go              # Implementation for `zenv tag` (handles normalized tags)
│   │   ├── report_cmd.go       # `zenv report` (new: handles subcommands like disk-usage, run alias)
│   │   ├── check_health.go     # `zenv check-health` (new)
│   │   └── check_db.go         # `zenv check-db` (new)
│   ├── service/                # Core business logic / Orchestration Layer
│   │   ├── service.go        # Service struct definition, NewService constructor, core method implementations (Install, Remove, Sync, Discover, GetLedgerData, etc.) managing transactions and orchestrating calls
│   │   ├── command_log.go    # Helper functions specifically for creating/updating `command_history` entries within the service layer
│   │   └── service_test.go   # Unit tests for the service layer logic (mocking store and pkgmgr)
│   ├── ledger/                 # SQLite Ledger Management
│   │   ├── db.go               # Database connection setup, runs migrations via library
│   │   ├── db_test.go          # Tests specifically for initialization/migration runner logic
│   │   ├── store.go            # Data access methods (Store struct), handles normalized tags, JOINs
│   │   ├── store_test.go       # Unit tests for store methods using sqlmock or test DB
│   │   ├── models.go           # Go struct definitions (Tag, PackageTag added, tags field removed)
│   │   ├── format.go           # Functions for formatting query results
│   │   ├── format_test.go      # Unit tests for formatting functions
│   │   └── migrations/         # SQL Migration files managed by golang-migrate (new)
│   │       ├── 001_initial_schema.up.sql
│   │       └── 001_initial_schema.down.sql
│   ├── pkgmgr/                 # Package Manager Abstraction & Implementations
│   │   ├── pkgmgr.go           # Defines PackageManager interface, PackageInfo, Capabilities, CheckAvailability()
│   │   └── dnf/                # Example Implementation for DNF/RPM
│   │       ├── dnf.go            # DNF adapter implementing PackageManager
│   │       └── dnf_test.go       # Unit tests for the DNF adapter
│   └── util/                   # Utility functions
│       ├── errors.go           # Custom error type definitions
│       ├── fs.go               # Filesystem path helpers (XDG base dirs, EnsureDir, config path)
│       ├── timeutil.go         # Time formatting/parsing helpers
│       ├── tags.go             # (Scope reduced or removed)
│       └── ..._test.go         # Unit tests for utility functions
├── scripts/                    # Helper scripts (e.g., for build, release, complex test setups - optional)
│   └── ...
├── .gitignore                  # Specifies intentionally untracked files for Git
├── go.mod                      # Go module definition (dependencies)
├── go.sum                      # Go module checksums
├── LICENSE                     # Project license file (e.g., MIT, Apache 2.0)
├── Makefile                    # Defines common development tasks (build, test, lint, man, completion, install)
├── README.md                   # Main project documentation (overview, installation, usage)
└── CONTRIBUTING.md             # Guidelines for contributors
```

## 12. Implementation Plan

1.  **Project Setup & Core Dependencies:**
    *   Initialize Go module.
    *   Add dependencies: `cobra`, `go-sqlite3`, `golang-migrate/migrate` (and drivers), TOML parser.
    *   Create initial directory structure including `internal/ledger/migrations/`.
    *   Set up basic `Makefile` (include build, test, lint, `migrate-create`, `db-migrate-*` targets). Set up `.gitignore`.
2.  **Ledger Schema & Migrations (`internal/ledger`):**
    *   Define Go structs (`models.go`) reflecting the normalized tag schema.
    *   Create initial migration files (`001_... .up.sql`, `.down.sql`) in `migrations/` for all tables and indexes.
    *   Implement database connection logic (`db.go`: `OpenDB`, `CloseDB`) and the `InitializeDB` function using `golang-migrate/migrate` API to apply migrations from embedded/loaded files. Write unit tests for initialization and migration runner logic.
3.  **Core Utilities & Logging (`internal/util`, `internal/applog`):**
    *   Implement essential helper functions (XDG paths, time, errors). Set up structured logging (`applog`).
    *   Write unit tests for utility functions.
    *   *Define custom sentinel error types (`util/errors.go`) early.*
4.  **Ledger Store (`internal/ledger/store.go`):**
    *   Implement `Store` struct, `NewStore`, `Querier` interface.
    *   Implement all DAO methods for `command_history`, `current_packages`, `package_log`, and normalized tags (`FindOrCreateTag`, `AddTagToPackage`, etc.), including necessary JOINs for tag queries. Implement `CheckDatabaseIntegrity`.
    *   *Implement store-level error handling: Wrap DB errors, return `util.ErrPackageNotFound` consistently, handle constraint violations.*
    *   Write comprehensive unit tests using mocks (`sqlmock`) or test DBs, covering CRUD, queries, filters, JOINs, and error conditions.
5.  **Package Manager Abstraction (`internal/pkgmgr`):**
    *   Define `PackageManager` interface, `PackageInfo`, `Capabilities`, `CheckAvailability()`.
    *   Implement the initial `dnf` adapter, including `CheckAvailability()`, robust metadata/origin capture, and logging PM version to `applog`.
    *   *Implement adapter-level error handling: Return specific errors like `util.ErrPkgMgrFailed`, wrap context from failed commands.*
    *   Write unit tests for the adapter, mocking `os/exec` and testing parsing logic and error paths.
6.  **Configuration Loading:**
    *   Implement config file loading (`$XDG_CONFIG_HOME/zenv/config.toml`), validation, and definition of the config struct. Load early in `main.go`.
7.  **Service Layer (`internal/service`):**
    *   Implement `Service` struct, `NewService` (injecting store, pkgmgrs, config).
    *   Implement core service methods (`Install`, `Remove`, `Sync`, `Discover`, `GetLedgerData`, `AddManual`, `Comment`, `Tag`, etc.). Ensure command start logged immediately (Tx1). Enhance `Sync` logic for reporting context based on incomplete commands.
    *   Implement new service methods (`CheckHealth`, `CheckDatabaseIntegrity`, `GetReportData`, `RunReportAlias`).
    *   *Implement central error handling: Coordinate errors from store/pkgmgr, manage transaction atomicity (rollback on failure), log internal errors thoroughly, ensure `command_history` reflects final status.*
    *   *Apply configuration settings*: Use loaded config to modify behavior (e.g., metadata fetching toggles, default report formats).
    *   Write unit tests for service methods, mocking dependencies and verifying logic, transaction handling, and error coordination.
8.  **CLI Layer (`internal/cli`):**
    *   Set up root command and subcommands using Cobra.
    *   Implement `RunE` functions: parse flags/args, create request structs, call `service` methods, handle streaming output, print results. Pass config down where needed. Implement new commands (`report`, `check-health`, `check-db`).
    *   *Implement user-facing error presentation: Translate service errors into clear messages, guide users on failures, use appropriate exit codes.*
9.  **Integration Testing:**
    *   *Develop integration tests concurrently with CLI/Service development.* Test key workflows (CLI -> Service -> Store (temp DB) -> Mocked PkgMgr), verifying data flow, state changes, output, and error handling across layers. Test various flag combinations and config file interactions.
10. **Error Handling Refinement:**
    *   *Review and refine error handling consistency across all layers.* Ensure errors are logged with sufficient context (`applog`) and user-facing messages are clear and helpful. Verify specific error types are handled appropriately (e.g., integrity check failures, PM failures). Detail specific error wrapping and propagation strategies between layers. Define expected user messages for common failure modes (PM unavailable, DB corrupt, package not found, etc.).
11. **Testing Refinement:**
    *   *Review and enhance test coverage.* Ensure unit and integration tests cover core logic, critical paths (including transaction rollbacks), specific error conditions identified in error handling refinement, edge cases (empty inputs, large datasets), and configuration variations. Define E2E test scenarios in `docs/TESTING.md` with more specificity about expected inputs/outputs and system states.
12. **Shell Completion & Build System:**
    *   Ensure Cobra completion generation works. Finalize `Makefile` targets (build, test, lint, completion, man, install, migrate-*). Ensure build includes embedded version info.
13. **Documentation:**
    *   Write/update `README.md`, man page, command help text, `CONTRIBUTING.md`, `TESTING.md` comprehensively, reflecting the final design, commands, config usage, migration strategy (`golang-migrate` use), backup requirements, error handling philosophy, and detailed usage examples.
14. **End-to-End Testing:**
    *   Execute E2E tests based on `docs/TESTING.md` in VM/container environments. Focus on multi-repo scenarios, PM update compatibility (using up-to-date environments), tag operations, health checks, reporting, config file variations, and specific error recovery paths (e.g., running `sync` after simulated failures).
15. **Build & Release:** Finalize `Makefile`, tag release.
