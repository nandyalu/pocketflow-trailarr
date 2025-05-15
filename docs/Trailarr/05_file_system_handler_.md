# Chapter 5: File System Handler

Welcome back to the Trailarr2 tutorial!

So far, we've learned how Trailarr2 structures its internal data using [Database Models](01_database_models_.md), manages its behavior with [Application Settings](02_application_settings_.md), how the whole application gets started ([Main Application Entry](03_main_application_entry_.md)), and how it talks to the database to save and retrieve information using [Database Managers](04_database_managers_.md).

But Trailarr2 isn't just about data *in* a database; it's about managing media files and their trailers *on your computer or server*. This means Trailarr2 needs a way to interact with the actual files and folders on your storage drives. It needs to know if a specific movie folder exists, look inside it to find the movie file, check if a trailer is already there, save a new trailer file, or maybe even delete an old one.

Directly working with files and folders can be tricky. Different operating systems (Windows, Linux, macOS) handle paths differently, permissions can be an issue, and performing operations like deleting requires caution to avoid accidental data loss.

This is where the **File System Handler** comes in.

## What is the File System Handler? The File Cabinet Assistant

Think of your computer's storage as a giant filing cabinet system. Your media files and folders are organized within it. The **File System Handler** is like a dedicated, cautious **assistant** who knows exactly how to navigate this filing cabinet for Trailarr2.

Its job is to provide a set of **safe and reliable tools** for Trailarr2 to perform file system operations without needing to worry about the low-level details or potential dangers.

The File System Handler:
*   Knows how to find specific folders or files.
*   Can tell you if a file or folder exists.
*   Helps check if a media file or a trailer is present in a folder based on specific rules.
*   Can safely handle renaming or deleting files and folders.
*   Provides structured information about files and folders (like their size or creation date).

Essentially, any time Trailarr2 needs to touch a file or folder on your system, it asks the File System Handler assistant to do it.

## Our Guiding Use Case: Checking for Media and Trailers

A core task for Trailarr2 is figuring out which of your media items (movies/series) need trailers. To do this, it needs to look at the folder where the media is stored and check two things:

1.  Does the actual movie/series file exist? (e.g., `Inception (2010).mkv`)
2.  Does a trailer for it already exist? (e.g., in a `Trailers` subfolder or named `Inception (2010)-trailer.mp4`)

This is a perfect job for the File System Handler.

## Using the File System Handler (The Simple Way)

The File System Handler in Trailarr2 is implemented as a Python class called `FilesHandler` (found in `backend/core/files_handler.py`). Many of its useful methods are designed to be called directly using the class name (these are called "static methods").

Here's how Trailarr2 code would typically use the `FilesHandler` to perform the checks for our use case:

```python
# Imagine this code checks a specific media folder

from core.files_handler import FilesHandler

media_folder_path = "/path/to/your/movie/Inception (2010)"

# Ask the FilesHandler: Does a media file exist in this folder?
media_exists = FilesHandler.check_media_exists(media_folder_path)

# Ask the FilesHandler: Does a trailer exist in this folder?
# (The default checks for a 'trailers' subfolder or inline file)
trailer_exists = await FilesHandler.check_trailer_exists(media_folder_path)

# Now Trailarr2 knows the status for this folder!
print(f"Media file found: {media_exists}")
print(f"Trailer file found: {trailer_exists}")
```

Let's break down this simple example:

1.  **`from core.files_handler import FilesHandler`**: This line imports our helpful File System Handler class.
2.  **`FilesHandler.check_media_exists(media_folder_path)`**: We call the `check_media_exists` method directly on the `FilesHandler` class. We give it the path to the folder. It returns `True` or `False` based on its internal logic (which checks for file size and extensions, as seen in the code snippet).
3.  **`await FilesHandler.check_trailer_exists(media_folder_path)`**: Similarly, we call the `check_trailer_exists` method. Notice the `await` keyword – some file operations can take a little time (especially on slow network drives), so the `FilesHandler` uses asynchronous operations (`aiofiles` library) to prevent the application from freezing while waiting. This method returns `True` or `False` based on whether it finds a file matching Trailarr2's trailer naming conventions.

Using the `FilesHandler` is straightforward: import it and call the method that describes the operation you need, passing the relevant file or folder path.

## Organizing File Information (`FolderInfo` and `FileInfo`)

When you ask the `FilesHandler` to list the contents of a folder (using the `get_folder_files` method), it doesn't just give you a raw list of names. It provides structured information about each item.

It uses simple blueprints, similar in concept to the [Database Models](01_database_models_.md) but specifically for describing file system entries:

```python
# Simplified FileInfo model blueprint
from pydantic import BaseModel, Field

class FileInfo(BaseModel):
    type: str = Field(default="file") # Always "file"
    name: str            # The name of the file
    size: str            # File size (e.g., "100 MB")
    created: str         # Creation date/time

# Simplified FolderInfo model blueprint
class FolderInfo(BaseModel):
    type: str = Field(default="folder") # Always "folder"
    name: str            # The name of the folder
    path: str            # The full path of the folder
    size: str = Field(default="0 KB") # Total size of contents
    files: list["FolderInfo"] = Field(default=[]) # List of items inside (can be files or folders)
    created: str         # Creation date/time
```

When you call `await FilesHandler.get_folder_files(some_path)`, you get back a `FolderInfo` object for `some_path`, and its `files` list contains `FileInfo` objects for files and nested `FolderInfo` objects for subfolders. This structured output makes it easy for the rest of the application (like the frontend that displays file lists) to work with the information.

## Other Helpful Assistant Tasks

Besides checking for media/trailers and listing contents, the `FilesHandler` offers other key methods:

*   `check_folder_exists(path)`: A simple check if a directory exists.
*   `delete_file(file_path)`: Safely deletes a specific file.
*   `delete_folder(folder_path)`: Safely deletes a folder and everything inside it.
*   `delete_trailer(folder_path)`: Specifically targets and deletes a trailer file or folder within a given media folder.
*   `rename_file_fol(old_path, new_path)`: Renames a file or folder.
*   `compute_file_hash(file_path)`: Calculates a unique identifier (hash) for a file's content (used, for example, to detect if a file has changed).
*   `cleanup_tmp_dir()`: Cleans up temporary files left over from operations like downloading.

Notice the names often include `file` or `folder` or `file_fol` (for file or folder) to indicate what they operate on.

A very important aspect is **safety**. The `FilesHandler` includes checks (like the `_is_path_safe` function seen in the `backend/api/v1/files.py` snippet) to prevent operations on critical system directories (like `/`, `/bin`, `/etc`). This adds a layer of protection against accidental deletions or modifications in sensitive areas, making Trailarr2 safer, especially when running in environments like Docker.

## How the File System Handler Works Under the Hood (The Assistant's Process)

When you ask the `FilesHandler` to do something, like check if a trailer exists, it performs a series of steps:

1.  It receives the path to the media folder you're interested in.
2.  It uses standard Python functions (often enhanced with asynchronous capabilities via libraries like `aiofiles.os`) to interact with the underlying operating system's file system.
3.  It might check if the path is a valid directory, list the contents of the directory, check if each item is a file or a directory, read the name and properties (like size or creation date) of each item.
4.  Based on the method you called, it applies specific logic (e.g., checking if a file name contains "trailer", or if a directory name is "trailers").
5.  It processes the results from the operating system's file system and returns a simple `True`/`False` or the structured information (`FileInfo`/`FolderInfo`).

Here's a simplified sequence for checking if a trailer exists:

```mermaid
sequenceDiagram
    participant AppCode as Trailarr2 Code
    participant FilesHandler as FilesHandler Utility
    participant OS_FS as Operating System Filesystem

    AppCode->>FilesHandler: Call check_trailer_exists("/path/to/movie")
    FilesHandler->>OS_FS: IsDir("/path/to/movie")?
    OS_FS-->>FilesHandler: Yes/No
    FilesHandler->>FilesHandler: If No, return False
    FilesHandler->>OS_FS: ScanDir("/path/to/movie")
    OS_FS-->>FilesHandler: List of entries (names, types)
    loop Each Entry
        FilesHandler->>FilesHandler: Is entry a directory named 'trailers'?
        alt If yes
            FilesHandler->>OS_FS: ScanDir("/path/to/movie/trailers")
            OS_FS-->>FilesHandler: List of entries
            loop Each Sub-entry
                FilesHandler->>FilesHandler: Is sub-entry a video file with "trailer" in name?
                alt If yes
                    FilesHandler-->>AppCode: Return True (Trailer found)
                    break loop
                end
            end
            FilesHandler->>FilesHandler: If loop finished without finding trailer, continue main loop
        end
        FilesHandler->>FilesHandler: Is entry a video file with "trailer" in name? (if check_inline_file is True)
        alt If yes
            FilesHandler-->>AppCode: Return True (Trailer found)
            break loop
        end
    end
    FilesHandler-->>AppCode: Return False (No trailer found)
```

This diagram shows how the `FilesHandler` orchestrates calls to the operating system's file system functions to figure out the answer to the question "Does a trailer exist?".

## Peeking Inside the FilesHandler Code

Let's look at a simplified version of the `check_media_exists` method to see how it uses standard Python/async functions to check for a media file:

```python
# From: backend/core/files_handler.py (Simplified)
import os
# Assume aiofiles.os is imported and used for async methods
# Assume VIDEO_EXTENSIONS and _convert_file_size are defined

class FilesHandler:
    VIDEO_EXTENSIONS = tuple([".avi", "mkv", ".mp4", ".webm"]) # Define valid video types

    @staticmethod
    def check_media_exists(path: str) -> bool:
        """Check if a media file exists in the specified folder."""
        # 1. Check if the provided path is even a directory
        _is_dir = os.path.isdir(path)
        if not _is_dir:
            return False # If it's not a folder, no media can be in it

        # 2. Scan the contents of the folder
        for entry in os.scandir(path):
            # Recursively check subdirectories (for series seasons, etc.)
            if entry.is_dir():
                if FilesHandler.check_media_exists(entry.path):
                    return True # Found media in a subfolder

            # Skip if it's not a file
            if not entry.is_file():
                continue

            # 3. Check file criteria: extension and size
            if not entry.name.endswith(FilesHandler.VIDEO_EXTENSIONS):
                continue # Skip if not a video file

            # Check if the file is large enough to be media (e.g., > 100 MB)
            # This helps skip small extra files like covers or NFOs
            if entry.stat().st_size < 100 * 1024 * 1024:  # 100 MB
                continue

            # If we reached here, we found a file that looks like media!
            return True

        # If we scanned everything and didn't find media, return False
        return False

    # ... other methods ...
```

This snippet shows how the `check_media_exists` method:
1.  Uses `os.path.isdir()` to confirm the path is a folder.
2.  Uses `os.scandir()` to efficiently list entries within the folder without loading all details immediately.
3.  Uses `entry.is_dir()` and `entry.is_file()` to tell if an item is a folder or file.
4.  Calls itself recursively (`FilesHandler.check_media_exists(entry.path)`) to look inside subfolders.
5.  Uses `entry.name.endswith()` to check the file extension against the `VIDEO_EXTENSIONS` list.
6.  Uses `entry.stat().st_size` to get the file size and compares it to a threshold (100 MB) to filter out small non-media files.
7.  If all checks pass for an entry, it immediately returns `True`. If the loop finishes without finding such a file, it returns `False`.

The `check_trailer_exists` and `get_folder_files` methods work similarly, using `os.scandir` or `aiofiles.os.scandir` and applying different logic to identify trailers or gather file/folder details respectively, populating the `FileInfo` and `FolderInfo` structures.

The File System Handler effectively wraps these standard operating system interactions with Trailarr2's specific rules and safety checks, providing a reliable interface for managing files.

## Summary and What's Next

In this chapter, we met the **File System Handler**, Trailarr2's cautious assistant for interacting with your computer's storage. We learned that it provides a safe and standardized way to perform file and folder operations like checking for existence, listing contents, renaming, and deleting. We focused on the use case of checking for media and trailer files and saw how methods like `check_media_exists` and `check_trailer_exists` are used. We also peeked under the hood to see how it uses standard Python functions and applies specific rules and safety checks (like the `_is_path_safe` logic).

Now that we understand how Trailarr2 manages its internal data ([Database Managers](04_database_managers_.md)) and interacts with files on disk (File System Handler), we are ready to explore how it connects to your media server applications like Radarr or Sonarr to discover your media library.

Let's move on to learn about [Arr Connection Managers](06_arr_connection_managers_.md).

[Chapter 6: Arr Connection Managers](06_arr_connection_managers_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)