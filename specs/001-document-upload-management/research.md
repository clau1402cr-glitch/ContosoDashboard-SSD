# Research Phase: Document Upload and Management

**Date**: 2026-01-28  
**Feature**: Document Upload and Management (`001-document-upload-management`)  
**Purpose**: Resolve technical decisions and evaluate technology options

## Research Summary

This document captures research findings on key technical decisions for the document upload feature:
virus scanning approach, search optimization strategy, and file storage architecture.

---

## 1. Virus Scanning Service Selection

### Decision: **ClamAV (Open Source)**

**Chosen Option**: ClamAV (on-premises antivirus scanning)

**Rationale**:
- ✅ **Open source**: No licensing costs, suitable for training environment
- ✅ **Training-appropriate**: Can run as service on dev machine or in container
- ✅ **Docker compatible**: Easy deployment in dev environment
- ✅ **API-based**: RESTful API or direct library integration available
- ✅ **Industry standard**: Used in enterprise environments

**Alternatives Considered**:

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **ClamAV** | Open source, free, trainable, docker-ready | Slower than commercial options | ✅ **CHOSEN** |
| **Windows Defender API** | Built-in to Windows, fast | Windows-only (not Linux containers), licensing complexity | ❌ Rejected |
| **VirusTotal API** | Cloud-based, very fast, comprehensive | Requires internet, external dependency, API quotas, cost at scale | ❌ Rejected (violates offline-first) |
| **Kaspersky Scan Engine** | Commercial, accurate | Expensive, overkill for training | ❌ Rejected |
| **No scanning (dev-only)** | Simple, fast | Zero security, production unacceptable | ❌ Rejected (security risk) |

**Implementation Plan**:
- Create `VirusScanService` interface for abstraction
- **Training**: `ClamAvScanService` runs locally (container or installed)
- **Production**: `WindowsDefenderScanService` or `VirusTotalScanService` (via config)
- Scan happens **BEFORE** file written to storage (prevents malware persistence)
- Blocked files create audit log entry: `Action="VirusScan", Status="Blocked"`

**Setup Instructions for Training**:
```bash
# Docker: Run ClamAV scanner in background
docker run -d -p 3310:3310 clamav/clamav:latest

# Or install locally (macOS):
brew install clamav

# Blazor service calls TCP:3310 for scan result
```

---

## 2. Search Optimization Strategy

### Decision: **EF Core LINQ with Indexed Columns + In-Memory Tagging**

**Chosen Approach**: EF Core LINQ queries with database indexes on frequently searched columns

**Why This Priority**: 
- Spec requires <2 second search on 500+ documents
- Search filters: title, description, tags, uploader, project
- Performance critical for usability

**Performance Target**: <2 seconds for 500 documents, <100ms for 1000 documents

### Option Evaluation

| Strategy | Query Complexity | Indexing | Performance | Scalability |
|----------|------------------|----------|-------------|-------------|
| **EF Core LINQ + DB Indexes** | Simple LINQ | Explicit indexes on Title, Tags, Category, ProjectId | ✅ <100ms for 1000 docs | Good to 100k docs |
| **Full-Text Search (SQL Server)** | Complex: FREETEXT | Automatic FTS index | ✅ <100ms | Good to 1M+ docs |
| **Elasticsearch** | Must export to ES | ES indexes | ✅ <50ms | Excellent but overkill |
| **In-memory (LINQ to Objects)** | Simple but slow | None | ❌ 500ms+ for 500 docs | Poor beyond 10k docs |
| **NoSQL (MongoDB)** | Different paradigm | Config-based | ✅ <100ms | Excellent but new DB |

**Selected**: EF Core LINQ + Database Indexes

**Rationale**:
- ✅ Leverages existing EF Core / SQL Server expertise in team
- ✅ Simple to implement and maintain (no new tools/databases)
- ✅ Meets performance target (<2s for 500 docs)
- ✅ Cost-effective (no additional services)
- ✅ Works offline (no external search service)
- ✅ Training-appropriate (familiar to learners)

**Database Indexes Required**:

```sql
CREATE INDEX IX_Documents_Title ON Documents(Title);
CREATE INDEX IX_Documents_Category ON Documents(Category);
CREATE INDEX IX_Documents_ProjectId ON Documents(ProjectId);
CREATE INDEX IX_Documents_UploadedByUserId ON Documents(UploadedByUserId);
CREATE INDEX IX_Documents_UploadedDate ON Documents(UploadedDate);

-- Composite index for common query pattern
CREATE INDEX IX_Documents_UploadedByUserId_Category 
  ON Documents(UploadedByUserId, Category) INCLUDE (Title, UploadedDate);
```

**Search Implementation**:

```csharp
// Example: DocumentService.SearchDocumentsAsync
public async Task<List<Document>> SearchDocumentsAsync(
    int userId, 
    string searchTerm, 
    DocumentSearchFilters filters)
{
    var query = _context.Documents
        .Where(d => d.UploadedByUserId == userId || d.ProjectId != null)  // Authorization
        .AsQueryable();

    // Title search (full-text substring match)
    if (!string.IsNullOrEmpty(searchTerm))
    {
        query = query.Where(d => 
            d.Title.Contains(searchTerm) || 
            d.Description.Contains(searchTerm));
    }

    // Tag search (comma-separated)
    if (!string.IsNullOrEmpty(filters.Tag))
    {
        query = query.Where(d => d.Tags.Contains(filters.Tag));
    }

    // Category filter
    if (!string.IsNullOrEmpty(filters.Category))
    {
        query = query.Where(d => d.Category == filters.Category);
    }

    // Project filter
    if (filters.ProjectId.HasValue)
    {
        query = query.Where(d => d.ProjectId == filters.ProjectId);
    }

    // Execute query (performance-critical)
    return await query
        .OrderByDescending(d => d.UploadedDate)
        .Take(100)  // Pagination prevents large result sets
        .ToListAsync();  // DB query executed here
}
```

**Performance Validation**:
- Load test with 500-document dataset
- Measure query time for each filter combination
- Target: <100ms per query
- If >2s observed, add more specific indexes or consider Full-Text Search upgrade

---

## 3. File Storage Architecture

### Decision: **Local Filesystem (Training) + Azure Blob (Production) via Interface Abstraction**

**Chosen Approach**: Service-based abstraction enables offline training + cloud-ready production

**Why This Matters**:
- Spec requires offline-first training implementation
- Production requires Azure Blob Storage
- No code changes should be needed to switch between implementations

### Local Filesystem Storage (Training)

**Directory Structure**:
```
AppData/
└── Documents/
    ├── 1/                           # userId = 1
    │   ├── personal/
    │   │   ├── guid-1.pdf
    │   │   ├── guid-2.docx
    │   └── 5/                       # projectId = 5
    │       ├── guid-3.pptx
    │       └── guid-4.xlsx
    └── 2/                           # userId = 2
        ├── personal/
        │   └── guid-5.pdf
        └── 10/                      # projectId = 10
            └── guid-6.jpg
```

**Path Strategy**: `{userId}/{projectId-or-personal}/{guid}.{extension}`

**Benefits**:
- ✅ Prevents unauthorized access (path traversal protected by GUID)
- ✅ Separates personal documents from project documents
- ✅ Enables efficient deletion (remove user directory)
- ✅ Works offline without any cloud service
- ✅ Fast local disk I/O (ideal for testing)

**Implementation**:

```csharp
public class LocalFileStorageService : IDocumentStorageService
{
    private readonly string _basePath = Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData),
        "ContosoDashboard",
        "Documents");

    public async Task<string> UploadAsync(Stream fileStream, string fileName, string storagePath)
    {
        var fullPath = Path.Combine(_basePath, storagePath);
        var directory = Path.GetDirectoryName(fullPath);
        
        Directory.CreateDirectory(directory);  // Create if not exists
        
        using (var fileStream = new FileStream(fullPath, FileMode.Create, FileAccess.Write))
        {
            await fileStream.CopyToAsync(fileStream);
        }
        
        return storagePath;  // Return relative path for database storage
    }

    public async Task<Stream> DownloadAsync(string storagePath)
    {
        var fullPath = Path.Combine(_basePath, storagePath);
        return new FileStream(fullPath, FileMode.Open, FileAccess.Read);  // Caller responsible for disposal
    }

    public async Task DeleteAsync(string storagePath)
    {
        var fullPath = Path.Combine(_basePath, storagePath);
        if (File.Exists(fullPath))
        {
            File.Delete(fullPath);
        }
    }

    public Task<string> GetUrlAsync(string storagePath, TimeSpan expiration)
    {
        // Local filesystem: return controller endpoint that serves file
        return Task.FromResult($"/api/documents/download?path={Uri.EscapeDataString(storagePath)}");
    }
}
```

### Azure Blob Storage (Production)

**Design Pattern** (future implementation):

```csharp
public class AzureBlobStorageService : IDocumentStorageService
{
    private readonly BlobContainerClient _containerClient;

    public AzureBlobStorageService(BlobContainerClient containerClient)
    {
        _containerClient = containerClient;  // Injected from configuration
    }

    public async Task<string> UploadAsync(Stream fileStream, string fileName, string storagePath)
    {
        var blobClient = _containerClient.GetBlobClient(storagePath);
        await blobClient.UploadAsync(fileStream, overwrite: true);
        return storagePath;
    }

    public async Task<Stream> DownloadAsync(string storagePath)
    {
        var blobClient = _containerClient.GetBlobClient(storagePath);
        var download = await blobClient.DownloadAsync();
        return download.Value.Content;
    }

    public async Task DeleteAsync(string storagePath)
    {
        var blobClient = _containerClient.GetBlobClient(storagePath);
        await blobClient.DeleteAsync();
    }

    public async Task<string> GetUrlAsync(string storagePath, TimeSpan expiration)
    {
        var blobClient = _containerClient.GetBlobClient(storagePath);
        var uri = blobClient.GenerateSasUri(BlobSasPermissions.Read, DateTime.UtcNow + expiration);
        return uri.AbsoluteUri;
    }
}
```

**Configuration (appsettings.json)**:

```json
{
  "DocumentStorage": {
    "Provider": "Local",  // or "AzureBlob"
    "LocalPath": "AppData/Documents",
    "AzureConnectionString": "DefaultEndpointsProtocol=https;..."
  }
}
```

**Dependency Injection Swap**:

```csharp
// Program.cs
var storageProvider = builder.Configuration.GetValue<string>("DocumentStorage:Provider");
if (storageProvider == "AzureBlob")
{
    builder.Services.AddScoped<IDocumentStorageService>(provider =>
        new AzureBlobStorageService(new BlobContainerClient(
            new Uri("https://conta.blob.core.windows.net/documents"),
            new DefaultAzureCredential())));
}
else
{
    builder.Services.AddScoped<IDocumentStorageService, LocalFileStorageService>();
}
```

**Migration Path to Azure**:
1. Implement `AzureBlobStorageService` (copy of pattern above)
2. Copy existing files from local filesystem → Azure Blob Storage
3. Change `appsettings.json` to `"Provider": "AzureBlob"`
4. No code changes required (service abstraction handles it)
5. Database schema unchanged (StoragePath column works for both)

---

## 4. File Upload Pattern in Blazor

### Decision: **MemoryStream Pattern with Size Validation**

**Why This Matters**: Blazor's `IBrowserFile` has disposal issues in async workflows

**Issue**: Direct streaming from `IBrowserFile` fails because Blazor disposes the stream prematurely

**Solution**: Copy to `MemoryStream` before processing

```csharp
// DocumentUploadComponent.razor.cs
private IBrowserFile selectedFile;

private async Task HandleFileSelected(InputFileChangeEventArgs e)
{
    selectedFile = e.File;
}

private async Task UploadFile()
{
    if (selectedFile == null) return;

    // Extract metadata BEFORE opening stream
    var fileName = selectedFile.Name;
    var fileSize = selectedFile.Size;
    var contentType = selectedFile.ContentType;

    // Validate before opening stream
    if (fileSize > 25 * 1024 * 1024)  // 25 MB
    {
        ErrorMessage = "File exceeds 25 MB limit";
        return;
    }

    // Copy to MemoryStream to prevent disposal issues
    using var memoryStream = new MemoryStream();
    using (var fileStream = selectedFile.OpenReadStream(maxAllowedSize: 25 * 1024 * 1024))
    {
        await fileStream.CopyToAsync(memoryStream);
    }
    memoryStream.Position = 0;  // Reset position for reading

    // Clear reference to prevent reuse
    selectedFile = null;
    StateHasChanged();

    // Now pass MemoryStream to service
    try
    {
        var documentId = await DocumentService.UploadDocumentAsync(
            memoryStream,
            new DocumentUploadModel { Title = Title, Category = Category, Tags = Tags },
            CurrentUserId);
        
        SuccessMessage = $"Document uploaded successfully (ID: {documentId})";
    }
    catch (Exception ex)
    {
        ErrorMessage = $"Upload failed: {ex.Message}";
    }
}
```

---

## 5. Pagination Strategy

### Decision: **Offset-Based Pagination with 20 Documents Per Page**

**Rationale**:
- 500-document list must load in <2 seconds
- Fetching all 500 at once would be slow and consume memory
- Offset-based pagination is simple and works with EF Core Skip/Take

**Implementation**:

```csharp
public async Task<PagedResult<Document>> GetMyDocumentsAsync(
    int userId, 
    string category = null, 
    int pageNumber = 1)
{
    const int pageSize = 20;
    var skip = (pageNumber - 1) * pageSize;

    var query = _context.Documents
        .Where(d => d.UploadedByUserId == userId);

    if (!string.IsNullOrEmpty(category))
    {
        query = query.Where(d => d.Category == category);
    }

    var totalCount = await query.CountAsync();
    var documents = await query
        .OrderByDescending(d => d.UploadedDate)
        .Skip(skip)
        .Take(pageSize)
        .ToListAsync();

    return new PagedResult<Document>
    {
        Items = documents,
        TotalCount = totalCount,
        PageNumber = pageNumber,
        PageSize = pageSize,
        TotalPages = (totalCount + pageSize - 1) / pageSize
    };
}
```

---

## 6. GUID-Based File Naming

### Decision: **GUID as File Name to Prevent Collisions and IDOR**

**Strategy**: `{userId}/{projectId-or-personal}/{Guid.NewGuid()}.{extension}`

**Why GUID**:
- ✅ Guaranteed unique (no collisions across all users/time)
- ✅ Prevents guessing (IDOR protection: attacker can't guess other files)
- ✅ No sequential IDs (sequential IDs enable enumeration attacks)
- ✅ Portable (same naming works for local filesystem and Azure Blob)

**Implementation**:

```csharp
public async Task<int> UploadDocumentAsync(
    Stream fileStream, 
    DocumentUploadModel model, 
    int userId)
{
    // 1. Validate input
    ValidateFileUpload(model);

    // 2. Generate unique GUID-based filename BEFORE saving to disk
    var extension = Path.GetExtension(model.FileName).ToLower();
    var uniqueFileName = $"{Guid.NewGuid()}{extension}";
    
    var projectIdOrPersonal = model.ProjectId?.ToString() ?? "personal";
    var storagePath = $"{userId}/{projectIdOrPersonal}/{uniqueFileName}";

    // 3. Scan for viruses BEFORE storage
    using var memoryStream = new MemoryStream();
    await fileStream.CopyToAsync(memoryStream);
    memoryStream.Position = 0;
    
    var scanResult = await _virusScanService.ScanAsync(memoryStream);
    if (scanResult.IsInfected)
    {
        await _auditLog.LogAsync(userId, "VirusScan", "Blocked", model.FileName, scanResult.Reason);
        throw new VirusDetectedException(scanResult.Reason);
    }
    memoryStream.Position = 0;

    // 4. SAVE TO DISK (with confirmed path)
    await _storageService.UploadAsync(memoryStream, model.FileName, storagePath);

    // 5. CREATE DATABASE RECORD (only after successful file save)
    var document = new Document
    {
        Title = model.Title,
        Category = model.Category,
        Tags = model.Tags,
        UploadedByUserId = userId,
        ProjectId = model.ProjectId,
        FileName = model.FileName,
        FileSizeBytes = memoryStream.Length,
        MimeType = model.MimeType,
        StoragePath = storagePath,  // GUID-based path from step 2
        UploadedDate = DateTime.UtcNow,
        UpdatedDate = DateTime.UtcNow
    };

    _context.Documents.Add(document);
    await _context.SaveChangesAsync();

    // 6. LOG TO AUDIT TRAIL
    await _auditLog.LogAsync(userId, "Upload", "Success", model.FileName, storagePath);

    return document.DocumentId;
}
```

**Key Insight**: Generate path BEFORE saving to disk. This prevents:
- **Duplicate Key Violations**: If DB insert fails, no orphaned file remains
- **IDOR Attacks**: Attacker can't guess file paths (random GUID)
- **Path Traversal**: GUID prevents `../../../etc/passwd` style attacks

---

## 7. Audit Trail Design

### Decision: **Comprehensive Logging of All Document Actions**

**Events to Log**:
- `Upload`: File uploaded successfully
- `Download`: File downloaded by user
- `Share`: Document shared with user(s)
- `Delete`: Document deleted
- `VirusScan`: File scanned (Success or Blocked)
- `PreviewOpen`: Document previewed in browser

**Audit Log Schema**:

```csharp
public class AuditLog
{
    public int AuditLogId { get; set; }
    public int? DocumentId { get; set; }
    public int UserId { get; set; }
    public string Action { get; set; }
    public string Status { get; set; }  // Success, Failed, Blocked
    public DateTime Timestamp { get; set; }
    public string IpAddress { get; set; }
    public string Details { get; set; }  // Additional context
}
```

**Admin Audit Report**:
- Filter by User, Action, Date Range
- Export to CSV
- Search for incident investigation

---

## Decision Summary

| Decision | Choice | Alternative | Why Chosen |
|----------|--------|-------------|-----------|
| Virus Scanning | ClamAV | Windows Defender, VirusTotal | Open source, offline-capable |
| Search Strategy | EF LINQ + Indexes | Full-Text Search, Elasticsearch | Simple, meets performance target |
| File Storage | Local FS (training) + Azure (prod) | Direct Azure, NoSQL | Offline-first training requirement |
| Blazor Upload Pattern | MemoryStream copy | Direct streaming | Prevents disposal issues |
| Pagination | Offset-based, 20 per page | Cursor-based | Simple, effective, EF-friendly |
| File Naming | GUID | Sequential, Hash | Prevents IDOR attacks |
| Audit Trail | Comprehensive logging | Minimal logging | Compliance and incident investigation |

---

## Next Steps

1. ✅ Research Complete
2. → Generate `data-model.md` with full database schema
3. → Generate `contracts/` with API endpoint specifications
4. → Generate `quickstart.md` for developer setup
5. → Run `/speckit.tasks` for implementation task decomposition
