# zenv - Visualized Architecture and Flows

This document provides visual representations (using Mermaid.js) of the `zenv` tool's architecture, database schema, and command flows.

*mermaid renderer is only available on github web, AFAIK the following diagrams/charts won't render on github mobile apps, view this page on a mobile browser if you need to* 

## 1. Overall Architecture

This diagram shows the high-level components of `zenv` and how they interact when a user executes a command.

```mermaid
graph TD
    subgraph "User Interaction"
        User([User]) --> CLI
    end

    subgraph "Core Components"
        CLI[CLI Interface] --> Service
        Service[Service Layer] --> Store
        Service --> PkgMgr
        Service --> AppLog
        Service --> Utils
    end
    
    subgraph "Data Management"
        Store[Ledger Store] --> DB[(SQLite DB)]
        Store --> Models[Data Models]
        DB -.-> MigrateLib[Migrations]
    end
    
    subgraph "Package Management"
        PkgMgr[Package Manager<br>Abstraction] --> PkgMgrImpl
        PkgMgrImpl[PM Implementations<br>DNF, APT, PIP...] --> SystemPM[System Package<br>Managers]
    end
    
    subgraph "Presentation"
        CLI --> Formatter[Formatter]
        Store --> Formatter
    end

    %% External connections
    SystemPM --> PkgMgrImpl
    PkgMgrImpl --> CLI
    AppLog[App Logger] --> LogFile[zenv.log]

    %% Styling
    classDef primary fill:#9cf,color:#000,stroke:#333,stroke-width:1px
    classDef secondary fill:#fec,color:#000,stroke:#333,stroke-width:1px
    classDef tertiary fill:#ccf,color:#000,stroke:#333,stroke-width:1px
    classDef storage fill:#f9d,color:#000,stroke:#333,stroke-width:1px
    classDef utility fill:#eee,color:#000,stroke:#333,stroke-width:1px
    
    class User,CLI tertiary
    class Service primary
    class Store,PkgMgr,PkgMgrImpl,Formatter secondary
    class DB storage
    class AppLog,Utils,Models,MigrateLib,SystemPM,LogFile utility
```

**Explanation:**

1.  **User Input:** The user interacts with `zenv` via the command line.
2.  **CLI Layer (`internal/cli`):** Parses the user's command, flags, and arguments using the Cobra library. Acts as a thin client.
3.  **Service Layer (`internal/service`):** The core orchestration engine. Receives requests from the CLI, manages database transactions, coordinates interactions between the ledger store and package manager adapters, handles core business logic, and logs command history.
4.  **Ledger Store (`internal/ledger/store`):** The Data Access Object (DAO) layer. Encapsulates all SQL logic for interacting with the SQLite database. Provides methods for CRUD operations and querying the ledger tables (`command_history`, `current_packages`, `package_log`, `tags`, `package_tags`).
5.  **Package Manager Abstraction (`internal/pkgmgr`):** Defines the `PackageManager` interface and contains specific implementations (adapters) for different package managers (e.g., `dnf`, `apt`, `pip`). Handles executing underlying PM commands, post-operation checks for metadata, and reporting capabilities.
6.  **SQLite Database (`ledger.db`):** The persistent storage for all tracked package information, logs, tags, and command history. Schema is managed via `golang-migrate/migrate`.
7.  **Underlying Package Manager:** The actual system or language package manager (e.g., `dnf`, `apt`) that `zenv` wraps.
8.  **Formatter (`internal/ledger/format`):** Takes data retrieved by the Store and formats it into user-requested outputs (table, JSON, CSV).
9.  **Configuration (`internal/util/fs`, `config.toml`):** Handles loading settings from a config file (e.g., defaults, report aliases).
10. **Logger (`internal/applog`):** Provides structured logging for debugging and diagnostics to `zenv.log`.
11. **Utilities (`internal/util`):** Contains shared helper functions (filesystem, time, errors).
12. **Models (`internal/ledger/models`):** Defines the Go structs mapping to database tables.

## 2. Ledger Database Schema (ER Diagram)

This diagram shows the structure of the SQLite database (`ledger.db`) and the relationships between the tables used to store the audit trail and package state.

```mermaid
erDiagram
    COMMAND_HISTORY ||--o{ PACKAGE_LOG : "Initiated"
    CURRENT_PACKAGES ||--o{ PACKAGE_TAGS : "Has"
    TAGS ||--o{ PACKAGE_TAGS : "Assigned To"
    PACKAGE_LOG ||--o{ COMMAND_HISTORY : "Linked To"

    COMMAND_HISTORY {
        INTEGER command_id PK
        TEXT start_timestamp "NOT NULL"
        TEXT end_timestamp "NULL"
        TEXT zenv_version "NOT NULL"
        TEXT command_string "NOT NULL"
        TEXT pm_command_string "NULL"
        INTEGER exit_code "NULL"
        TEXT error_message "NULL"
        TEXT details "NULL"
    }

    CURRENT_PACKAGES {
        TEXT package_name PK
        TEXT manager PK
        TEXT origin "NULL"
        TEXT package_version "NULL"
        TEXT install_reason "NULL"
        TEXT location "NULL"
        TEXT comment "NULL"
        TEXT install_timestamp "NULL"
        TEXT last_updated_timestamp "NOT NULL"
        TEXT checksum "NULL"
        TEXT signature "NULL"
        TEXT license "NULL"
        INTEGER size "NULL"
    }

    PACKAGE_LOG {
        INTEGER id PK
        INTEGER command_id "FK NULL"
        TEXT timestamp "NOT NULL"
        TEXT action "NOT NULL"
        TEXT package_name "NOT NULL"
        TEXT manager "NOT NULL"
        TEXT origin "NULL"
        TEXT package_version "NULL"
        TEXT install_reason "NULL"
        TEXT location "NULL"
        TEXT comment "NULL"
        TEXT checksum "NULL"
        TEXT signature "NULL"
        TEXT license "NULL"
        INTEGER size "NULL"
    }

    TAGS {
        INTEGER tag_id PK
        TEXT name "NOT NULL UNIQUE"
    }

    PACKAGE_TAGS {
        TEXT package_name "PK FK"
        TEXT manager "PK FK"
        INTEGER tag_id "PK FK"
    }
```

**Explanation:**

*   `COMMAND_HISTORY`: Records every invocation of the `zenv` command.
*   `CURRENT_PACKAGES`: Stores the last known state of each package tracked by `zenv`. Uniquely identified by `package_name` and `manager`.
*   `PACKAGE_LOG`: An append-only log of specific events (install, remove, tag update, sync change) affecting packages. Each event is linked back to the `COMMAND_HISTORY` entry that caused it (if applicable).
*   `TAGS`: Stores unique tag names.
*   `PACKAGE_TAGS`: A linking table connecting packages in `CURRENT_PACKAGES` to their assigned tags in the `TAGS` table, enabling many-to-many relationships. Foreign key constraints with `ON DELETE CASCADE` ensure data integrity when packages or tags are removed.

## 3. Command User Flows (Sequence Diagrams)

These diagrams illustrate the typical sequence of interactions between components for key `zenv` commands.

### 3.1 `zenv init`

```mermaid
sequenceDiagram
    actor U as User
    participant CLI
    participant SVC as Service
    participant UT as Utils
    participant DB
    participant MG as Migrate
    participant ST as Store
    participant LOG as AppLog

    U->>CLI: zenv init
    CLI->>SVC: Initialize(ctx)
    
    activate SVC
    SVC->>UT: GetDBPath/GetLogPath()
    activate UT
    UT-->>SVC: Paths
    deactivate UT
    
    SVC->>UT: EnsureDir(paths)
    activate UT
    UT-->>SVC: Directories created
    deactivate UT
    
    SVC->>LOG: ConfigureLogging(path)
    activate LOG
    LOG-->>SVC: Logger configured
    deactivate LOG
    
    SVC->>DB: OpenDB(path)
    activate DB
    DB-->>SVC: Connection pool
    deactivate DB
    
    SVC->>DB: InitializeDB(pool)
    activate DB
    DB->>DB: PRAGMA foreign_keys = ON
    
    DB->>MG: NewWithDatabaseInstance
    activate MG
    MG-->>DB: Migration instance
    deactivate MG
    
    DB->>MG: Up()
    activate MG
    Note over MG: Apply migrations
    MG-->>DB: Result
    deactivate MG
    
    alt Migration Needed or New DB
        DB-->>SVC: Migrations applied
        SVC->>SVC: Start transaction
        SVC->>ST: AddLogEntry(init)
        activate ST
        ST-->>SVC: Entry added
        deactivate ST
        SVC->>SVC: Commit transaction
    else Schema Up-to-Date
        DB-->>SVC: Schema OK
    end
    deactivate DB
    
    SVC->>DB: CloseDB()
    activate DB
    DB-->>SVC: Closed
    deactivate DB
    
    SVC-->>CLI: Result
    deactivate SVC
    
    CLI-->>U: Initialization complete
```

### 3.2 `zenv install <pkg> --tag <tag>`

```mermaid
sequenceDiagram
    actor U as User
    participant CLI
    participant SVC as Service
    participant PM as PkgMgr
    participant DNF
    participant SYS as System PM
    participant ST as Store

    U->>CLI: zenv install pkg --tag mytag
    CLI->>SVC: Install(ctx, args, stdout, stderr)
    
    activate SVC
    SVC->>SVC: Start cmd_start transaction
    SVC->>ST: AddCommandStart
    activate ST
    ST-->>SVC: commandID
    deactivate ST
    SVC->>SVC: Commit cmd_start transaction

    SVC->>PM: Get PackageManager
    activate PM
    PM-->>SVC: DNF adapter
    deactivate PM
    
    SVC->>DNF: Install(packages, options)
    activate DNF
    DNF->>SYS: sudo dnf install -y pkg
    activate SYS
    Note over SYS: Execute & stream output
    SYS-->>DNF: Exit code 0
    deactivate SYS
    DNF-->>SVC: Command string, nil error
    deactivate DNF

    SVC->>SVC: Start main transaction
    SVC->>ST: UpdateCommandPMString
    activate ST
    ST-->>SVC: OK
    deactivate ST

    SVC->>DNF: GetPackageDetails
    activate DNF
    DNF->>SYS: rpm -q pkg / dnf info
    activate SYS
    SYS-->>DNF: Package details
    deactivate SYS
    DNF-->>SVC: PackageInfo
    deactivate DNF

    SVC->>ST: ReplaceCurrentPackage
    activate ST
    ST-->>SVC: OK
    deactivate ST
    
    SVC->>ST: AddLogEntry(installed)
    activate ST
    ST-->>SVC: OK
    deactivate ST

    SVC->>ST: FindOrCreateTag(mytag)
    activate ST
    ST-->>SVC: tagID
    deactivate ST
    
    SVC->>ST: AddTagToPackage
    activate ST
    ST-->>SVC: OK
    deactivate ST

    SVC->>SVC: Commit main transaction
    SVC->>SVC: Start cmd_end transaction
    SVC->>ST: UpdateCommandEnd
    activate ST
    ST-->>SVC: OK
    deactivate ST
    SVC->>SVC: Commit cmd_end transaction

    SVC-->>CLI: Success result
    deactivate SVC
    
    CLI-->>U: Installation successful
```

### 3.3 `zenv remove <pkg>`

```mermaid
sequenceDiagram
    actor U as User
    participant CLI
    participant SVC as Service
    participant PM as PkgMgr
    participant DNF
    participant SYS as System PM
    participant ST as Store

    U->>CLI: zenv remove pkg
    CLI->>SVC: Remove(ctx, args, stdout, stderr)
    
    activate SVC
    SVC->>SVC: Start cmd_start transaction
    SVC->>ST: AddCommandStart
    activate ST
    ST-->>SVC: commandID
    deactivate ST
    SVC->>SVC: Commit cmd_start transaction

    SVC->>PM: Get PackageManager
    activate PM
    PM-->>SVC: DNF adapter
    deactivate PM

    SVC->>SVC: Start main transaction
    SVC->>ST: GetCurrentPackage(pkg)
    activate ST
    ST-->>SVC: priorDetails
    deactivate ST
    
    SVC->>DNF: Remove(packages, options)
    activate DNF
    DNF->>SYS: sudo dnf remove -y pkg
    activate SYS
    Note over SYS: Execute & stream output
    SYS-->>DNF: Exit code 0
    deactivate SYS
    DNF-->>SVC: Command string, nil error
    deactivate DNF

    SVC->>ST: UpdateCommandPMString
    activate ST
    ST-->>SVC: OK
    deactivate ST

    SVC->>ST: DeleteCurrentPackage
    activate ST
    ST-->>SVC: OK
    deactivate ST
    
    SVC->>ST: AddLogEntry(removed)
    activate ST
    ST-->>SVC: OK
    deactivate ST

    SVC->>SVC: Commit main transaction
    SVC->>SVC: Start cmd_end transaction
    SVC->>ST: UpdateCommandEnd
    activate ST
    ST-->>SVC: OK
    deactivate ST
    SVC->>SVC: Commit cmd_end transaction

    SVC-->>CLI: Success result
    deactivate SVC
    
    CLI-->>U: Removal successful
```

### 3.4 `zenv ledger installed --tag <tag>`

```mermaid
sequenceDiagram
    actor U as User
    participant C as CLI
    participant S as Service
    participant D as Store
    participant F as Formatter

    U->>C: ledger installed --tag mytag
    C->>S: GetLedgerData(args)
    
    activate S
    S->>S: Start transaction
    S->>D: AddCommandStart
    activate D
    D-->>S: commandID
    deactivate D
    S->>S: Commit transaction

    S->>D: GetCurrentPackages
    activate D
    Note over D: Query with JOINs
    D-->>S: results
    deactivate D

    S->>S: Start transaction
    S->>D: UpdateCommandEnd
    activate D
    D-->>S: OK
    deactivate D
    S->>S: Commit transaction

    S-->>C: Return results
    deactivate S
    
    C->>F: Format results
    activate F
    F-->>C: Formatted data
    deactivate F
    
    C-->>U: Display results
```

### 3.5 `zenv tag <pkg> --add-tag <t1> --remove-tag <t2>`

```mermaid
sequenceDiagram
    actor U as User
    participant C as CLI
    participant S as Service
    participant D as Store

    U->>C: tag pkg --add/remove tags
    C->>S: Tag(args)
    
    activate S
    S->>S: Start transaction
    S->>D: AddCommandStart
    activate D
    D-->>S: commandID
    deactivate D
    S->>S: Commit transaction

    S->>S: Start transaction
    S->>D: GetCurrentPackage
    activate D
    D-->>S: package
    deactivate D

    S->>D: FindOrCreateTag
    activate D
    D-->>S: tagID
    deactivate D
    
    S->>D: AddTagToPackage
    activate D
    D-->>S: OK
    deactivate D

    S->>D: FindTagByName
    activate D
    D-->>S: tagID
    deactivate D
    
    S->>D: RemoveTagFromPackage
    activate D
    D-->>S: OK
    deactivate D

    S->>D: AddLogEntry
    activate D
    D-->>S: OK
    deactivate D

    S->>S: Commit transaction
    S->>S: Start transaction
    S->>D: UpdateCommandEnd
    activate D
    D-->>S: OK
    deactivate D
    S->>S: Commit transaction

    S-->>C: Success
    deactivate S
    
    C-->>U: Tags updated
```

### 3.6 `zenv sync`

```mermaid
sequenceDiagram
    actor User
    participant CLI
    participant Service
    participant PKM as Package Managers
    participant DNF as DNF Adapter
    participant PIP as Pip Adapter
    participant Store

    User->>CLI: zenv sync [--apply]
    CLI->>Service: Sync(ctx, {Apply: true/false})
    
    activate Service
    Service->>Service: Start cmd_start transaction
    Service->>Store: AddCommandStart
    activate Store
    Store-->>Service: commandID=127
    deactivate Store
    Service->>Service: Commit cmd_start transaction

    Service->>PKM: Get Available Package Managers
    activate PKM
    PKM-->>Service: [DNF, PIP]
    deactivate PKM
    Service->>Service: Check availability

    par Get DNF Packages
        Service->>DNF: GetAllInstalledPackages()
        activate DNF
        DNF->>DNF: Query system
        DNF-->>Service: dnfPkgs
        deactivate DNF
    and Get Pip Packages
        Service->>PIP: GetAllInstalledPackages()
        activate PIP
        PIP->>PIP: Query system
        PIP-->>Service: pipPkgs
        deactivate PIP
    end

    Service->>Store: GetCurrentPackages
    activate Store
    Store-->>Service: dbState
    deactivate Store

    Service->>Service: Compare states
    Service->>Service: Identify changes

    alt Apply is true
        Service->>Service: Start main transaction
        
        loop Apply Adds
            Service->>Store: ReplaceCurrentPackage
            activate Store
            Store-->>Service: OK
            deactivate Store
            Service->>Store: AddLogEntry
            activate Store
            Store-->>Service: OK
            deactivate Store
        end

        loop Apply Removals
            Service->>Store: GetCurrentPackage
            activate Store
            Store-->>Service: priorDetails
            deactivate Store
            Service->>Store: DeleteCurrentPackage
            activate Store
            Store-->>Service: OK
            deactivate Store
            Service->>Store: AddLogEntry
            activate Store
            Store-->>Service: OK
            deactivate Store
        end

        loop Apply Updates
            Service->>Store: GetCurrentPackage
            activate Store
            Store-->>Service: priorDetails
            deactivate Store
            Service->>Store: ReplaceCurrentPackage
            activate Store
            Store-->>Service: OK
            deactivate Store
            Service->>Store: AddLogEntry
            activate Store
            Store-->>Service: OK
            deactivate Store
        end

        Service->>Service: Commit main transaction
        Service->>Service: Create summary
        Service->>Service: Start cmd_end transaction
        Service->>Store: UpdateCommandEnd
        activate Store
        Store-->>Service: OK
        deactivate Store
        Service->>Service: Commit cmd_end transaction

        Service-->>CLI: Success result
    else Apply is false
        Service->>Service: Generate report
        Service->>Service: Start cmd_end transaction
        Service->>Store: UpdateCommandEnd
        activate Store
        Store-->>Service: OK
        deactivate Store
        Service->>Service: Commit cmd_end transaction

        Service-->>CLI: Report Data
    end
    deactivate Service
    
    alt Apply is true
        CLI-->>User: Success message
    else Apply is false
        CLI-->>User: Formatted report
    end
```
**Explanation (`sync`):**
1. User runs `zenv sync` (optionally with `--apply`).
2. CLI calls `Service.Sync`.
3. Service logs command start.
4. Service identifies available package managers.
5. Concurrently (or sequentially) calls `GetAllInstalledPackages` on each adapter to get the current system state (`systemState`).
6. Fetches the current ledger state from the database using `Store.GetCurrentPackages` (`dbState`).
7. Compares `systemState` and `dbState` to identify packages added, removed, or updated outside of `zenv`.
8. If `--apply` is **false**:
    * Generates a report detailing the differences.
    * Logs command end (with report details).
    * Returns the report to the CLI for display.
9. If `--apply` is **true**:
    * Starts the main DB transaction.
    * For **Adds**: Creates `CurrentPackage` from system state, calls `Store.ReplaceCurrentPackage`, logs `sync_add` event.
    * For **Removals**: Fetches prior details, calls `Store.DeleteCurrentPackage`, logs `sync_remove` event.
    * For **Updates**: Creates updated `CurrentPackage` from system state (preserving original install time), calls `Store.ReplaceCurrentPackage`, logs `sync_update` event. (All operations update `last_updated_timestamp`).
    * Commits the main transaction.
    * Logs command end (potentially with a summary of changes).
    * Returns success to the CLI.

## 4. Requirements Sections Visualization

Visualizing key aspects mentioned in the requirements document.

### 4.1 Core Components (Simplified Relationship)

```mermaid
graph LR
    CLI(CLI Interface) --> Service(Service Layer)
    Service --> Ledger(Ledger Management)
    Service --> PkgMgr(Package Manager Abstraction)
    Ledger --> DB[(SQLite DB)]
    PkgMgr --> Adapters(Adapters: DNF, APT, ...)
    Adapters --> SystemPM(System Package Managers)
    Service --> Logging(App Logging)
    Service --> Utils(Utilities)
    CLI --> Formatting(Output Formatting)
    Ledger --> Formatting

    style CLI fill:#ccf,color:#000
    style Service fill:#9cf,color:#000
    style Ledger fill:#fec,color:#000
    style PkgMgr fill:#fec,color:#000
    style Adapters fill:#fec,color:#000
    style Formatting fill:#fec,color:#000
    style DB fill:#f9d,color:#000
    style SystemPM fill:#bbf,color:#000
    style Logging fill:#eee,color:#000
    style Utils fill:#eee,color:#000
```
**Explanation:** This mirrors the main architecture diagram but focuses solely on the internal components and their primary relationships as described in Section 3 of the ADD.

### 4.2 Goals Overview (Mind Map)

```mermaid
mindmap
  root((zenv Goals))
    (Audit Trail)
      ::icon(fa fa-history)
      Log Commands
      Log Package Events
      Link Events to Commands
    (Rich Metadata)
      ::icon(fa fa-database)
      Core Metadata
      Extra Metadata
      User Tags
      Installation Info
    (Unified Interface)
      ::icon(fa fa-layer-group)
      Wrap Multiple PMs
      Manual Entry Support
    (State Reconciliation)
      ::icon(fa fa-sync)
      discover Command
      sync Command
      Health Checks
    (Extensibility)
      ::icon(fa fa-plug)
      PM Interface
      Capabilities Method
    (User Control)
      ::icon(fa fa-search)
      Manual Management
      Ledger Querying
      Reporting
    (Robustness)
      ::icon(fa fa-shield-alt)
      Post-Operation Checks
      DB Transactions
      Error Handling
      Migrations
      Command Logging
    (Maintainability)
      ::icon(fa fa-wrench)
      Service Layer
      Component Boundaries
    (User Experience)
      ::icon(fa fa-smile)
      Intuitive CLI
      Help & Completion
      PM Output Streaming
      Confirmations
```
**Explanation:** This mind map categorizes and lists the primary goals defined in Section 2 of the ADD (`requirements.md`), providing a quick visual summary of the project's objectives.

This concludes the visual overview. These diagrams should help in understanding the structure, data flow, and intended functionality of the `zenv` tool.
