# Contract: IFileScanService

**Feature**: `001-document-upload-management`  
**Namespace**: `ContosoDashboard.Services`  
**Layer**: Infrastructure (security validation abstraction)

---

## Purpose

`IFileScanService` abstracts malware / content validation so the training-safe local stub can be replaced with a production-grade scanner (ClamAV or cloud-based AV) by DI configuration alone (Principle IV). It satisfies FR-005 while remaining offline-capable (Principle I).

---

## Interface

```csharp
public interface IFileScanService
{
    /// <summary>
    /// Scans the provided stream for threats/invalid content.
    /// The stream position is reset to 0 before and after scanning.
    /// Returns a FileScanResult indicating whether the content is safe to store.
    /// </summary>
    Task<FileScanResult> ScanFileAsync(Stream fileStream, string originalFileName);
}

public record FileScanResult(bool IsClean, string Reason);
```

---

## Local Implementation: `LocalFileScanService`

Validates magic bytes of the file stream against the declared file extension.

| Extension | Magic bytes (hex) | ASCII hint |
|-----------|-------------------|------------|
| `.pdf` | `25 50 44 46` | `%PDF` |
| `.jpg` / `.jpeg` | `FF D8 FF` | — |
| `.png` | `89 50 4E 47 0D 0A 1A 0A` | `‰PNG` |
| `.docx` / `.xlsx` / `.pptx` | `50 4B 03 04` | `PK` (ZIP) |
| `.txt` | *(no signature)* | Extension-only check |

Returns `FileScanResult(IsClean: false, ...)` for:
- File whose first bytes do not match the declared extension
- Extensions not on the whitelist

```csharp
// TRAINING NOTE: LocalFileScanService performs magic-byte validation only.
// In a production deployment, replace this with:
//   - Offline: ClamAV via ClamAV.NET package (requires ClamD daemon)
//   - Cloud: Azure Defender for Storage (requires Azure subscription)
// Register the alternative implementation in Program.cs — no other code changes needed.
public class LocalFileScanService : IFileScanService { ... }
```

**Registered in `Program.cs`**:
```csharp
builder.Services.AddScoped<IFileScanService, LocalFileScanService>();
```
