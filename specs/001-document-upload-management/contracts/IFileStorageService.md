# Contract: IFileStorageService

**Feature**: `001-document-upload-management`  
**Namespace**: `ContosoDashboard.Services`  
**Layer**: Infrastructure (file I/O abstraction)

---

## Purpose

`IFileStorageService` abstracts all file system operations so that the storage backend can be swapped from local filesystem to cloud blob storage (e.g., Azure Blob Storage) by changing only DI registration in `Program.cs` — no business logic changes (Principle IV).

---

## Interface

```csharp
public interface IFileStorageService
{
    /// <summary>
    /// Writes a file stream to storage using the provided storedFileName.
    /// Returns the relative path that should be persisted in Document.StoragePath.
    /// </summary>
    Task<string> SaveFileAsync(string storedFileName, Stream fileStream);

    /// <summary>
    /// Opens a readable stream for an existing file by its StoragePath.
    /// Throws FileNotFoundException if the file does not exist.
    /// </summary>
    Task<Stream> GetFileStreamAsync(string storagePath);

    /// <summary>
    /// Deletes a file by its StoragePath.
    /// No-ops silently if the file does not exist (idempotent cleanup on delete).
    /// </summary>
    Task DeleteFileAsync(string storagePath);

    /// <summary>Returns true if a file exists at the given StoragePath.</summary>
    Task<bool> FileExistsAsync(string storagePath);

    /// <summary>
    /// Returns the absolute filesystem path for a given StoragePath.
    /// Used only by DocumentController to call PhysicalFile().
    /// </summary>
    string GetAbsolutePath(string storagePath);
}
```

---

## Local Implementation: `LocalFileStorageService`

- **Storage root**: configurable via `appsettings.json` key `DocumentStoragePath` (default: `./document-uploads`)
- **Sub-directory layout**: `{year}/{month}/` (e.g., `2026/04/`) — created automatically if absent
- **Stored filename**: caller-provided GUID-based name (e.g., `3fa85f64-...-c2963f66afa6.pdf`)
- **Full path example**: `./document-uploads/2026/04/3fa85f64-...-c2963f66afa6.pdf`

```csharp
public class LocalFileStorageService : IFileStorageService
{
    private readonly string _rootPath;

    public LocalFileStorageService(IConfiguration configuration)
    {
        _rootPath = configuration["DocumentStoragePath"] ?? "./document-uploads";
    }
    // ...
}
```

**Registered in `Program.cs`**:
```csharp
builder.Services.AddScoped<IFileStorageService, LocalFileStorageService>();
```

**Production note**: To switch to Azure Blob Storage, create `AzureBlobFileStorageService : IFileStorageService` and update the DI registration. No changes to `IDocumentService` or pages are required.
