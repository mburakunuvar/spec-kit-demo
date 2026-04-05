# Research: Document Upload and Management

**Feature**: `001-document-upload-management`  
**Generated**: 2026-04-05  
**Purpose**: Resolve all unknowns from Technical Context before Phase 1 design

---

## R-001: Offline-Compatible Malware Scanning (FR-005)

**Question**: The spec requires malware scanning (FR-005), but the constitution mandates offline-first. How can scanning work without cloud connectivity?

**Decision**: Implement a local `IFileScanService` with `LocalFileScanService` that validates uploaded file content against expected magic-byte signatures for each whitelisted type.

**Rationale**: Magic-byte validation ensures the file's actual binary format matches its declared extension, preventing MIME-type spoofing (e.g., a renamed `.exe` submitted as `.pdf`). This is an effective first layer of defence that works fully offline and aligns with the constitution's extension-whitelist requirement. The stub documents a production note explaining that a real deployment would integrate ClamAV (offline) or Azure Defender for Storage (cloud) behind the same interface.

**Alternatives considered**:
- Cloud-based AV scanning (e.g., Azure Defender for Storage) — rejected because it violates Principle I (Offline-First); requires network access
- Full ClamAV integration — rejected for training context because it requires an external daemon process; architecture is demonstrated through the interface; LocalFileScanService is the training-suitable placeholder

**Implementation notes**:
- PDF: first 4 bytes `25 50 44 46` (`%PDF`)
- JPEG: first 3 bytes `FF D8 FF`
- PNG: first 8 bytes `89 50 4E 47 0D 0A 1A 0A`
- Office Open XML (DOCX/XLSX/PPTX): first 4 bytes `50 4B 03 04` (ZIP container)
- Plain text: no fixed magic bytes; validate by checking extension only (`.txt`) since content is ASCII
- `FileScanResult` carries `bool IsClean` and `string Reason`

---

## R-002: EF Core Full-Text Search on SQLite (FR-014)

**Question**: SQLite has FTS5 support, but EF Core has no built-in FTS5 provider in the SQLite package. What search approach is appropriate for training scale?

**Decision**: Use EF Core `Contains`-based LINQ queries on indexed columns (`Title`, `Description`, `Tags`, `StoredFileName`), with a join to `User` for uploader-name search and a join to `Project` for project-name search.

**Rationale**: At training scale (up to 500 documents per user, spec SC-006), `LIKE`-based string matching on indexed columns performs well within the 2-second target (SC-007). It avoids introducing raw SQL, external packages, or manual FTS5 virtual-table migrations, preserving training clarity (Principle V). A production comment documents that SQLite FTS5 virtual tables or migrating to SQL Server full-text indexing would be used at larger scale.

**Alternatives considered**:
- Raw SQL with FTS5 virtual table — rejected because EF Core migrations cannot scaffold FTS5 tables automatically; requires raw SQL scripts that obscure the EF Core learning path
- Adding `Microsoft.EntityFrameworkCore.SqlServer` FTS extension — rejected because the project uses SQLite and switching would break the offline-simple setup

---

## R-003: Blazor Server File Upload Pattern (FR-001, FR-008)

**Question**: How should file uploads be handled in Blazor Server to show progress and enforce size limits?

**Decision**: Use the `InputFile` component (`IBrowserFile`) with `OpenReadStream(maxAllowedSize: 26_214_400)` (25 MB + 1 byte to safely detect over-limit files). Stream the content through `IFileStorageService.SaveFileAsync`, which writes to the local filesystem outside `wwwroot`. Progress is reported by wrapping the stream in a byte-counting adapter that updates a Blazor component state variable.

**Rationale**: The `InputFile` component is the standard Blazor Server file upload primitive. Streaming avoids loading the entire file into memory. Size enforcement at the stream level is enforced by the `maxAllowedSize` parameter before the stream is passed to storage. This avoids loading a 25 MB file into memory just to check its size.

**Alternatives considered**:
- Multipart HTTP POST to a controller endpoint — rejected because mixing Blazor real-time UI with a standard HTTP form upload requires additional wiring; `InputFile` with streaming is the idiomatic Blazor approach
- Client-side size check via JavaScript interop — rejected as a sole guard; kept as a UX pre-flight check in addition to server-side validation (defense-in-depth)

---

## R-004: Secure File Download and In-Browser Preview (FR-016, FR-017, FR-018)

**Question**: Files stored outside `wwwroot` cannot be served by the static files middleware. How are downloads and previews served securely?

**Decision**: Add a `DocumentController : Controller` with two endpoints:
- `GET /documents/download/{documentId}` — returns `PhysicalFile(path, contentType, fileDownloadName)` with `[Authorize]` and service-layer IDOR check
- `GET /documents/preview/{documentId}` — same but returns inline (`Content-Disposition: inline`) for PDF and images; the browser renders them natively

The controller is registered by adding `builder.Services.AddControllersWithViews()` and `app.MapControllers()` to `Program.cs`.

**Rationale**: An MVC controller endpoint is the standard pattern for serving authorization-gated files in ASP.NET Core. Cookie authentication is shared between Blazor and MVC controllers in the same app, so the existing session is honoured automatically. Serving with `Content-Disposition: inline` enables native browser rendering of PDFs and images without client-side libraries, satisfying the 3-second preview target (SC-008) and training clarity (no added JS libraries needed).

**Alternatives considered**:
- Blazor `NavigationManager` download link — requires the file to be in `wwwroot` or base64-encoded, which violates the storage constraint and would load the entire file into memory
- pdf.js or similar JS library — rejected as unnecessary for the training use case; native browser PDF rendering is sufficient and simpler

---

## R-005: TeamLead Team Membership Resolution (FR-019a)

**Question**: FR-019a grants TeamLeads edit/replace rights on documents uploaded by "their team members." How is "team" defined given the current data model?

**Decision**: A TeamLead's team = all `UserRole.Employee` users who share at least one project with the TeamLead (they appear together in `ProjectMember` rows for the same `ProjectId`). This is computed in `DocumentService` using a join on `ProjectMember`.

**Rationale**: The `ProjectMember` table is the only existing team-scoping construct in the data model. Using shared project membership is consistent with how team collaboration is already modelled and requires no new schema changes. A training comment notes that a production system might have explicit team/department relationships.

**Edge case**: A TeamLead who shares no project with the document uploader cannot edit that document (correct behaviour — they are not part of the same team in any active project context).

**Alternatives considered**:
- Using `User.Department` for team scoping — rejected because department is free-text and not enforced as a relationship; two users in "Engineering" may be on entirely different teams
- Adding a new `TeamMembership` join table — rejected as out of scope for this feature; would require a separate spec

---

## R-006: NotificationType Enum Extension (FR-025, FR-031)

**Question**: The existing `NotificationType` enum does not include document-specific notification types. How should document notifications be sent?

**Decision**: Extend `NotificationType` with two new values:
- `DocumentShared` — used when a document is shared with a user (FR-025)
- `DocumentAddedToProject` — used when a document is added to a project the user belongs to (FR-031)

**Rationale**: Adding enum values to `NotificationType` is a non-breaking change and keeps notification handling in the existing `NotificationService` pattern. No schema migration is needed because the column is stored as integer ordinal in SQLite. Both new types will follow the existing `NotificationService.CreateNotificationAsync` pattern.

---

## R-007: File Storage Path Layout

**Question**: Where on the local filesystem should uploaded files be stored, and how should the path be configured?

**Decision**: Store files under a configurable `DocumentStoragePath` key in `appsettings.json`, defaulting to `./document-uploads` (relative to the application root, outside `wwwroot`). Sub-directory: `{year}/{month}/` to avoid flat-directory performance issues at scale. Stored filename: `{guid}{original-extension}`.

Example: `document-uploads/2026/04/3fa85f64-5717-4562-b3fc-2c963f66afa6.pdf`

**Rationale**: A configurable path satisfies Principle I (all environment-specific values in `appsettings.json`) and allows the path to be overridden in dev containers / CI without code changes. Sub-directory sharding prevents filesystem slowdown with large numbers of files. GUID filenames prevent collisions and path-traversal attacks.

**Alternatives considered**:
- Hardcoded path constant — rejected; violates Principle I
- Database BLOB storage — rejected; would bloat the SQLite file and make backup/migration of files harder; IFileStorageService interface still supports a future cloud-blob implementation
