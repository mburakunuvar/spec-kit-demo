# Contract: IDocumentService

**Feature**: `001-document-upload-management`  
**Namespace**: `ContosoDashboard.Services`  
**Layer**: Service (business logic; no UI dependencies)

---

## Purpose

`IDocumentService` is the single point of entry for all document management operations. It enforces authorization, validation, and audit logging before delegating to `IFileStorageService` and `IFileScanService`. Pages and controllers must not access `ApplicationDbContext` directly (Principle III).

---

## Methods

### Query

```csharp
/// <summary>Returns all documents uploaded by the specified user, with optional filtering.</summary>
Task<List<Document>> GetMyDocumentsAsync(int userId, DocumentFilter? filter = null);

/// <summary>Returns all documents shared directly with the specified user.</summary>
Task<List<Document>> GetSharedWithMeAsync(int userId);

/// <summary>
/// Returns all documents associated with a project.
/// Returns empty list if the requesting user is not a project member (IDOR prevention).
/// </summary>
Task<List<Document>> GetProjectDocumentsAsync(int projectId, int requestingUserId);

/// <summary>
/// Returns all documents associated with a task's project that the requesting user can access.
/// </summary>
Task<List<Document>> GetTaskDocumentsAsync(int taskId, int requestingUserId);

/// <summary>
/// Full-text search across Title, Description, Tags, OriginalFileName, uploader DisplayName,
/// and associated Project Name. Returns only documents the requesting user has access to.
/// </summary>
Task<List<Document>> SearchDocumentsAsync(string query, int requestingUserId);

/// <summary>
/// Loads a single document by ID. Returns null if not found or requesting user lacks access.
/// </summary>
Task<Document?> GetDocumentByIdAsync(int documentId, int requestingUserId);

/// <summary>Returns the N most recently uploaded documents for the user. Default N = 5.</summary>
Task<List<Document>> GetRecentDocumentsAsync(int userId, int count = 5);

/// <summary>Returns the total number of documents uploaded by the user.</summary>
Task<int> GetDocumentCountAsync(int userId);

/// <summary>
/// Returns the full audit activity log. Throws UnauthorizedAccessException if
/// requestingUserId is not an Administrator.
/// </summary>
Task<List<DocumentActivity>> GetActivityLogAsync(int requestingUserId);
```

### Commands

```csharp
/// <summary>
/// Uploads a new document. Validates size, extension, and magic bytes before storing.
/// Returns the saved Document or throws on validation failure.
/// </summary>
/// <exception cref="DocumentValidationException">File type, size, or scan failure.</exception>
Task<Document> UploadDocumentAsync(
    DocumentUploadRequest request,
    Stream fileStream,
    int requestingUserId);

/// <summary>
/// Updates editable metadata (Title, Description, Category, Tags).
/// Authorized for: document owner, TeamLead of the uploader, ProjectManager of associated
/// project, Administrator.
/// </summary>
/// <exception cref="UnauthorizedAccessException">Caller lacks permission.</exception>
Task<Document> UpdateMetadataAsync(
    int documentId,
    DocumentMetadataUpdate update,
    int requestingUserId);

/// <summary>
/// Replaces the stored file with a new version while preserving metadata.
/// Authorized for: document owner, TeamLead of the uploader, Administrator.
/// </summary>
/// <exception cref="UnauthorizedAccessException">Caller lacks permission.</exception>
/// <exception cref="DocumentValidationException">New file fails validation.</exception>
Task<Document> ReplaceFileAsync(
    int documentId,
    Stream newFileStream,
    string originalFileName,
    int requestingUserId);

/// <summary>
/// Permanently deletes a document and its file from storage.
/// Authorized for: document owner, ProjectManager of associated project, Administrator.
/// (TeamLeads cannot delete documents they did not upload — FR-019a.)
/// </summary>
/// <exception cref="UnauthorizedAccessException">Caller lacks permission.</exception>
Task DeleteDocumentAsync(int documentId, int requestingUserId);

/// <summary>
/// Shares a document with one or more users. Creates DocumentShare rows and
/// sends in-app notifications to each recipient.
/// Duplicate shares are silently skipped.
/// Authorized for: document owner only.
/// </summary>
/// <exception cref="UnauthorizedAccessException">Caller is not the document owner.</exception>
Task ShareDocumentAsync(int documentId, List<int> recipientUserIds, int requestingUserId);
```

---

## Supporting Types

```csharp
public record DocumentUploadRequest(
    string Title,
    string? Description,
    string Category,
    string? Tags,
    string OriginalFileName,
    long FileSizeBytes,
    string ContentType,
    int? ProjectId
);

public record DocumentMetadataUpdate(
    string Title,
    string? Description,
    string Category,
    string? Tags
);

public record DocumentFilter(
    string? Category = null,
    int? ProjectId = null,
    DateTime? UploadedAfter = null,
    DateTime? UploadedBefore = null,
    string? SortBy = "UploadedDate",   // Title | UploadedDate | Category | FileSizeBytes
    bool Descending = true
);

public class DocumentValidationException(string message) : Exception(message);
```

---

## Authorization Matrix

| Operation | Owner | TeamLead (of uploader) | PM (of project) | Admin |
|-----------|:-----:|:---------------------:|:---------------:|:-----:|
| View / Download | ✅ | ✅ | ✅ | ✅ |
| Edit metadata | ✅ | ✅ | ✅ | ✅ |
| Replace file | ✅ | ✅ | ✅ | ✅ |
| Delete | ✅ | ❌ | ✅ | ✅ |
| Share | ✅ | ❌ | ❌ | ✅ |
| Audit log | ❌ | ❌ | ❌ | ✅ |

> TeamLead access is determined by shared project membership (see R-005 in research.md).
