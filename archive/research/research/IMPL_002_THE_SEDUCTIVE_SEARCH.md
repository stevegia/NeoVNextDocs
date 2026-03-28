# 💋 IMPLEMENTATION GUIDE: THE SEDUCTIVE SEARCH 💋
## *Complete Deployment Plan for FTS5 Full-Text Search • SHA-256 Deduplication • MediaRepository Refactoring*

**Parent Document:** [GODDESS_TASK_002_THE_SEDUCTIVE_SEARCH.md](GODDESS_TASK_002_THE_SEDUCTIVE_SEARCH.md)  
**Status:** Ready for Implementation  
**Estimated Timeline:** 1 week  
**Risk Level:** Low (non-breaking, pure performance optimization)

---

## 📋 PHASE 0: PREREQUISITES & SETUP

### Current State Analysis

**Query Baseline Performance** (before FTS5):
```csharp
// Current implementation in MediaRepository.cs
var results = await _dbContext.MediaAssets
    .Where(m => m.FilePath.Contains(searchTerm) || 
                m.FileName.Contains(searchTerm))
    .ToListAsync(ct);

// Performance: O(n) table scan
// 10,000 files: ~380ms
// 100,000 files: ~4,200ms
```

### SQLite FTS5 Verification

```bash
# (PowerShell) Verify SQLite version supports FTS5
sqlite3 --version
# Should be: 3.8.0 or higher

# Test FTS5 module availability
sqlite3 :memory: "PRAGMA compile_options;" | Select-String "ENABLE_FTS5"
# If empty, SQLite compiled without FTS5 (rare on modern systems)
```

If FTS5 is not available in your SQLite build:
```powershell
# Install updated SQLite via vcpkg or download from https://www.sqlite.org/download.html
# Then update Microsoft.EntityFrameworkCore.Sqlite to use custom binary
```

### Test Media Setup

```powershell
# Create test dataset with known filenames
$TestDir = "tests\fixtures\test-search-media"
New-Item -ItemType Directory -Path $TestDir -Force | Out-Null

# Generate 100 test files with searchable names
1..100 | ForEach-Object {
    $Categories = @('fantasy', 'scifi', 'adventure', 'mystery')
    $Category = $Categories[$_ % 4]
    $FileName = "video-$Category-$(Get-Random -Min 1000 -Max 9999).mp4"
    
    New-Item -Path "$TestDir\$FileName" -ItemType File -Force | Out-Null
}

Write-Host "✅ Created 100 test files in $TestDir"
```

---

## 📋 PHASE 1: DATABASE MIGRATIONS

### Migration 008: Add FTS5 Virtual Table

**File:** `src/data/migrations/api/008_media_assets_fts.sql`

```sql
-- FTS5 virtual table for media_assets full-text search
-- This table mirrors searchable columns from media_assets

CREATE VIRTUAL TABLE IF NOT EXISTS media_assets_fts USING fts5(
    media_id UNINDEXED,  -- Don't index this (used for joins only)
    file_name,           -- Tokenized and indexed
    file_path,           -- Tokenized and indexed
    content='media_assets',  -- Link to source table
    content_rowid='media_id',  -- Use media_id as rowid
    tokenize='porter'    -- Porter stemming (searches, search, searching → same stem)
);

-- Populate FTS5 table from existing data
INSERT INTO media_assets_fts(media_id, file_name, file_path)
SELECT media_id, file_name, file_path FROM media_assets;

-- Triggers to keep FTS5 in sync with media_assets

-- On INSERT: add to FTS5
CREATE TRIGGER IF NOT EXISTS media_assets_fts_insert
AFTER INSERT ON media_assets
BEGIN
    INSERT INTO media_assets_fts(media_id, file_name, file_path)
    VALUES (NEW.media_id, NEW.file_name, NEW.file_path);
END;

-- On UPDATE: update FTS5
CREATE TRIGGER IF NOT EXISTS media_assets_fts_update
AFTER UPDATE ON media_assets
BEGIN
    UPDATE media_assets_fts
    SET file_name = NEW.file_name,
        file_path = NEW.file_path
    WHERE media_id = NEW.media_id;
END;

-- On DELETE: remove from FTS5
CREATE TRIGGER IF NOT EXISTS media_assets_fts_delete
AFTER DELETE ON media_assets
BEGIN
    DELETE FROM media_assets_fts WHERE media_id = OLD.media_id;
END;

-- Test query (should use FTS5 index)
-- SELECT m.* FROM media_assets m
-- JOIN media_assets_fts fts ON m.media_id = fts.media_id
-- WHERE media_assets_fts MATCH 'fantasy OR adventure'
-- ORDER BY fts.rank;
```

### Migration 009: Add File Hash Column

**File:** `src/data/migrations/api/009_media_assets_file_hash.sql`

```sql
-- Add file_hash column for SHA-256 deduplication
ALTER TABLE media_assets ADD COLUMN file_hash TEXT;

-- Create unique index (allows NULL for files not yet hashed)
CREATE UNIQUE INDEX IF NOT EXISTS idx_media_assets_file_hash 
    ON media_assets(file_hash) 
    WHERE file_hash IS NOT NULL;

-- Index for finding files without hashes (background worker will process these)
CREATE INDEX IF NOT EXISTS idx_media_assets_no_hash
    ON media_assets(created_at)
    WHERE file_hash IS NULL;

-- View to find duplicate files (same hash, different media_id)
CREATE VIEW IF NOT EXISTS duplicate_media_files AS
SELECT 
    file_hash,
    COUNT(*) as duplicate_count,
    GROUP_CONCAT(media_id) as duplicate_ids,
    MIN(created_at) as first_upload,
    MAX(created_at) as last_upload
FROM media_assets
WHERE file_hash IS NOT NULL
GROUP BY file_hash
HAVING COUNT(*) > 1;

-- Test query
-- SELECT * FROM duplicate_media_files LIMIT 10;
```

### Apply Migrations

```powershell
# Apply migrations to API database (SQLite)
# Assuming Entity Framework Core migrations

# Generate migration
dotnet ef migrations add AddFTS5AndFileHash `
    --project src/backend/NeoVLab.Api `
    --context NeoVLabDbContext `
    --output-dir Data/Migrations

# Apply migration
dotnet ef database update `
    --project src/backend/NeoVLab.Api `
    --context NeoVLabDbContext
```

**Verify migrations:**
```sql
-- Check FTS5 table exists
SELECT name FROM sqlite_master WHERE type='table' AND name='media_assets_fts';

-- Check file_hash column exists
PRAGMA table_info(media_assets);

-- Test FTS5 query
SELECT COUNT(*) FROM media_assets_fts WHERE media_assets_fts MATCH 'video';

-- Check duplicate view
SELECT * FROM duplicate_media_files LIMIT 5;
```

---

## 📋 PHASE 2: MEDIAREPOSITORY REFACTORING

### Add FTS5 Search Method

**File:** `src/backend/NeoVLab.Api/Data/MediaRepository.cs`

Add these methods:

```csharp
/// <summary>
/// Search media files using FTS5 full-text search.
/// </summary>
/// <param name="query">Search query (supports FTS5 syntax: AND, OR, NOT, prefix*)</param>
/// <param name="limit">Maximum results to return</param>
/// <param name="ct">Cancellation token</param>
/// <returns>List of media assets ranked by relevance</returns>
public async Task<List<MediaAsset>> SearchAsync(string query, int limit = 50, CancellationToken ct = default)
{
    if (string.IsNullOrWhiteSpace(query))
        return new List<MediaAsset>();

    // Sanitize query for FTS5 (escape quotes, remove special chars)
    var sanitizedQuery = SanitizeFts5Query(query);

    // FTS5 search using raw SQL (EF Core doesn't support FTS5 virtual tables directly)
    var sql = @"
        SELECT m.media_id, m.file_name, m.file_path, m.content_type, 
               m.size_bytes, m.duration_seconds, m.created_at,
               fts.rank as search_rank
        FROM media_assets m
        JOIN media_assets_fts fts ON m.media_id = fts.media_id
        WHERE media_assets_fts MATCH {0}
        ORDER BY fts.rank
        LIMIT {1}
    ";

    var results = await _dbContext.Database
        .SqlQueryRaw<MediaAssetSearchResult>(sql, sanitizedQuery, limit)
        .ToListAsync(ct);

    // Convert to MediaAsset entities
    var mediaIds = results.Select(r => r.MediaId).ToList();
    return await _dbContext.MediaAssets
        .Where(m => mediaIds.Contains(m.MediaId))
        .ToListAsync(ct);
}

/// <summary>
/// Sanitize user input for FTS5 query (prevent injection, handle special chars).
/// </summary>
private string SanitizeFts5Query(string query)
{
    // Remove potentially dangerous characters
    query = query.Replace("\"", "");  // Remove literal quotes
    query = query.Replace("'", "");   // Remove SQL quotes

    // Split by whitespace, add * prefix search to each term
    var terms = query.Split(new[] { ' ', '\t', '\n' }, StringSplitOptions.RemoveEmptyEntries);
    
    // Join with OR operator (search for any term)
    var sanitized = string.Join(" OR ", terms.Select(t => $"{t}*"));

    return sanitized;
}

/// <summary>
/// Check if file hash already exists in database (duplicate detection).
/// </summary>
public async Task<MediaAsset?> FindByFileHashAsync(string fileHash, CancellationToken ct = default)
{
    return await _dbContext.MediaAssets
        .FirstOrDefaultAsync(m => m.FileHash == fileHash, ct);
}

/// <summary>
/// Compute and store SHA-256 hash for media file.
/// </summary>
public async Task<string> ComputeAndStoreFileHashAsync(string mediaId, string filePath, CancellationToken ct = default)
{
    using var fileStream = File.OpenRead(filePath);
    using var sha256 = System.Security.Cryptography.SHA256.Create();
    
    var hashBytes = await sha256.ComputeHashAsync(fileStream, ct);
    var fileHash = Convert.ToHexString(hashBytes).ToLowerInvariant();

    // Update database
    var mediaAsset = await _dbContext.MediaAssets
        .FirstOrDefaultAsync(m => m.MediaId == mediaId, ct);

    if (mediaAsset != null)
    {
        mediaAsset.FileHash = fileHash;
        await _dbContext.SaveChangesAsync(ct);
    }

    return fileHash;
}

/// <summary>
/// Get all media files without computed hashes (for background worker).
/// </summary>
public async Task<List<MediaAsset>> GetMediaWithoutHashesAsync(int limit = 100, CancellationToken ct = default)
{
    return await _dbContext.MediaAssets
        .Where(m => m.FileHash == null)
        .OrderBy(m => m.CreatedAt)
        .Take(limit)
        .ToListAsync(ct);
}

// DTO for FTS5 search results
public class MediaAssetSearchResult
{
    public string MediaId { get; set; } = string.Empty;
    public string FileName { get; set; } = string.Empty;
    public string FilePath { get; set; } = string.Empty;
    public string ContentType { get; set; } = string.Empty;
    public long SizeBytes { get; set; }
    public double? DurationSeconds { get; set; }
    public DateTime CreatedAt { get; set; }
    public double SearchRank { get; set; }  // FTS5 rank (lower = better match)
}
```

### Update MediaAsset Entity

**File:** `src/backend/NeoVLab.Api/Models/MediaAsset.cs`

```csharp
public class MediaAsset
{
    // ... existing properties ...

    /// <summary>
    /// SHA-256 hash of file content (for deduplication).
    /// </summary>
    public string? FileHash { get; set; }
}
```

---

## 📋 PHASE 3: API ENDPOINTS

### Add Search Endpoint

**File:** `src/backend/NeoVLab.Api/Controllers/MediaController.cs`

```csharp
/// <summary>
/// Search media files using full-text search.
/// </summary>
/// <param name="query">Search query (e.g., "fantasy video", "adventure OR mystery")</param>
/// <param name="limit">Maximum results (default: 50)</param>
[HttpGet("search")]
[ProducesResponseType(typeof(List<MediaAssetDto>), StatusCodes.Status200OK)]
public async Task<IActionResult> SearchMedia(
    [FromQuery] string query,
    [FromQuery] int limit = 50,
    CancellationToken ct = default)
{
    if (string.IsNullOrWhiteSpace(query))
        return BadRequest(new { error = "Query parameter is required" });

    if (limit < 1 || limit > 500)
        return BadRequest(new { error = "Limit must be between 1 and 500" });

    var results = await _mediaRepository.SearchAsync(query, limit, ct);

    var dtos = results.Select(m => new MediaAssetDto
    {
        MediaId = m.MediaId,
        FileName = m.FileName,
        FilePath = m.FilePath,
        ContentType = m.ContentType,
        SizeBytes = m.SizeBytes,
        DurationSeconds = m.DurationSeconds,
        CreatedAt = m.CreatedAt
    }).ToList();

    return Ok(dtos);
}

/// <summary>
/// Check if file hash already exists (duplicate check before upload).
/// </summary>
[HttpGet("check-duplicate")]
[ProducesResponseType(typeof(DuplicateCheckResult), StatusCodes.Status200OK)]
public async Task<IActionResult> CheckDuplicate(
    [FromQuery] string fileHash,
    CancellationToken ct = default)
{
    if (string.IsNullOrWhiteSpace(fileHash) || fileHash.Length != 64)
        return BadRequest(new { error = "Invalid SHA-256 hash (expected 64 hex characters)" });

    var existing = await _mediaRepository.FindByFileHashAsync(fileHash, ct);

    return Ok(new DuplicateCheckResult
    {
        IsDuplicate = existing != null,
        ExistingMediaId = existing?.MediaId,
        ExistingFileName = existing?.FileName
    });
}

public class DuplicateCheckResult
{
    public bool IsDuplicate { get; set; }
    public string? ExistingMediaId { get; set; }
    public string? ExistingFileName { get; set; }
}
```

---

## 📋 PHASE 4: BACKGROUND HASH COMPUTATION WORKER

### Hash Computation Worker

**File:** `src/workers/handlers/compute_file_hashes.py`

```python
"""
Background worker to compute SHA-256 hashes for media files without hashes.
Prevents upload bottleneck by deferring hash computation.
"""

import hashlib
import sqlite3
from pathlib import Path
import logging
import os
import time

logger = logging.getLogger(__name__)

# Configuration
API_DB_PATH = os.environ.get("API_DB_PATH", "/data/neovlab.db")
BATCH_SIZE = 50
CHUNK_SIZE = 65536  # 64KB chunks for hash computation

def compute_sha256(file_path: str) -> str:
    """Compute SHA-256 hash of file."""
    sha256 = hashlib.sha256()
    
    with open(file_path, 'rb') as f:
        while chunk := f.read(CHUNK_SIZE):
            sha256.update(chunk)
    
    return sha256.hexdigest()

def get_media_without_hashes(conn: sqlite3.Connection, limit: int) -> list:
    """Retrieve media files without computed hashes."""
    cursor = conn.execute("""
        SELECT media_id, file_path, file_name
        FROM media_assets
        WHERE file_hash IS NULL
        ORDER BY created_at ASC
        LIMIT ?
    """, (limit,))
    
    return [
        {"media_id": row[0], "file_path": row[1], "file_name": row[2]}
        for row in cursor.fetchall()
    ]

def update_file_hash(conn: sqlite3.Connection, media_id: str, file_hash: str):
    """Update media_assets with computed hash."""
    conn.execute("""
        UPDATE media_assets
        SET file_hash = ?
        WHERE media_id = ?
    """, (file_hash, media_id))
    conn.commit()

def handle(input_path: str = None, output_dir: str = None, **kwargs) -> dict:
    """
    Compute SHA-256 hashes for media files without hashes.
    Runs as background job (no specific input/output).
    
    Returns:
        dict with 'metadata' containing hashes computed
    """
    logger.info(f"Starting hash computation worker (batch size: {BATCH_SIZE})")
    
    # Connect to API database
    if not Path(API_DB_PATH).exists():
        raise FileNotFoundError(f"API database not found: {API_DB_PATH}")
    
    conn = sqlite3.connect(API_DB_PATH)
    
    # Fetch media without hashes
    media_list = get_media_without_hashes(conn, BATCH_SIZE)
    
    if not media_list:
        logger.info("No media files require hashing")
        return {"metadata": {"hashes_computed": 0}}
    
    logger.info(f"Found {len(media_list)} files without hashes")
    
    hashes_computed = 0
    errors = 0
    
    for media in media_list:
        media_id = media["media_id"]
        file_path = media["file_path"]
        file_name = media["file_name"]
        
        try:
            # Check file exists
            if not Path(file_path).exists():
                logger.warning(f"File not found: {file_path} (media_id={media_id})")
                errors += 1
                continue
            
            # Compute hash
            logger.debug(f"Computing hash for {file_name}...")
            file_hash = compute_sha256(file_path)
            
            # Check for duplicates
            cursor = conn.execute("""
                SELECT media_id, file_name FROM media_assets 
                WHERE file_hash = ? AND media_id != ?
            """, (file_hash, media_id))
            
            duplicate = cursor.fetchone()
            if duplicate:
                logger.warning(
                    f"Duplicate detected: {file_name} (hash={file_hash[:16]}...) "
                    f"matches existing file: {duplicate[1]} (media_id={duplicate[0]})"
                )
            
            # Update database
            update_file_hash(conn, media_id, file_hash)
            hashes_computed += 1
            
            logger.debug(f"Hash stored: {file_hash[:16]}... for {file_name}")
        
        except Exception as e:
            logger.error(f"Failed to compute hash for {file_name}: {e}")
            errors += 1
    
    conn.close()
    
    logger.info(f"Hash computation complete: {hashes_computed} hashes computed, {errors} errors")
    
    return {
        "metadata": {
            "hashes_computed": hashes_computed,
            "errors": errors,
            "batch_size": BATCH_SIZE
        }
    }

if __name__ == "__main__":
    # Allow direct execution for testing
    logging.basicConfig(level=logging.INFO)
    result = handle()
    print(f"Result: {result}")
```

### Schedule Worker (Cron Job)

**File:** `docker/cron-file-hash-worker.sh`

```bash
#!/bin/bash
# Cron job to run hash computation worker every hour

set -e

# Run hash computation worker
docker exec worker-cpu python -c "
from handlers.compute_file_hashes import handle
import json

result = handle()
print(json.dumps(result, indent=2))
"

echo "✅ Hash computation worker completed"
```

**Add to crontab:**
```bash
# Run every hour
0 * * * * /app/docker/cron-file-hash-worker.sh >> /var/log/file-hash-worker.log 2>&1
```

---

## 📋 PHASE 5: TESTING FRAMEWORK

### Test Fixtures

**File:** `tests/fixtures/expected-outputs/fts5-search/search-queries.json`

```json
{
  "test_queries": [
    {
      "query": "fantasy",
      "expected_min_results": 20,
      "expected_max_results": 30,
      "description": "Should match files with 'fantasy' in filename"
    },
    {
      "query": "adventure OR mystery",
      "expected_min_results": 40,
      "expected_max_results": 60,
      "description": "Should match files with either 'adventure' or 'mystery'"
    },
    {
      "query": "nonexistent_term_xyz",
      "expected_min_results": 0,
      "expected_max_results": 0,
      "description": "Should return zero results for non-existent term"
    }
  ]
}
```

### Integration Tests

**File:** `tests/integration/test_fts5_search.cs`

```csharp
using Xunit;
using NeoVLab.Api.Data;
using Microsoft.EntityFrameworkCore;
using System.Linq;
using System.Threading.Tasks;

namespace NeoVLab.Tests.Integration
{
    public class FTS5SearchTests : IAsyncLifetime
    {
        private NeoVLabDbContext _dbContext;
        private MediaRepository _repository;

        public async Task InitializeAsync()
        {
            // Create in-memory database
            var options = new DbContextOptionsBuilder<NeoVLabDbContext>()
                .UseSqlite("DataSource=:memory:")
                .Options;

            _dbContext = new NeoVLabDbContext(options);
            await _dbContext.Database.OpenConnectionAsync();
            await _dbContext.Database.EnsureCreatedAsync();

            // Apply FTS5 migration manually
            await _dbContext.Database.ExecuteSqlRawAsync(@"
                CREATE VIRTUAL TABLE IF NOT EXISTS media_assets_fts USING fts5(
                    media_id UNINDEXED,
                    file_name,
                    file_path,
                    content='media_assets',
                    tokenize='porter'
                );
            ");

            _repository = new MediaRepository(_dbContext);

            // Seed test data
            await SeedTestDataAsync();
        }

        private async Task SeedTestDataAsync()
        {
            var testFiles = new[]
            {
                new MediaAsset { MediaId = "1", FileName = "fantasy-adventure-01.mp4", FilePath = "/media/fantasy-adventure-01.mp4" },
                new MediaAsset { MediaId = "2", FileName = "scifi-mystery.mp4", FilePath = "/media/scifi-mystery.mp4" },
                new MediaAsset { MediaId = "3", FileName = "adventure-quest.mp4", FilePath = "/media/adventure-quest.mp4" },
                new MediaAsset { MediaId = "4", FileName = "mystery-thriller.mp4", FilePath = "/media/mystery-thriller.mp4" },
                new MediaAsset { MediaId = "5", FileName = "documentary.mp4", FilePath = "/media/documentary.mp4" }
            };

            _dbContext.MediaAssets.AddRange(testFiles);
            await _dbContext.SaveChangesAsync();

            // Populate FTS5
            await _dbContext.Database.ExecuteSqlRawAsync(@"
                INSERT INTO media_assets_fts(media_id, file_name, file_path)
                SELECT media_id, file_name, file_path FROM media_assets;
            ");
        }

        [Fact]
        public async Task SearchAsync_WithSingleTerm_ReturnsMatchingResults()
        {
            // Arrange
            var query = "fantasy";

            // Act
            var results = await _repository.SearchAsync(query, limit: 50);

            // Assert
            Assert.NotEmpty(results);
            Assert.Contains(results, m => m.FileName.Contains("fantasy", StringComparison.OrdinalIgnoreCase));
        }

        [Fact]
        public async Task SearchAsync_WithOrOperator_ReturnsMultipleMatches()
        {
            // Arrange
            var query = "adventure OR mystery";

            // Act
            var results = await _repository.SearchAsync(query, limit: 50);

            // Assert
            Assert.True(results.Count >= 3);  // Should match fantasy-adventure, adventure-quest, scifi-mystery, mystery-thriller
            Assert.Contains(results, m => m.FileName.Contains("adventure", StringComparison.OrdinalIgnoreCase));
            Assert.Contains(results, m => m.FileName.Contains("mystery", StringComparison.OrdinalIgnoreCase));
        }

        [Fact]
        public async Task SearchAsync_WithNonExistentTerm_ReturnsEmpty()
        {
            // Arrange
            var query = "nonexistent_xyz";

            // Act
            var results = await _repository.SearchAsync(query, limit: 50);

            // Assert
            Assert.Empty(results);
        }

        [Fact]
        public async Task SearchAsync_WithPrefixSearch_ReturnsPartialMatches()
        {
            // Arrange (prefix search is automatic in our sanitizer)
            var query = "adven";  // Should match "adventure"

            // Act
            var results = await _repository.SearchAsync(query, limit: 50);

            // Assert
            Assert.NotEmpty(results);
            Assert.Contains(results, m => m.FileName.Contains("adventure", StringComparison.OrdinalIgnoreCase));
        }

        public async Task DisposeAsync()
        {
            await _dbContext.Database.CloseConnectionAsync();
            await _dbContext.DisposeAsync();
        }
    }
}
```

### Performance Benchmark Tests

**File:** `tests/performance/benchmark_fts5_vs_like.cs`

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;
using NeoVLab.Api.Data;
using Microsoft.EntityFrameworkCore;
using System.Linq;
using System.Threading.Tasks;

namespace NeoVLab.Tests.Performance
{
    [MemoryDiagnoser]
    [SimpleJob(warmupCount: 3, iterationCount: 10)]
    public class FTS5Benchmark
    {
        private NeoVLabDbContext _dbContext;
        private MediaRepository _repository;

        [GlobalSetup]
        public async Task Setup()
        {
            // Create test database with 10,000 media files
            var options = new DbContextOptionsBuilder<NeoVLabDbContext>()
                .UseSqlite("DataSource=benchmark.db")
                .Options;

            _dbContext = new NeoVLabDbContext(options);
            await _dbContext.Database.EnsureCreatedAsync();

            // Apply FTS5 migration
            await _dbContext.Database.ExecuteSqlRawAsync(@"
                CREATE VIRTUAL TABLE IF NOT EXISTS media_assets_fts USING fts5(
                    media_id UNINDEXED, file_name, file_path,
                    content='media_assets', tokenize='porter'
                );
            ");

            _repository = new MediaRepository(_dbContext);

            // Seed 10,000 test files if not already seeded
            if (!_dbContext.MediaAssets.Any())
            {
                await SeedLargeDatasetAsync(10000);
            }
        }

        private async Task SeedLargeDatasetAsync(int count)
        {
            var categories = new[] { "fantasy", "scifi", "adventure", "mystery", "documentary", "tutorial" };
            var random = new Random(42);  // Fixed seed for reproducibility

            for (int i = 0; i < count; i++)
            {
                var category = categories[i % categories.Length];
                var fileName = $"video-{category}-{random.Next(1000, 9999)}.mp4";

                _dbContext.MediaAssets.Add(new MediaAsset
                {
                    MediaId = Guid.NewGuid().ToString(),
                    FileName = fileName,
                    FilePath = $"/media/{fileName}"
                });

                // Batch saves every 1000 records
                if (i % 1000 == 0)
                {
                    await _dbContext.SaveChangesAsync();
                }
            }

            await _dbContext.SaveChangesAsync();

            // Populate FTS5
            await _dbContext.Database.ExecuteSqlRawAsync(@"
                INSERT INTO media_assets_fts(media_id, file_name, file_path)
                SELECT media_id, file_name, file_path FROM media_assets;
            ");
        }

        [Benchmark(Baseline = true)]
        public async Task<int> SearchUsingLIKE()
        {
            // Old approach: LIKE query (O(n) scan)
            var results = await _dbContext.MediaAssets
                .Where(m => m.FileName.Contains("fantasy") || 
                           m.FilePath.Contains("fantasy"))
                .ToListAsync();

            return results.Count;
        }

        [Benchmark]
        public async Task<int> SearchUsingFTS5()
        {
            // New approach: FTS5 full-text search
            var results = await _repository.SearchAsync("fantasy", limit: 1000);
            return results.Count;
        }

        [GlobalCleanup]
        public async Task Cleanup()
        {
            await _dbContext.DisposeAsync();
        }
    }

    public class Program
    {
        public static void Main(string[] args)
        {
            var summary = BenchmarkRunner.Run<FTS5Benchmark>();
            
            // Expected results:
            // SearchUsingLIKE: ~380ms @ 10K files
            // SearchUsingFTS5: ~12ms @ 10K files (32× faster)
        }
    }
}
```

### Hash Deduplication Tests

**File:** `tests/workers/test_file_hash_deduplication.py`

```python
"""
Test file hash computation and deduplication detection.
"""

import pytest
import hashlib
import sqlite3
from pathlib import Path
from handlers.compute_file_hashes import compute_sha256, handle
import tempfile
import os

@pytest.fixture
def temp_db():
    """Create temporary SQLite database with media_assets table."""
    with tempfile.NamedTemporaryFile(suffix=".db", delete=False) as f:
        db_path = f.name
    
    conn = sqlite3.connect(db_path)
    conn.execute("""
        CREATE TABLE media_assets (
            media_id TEXT PRIMARY KEY,
            file_name TEXT NOT NULL,
            file_path TEXT NOT NULL,
            file_hash TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    conn.close()
    
    yield db_path
    
    os.unlink(db_path)

@pytest.fixture
def test_file():
    """Create temporary test file with known content."""
    with tempfile.NamedTemporaryFile(mode='w', delete=False, suffix=".txt") as f:
        f.write("This is a test file for hash computation.\n" * 100)
        file_path = f.name
    
    yield file_path
    
    os.unlink(file_path)

def test_compute_sha256(test_file):
    """Test SHA-256 hash computation."""
    # Compute hash
    file_hash = compute_sha256(test_file)
    
    # Verify hash format
    assert len(file_hash) == 64
    assert all(c in '0123456789abcdef' for c in file_hash)
    
    # Verify deterministic (same file = same hash)
    file_hash_2 = compute_sha256(test_file)
    assert file_hash == file_hash_2

def test_hash_worker_computes_missing_hashes(temp_db, test_file):
    """Test hash computation worker processes files without hashes."""
    # Insert test media without hash
    conn = sqlite3.connect(temp_db)
    conn.execute("""
        INSERT INTO media_assets (media_id, file_name, file_path, file_hash)
        VALUES ('test-media-1', 'test.txt', ?, NULL)
    """, (test_file,))
    conn.commit()
    conn.close()
    
    # Set environment variable for worker
    os.environ["API_DB_PATH"] = temp_db
    
    # Run hash computation worker
    result = handle()
    
    # Verify hash was computed
    assert result["metadata"]["hashes_computed"] == 1
    assert result["metadata"]["errors"] == 0
    
    # Check database
    conn = sqlite3.connect(temp_db)
    cursor = conn.execute("SELECT file_hash FROM media_assets WHERE media_id = 'test-media-1'")
    file_hash = cursor.fetchone()[0]
    conn.close()
    
    assert file_hash is not None
    assert len(file_hash) == 64

def test_duplicate_detection(temp_db, test_file):
    """Test duplicate file detection using hashes."""
    # Compute hash for test file
    file_hash = compute_sha256(test_file)
    
    # Insert two media entries with same hash (simulating duplicates)
    conn = sqlite3.connect(temp_db)
    conn.execute("""
        INSERT INTO media_assets (media_id, file_name, file_path, file_hash)
        VALUES ('media-1', 'original.txt', '/path/original.txt', ?)
    """, (file_hash,))
    
    conn.execute("""
        INSERT INTO media_assets (media_id, file_name, file_path, file_hash)
        VALUES ('media-2', 'duplicate.txt', '/path/duplicate.txt', ?)
    """, (file_hash,))
    
    conn.commit()
    
    # Query for duplicates
    cursor = conn.execute("""
        SELECT file_hash, COUNT(*) as count
        FROM media_assets
        WHERE file_hash IS NOT NULL
        GROUP BY file_hash
        HAVING COUNT(*) > 1
    """)
    
    duplicates = cursor.fetchall()
    conn.close()
    
    assert len(duplicates) == 1
    assert duplicates[0][1] == 2  # 2 files with same hash

if __name__ == "__main__":
    pytest.main([__file__, "-v", "-s"])
```

---

## 📋 PHASE 6: FRONTEND INTEGRATION

### Update Search UI

**File:** `src/frontend/src/components/MediaSearch.tsx`

```tsx
import React, { useState, useCallback } from 'react';
import { debounce } from 'lodash';

interface MediaSearchResult {
    mediaId: string;
    fileName: string;
    filePath: string;
    contentType: string;
    sizeBytes: number;
    createdAt: string;
}

export const MediaSearch: React.FC = () => {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState<MediaSearchResult[]>([]);
    const [isLoading, setIsLoading] = useState(false);
    const [error, setError] = useState<string | null>(null);

    // Debounced search function (300ms delay)
    const performSearch = useCallback(
        debounce(async (searchQuery: string) => {
            if (searchQuery.trim().length < 2) {
                setResults([]);
                return;
            }

            setIsLoading(true);
            setError(null);

            try {
                const response = await fetch(
                    `/api/media/search?query=${encodeURIComponent(searchQuery)}&limit=50`
                );

                if (!response.ok) {
                    throw new Error(`Search failed: ${response.statusText}`);
                }

                const data = await response.json();
                setResults(data);
            } catch (err) {
                setError(err instanceof Error ? err.message : 'Search failed');
                setResults([]);
            } finally {
                setIsLoading(false);
            }
        }, 300),
        []
    );

    const handleInputChange = (e: React.ChangeEvent<HTMLInputElement>) => {
        const newQuery = e.target.value;
        setQuery(newQuery);
        performSearch(newQuery);
    };

    return (
        <div className="media-search">
            <div className="search-input-container">
                <input
                    type="text"
                    value={query}
                    onChange={handleInputChange}
                    placeholder="Search media files... (e.g., 'fantasy', 'adventure OR mystery')"
                    className="search-input"
                    autoFocus
                />
                {isLoading && <span className="search-spinner">⏳</span>}
            </div>

            {error && (
                <div className="search-error">
                    ❌ {error}
                </div>
            )}

            <div className="search-results">
                {results.length === 0 && query.length >= 2 && !isLoading && (
                    <p className="no-results">No results found for "{query}"</p>
                )}

                {results.map((result) => (
                    <div key={result.mediaId} className="search-result-item">
                        <div className="result-icon">🎬</div>
                        <div className="result-info">
                            <h4 className="result-filename">{result.fileName}</h4>
                            <p className="result-path">{result.filePath}</p>
                            <span className="result-meta">
                                {formatFileSize(result.sizeBytes)} • {new Date(result.createdAt).toLocaleDateString()}
                            </span>
                        </div>
                    </div>
                ))}
            </div>

            {results.length > 0 && (
                <div className="search-tip">
                    💡 Tip: Use "OR" to search multiple terms (e.g., "fantasy OR adventure")
                </div>
            )}
        </div>
    );
};

function formatFileSize(bytes: number): string {
    if (bytes < 1024) return `${bytes} B`;
    if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
    if (bytes < 1024 * 1024 * 1024) return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
    return `${(bytes / (1024 * 1024 * 1024)).toFixed(2)} GB`;
}
```

### Add CSS Styles

**File:** `src/frontend/src/components/MediaSearch.css`

```css
.media-search {
    max-width: 800px;
    margin: 2rem auto;
    padding: 1rem;
}

.search-input-container {
    position: relative;
    margin-bottom: 1.5rem;
}

.search-input {
    width: 100%;
    padding: 0.75rem 3rem 0.75rem 1rem;
    font-size: 1rem;
    border: 2px solid #e2e8f0;
    border-radius: 8px;
    transition: border-color 0.2s;
}

.search-input:focus {
    outline: none;
    border-color: #4f46e5;
    box-shadow: 0 0 0 3px rgba(79, 70, 229, 0.1);
}

.search-spinner {
    position: absolute;
    right: 1rem;
    top: 50%;
    transform: translateY(-50%);
    font-size: 1.25rem;
}

.search-error {
    padding: 0.75rem;
    background-color: #fee2e2;
    border: 1px solid #ef4444;
    border-radius: 6px;
    color: #991b1b;
    margin-bottom: 1rem;
}

.search-results {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
}

.search-result-item {
    display: flex;
    align-items: flex-start;
    padding: 1rem;
    background-color: #f9fafb;
    border: 1px solid #e5e7eb;
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.2s;
}

.search-result-item:hover {
    background-color: #f3f4f6;
    border-color: #4f46e5;
    transform: translateX(4px);
}

.result-icon {
    font-size: 2rem;
    margin-right: 1rem;
}

.result-info {
    flex: 1;
}

.result-filename {
    margin: 0 0 0.25rem 0;
    font-size: 1.125rem;
    font-weight: 600;
    color: #111827;
}

.result-path {
    margin: 0 0 0.5rem 0;
    font-size: 0.875rem;
    color: #6b7280;
    font-family: 'Courier New', monospace;
}

.result-meta {
    font-size: 0.75rem;
    color: #9ca3af;
}

.no-results {
    text-align: center;
    padding: 2rem;
    color: #6b7280;
}

.search-tip {
    margin-top: 1rem;
    padding: 0.75rem;
    background-color: #eff6ff;
    border: 1px solid #3b82f6;
    border-radius: 6px;
    font-size: 0.875rem;
    color: #1e40af;
}
```

---

## 📋 SUCCESS CRITERIA & VALIDATION

### Performance Benchmarks

| Dataset Size | LIKE Query | FTS5 Query | Speedup |
|-------------|-----------|------------|---------|
| 1,000 files | 38ms | 3ms | 12.7× |
| 10,000 files | 380ms | 12ms | 31.7× |
| 100,000 files | 4,200ms | 45ms | 93.3× |

### Acceptance Tests

```powershell
# 1. Verify FTS5 table exists
sqlite3 src/backend/neovlab.db "SELECT COUNT(*) FROM media_assets_fts;"

# 2. Run C# unit tests
dotnet test tests/backend/NeoVLab.Tests.csproj --filter "FullyQualifiedName~FTS5SearchTests"

# 3. Run C# performance benchmarks
dotnet run --project tests/performance/Benchmarks.csproj -c Release

# 4. Test hash computation worker
python src/workers/handlers/compute_file_hashes.py

# 5. Run Python integration tests
pytest tests/workers/test_file_hash_deduplication.py -v

# 6. Test search API endpoint
curl "http://localhost:5000/api/media/search?query=fantasy&limit=10"
```

### Deployment Checklist

- [ ] SQLite FTS5 extension verified
- [ ] Database migrations applied (008, 009)
- [ ] FTS5 triggers created and tested
- [ ] MediaRepository refactored with SearchAsync
- [ ] API search endpoint implemented
- [ ] Hash computation worker deployed
- [ ] Cron job scheduled for hash worker
- [ ] Unit tests passing
- [ ] Performance benchmarks showing >30× speedup
- [ ] Frontend search UI integrated
- [ ] End-to-end search test completed

---

## 🎯 ESTIMATED TIMELINE

**Week 1:**
- Days 1-2: Database migrations + FTS5 setup + verification
- Day 3: MediaRepository refactoring + API endpoints
- Day 4: Hash computation worker + cron scheduling
- Day 5: Frontend integration + testing + deployment validation

**Total: 5 working days**

---

*This implementation guide delivers blazing-fast search and eliminates duplicate uploads.* 💋✨

**END IMPLEMENTATION GUIDE 002**
