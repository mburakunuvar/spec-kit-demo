# Data Model: Document Upload and Management

**Feature**: `001-document-upload-management`  
**Generated**: 2026-04-05  
**Depends on**: [research.md](research.md)

---

## New Entities

### Document

Represents a file stored in the system.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `DocumentId` | `int` | PK, auto-increment | Integer PK per constitution convention |
| `Title` | `string` | Required, MaxLength(255) | User-provided display name |
| `Description` | `string?` | MaxLength(2000) | Optional summary |
| `Category` | `string` | Required, MaxLength(50) | One of: `Project Documents`, `Team Resources`, `Personal Files`, `Reports`, `Presentations`, `Other` |
| `Tags` | `string?` | MaxLength(500) | Comma-separated tags; stored as text for training simplicity |
| `OriginalFileName` | `string` | Required, MaxLength(255) | Original filename shown to users (e.g. `report.pdf`) |
| `StoredFileName` | `string` | Required, MaxLength(255) | GUID-based filename on disk (e.g. `3fa85...6.pdf`) |
| `StoragePath` | `string` | Required, MaxLength(1000) | Relative path from configured root (e.g. `2026/04/3fa85...6.pdf`) |
| `FileSizeBytes` | `long` | Required | File size in bytes; recorded at upload time |
| `ContentType` | `string` | Required, MaxLength(100) | MIME type (e.g. `application/pdf`, `image/jpeg`) |
| `UploadedByUserId` | `int` | Required, FK → `User.UserId` | Uploader identity |
| `ProjectId` | `int?` | FK → `Project.ProjectId`, nullable | Optional project association |
| `UploadedDate` | `DateTime` | Required, UTC | Set automatically at upload |
| `UpdatedDate` | `DateTime` | Required, UTC | Updated on metadata edit or file replace |

**Navigation properties**:
- `UploadedByUser: User` (via `UploadedByUserId`)
- `Project: Project?` (via `ProjectId`)
- `Shares: ICollection<DocumentShare>`
- `Activities: ICollection<DocumentActivity>`

**Indexes**:
- `(UploadedByUserId)` — supports "My Documents" query
- `(ProjectId)` — supports project document listing
- `(UploadedDate DESC)` — supports recent documents widget and date-range filter
- `(Category)` — supports category filter

---

### DocumentShare

Represents a sharing relationship between a document and a recipient user.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `DocumentShareId` | `int` | PK, auto-increment | |
| `DocumentId` | `int` | Required, FK → `Document.DocumentId` | Cascades delete on document removal |
| `SharedWithUserId` | `int` | Required, FK → `User.UserId` | Recipient |
| `SharedByUserId` | `int` | Required, FK → `User.UserId` | The user who performed the share action |
| `SharedDate` | `DateTime` | Required, UTC | Set automatically |

**Navigation properties**:
- `Document: Document`
- `SharedWithUser: User`
- `SharedByUser: User`

**Indexes**:
- `(SharedWithUserId)` — supports "Shared with Me" query
- `(DocumentId, SharedWithUserId)` — unique share relationship check

---

### DocumentActivity

An append-only audit log entry for a document action.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `DocumentActivityId` | `int` | PK, auto-increment | |
| `DocumentId` | `int` | Required, FK → `Document.DocumentId` | Non-cascading delete; keep log after doc deleted |
| `ActingUserId` | `int` | Required, FK → `User.UserId` | User who performed the action |
| `ActionType` | `DocumentActionType` | Required | Enum: `Upload`, `Download`, `Delete`, `Share`, `MetadataEdit`, `FileReplace` |
| `Timestamp` | `DateTime` | Required, UTC | Set automatically |
| `Notes` | `string?` | MaxLength(500) | Optional context (e.g., shared-with user count) |

**Navigation properties**:
- `Document: Document`
- `ActingUser: User`

**Indexes**:
- `(DocumentId)` — supports per-document audit trail
- `(Timestamp DESC)` — supports admin audit log

---

## Enum Extensions

### DocumentActionType (new)

```csharp
public enum DocumentActionType
{
    Upload,
    Download,
    Delete,
    Share,
    MetadataEdit,
    FileReplace
}
```

### DocumentCategory (new — used as string constants)

```csharp
public static class DocumentCategory
{
    public const string ProjectDocuments = "Project Documents";
    public const string TeamResources    = "Team Resources";
    public const string PersonalFiles    = "Personal Files";
    public const string Reports          = "Reports";
    public const string Presentations    = "Presentations";
    public const string Other            = "Other";

    public static readonly string[] All = [
        ProjectDocuments, TeamResources, PersonalFiles,
        Reports, Presentations, Other
    ];
}
```

> Categories are stored as text (not int enum) per spec and constitution (text values for categorical data).

### NotificationType extension (existing enum — add two values)

```csharp
// add to existing NotificationType enum in Notification.cs
DocumentShared,          // recipient of a document share (FR-025)
DocumentAddedToProject   // project member when a doc is added to their project (FR-031)
```

---

## DbContext Changes (`ApplicationDbContext`)

New `DbSet` properties:

```csharp
public DbSet<Document>         Documents         { get; set; } = null!;
public DbSet<DocumentShare>    DocumentShares    { get; set; } = null!;
public DbSet<DocumentActivity> DocumentActivities { get; set; } = null!;
```

New `OnModelCreating` configuration:

```csharp
// Document → User (uploader) — restrict delete
modelBuilder.Entity<Document>()
    .HasOne(d => d.UploadedByUser)
    .WithMany()
    .HasForeignKey(d => d.UploadedByUserId)
    .OnDelete(DeleteBehavior.Restrict);

// Document → Project — set null on project delete
modelBuilder.Entity<Document>()
    .HasOne(d => d.Project)
    .WithMany()
    .HasForeignKey(d => d.ProjectId)
    .OnDelete(DeleteBehavior.SetNull);

// DocumentShare → Document — cascade delete shares when doc deleted
modelBuilder.Entity<DocumentShare>()
    .HasOne(ds => ds.Document)
    .WithMany(d => d.Shares)
    .HasForeignKey(ds => ds.DocumentId)
    .OnDelete(DeleteBehavior.Cascade);

// DocumentShare → SharedWithUser — restrict
modelBuilder.Entity<DocumentShare>()
    .HasOne(ds => ds.SharedWithUser)
    .WithMany()
    .HasForeignKey(ds => ds.SharedWithUserId)
    .OnDelete(DeleteBehavior.Restrict);

// DocumentShare → SharedByUser — restrict
modelBuilder.Entity<DocumentShare>()
    .HasOne(ds => ds.SharedByUser)
    .WithMany()
    .HasForeignKey(ds => ds.SharedByUserId)
    .OnDelete(DeleteBehavior.Restrict);

// DocumentActivity → Document — restrict (keep audit after delete)
modelBuilder.Entity<DocumentActivity>()
    .HasOne(da => da.Document)
    .WithMany(d => d.Activities)
    .HasForeignKey(da => da.DocumentId)
    .OnDelete(DeleteBehavior.Restrict);

// DocumentActivity → User — restrict
modelBuilder.Entity<DocumentActivity>()
    .HasOne(da => da.ActingUser)
    .WithMany()
    .HasForeignKey(da => da.ActingUserId)
    .OnDelete(DeleteBehavior.Restrict);

// Indexes
modelBuilder.Entity<Document>()
    .HasIndex(d => d.UploadedByUserId);
modelBuilder.Entity<Document>()
    .HasIndex(d => d.ProjectId);
modelBuilder.Entity<Document>()
    .HasIndex(d => d.UploadedDate);
modelBuilder.Entity<Document>()
    .HasIndex(d => d.Category);

modelBuilder.Entity<DocumentShare>()
    .HasIndex(ds => ds.SharedWithUserId);
modelBuilder.Entity<DocumentShare>()
    .HasIndex(ds => new { ds.DocumentId, ds.SharedWithUserId })
    .IsUnique();

modelBuilder.Entity<DocumentActivity>()
    .HasIndex(da => da.DocumentId);
modelBuilder.Entity<DocumentActivity>()
    .HasIndex(da => da.Timestamp);
```

---

## Validation Rules

| Entity | Rule | Enforcement |
|--------|------|-------------|
| `Document` | File size ≤ 25 MB (26,214,400 bytes) | `IFileStorageService` stream + `DocumentService` |
| `Document` | Extension whitelist: `.pdf`, `.docx`, `.xlsx`, `.pptx`, `.txt`, `.jpg`, `.jpeg`, `.png` | `DocumentService` — validated against lower-cased extension |
| `Document` | Magic-byte check matches declared extension | `IFileScanService` — see R-001 in research.md |
| `Document` | `Category` must be one of `DocumentCategory.All` | `DocumentService` validation before save |
| `DocumentShare` | Cannot share with yourself (`SharedWithUserId != SharedByUserId`) | `DocumentService` |
| `DocumentShare` | Duplicate share is idempotent (no error, no duplicate row) | `DocumentService` — query-before-insert |

---

## State Transitions

### Document lifecycle

```
[Uploaded] ──edit metadata──► [Updated]
           ──replace file───► [Updated]
           ──delete─────────► [Deleted] (permanent; no soft-delete)
           ──share──────────► [Shared] (creates DocumentShare rows)
```

> No soft-delete / recycle bin. Deletion is permanent per FR-023. The audit log (`DocumentActivity`) retains a record of the deletion event.

---

## Entity Relationship Diagram (text)

```
User ──────────────────────── uploads ──────────────► Document
                                                          │
Project ──────────────────── associated with ─────────── ┘
                                                          │
DocumentShare ─── links ──► Document (DocumentId)         │
              └──────────► SharedWithUser (User)           │
              └──────────► SharedByUser (User)             │
                                                          │
DocumentActivity ─ links ─► Document (DocumentId)  ◄──── ┘
                 └─────────► ActingUser (User)
```
