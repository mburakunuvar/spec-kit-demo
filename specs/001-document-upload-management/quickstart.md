# Quickstart: Document Upload and Management

**Feature**: `001-document-upload-management`  
**Generated**: 2026-04-05

This guide explains how to get the feature running in the local dev environment and how each major capability works.

---

## Prerequisites

- .NET 10 SDK
- VS Code with C# Dev Kit (or Rider)
- The dev container already satisfies all dependencies

---

## 1. Add Storage Configuration

Add the following to `ContosoDashboard/appsettings.Development.json`:

```json
{
  "DocumentStoragePath": "./document-uploads"
}
```

The folder will be created automatically on first upload. It is outside `wwwroot`, so files are never served statically.

---

## 2. Apply the EF Core Migration

After the three new models (`Document`, `DocumentShare`, `DocumentActivity`) are added and `ApplicationDbContext` is updated:

```bash
cd ContosoDashboard
dotnet ef migrations add AddDocumentManagement
dotnet ef database update
```

> The SQLite database file is in the project directory (see `appsettings.json` connection string). The migration creates three new tables and two new indexes on the existing `Notifications` table (new enum values require no schema change — stored as int ordinal).

---

## 3. Run the Application

```bash
cd ContosoDashboard
dotnet run
```

Navigate to `https://localhost:{port}` (see terminal output). Log in as any seeded user.

---

## 4. Feature Walkthrough

### Upload a Document
1. Click **Documents** in the left nav → **My Documents** tab
2. Click **Upload** → select a PDF, DOCX, XLSX, PPTX, TXT, JPG, or PNG (≤ 25 MB)
3. Fill in **Title** and **Category** (required); optionally add description, tags, project link
4. Watch the progress bar; receive success/error toast on completion

### Browse and Filter
- Use the sort dropdown (Title / Upload Date / Category / File Size) and filter panel (Category, Project, Date Range) on the **My Documents** tab

### Download or Preview
- **Download**: click the download icon — the browser downloads the file via `/documents/download/{id}`
- **Preview** (PDF/images): click the preview icon — opens inline at `/documents/preview/{id}`; the file is displayed natively by the browser

### Search
- Use the search bar in the Documents page header; searches Title, Description, Tags, OriginalFileName, uploader name, and project name

### Edit Metadata / Replace File
- Click a document row → **Edit** button; change title, description, category, or tags; or upload a replacement file

### Delete
- Click a document row → **Delete** (trash icon) → confirm in the modal prompt; deletion is permanent

### Share
- Click a document row → **Share** button; search for colleagues by name; select and confirm; recipients receive an in-app notification and see the document under their **Shared with Me** tab

### Project Documents Tab
- Open any project in **Projects** → **Documents** tab; shows all documents associated with that project; Project Managers can upload directly from here

### Task Document Attachments
- Open a task in **Tasks** → scroll to **Attachments** section; upload a document (auto-linked to the task's project)

### Dashboard Widget
- The home page shows the **Recent Documents** widget (last 5 uploads) and total document count badge

---

## 5. Admin Audit Log

Log in as `admin@contoso.com` (password: any — mock auth). Navigate to **Documents** → **Audit Log** tab to view all system-wide document actions.

---

## 6. Architecture Notes (for students)

| Layer | Location | Responsibility |
|-------|----------|----------------|
| Models | `ContosoDashboard/Models/` | `Document`, `DocumentShare`, `DocumentActivity` — data annotations only |
| Services | `ContosoDashboard/Services/` | `IDocumentService` / `DocumentService`, `IFileStorageService` / `LocalFileStorageService`, `IFileScanService` / `LocalFileScanService` |
| Data | `ContosoDashboard/Data/ApplicationDbContext.cs` | EF Core relationships, indexes, DbSets |
| Pages | `ContosoDashboard/Pages/` | `Documents.razor`, `DocumentDetails.razor`, extended `ProjectDetails.razor`, `Tasks.razor` |
| Controller | `ContosoDashboard/Controllers/DocumentController.cs` | Auth-gated `/documents/download` and `/documents/preview` endpoints |
| Config | `appsettings.json` | `DocumentStoragePath` — swap storage root without code changes |

**Production differences** (documented in code comments):
- `LocalFileScanService` magic-byte check → replace with ClamAV or Azure Defender for Storage
- `LocalFileStorageService` → replace with `AzureBlobFileStorageService`
- `Contains` search queries → replace with SQLite FTS5 virtual tables or SQL Server full-text indexing
