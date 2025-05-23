# Tutorial: trailarr2

Trailarr is an application that helps you manage *trailers* for your media library.
It connects to **Radarr** and **Sonarr** to sync your movie and series lists,
automatically finds and **downloads trailers** from sources like YouTube,
and organizes them alongside your media files. It also provides a **web interface**
to view your media, manage connections, configure download settings, and monitor tasks.


**Source Repository:** [None](None)

```mermaid
flowchart TD
    A0["Main Application Entry
"]
    A1["Application Settings
"]
    A2["Database Models
"]
    A3["Database Managers
"]
    A4["Arr Connection Managers
"]
    A5["Trailer Download Core
"]
    A6["Task Scheduling
"]
    A7["File System Handler
"]
    A8["Frontend API Services (Generated)
"]
    A9["Frontend UI Components
"]
    A0 -- "Loads configuration" --> A1
    A0 -- "Manages scheduler lifecycle" --> A6
    A0 -- "Mounts static directories" --> A7
    A0 -- "Provides API endpoints for" --> A8
    A9 -- "Uses to interact with backend" --> A8
    A8 -- "Call API endpoints hosted by" --> A0
    A6 -- "Triggers data synchronizati..." --> A4
    A6 -- "Schedules and triggers down..." --> A5
    A6 -- "Triggers disk scanning task..." --> A7
    A4 -- "Uses to save fetched media ..." --> A3
    A5 -- "Uses to update media status" --> A3
    A5 -- "Performs file operations using" --> A7
    A3 -- "Interacts with data structu..." --> A2
    A1 -- "Provides configuration to" --> A4
    A1 -- "Provides configuration to" --> A5
```

## Chapters

1. [Database Models
](01_database_models_.md)
2. [Application Settings
](02_application_settings_.md)
3. [Main Application Entry
](03_main_application_entry_.md)
4. [Database Managers
](04_database_managers_.md)
5. [File System Handler
](05_file_system_handler_.md)
6. [Arr Connection Managers
](06_arr_connection_managers_.md)
7. [Trailer Download Core
](07_trailer_download_core_.md)
8. [Task Scheduling
](08_task_scheduling_.md)
9. [Frontend API Services (Generated)
](09_frontend_api_services__generated__.md)
10. [Frontend UI Components
](10_frontend_ui_components_.md)


---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)