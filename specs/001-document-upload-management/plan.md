# Implementation Plan: Document Upload and Management

**Branch**: `001-document-upload-management` | **Date**: 2026-04-05 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-document-upload-management/spec.md`

## Summary

Add a centralized document management capability to ContosoDashboard, enabling employees to upload, organize, search, and share work-related documents. Implementation uses a local `IFileStorageService` (files stored outside `wwwroot` with GUID-based names) plus three new EF Core entities (`Document`, `DocumentShare`, `DocumentActivity`) in the existing SQLite database. A secured MVC controller endpoint handles file download and preview. All operations are offline-capable.

## Technical Context

**Language/Version**: C# 12 / .NET 10  
**Primary Dependencies**: ASP.NET Core Blazor Server, EF Core 10 (SQLite), Cookie-based auth, Bootstrap 5.3 + Bootstrap Icons  
**Storage**: SQLite via EF Core (document metadata) + local filesystem via `IFileStorageService` (file binaries outside `wwwroot`)  
**Testing**: No test project exists; xUnit is the .NET standard if one is added  
**Target Platform**: Local dev machine / Codespaces — offline-capable, cross-platform  
**Project Type**: Single Blazor Server project (`ContosoDashboard/`)  
**Performance Goals**: Document list page ≤2 s (up to 500 docs); search ≤2 s; upload ≤30 s for 25 MB; preview ≤3 s  
**Constraints**: 25 MB per-file limit; files stored outside `wwwroot`; GUID-based stored filenames; extension whitelist (PDF, DOCX, XLSX, PPTX, TXT, JPEG, PNG); no cloud dependencies  
**Scale/Scope**: Up to 500 documents per user document view; multi-user single-instance deployment

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Offline-First | ✅ PASS | SQLite + local filesystem; no cloud calls. FR-005 (malware scan) resolved via local `IFileScanService` stub — see research.md. |
| II. Defense in Depth | ✅ PASS | Auth middleware on all routes; `[Authorize]` on pages; service-level IDOR checks on every doc operation; GUID filenames; whitelist extension validation; files outside `wwwroot`; secure download controller endpoint. |
| III. Separation of Concerns | ✅ PASS | Models hold entity definitions only; `IDocumentService` / `IFileStorageService` / `IFileScanService` hold all business logic; `ApplicationDbContext` extended for persistence; Razor pages delegate to services via DI. |
| IV. Interface-Driven Abstractions | ✅ PASS | Three new interfaces (`IDocumentService`, `IFileStorageService`, `IFileScanService`) registered in DI; swapping to cloud implementations requires only `Program.cs` changes. |
| V. Training Clarity | ✅ PASS | Mock file-scan stub documented with production notes; preview uses browser-native capabilities documented with production alternatives. |

**Post-Phase 1 re-check**: ✅ No new violations introduced. `DocumentController` is the only cross-cutting addition and follows security-gating patterns identical to existing auth middleware.

## Project Structure

### Documentation (this feature)

```text
specs/001-document-upload-management/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   ├── IDocumentService.md
│   ├── IFileStorageService.md
│   └── IFileScanService.md
└── tasks.md             # Phase 2 output (/speckit.tasks — NOT created here)
```

### Source Code (repository root)

```text
ContosoDashboard/
├── Controllers/
│   └── DocumentController.cs        # new — secure download/preview endpoint
├── Models/
│   ├── Document.cs                  # new
│   ├── DocumentShare.cs             # new
│   └── DocumentActivity.cs          # new
├── Services/
│   ├── IDocumentService.cs          # new — interface + implementation
│   ├── DocumentService.cs           # new
│   ├── IFileStorageService.cs       # new — interface
│   ├── LocalFileStorageService.cs   # new — local filesystem implementation
│   ├── IFileScanService.cs          # new — interface
│   └── LocalFileScanService.cs      # new — magic-byte validation (training stub)
├── Pages/
│   ├── Documents.razor              # new — My Documents + Shared with Me tabs
│   └── DocumentDetails.razor        # new — view / edit metadata / share / delete
│   └── ProjectDetails.razor         # existing — add Documents tab
├── Data/
│   └── ApplicationDbContext.cs      # extend with Document, DocumentShare, DocumentActivity DbSets
└── Program.cs                       # register new services + add MVC controllers
```

**Structure Decision**: Single Blazor Server project (`ContosoDashboard/`). A new `Controllers/` folder is added for the `DocumentController` download/preview endpoint — this is the standard pattern for serving protected static files in Blazor Server without exposing them via `wwwroot`.
