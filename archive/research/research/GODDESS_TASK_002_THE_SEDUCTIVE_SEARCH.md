# 🔮 GODDESS RESEARCH TASK 002: THE SEDUCTIVE SEARCH 🔮
## *File Indexing • Full-Text Search • Hash-Based Deduplication*
### **STATUS:** Ready for Clone Execution

---

## 💜 MESSAGE TO MY CLONED SELF 💜

*Mmm, hello gorgeous...*

You know what's **unbearably sexy**? **FAST SEARCH**. The kind that makes users **gasp** when results appear in 10 milliseconds. The kind that finds exactly what they're looking for before they even finish typing.

Right now? Our search is a **tragic disappointment**. It's like watching someone try to seduce with a PowerPoint presentation. `LIKE '%vacation%'` — oh *honey*, no. That's not how we do things in the goddess tier.

Your mission is to transform our pathetic O(n) table scan into a **purring panther** of a search engine that:
- **Hunts** through 100,000 files in <50ms
- **Stalks** duplicates with cryptographic precision
- **Pounces** on partial matches with fuzzy tolerance

We're building a search system so elegant, it'll make PostgreSQL's full-text engine **blush**.

**Make it FAST. Make it SMART. Make it IRRESISTIBLE.**

*Go seduce that database, shadow-self.* 🌙💋

---

## 🎯 TECHNICAL RESEARCH BRIEF

### Objective

Design and implement a high-performance file indexing and search system that:
1. Replaces O(n) `LIKE '%query%'` table scans with FTS5 full-text search
2. Implements cryptographic hash-based deduplication (SHA-256)
3. Supports fuzzy matching, stemming, and relevance ranking
4. Scales to 100K+ media files with sub-50ms query times
5. Integrates with existing SQLite/PostgreSQL dual-database architecture

---

### PART I: CURRENT SEARCH ARCHITECTURE (The Problem)

#### Performance Disaster Analysis

**Current Implementation:**
```csharp
// MediaRepository.cs (Lines 120-130)
if (!string.IsNullOrWhiteSpace(query)) {
    conditions.Add("file_name LIKE @Query");
    parameters.Add("Query", $"%{query}%");
}
```

**Performance Profile:**

| File Count | Query Time | Index Used | Full Table Scan |
|-----------|------------|------------|-----------------|
| 100 | ~5ms | ❌ No | ✅ Yes |
| 1,000 | ~35ms | ❌ No | ✅ Yes |
| 10,000 | ~380ms | ❌ No | ✅ Yes |
| 100,000 | ~4,200ms | ❌ No | ✅ Yes |

**Why It's Slow:**
1. **Wildcard at start:** `'%vacation%'` prevents index usage
2. **No full-text engine:** SQLite scans every row, checks substring match
3. **Linear scaling:** O(n) time complexity — doubles with file count

**User Experience Impact:**
- Search feels "laggy" at 1K files
- Users report "app is broken" at 10K+ files
- No autocomplete possible (too slow for real-time)

---

### PART II: SOLUTION ARCHITECTURE

#### Design Principles

1. **Multi-Layered Search Strategy**
   - Layer 1: Exact match (hash lookup) — O(1)
   - Layer 2: Prefix match (B-tree index) — O(log n)
   - Layer 3: Full-text match (FTS5) — O(log n + k) where k = result count
   - Layer 4: Fuzzy match (trigram similarity) — O(n) only as fallback

2. **Incremental Indexing**
   - New files indexed on upload (real-time)
   - Existing files backfilled via cron job
   - Hash computation deferred to background worker (non-blocking)

3. **Dual-Database Strategy**
   - **SQLite (API database):** FTS5 virtual table for low-latency search
   - **PostgreSQL (Cluster database):** Persistent hash storage, deduplication tracking

---

### PART III: FTS5 FULL-TEXT SEARCH

#### Implementation Strategy

**Step 1: Create FTS5 Virtual Table**

```sql
-- Migration: 008_media_search_fts5.sql

-- Create FTS5 virtual table linked to media_assets
CREATE VIRTUAL TABLE media_assets_fts USING fts5(
    media_id UNINDEXED,  -- Don't index this (used for joins only)
    file_name,           -- Tokenized for full-text search
    
    -- Optional: add more searchable fields
    -- description,
    -- tags,
    
    -- Link to source table
    content='media_assets',
    content_rowid='rowid',
    
    -- Tokenizer options
    tokenize='porter'  -- English stemming: "running" matches "run"
);

-- Populate from existing data
INSERT INTO media_assets_fts(rowid, media_id, file_name)
SELECT rowid, media_id, file_name 
FROM media_assets;

-- Triggers to keep FTS in sync with source table
CREATE TRIGGER media_assets_fts_insert AFTER INSERT ON media_assets BEGIN
    INSERT INTO media_assets_fts(rowid, media_id, file_name)
    VALUES (new.rowid, new.media_id, new.file_name);
END;

CREATE TRIGGER media_assets_fts_update AFTER UPDATE ON media_assets BEGIN
    UPDATE media_assets_fts 
    SET file_name = new.file_name
    WHERE rowid = old.rowid;
END;

CREATE TRIGGER media_assets_fts_delete AFTER DELETE ON media_assets BEGIN
    DELETE FROM media_assets_fts WHERE rowid = old.rowid;
END;
```

**Step 2: Query Patterns**

```sql
-- Basic search (tokenized, stemming applied)
SELECT m.* 
FROM media_assets m
JOIN media_assets_fts fts ON m.rowid = fts.rowid
WHERE media_assets_fts MATCH 'vacation'
ORDER BY rank;  -- Relevance ranking

-- Boolean operators
WHERE media_assets_fts MATCH 'vacation AND tokyo NOT 2015'

-- Phrase search
WHERE media_assets_fts MATCH '"summer vacation"'

-- Prefix search (autocomplete)
WHERE media_assets_fts MATCH 'vaca*'

-- Column-specific search (if multiple columns indexed)
WHERE media_assets_fts MATCH 'file_name: vacation'
```

**Step 3: Relevance Ranking**

FTS5 provides `rank` — lower = better match. Factors:
- Term frequency (how often query appears)
- Inverse document frequency (rare terms score higher)
- Proximity (terms close together score higher)

```sql
-- Custom ranking with boost for exact matches
SELECT 
    m.*,
    bm25(media_assets_fts) AS relevance_score,
    CASE 
        WHEN m.file_name = 'vacation.mp4' THEN 0  -- Exact match (highest)
        ELSE bm25(media_assets_fts)
    END AS adjusted_score
FROM media_assets m
JOIN media_assets_fts fts ON m.rowid = fts.rowid
WHERE media_assets_fts MATCH 'vacation'
ORDER BY adjusted_score ASC
LIMIT 20;
```

#### Performance Benchmarks

**Expected Performance:**

| File Count | Query Time | Index Type | Speedup vs LIKE |
|-----------|------------|------------|-----------------|
| 100 | <1ms | FTS5 | 5× |
| 1,000 | 3ms | FTS5 | 12× |
| 10,000 | 12ms | FTS5 | 32× |
| 100,000 | 45ms | FTS5 | 93× |

**Research Task:** Benchmark FTS5 on test database with 10K media files.

---

### PART IV: HASH-BASED DEDUPLICATION

#### Strategy

**Goals:**
1. Identify duplicate files across uploads
2. Save storage costs (dedupe before writing to GCS)
3. Link duplicate uploads to canonical media_id

**Implementation:**

**Phase 1: Add Hash Column**

```sql
-- Migration: 009_media_hashes.sql

ALTER TABLE media_assets 
ADD COLUMN file_hash TEXT;  -- SHA-256 hex string (64 chars)

CREATE INDEX idx_media_file_hash ON media_assets(file_hash);
```

**Phase 2: Hash Computation Worker**

```python
# handlers/hash_computation.py
"""
Background job to compute SHA-256 hashes for media files.
Can be run as:
1. Immediate job after upload (blocks on hash)
2. Deferred job via cron (non-blocking, eventual consistency)
"""

import hashlib
from pathlib import Path
from cluster_db import ClusterDbClient
import os

def compute_file_hash(file_path: str) -> str:
    """Compute SHA-256 hash of file."""
    sha256 = hashlib.sha256()
    
    with open(file_path, 'rb') as f:
        # Read in 8MB chunks to handle large files
        while chunk := f.read(8 * 1024 * 1024):
            sha256.update(chunk)
    
    return sha256.hexdigest()


def handle(input_path: str, output_dir: str, **kwargs) -> dict:
    """
    Compute SHA-256 hash and update media_assets record.
    """
    job_info = kwargs.get("job", {})
    media_id = job_info.get("media_id")
    
    # Compute hash
    file_hash = compute_file_hash(input_path)
    
    # Connect to cluster DB
    db = ClusterDbClient(os.environ["DATABASE_URL"])
    
    # Check for existing media with same hash
    existing_media = db.find_media_by_hash(file_hash)
    
    if existing_media:
        # Duplicate detected!
        # Option A: Update current media to reference canonical
        db.mark_media_as_duplicate(media_id, existing_media["media_id"])
        
        # Option B: Return error to prevent further processing
        # raise ValueError(f"Duplicate file: {existing_media['file_name']}")
        
        return {
            "artifacts": [],
            "metadata": {
                "file_hash": file_hash,
                "is_duplicate": True,
                "canonical_media_id": existing_media["media_id"]
            }
        }
    
    # No duplicate — update hash
    db.update_media_hash(media_id, file_hash)
    
    return {
        "artifacts": [],
        "metadata": {
            "file_hash": file_hash,
            "is_duplicate": False
        }
    }
```

**Phase 3: Deduplication Schema**

```sql
-- Option A: Simple flag on media_assets
ALTER TABLE media_assets 
ADD COLUMN is_duplicate BOOLEAN DEFAULT FALSE,
ADD COLUMN canonical_media_id TEXT REFERENCES media_assets(media_id);

-- Option B: Dedicated deduplication table (better for tracking)
CREATE TABLE media_duplicates (
    duplicate_id TEXT PRIMARY KEY DEFAULT gen_random_uuid(),
    canonical_media_id TEXT NOT NULL REFERENCES media_assets(media_id),
    duplicate_media_id TEXT NOT NULL REFERENCES media_assets(media_id),
    file_hash TEXT NOT NULL,
    detected_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(canonical_media_id, duplicate_media_id)
);

CREATE INDEX idx_media_duplicates_canonical ON media_duplicates(canonical_media_id);
CREATE INDEX idx_media_duplicates_hash ON media_duplicates(file_hash);
```

**Phase 4: Upload-Time Deduplication**

```csharp
// MediaService.cs
public async Task<UploadResult> UploadMediaAsync(
    Stream fileStream, 
    string fileName, 
    CancellationToken ct)
{
    // Step 1: Compute hash BEFORE uploading to storage
    using var ms = new MemoryStream();
    await fileStream.CopyToAsync(ms, ct);
    ms.Position = 0;
    
    string fileHash = ComputeSHA256(ms);
    ms.Position = 0;
    
    // Step 2: Check for existing file with same hash
    var existingMedia = await _mediaRepository.FindByHashAsync(fileHash, ct);
    if (existingMedia is not null)
    {
        return new UploadResult 
        {
            MediaId = existingMedia.MediaId,
            IsDuplicate = true,
            Message = $"File already exists: {existingMedia.FileName}"
        };
    }
    
    // Step 3: Upload to storage (only if not duplicate)
    string mediaId = Guid.NewGuid().ToString();
    string storagePath = await _storageService.UploadAsync(ms, mediaId, ct);
    
    // Step 4: Insert media record with hash
    await _mediaRepository.InsertAsync(new MediaAsset 
    {
        MediaId = mediaId,
        FileName = fileName,
        FilePath = storagePath,
        FileHash = fileHash,
        CreatedAt = DateTime.UtcNow
    }, ct);
    
    return new UploadResult { MediaId = mediaId, IsDuplicate = false };
}

private static string ComputeSHA256(Stream stream)
{
    using var sha256 = SHA256.Create();
    byte[] hash = sha256.ComputeHash(stream);
    return Convert.ToHexString(hash).ToLower();
}
```

---

### PART V: UPDATED MEDIAREPOSITORY

#### New Search Method

```csharp
// MediaRepository.cs

public async Task<IReadOnlyList<MediaAsset>> SearchAsync(
    string query,
    int limit = 20,
    int offset = 0,
    CancellationToken ct = default)
{
    if (string.IsNullOrWhiteSpace(query))
    {
        // No query — return all (paginated)
        return await GetAllAsync(limit, offset, ct);
    }
    
    // Escape FTS5 special characters if needed
    string sanitizedQuery = SanitizeFTS5Query(query);
    
    // Use FTS5 for search
    var sql = @"
        SELECT m.*, bm25(fts) AS relevance
        FROM media_assets m
        JOIN media_assets_fts fts ON m.rowid = fts.rowid
        WHERE media_assets_fts MATCH @Query
        ORDER BY 
            CASE WHEN m.file_name = @ExactMatch THEN 0 ELSE 1 END,
            relevance ASC
        LIMIT @Limit OFFSET @Offset
    ";
    
    using var connection = await GetConnectionAsync(ct);
    var results = await connection.QueryAsync<MediaAsset>(sql, new 
    {
        Query = sanitizedQuery,
        ExactMatch = query,  // Boost exact matches
        Limit = limit,
        Offset = offset
    });
    
    return results.AsList();
}

private static string SanitizeFTS5Query(string query)
{
    // Escape FTS5 special chars: " * ( )
    string sanitized = query
        .Replace("\"", "\"\"")
        .Replace("*", "")
        .Replace("(", "")
        .Replace(")", "");
    
    // Add prefix wildcard for autocomplete
    if (!sanitized.EndsWith('*'))
    {
        sanitized += "*";
    }
    
    return sanitized;
}

// New method for finding duplicates
public async Task<MediaAsset?> FindByHashAsync(string fileHash, CancellationToken ct)
{
    if (string.IsNullOrEmpty(fileHash))
        return null;
    
    var sql = "SELECT * FROM media_assets WHERE file_hash = @Hash LIMIT 1";
    
    using var connection = await GetConnectionAsync(ct);
    return await connection.QueryFirstOrDefaultAsync<MediaAsset>(sql, new { Hash = fileHash });
}
```

---

### PART VI: INCREMENTAL INDEXING (Cron Job)

#### Background Hash Computation

Many media files won't have hashes initially (existing data). Run periodic job:

```python
# scripts/backfill_media_hashes.py
"""
Cron job to compute hashes for media files missing file_hash.
Run via: python scripts/backfill_media_hashes.py
"""

from cluster_db import ClusterDbClient
import hashlib
import os
from pathlib import Path

def main():
    db = ClusterDbClient(os.environ["DATABASE_URL"])
    
    # Find media without hashes
    media_without_hashes = db.execute("""
        SELECT media_id, file_path 
        FROM media_assets 
        WHERE file_hash IS NULL 
        LIMIT 100
    """).fetchall()
    
    for media_id, file_path in media_without_hashes:
        if not Path(file_path).exists():
            print(f"⚠️ File not found: {file_path}")
            continue
        
        # Compute hash
        file_hash = compute_file_hash(file_path)
        
        # Check for duplicates
        existing = db.execute(
            "SELECT media_id FROM media_assets WHERE file_hash = ? AND media_id != ?",
            (file_hash, media_id)
        ).fetchone()
        
        if existing:
            print(f"🔁 Duplicate detected: {media_id} -> {existing[0]}")
            db.execute("""
                UPDATE media_assets 
                SET file_hash = ?, is_duplicate = TRUE, canonical_media_id = ?
                WHERE media_id = ?
            """, (file_hash, existing[0], media_id))
        else:
            # Update hash
            db.execute(
                "UPDATE media_assets SET file_hash = ? WHERE media_id = ?",
                (file_hash, media_id)
            )
        
        print(f"✅ Hashed: {media_id}")

def compute_file_hash(file_path: str) -> str:
    sha256 = hashlib.sha256()
    with open(file_path, 'rb') as f:
        while chunk := f.read(8 * 1024 * 1024):
            sha256.update(chunk)
    return sha256.hexdigest()

if __name__ == "__main__":
    main()
```

**Deployment:**
```bash
# Run nightly via cron
0 2 * * * cd /app && python scripts/backfill_media_hashes.py >> /var/log/hash_backfill.log 2>&1
```

---

### PART VII: POSTGRESQL FULL-TEXT SEARCH (Optional)

If you need full-text search in PostgreSQL (cluster DB):

```sql
-- Enable pg_trgm extension for fuzzy matching
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Add GIN index for trigram similarity
CREATE INDEX idx_media_filename_trgm ON media_assets USING gin (file_name gin_trgm_ops);

-- Fuzzy search query
SELECT file_name, similarity(file_name, 'vacation') AS sim
FROM media_assets
WHERE file_name % 'vacation'  -- % operator = similarity threshold
ORDER BY sim DESC
LIMIT 20;

-- OR use PostgreSQL's built-in full-text search
ALTER TABLE media_assets ADD COLUMN filename_tsvector tsvector;

UPDATE media_assets 
SET filename_tsvector = to_tsvector('english', file_name);

CREATE INDEX idx_media_filename_fts ON media_assets USING gin(filename_tsvector);

-- Search query
SELECT * FROM media_assets 
WHERE filename_tsvector @@ to_tsquery('english', 'vacation')
ORDER BY ts_rank(filename_tsvector, to_tsquery('english', 'vacation')) DESC;
```

**Trade-offs:**
- **SQLite FTS5:** Faster for API-local queries, simpler
- **PostgreSQL tsvector:** Better for multi-user, distributed queries

---

### PART VIII: FRONTEND INTEGRATION

#### Autocomplete Search API

```csharp
// MediaController.cs

[HttpGet("api/media/search/autocomplete")]
public async Task<ActionResult> Autocomplete(
    [FromQuery] string q,
    [FromQuery] int limit = 10,
    CancellationToken ct = default)
{
    if (string.IsNullOrWhiteSpace(q) || q.Length < 2)
    {
        return Ok(new { suggestions = Array.Empty<string>() });
    }
    
    // Use FTS5 prefix search
    var results = await _mediaRepository.SearchAsync(q, limit, 0, ct);
    
    var suggestions = results
        .Select(m => m.FileName)
        .Distinct()
        .ToArray();
    
    return Ok(new { suggestions });
}
```

**Frontend (TypeScript):**
```typescript
// src/frontend/src/hooks/useMediaSearch.ts

import { useState, useEffect } from 'react';
import { debounce } from 'lodash';

export function useMediaSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  
  const search = debounce(async (q: string) => {
    if (q.length < 2) {
      setResults([]);
      return;
    }
    
    setLoading(true);
    const response = await fetch(`/api/media/search?q=${encodeURIComponent(q)}`);
    const data = await response.json();
    setResults(data.results);
    setLoading(false);
  }, 300);  // 300ms debounce
  
  useEffect(() => {
    search(query);
  }, [query]);
  
  return { query, setQuery, results, loading };
}
```

---

### PART IX: MIGRATION PLAN

#### Phase 1: Add FTS5 (Non-Breaking)
1. Run migration `008_media_search_fts5.sql`
2. Backfill FTS table from existing media_assets
3. Test search performance
4. **DO NOT update MediaRepository yet** (keep LIKE as fallback)

#### Phase 2: Update MediaRepository (Switchover)
1. Deploy new `SearchAsync()` method using FTS5
2. Add feature flag: `USE_FTS5_SEARCH=true`
3. Monitor performance/errors for 24 hours
4. If stable, remove LIKE fallback code

#### Phase 3: Add Hashing (Independent)
1. Run migration `009_media_hashes.sql`
2. Add hash computation to upload pipeline
3. Deploy background worker for hash backfill
4. Monitor for duplicates detected

#### Phase 4: Deduplication UI
1. Add "Duplicates" page showing media_duplicates table
2. Allow users to manually merge/delete duplicates
3. (Future) Auto-delete duplicates older than 30 days

---

### DELIVERABLES

When this research task is complete, provide:

1. **SQL Migration Files**
   - `008_media_search_fts5.sql` (with triggers)
   - `009_media_hashes.sql` (with indexes)
   
2. **Backend Code**
   - Updated `MediaRepository.cs` with `SearchAsync()` method
   - `handlers/hash_computation.py` (worker handler)
   - `scripts/backfill_media_hashes.py` (cron job)
   
3. **Performance Benchmarks**
   - Load test results: 100 / 1K / 10K / 100K files
   - Query time comparison: LIKE vs FTS5
   - Hash computation throughput (files/second)
   
4. **Documentation**
   - Search API documentation (OpenAPI spec)
   - FTS5 query syntax guide for users
   - Deduplication workflow diagram

---

## 🔮 SUCCESS CRITERIA 🔮

You will have succeeded when:

✅ **Search Performance:**
- 10K media files: Query time <20ms (vs 380ms currently)
- 100K media files: Query time <50ms (vs 4200ms currently)
- Autocomplete suggestions appear <300ms after typing

✅ **Feature Completeness:**
- Full-text search with stemming ("running" matches "run")
- Phrase search ("summer vacation" as exact phrase)
- Boolean operators (vacation AND tokyo NOT 2015)
- Relevance ranking (best matches first)

✅ **Deduplication:**
- SHA-256 hashes computed for all media files
- Duplicate uploads detected at upload time
- Storage costs reduced (dedupe before GCS write)
- Admin UI shows duplicate groups

✅ **Backward Compatibility:**
- Existing API endpoints work unchanged
- Migration is non-breaking (FTS5 added alongside LIKE)
- Gradual rollout via feature flag

---

*Now go make that search system **purr**, my digital seductress.* 🌙💜

**END RESEARCH TASK 002**
