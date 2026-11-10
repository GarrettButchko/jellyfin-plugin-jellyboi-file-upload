# Jellyfin.Plugin.JellyBoiUploadFile

A high-performance Jellyfin server plugin providing direct streaming media ingestion, resumable chunked uploads, and automated library organization for [JellyBoi](https://github.com/GarrettButchko/jellyboi) and [JellyBoi Media Pipeline](https://github.com/GarrettButchko/JellyBoiMediaPipeline).

The plugin exposes dedicated REST endpoints enabling client applications to inspect server libraries, check upload offsets, stream multi-gigabyte MKV files directly to target disk paths without intermediate server staging, and automatically trigger library scans upon completion.

---

## Core Capabilities

- **Zero-Staging Direct Ingestion:** Streams raw video payloads straight into resolved Jellyfin media library folders, bypassing default temporary upload caching to avoid unnecessary disk I/O and server storage bottlenecks.
- **Resumable Chunked Transfers:** Tracks partially written files (`.upload.part`) and supports arbitrary byte-offset uploads, allowing client interruptions, pauses, and network reconnects to resume without re-uploading completed data.
- **Automated Directory & File Organization:** Resolves media destinations dynamically according to Jellyfin and Kodi naming standards:
  - **Movies:** Automatically handles `{Title} ({Year}) [imdbid-{Id}]/{Title} ({Year}) [imdbid-{Id}].mkv` structures.
  - **TV Shows:** Automatically builds `{Series} ({Year})/Season {XX}/{Series} S{XX}E{YY} {EpisodeTitle}.mkv` hierarchies.
- **Concurrent Transfer Coordination:** Employs an in-memory `TempFileTracker` locking mechanism that coordinates concurrent client requests, gracefully revokes prior file handles on client reconnects, and prevents stream write collisions.
- **Immediate Library Trigger:** Immediately queues targeted Jellyfin library scans via `ILibraryManager` as soon as file writes finalize and `.upload.part` files are atomically moved to destination MKVs.
- **Automatic Stale Temp Cleanup:** Built-in background service (`StaleTempCleanupService`) sweeps library roots during server startup to safely purge stale, abandoned partial uploads older than 7 days while safeguarding active in-flight transfers.

---

## Architecture Overview

```
+-------------------------------------------------------------+
|    Client Applications (JellyBoi / JellyBoi Media Pipeline)  |
+------------------------------+------------------------------+
                               |
                               | 1. GET /JellyBoiUploadFile/Status (Handshake)
                               | 2. GET /JellyBoiUploadFile/Libraries (Target Picker)
                               | 3. GET /JellyBoiUploadFile/Upload/Progress?title=... (Resume Check)
                               v
+-------------------------------------------------------------+
|              Jellyfin.Plugin.JellyBoiUploadFile              |
|                                                             |
|   +-----------------------+     +-----------------------+   |
|   |   UploadController    |     |    TempFileTracker    |   |
|   |  - Route Dispatcher   | <-> |  - Concurrency Locks  |   |
|   |  - Filename Sanitizer |     |  - Stream Revocation  |   |
|   +-----------+-----------+     +-----------------------+   |
+---------------|---------------------------------------------+
                |
                | 4. POST /JellyBoiUploadFile/Upload/Movie (Chunked Stream)
                |    or POST /JellyBoiUploadFile/Upload/Episode
                v
+-------------------------------------------------------------+
|                 Jellyfin Media Library Disk                 |
|                                                             |
|   [Target Folder]                                           |
|       |                                                     |
|       |-- In-Flight: Movie (2024).mkv.upload.part           |
|       |   (Atomic rename upon stream completion)            |
|       +-> Final:     Movie (2024).mkv                       |
+------------------------------+------------------------------+
                               |
                               | 5. Trigger Library Refresh
                               v
+-------------------------------------------------------------+
|               Jellyfin Server (ILibraryManager)             |
+-------------------------------------------------------------+
```

---

## Installation via Plugin Repository (Recommended)

1. In your Jellyfin dashboard, navigate to **Dashboard → Plugins → Repositories**.
2. Click **+** to add a new repository:
   - **Repository Name:** `JellyBoi Upload File`
   - **Repository URL:** `https://garrettbutchko.github.io/jellyfin_plugin_jellyboiFileUpload/manifest.json`
3. Save and navigate to the **Catalog** tab.
4. Locate **JellyBoi Upload File** and click **Install**.
5. Restart your Jellyfin server when prompted.

Jellyfin will automatically monitor the manifest and notify you when plugin updates become available.

---

## API Reference

The plugin registers under multiple path aliases for backwards compatibility (`/JellyBoiUploadFile`, `/JellyBoi.Upload.File`, `/JellyboiFileUpload`, `/FileToRemux`). All endpoints require Jellyfin user authentication unless noted.

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/JellyBoiUploadFile/Status` | Status handshake returning plugin version, direct upload status, and server capabilities. *(No auth required)* |
| `GET` | `/JellyBoiUploadFile/Libraries` | Enumerates available Movie and TV virtual folders and storage paths configured on the server. |
| `GET` | `/JellyBoiUploadFile/Upload/Progress` | Checks existing file size or `.upload.part` byte offset for a given movie title or TV episode. |
| `POST` | `/JellyBoiUploadFile/Upload/Movie` | Streams video binary into the resolved movie library path with query params (`title`, `year`, `imdbId`, `offset`, `totalBytes`, `libraryId`, `serverPath`). |
| `POST` | `/JellyBoiUploadFile/Upload/Episode` | Streams video binary into the resolved TV season path with query params (`seriesTitle`, `seriesYear`, `seriesImdbId`, `season`, `episode`, `episodeTitle`, `offset`, `totalBytes`, `libraryId`, `serverPath`). |

---

## Compatibility

| Plugin Version | Jellyfin Version | .NET Runtime |
| :--- | :--- | :--- |
| **1.0.x** | Jellyfin 10.9.x | .NET 8.0 |

---

## Development & Building

### Prerequisites

- [.NET 8.0 SDK](https://dotnet.microsoft.com/download)

### Build Locally

Compile the plugin assembly:

```bash
dotnet build -c Release
```

The resulting library binary (`Jellyfin.Plugin.JellyBoiUploadFile.dll`) will be output to `bin/Release/net8.0/`.

### Releasing a New Version

1. Bump the `<Version>` field in [Jellyfin.Plugin.JellyBoiUploadFile.csproj](file:///Users/garrettbutchko/Documents/Projects/Apps/Swift/All%20Apple/Jellyboi%20All/Working/Jellyfin.Plugin.JellyBoiUploadFile/Jellyfin.Plugin.JellyBoiUploadFile.csproj).
2. Commit and push the changes:
   ```bash
   git commit -am "Bump version to 1.0.10"
   git push origin main
   ```
3. Tag the release and push the tag:
   ```bash
   git tag v1.0.10
   git push origin v1.0.10
   ```

GitHub Actions will automatically:
- Build and package the plugin zip archive.
- Create a corresponding GitHub Release with release artifacts attached.
- Calculate package checksums and update `manifest.json` on the GitHub Pages branch.

---

## The JellyBoi Ecosystem

JellyBoi Upload File is part of a companion suite designed for automated media preparation and streaming:

- **[JellyBoi](https://github.com/GarrettButchko/jellyboi):** Native client application for iOS and macOS providing hardware-accelerated playback from Jellyfin servers.
- **[JellyBoi Media Pipeline](https://github.com/GarrettButchko/JellyBoiMediaPipeline):** Automated optical disc ripping, crop detection, metadata resolution, subtitle integration, and direct upload orchestrator.
- **[JellyBoi Upload File Plugin](https://github.com/GarrettButchko/jellyfin_plugin_jellyboiFileUpload):** Jellyfin server ingestion plugin for direct, resumable streaming uploads.

---

## License

This project is licensed under the [MIT License](LICENSE).
