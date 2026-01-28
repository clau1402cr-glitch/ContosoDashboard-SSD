# Implementation Plan: Document Upload and Management

**Branch**: `001-document-upload-management` | **Date**: 2026-01-28 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-document-upload-management/spec.md`

## Summary

Enable Contoso employees to upload, organize, search, and share work-related documents (PDF, Office, images, text)
within ContosoDashboard. System enforces role-based access control via service-layer authorization, maintains
comprehensive audit logs, and implements Defense-in-Depth security with encryption, virus scanning, and IDOR protection.

**Technical Approach**:
- Service-Oriented Architecture: DocumentService encapsulates all business logic with authorization checks
- Infrastructure Abstraction: IDocumentStorageService enables local filesystem (training) ↔ Azure Blob (production) swap
- Security by Defense-in-Depth: Authorization at middleware, page, service, and data layers
- Async/Await Pattern: All file operations non-blocking using MemoryStream for Blazor compatibility
- Database: New entities (Document, DocumentShare, AuditLog) with explicit indexes for performance

## Technical Context

**Language/Version**: ASP.NET Core 8.0  
**Primary Dependencies**: 
- `Azure.Storage.Blobs` 11.0.0 (production)
- `System.IO` (training local filesystem)
- `Microsoft.EntityFrameworkCore` 8.0 (already in project)

**Storage**: 
- **Training**: Local filesystem at `{AppDataDir}/Documents/{userId}/{projectId-or-personal}/{guid}.{ext}`
- **Production**: Azure Blob Storage with container-per-tenant structure

**Testing**: 
- Unit tests: DocumentService with mock IDocumentStorageService
- Integration tests: End-to-end upload/download/share workflows
- Security tests: IDOR vulnerability, authorization layer, cross-user isolation
- Performance tests: 500-document load time, search <2s, upload <30s

**Target Platform**: ASP.NET Core web application (Blazor Server)

**Project Type**: Web application (backend service + Blazor frontend)

**Performance Goals**:
- Upload 25 MB: <30 seconds
- List documents (500 docs): <2 seconds
- Search queries: <2 seconds
- PDF/image preview: <3 seconds
- Virus scan: <5 seconds (before storage)
- Pagination: 20 documents per page

**Constraints**:
- 8-10 week delivery timeline
- No external cloud integration for training environment
- Entra ID authentication required for production
- All file operations must be async/await (Blazor compatibility)
- Support file types: PDF, Office (Word, Excel, PowerPoint), text, images (JPEG, PNG)
- Max file size: 25 MB per file
- Max total storage per user: Unlimited (storage quotas out of scope)

**Scale/Scope**:
- 5,000 target users
- Expected 50,000-250,000 documents total
- 10-50 documents per user over 3-month ramp
- 500+ documents per power user

## Constitution Check

**Validating against ContosoDashboard Constitution v1.0.0**:

### ✅ Gate 1: Training-First Design (PASS)
- ✅ Local filesystem implementation for offline training without Azure costs
- ✅ Clear code comments explaining design decisions documented
- ✅ Migration path to Azure documented in plan

### ✅ Gate 2: Offline-First with Cloud Migration Path (PASS)
- ✅ `IDocumentStorageService` interface abstracts storage (local vs. Azure)
- ✅ `LocalFileStorageService` implements for training with System.IO
- ✅ `AzureBlobStorageService` design prepared for production migration
- ✅ Dependency injection enables configuration-driven swap (no code changes)

### ✅ Gate 3: Security by Defense-in-Depth (PASS)
- ✅ Middleware Layer: `[Authorize]` attributes on all document endpoints
- ✅ Page Layer: Blazor components require authentication state
- ✅ Service Layer: `DocumentService.GetDocumentsAsync()` filters by current user authorization
- ✅ Data Layer: Explicit foreign keys (Document.UploadedByUserId, Document.ProjectId)
- ✅ Virus Scanning: ClamAV scan before persistence prevents malware storage
- ✅ Audit Trail: All actions logged (upload, download, share, delete, virus blocks)
- ✅ IDOR Protection: Service validates project membership before returning project documents

### ✅ Gate 4: Service-Oriented Architecture (PASS)
- ✅ `IDocumentService` interface: Upload, Search, Get, Share, Delete, Audit methods
- ✅ All authorization checks in service layer (service layer is single authority for access control)
- ✅ No direct EF Core queries from Blazor pages (all through DocumentService)
- ✅ Service independently testable with mock IDocumentStorageService

### ✅ Gate 5: Role-Based Access Control (PASS)
- ✅ **Employee**: Upload, view own docs, search, download, view shared docs
- ✅ **TeamLead**: All Employee + view team member uploads
- ✅ **ProjectManager**: All TeamLead + manage project documents, share across team
- ✅ **Administrator**: Full system access, audit reports, delete any document
- ✅ Service methods enforce role checks: `DocumentService` checks `currentUser.Role` before operations

**Constitution Status**: ✅ **ALL GATES PASS** - Feature fully compliant with ContosoDashboard Constitution v1.0.0

## Project Structure

### Documentation (this feature)

```text
specs/001-document-upload-management/
├── spec.md                          # Feature specification (6 user stories)
├── plan.md                          # This file - technical architecture
├── research.md                      # Phase 0: Technology research (virus scanning, search optimization)
├── data-model.md                    # Phase 1: Complete database schema with indexes
├── contracts/                       # Phase 1: API endpoint definitions
│   ├── document-service.md          # DocumentService interface + method signatures
│   ├── endpoints.md                 # REST/Blazor service endpoints
│   └── document-models.md           # DTOs and request/response models
├── quickstart.md                    # Phase 1: Getting started guide for developers
└── checklists/
    └── requirements.md              # Quality validation checklist (APROBADO)
```

### Source Code (repository root)

```text
ContosoDashboard/
├── Models/
│   ├── Document.cs                  # NEW: Document entity
│   ├── DocumentShare.cs             # NEW: Share relationship entity
│   ├── AuditLog.cs                  # NEW: Audit logging entity
│   └── [Existing models...]
├── Services/
│   ├── IDocumentService.cs          # NEW: Service interface
│   ├── DocumentService.cs           # NEW: Business logic + authorization
│   ├── IDocumentStorageService.cs   # NEW: Storage abstraction interface
│   ├── LocalFileStorageService.cs   # NEW: Local filesystem implementation (training)
│   ├── AzureBlobStorageService.cs   # NEW: Azure Blob implementation stub (production)
│   ├── VirusScanService.cs          # NEW: ClamAV integration
│   └── [Existing services...]
├── Pages/
│   ├── Documents/                   # NEW: Document management pages
│   │   ├── MyDocuments.razor        # View my uploaded documents
│   │   ├── ProjectDocuments.razor   # View project documents
│   │   ├── DocumentUpload.razor     # Upload form component
│   │   ├── DocumentShare.razor      # Share dialog
│   │   ├── DocumentPreview.razor    # PDF/image preview viewer
│   │   └── AdminAuditLogs.razor     # Admin audit report page
│   └── [Existing pages...]
├── Data/
│   ├── ApplicationDbContext.cs      # MODIFIED: Add DbSet<Document>, DbSet<DocumentShare>, DbSet<AuditLog>
│   └── Migrations/
│       └── [AUTO-GENERATED: Add document tables]
├── Program.cs                       # MODIFIED: Register DocumentService, DocumentStorageService
├── appsettings.json                 # MODIFIED: Add document storage settings
└── [Existing structure...]

Tests/
├── Unit/
│   ├── DocumentServiceTests.cs      # NEW: Service logic + authorization tests
│   ├── DocumentStorageTests.cs      # NEW: Storage abstraction tests (mocked)
│   └── VirusScanTests.cs            # NEW: Virus scanning logic
├── Integration/
│   ├── DocumentUploadE2ETests.cs    # NEW: Full upload workflow
│   ├── DocumentSearchTests.cs       # NEW: Search functionality
│   ├── DocumentShareTests.cs        # NEW: Sharing + notifications
│   └── AuditLoggingTests.cs         # NEW: Audit trail verification
└── Security/
    ├── IDORTests.cs                 # NEW: IDOR vulnerability tests (cross-user access)
    └── AuthorizationTests.cs        # NEW: Role-based access control tests
```

**Structure Decision**: Web application structure with backend services (DocumentService, DocumentStorageService)
and Blazor frontend components (MyDocuments, DocumentUpload, etc.). Separates concerns: models in Models/,
business logic in Services/, UI in Pages/, data access through EF Core in Data/. Tests organized by type
(Unit, Integration, Security) for clarity and maintainability.

## Implementation Phases

### Phase 1: Core Infrastructure (Week 1-2)

**Deliverables**:
- Database schema (Document, DocumentShare, AuditLog tables with indexes)
- Service interfaces (IDocumentService, IDocumentStorageService, IVirusScanService)
- Local filesystem storage implementation
- Basic Blazor upload page
- Unit tests for DocumentService authorization

**Acceptance Gates**:
- ✅ Upload creates Document record in database
- ✅ Service-layer authorization prevents unauthorized access
- ✅ File stored with GUID-based name preventing collisions
- ✅ Database migrations execute cleanly on fresh database

### Phase 2: Search & Organization (Week 2-3)

**Deliverables**:
- Search implementation (title, tags, uploader, project filters)
- My Documents page with category filtering
- Pagination for 500+ document lists
- Performance testing (search <2s, list load <2s)

**Acceptance Gates**:
- ✅ Search returns results in <2 seconds
- ✅ 500-document list loads in <2 seconds
- ✅ Users only see documents they're authorized for

### Phase 3: Sharing & Notifications (Week 3-4)

**Deliverables**:
- Document sharing logic with role validation
- Notification creation when sharing
- "Shared with Me" view
- Share authorization (ProjectManager/Admin only)

**Acceptance Gates**:
- ✅ Share creates notification for recipient
- ✅ Non-ProjectManager sees disabled Share button
- ✅ Deleted document removes access for shared users

### Phase 4: Preview & Download (Week 4-5)

**Deliverables**:
- PDF/image preview viewer component (3s load time)
- Download with original filename preservation
- Unsupported file type messaging

**Acceptance Gates**:
- ✅ PDF/image preview loads in <3 seconds
- ✅ Download preserves file format and name

### Phase 5: Audit & Admin Features (Week 5-6)

**Deliverables**:
- Audit log recording (upload, download, share, delete, virus scan)
- Admin audit report page
- CSV export functionality
- Virus scan event logging

**Acceptance Gates**:
- ✅ All document actions appear in audit log
- ✅ Admin can filter logs by user/date
- ✅ CSV export contains all fields

### Phase 6: Security Hardening & Performance (Week 6-8)

**Deliverables**:
- Virus scanning integration (ClamAV or Windows Defender)
- IDOR vulnerability testing and fixes
- Security headers (CSP, X-Frame-Options)
- Performance load testing (25MB upload <30s, search <2s)
- Code security review

**Acceptance Gates**:
- ✅ Zero IDOR vulnerabilities found
- ✅ Virus-infected file blocks upload
- ✅ Performance targets met (upload <30s, search <2s)
- ✅ Security code review passed

### Phase 7: Testing & UAT (Week 8-10)

**Deliverables**:
- End-to-end test scenarios
- User acceptance testing with pilot users
- Bug fixes and performance tuning
- Documentation and training materials

**Acceptance Gates**:
- ✅ UAT sign-off from stakeholders
- ✅ All critical bugs resolved
- ✅ Performance meets targets
- ✅ Documentation complete

## Database Schema

### New Tables

```sql
-- Documents table
CREATE TABLE Documents (
    DocumentId INT PRIMARY KEY IDENTITY,
    Title NVARCHAR(255) NOT NULL,
    Description NVARCHAR(2000),
    Category NVARCHAR(50) NOT NULL,  -- Project Documents, Team Resources, Personal Files, Reports, etc.
    Tags NVARCHAR(500),  -- Comma-separated
    UploadedByUserId INT NOT NULL,
    ProjectId INT,
    FileName NVARCHAR(255) NOT NULL,
    FileSizeBytes BIGINT NOT NULL,
    MimeType NVARCHAR(255) NOT NULL,  -- application/pdf, image/jpeg, etc.
    StoragePath NVARCHAR(500) NOT NULL,  -- GUID-based path {userId}/{projectId}/{guid}.ext
    UploadedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    FOREIGN KEY (UploadedByUserId) REFERENCES Users(UserId),
    FOREIGN KEY (ProjectId) REFERENCES Projects(ProjectId),
    CREATE INDEX IX_Documents_UploadedByUserId ON Documents(UploadedByUserId),
    CREATE INDEX IX_Documents_ProjectId ON Documents(ProjectId),
    CREATE INDEX IX_Documents_Category ON Documents(Category),
    CREATE INDEX IX_Documents_UploadedDate ON Documents(UploadedDate)
);

-- Document Shares table
CREATE TABLE DocumentShares (
    ShareId INT PRIMARY KEY IDENTITY,
    DocumentId INT NOT NULL,
    SharedWithUserId INT NOT NULL,
    SharedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    FOREIGN KEY (DocumentId) REFERENCES Documents(DocumentId) ON DELETE CASCADE,
    FOREIGN KEY (SharedWithUserId) REFERENCES Users(UserId) ON DELETE CASCADE,
    CREATE UNIQUE INDEX IX_DocumentShares_Unique ON DocumentShares(DocumentId, SharedWithUserId),
    CREATE INDEX IX_DocumentShares_SharedWithUserId ON DocumentShares(SharedWithUserId)
);

-- Audit Logs table
CREATE TABLE AuditLogs (
    AuditLogId INT PRIMARY KEY IDENTITY,
    DocumentId INT,
    UserId INT NOT NULL,
    Action NVARCHAR(50) NOT NULL,  -- Upload, Download, Share, Delete, VirusScan
    Status NVARCHAR(50) NOT NULL,  -- Success, Failed, Blocked
    Timestamp DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    IpAddress NVARCHAR(50),
    Details NVARCHAR(500),
    FOREIGN KEY (DocumentId) REFERENCES Documents(DocumentId),
    FOREIGN KEY (UserId) REFERENCES Users(UserId),
    CREATE INDEX IX_AuditLogs_UserId ON AuditLogs(UserId),
    CREATE INDEX IX_AuditLogs_Timestamp ON AuditLogs(Timestamp),
    CREATE INDEX IX_AuditLogs_Action ON AuditLogs(Action)
);
```

### Modified Table

```sql
-- ApplicationDbContext: Add DbSets
public DbSet<Document> Documents { get; set; }
public DbSet<DocumentShare> DocumentShares { get; set; }
public DbSet<AuditLog> AuditLogs { get; set; }
```

## Service Layer Design

### IDocumentService Interface

```csharp
public interface IDocumentService
{
    // Upload & Management
    Task<int> UploadDocumentAsync(Stream fileStream, DocumentUploadModel model, int userId);
    Task<Document> GetDocumentAsync(int documentId, int userId);  // Authorization check
    Task<List<Document>> GetMyDocumentsAsync(int userId, string category = null, int page = 1);
    Task<List<Document>> SearchDocumentsAsync(int userId, string searchTerm, DocumentSearchFilters filters);
    Task UpdateDocumentMetadataAsync(int documentId, DocumentUpdateModel model, int userId);
    Task DeleteDocumentAsync(int documentId, int userId);
    
    // Sharing
    Task ShareDocumentAsync(int documentId, int[] userIds, int userId);  // Only ProjectManager/Admin
    Task<List<Document>> GetSharedWithMeAsync(int userId);
    
    // Audit
    Task<List<AuditLog>> GetAuditLogsAsync(int? userId, DateTime? startDate, int? limit);
    Task ExportAuditLogsAsync(Stream outputStream, AuditLogExportModel model);
}

public interface IDocumentStorageService
{
    Task<string> UploadAsync(Stream fileStream, string fileName, string storagePath);
    Task DeleteAsync(string storagePath);
    Task<Stream> DownloadAsync(string storagePath);
    Task<string> GetUrlAsync(string storagePath, TimeSpan expiration);
}

public interface IVirusScanService
{
    Task<VirusScanResult> ScanAsync(Stream fileStream);  // Result: Safe, Infected, Error
}
```

## API Endpoints

```
POST /api/documents/upload
  Body: { file, title, category, tags, projectId? }
  Response: { documentId, uploadedDate, uploadedBy }
  Auth: [Authorize] Any authenticated user

GET /api/documents
  Query: { page, category?, projectId? }
  Response: [{ documentId, title, category, size, uploadedDate, uploadedBy }]
  Auth: [Authorize] Filters by user authorization

GET /api/documents/search
  Query: { term, tags?, uploader?, projectId?, dateFrom?, dateTo? }
  Response: [{ documentId, title, category, matchedFields }]
  Auth: [Authorize] <2 second SLA

GET /api/documents/{id}/preview
  Response: Stream (PDF/image MIME type)
  Auth: [Authorize] With download tracking for audit

POST /api/documents/{id}/share
  Body: { userIds: [int] }
  Response: { sharedCount, notifications: [int] }
  Auth: [Authorize] ProjectManager+ only

DELETE /api/documents/{id}
  Auth: [Authorize] Owner or Admin

GET /api/admin/audit-logs
  Query: { userId?, action?, startDate?, limit }
  Response: [{ timestamp, user, action, document, status }]
  Auth: [Authorize(Roles = "Administrator")]

GET /api/admin/audit-logs/export
  Query: { format: 'csv' }
  Response: CSV Stream
  Auth: [Authorize(Roles = "Administrator")]
```

## Dependency Injection Configuration

```csharp
// Program.cs
builder.Services.AddScoped<IDocumentService, DocumentService>();
builder.Services.AddScoped<IDocumentStorageService, LocalFileStorageService>();  // Training
// builder.Services.AddScoped<IDocumentStorageService, AzureBlobStorageService>();  // Production
builder.Services.AddScoped<IVirusScanService, ClamAvScanService>();
```

## Next Steps

1. **Phase 0: Research** → Generate `research.md` with virus scanning evaluation, search optimization
2. **Phase 1: Data Model** → Generate `data-model.md` with complete schema, indexes, relationships
3. **Phase 1: Contracts** → Generate `contracts/` with API endpoint details, DTOs, error handling
4. **Phase 1: Quickstart** → Generate `quickstart.md` for developer onboarding
5. **Phase 2: Tasks** → Run `/speckit.tasks` to decompose into development tasks by user story
6. **Implementation** → Run `/speckit.implement` to generate code scaffolds
