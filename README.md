# Cloud Storage for BoxLang

Enhanced S3 file and directory operations for BoxLang applications, with modern, env-driven tests and robust transfer strategies:
- Large file uploads (multipart, retries, backoff, bounded memory)
- Large file downloads (async multipart range reads with throttling and fail‑fast)
- Batch parallel transfers with concurrency control and `failFast`
- Directory semantics (create, exists, list, delete, copy) over S3 prefixes
- Local ↔ S3 and S3 ↔ S3 copy convenience BIFs
- Presigned URL generation for GET/PUT (with headers/metadata)

Built for BoxLang (minimum version 1.7.0) and designed to plug into ColdBox / BoxLang module ecosystems.

## Features
- Uploads: single‑part for small files, multipart for large (adaptive part sizing, retry/backoff).
- Downloads: single‑part for small files, async multipart range downloads for large files with dynamic throttling and fail‑fast semantics.
- Batch operations: async execution with concurrency control and `failFast` option.
- Directory helpers: treat trailing slash keys as logical folders (zero‑byte placeholder objects).
- Server‑side S3→S3 copy using `copyObject` with fallback two‑phase temp download/upload where needed.
- Presigned URL support for GET/PUT, including optional content type, metadata, and response headers.
- Cloud path utilities: resolve cloud vs local, map `s3:///bucket/key` and relative keys (default bucket).
- Test suite supports env‑driven integration flows (and can be mocked when desired).

## Provided BIFs
| BIF | Purpose |
|-----|---------|
| `CSFileUpload` | Upload local file to S3 (small or multipart). |
| `CSFileDownload` | Download S3 object to local disk (streaming large objects). |
| `CSFileUploadBatch` | Parallel batch uploads (local→S3). |
| `CSFileDownloadBatch` | Parallel batch downloads (S3→local). |
| `CSFileCopy` | Copy local↔S3 or S3→S3 single object. |
| `CSFileDelete` | Delete single S3 object. |
| `CSFileGet` | Read object bytes (returns binary) |
| `CSFileInfo` | Get object metadata (size, contentType, eTag, etc.) or directory prefix aggregate (objectCount, totalSize, latest modified). |
| `CSFilePresign` | Generate presigned URL (GET/PUT) with optional headers |
| `CSDirectoryCreate` | Create (ensure) S3 "directory" placeholder. |
| `CSDirectoryExists` | Boolean existence check for prefix placeholder / any object under prefix. |
| `CSDirectoryList` | List objects under a prefix with filters & formats. |
| `CSDirectoryDelete` | Delete all objects under a prefix. |
| `CSDirectoryCopy` | Copy whole directory local↔S3 or S3→S3 (two‑phase). |
 

Notes:
- Use `s3:///bucket/key...` URIs OR relative keys with a configured default bucket (see settings below).
- All directory paths are normalized to a trailing `/`.
- Cloud paths and resolution are handled by `CloudStorageService` (see Utilities below).

## Module Settings
Configure in your module or application (e.g. `ModuleConfig.bx`):

```boxlang
settings = {
  // Timeout (seconds) passed to SDK / HTTP operations
  FileTransferTimeout : 300,
  // Multipart upload threshold (bytes) – above triggers multipart
  UploadMultipartThreshold : 25 * 1024 * 1024,
  // Multipart part size (bytes) – minimum 5MB per S3 requirements (default ~16MB)
  UploadPartSize : 16 * 1024 * 1024,
  // Optional default bucket if callers pass relative keys only
  s3Bucket : 'my-app-bucket'
};
```

### Behavior Summary
- Downloads: Small files use a single SDK read; large files use async multipart range reads with concurrency control, dynamic backoff when connection exhaustion is detected, and fail‑fast to abort quickly on hard failures.
- Uploads: Files `<= UploadMultipartThreshold` use single `putObject`; larger files use multipart (`createMultipartUpload` → `putPart` → `completeMultipartUpload` with abort on error). Each part uses exponential backoff.
- Batch: Uses async futures with configurable concurrency; `failFast=true` aborts remaining items when a failure is detected.

## Installation
Download the module from github and place it in your `boxlang_modules/` directory or other configured BoxLang Module directory, or install via CommandBox once released.

From CommandBox (Pending module release):
```bash
box install bx-cloudstorage
```
Ensure BoxLang runtime & AWS credentials (via environment or configured SDK) are available. The module selects an S3 strategy under the hood.

Install Required Java packages using Maven (Maven needs to be installed):
```bash
mvn install
```
## Usage Examples
### Upload a File
```boxlang
var r = CSFileUpload( source='/tmp/big.iso', destination='s3:///backups/big.iso' );
if( r.error ) writeOutput( 'Upload failed: ' & r.message );
```

### Download a File
```boxlang
var d = CSFileDownload( source='s3:///media/large-video.mp4', destination='/data/large-video.mp4' );
if( d.error ?: false ) writeOutput( 'Download failed: ' & (d.message ?: '') );
```

### Batch Upload
```boxlang
var sources = [ '/tmp/a.log', '/tmp/b.log' ];
var dests   = [ 's3:///logs/a.log', 's3:///logs/b.log' ];
var batch   = CSFileUploadBatch( sources=sources, destinations=dests, concurrency=4 );
if( !batch.success ) dump( batch.errors );
```

### Batch Download
```boxlang
var srcs = [ 's3:///logs/a.log', 's3:///logs/b.log' ];
var dsts = [ '/tmp/restores/a.log', '/tmp/restores/b.log' ];
var dl   = CSFileDownloadBatch( sources=srcs, destinations=dsts, concurrency=4 );
if( !dl.success ) dump( dl.errors );
```

### Copy Local ↔ S3 and S3 ↔ S3
```boxlang
// Local → S3
CSFileCopy( source='/tmp/report.csv', destination='s3:///exports/report.csv' );
// S3 → Local
CSFileCopy( source='s3:///exports/report.csv', destination='/tmp/report.csv' );
// S3 → S3 (server-side copyObject fallback to two‑phase if needed)
CSFileCopy( source='s3:///exports/report.csv', destination='s3:///archive/report.csv' );
```

### Directory Copy (Recursive)
```boxlang
// Upload local directory tree
CSDirectoryCopy( '/data/images', 's3:///cdn/images', recurse=true );
// Download prefix
CSDirectoryCopy( 's3:///cdn/images', '/restore/images', recurse=true );
```

### List Objects with Filter
```boxlang
var files = CSDirectoryList( 's3:///logs/2025', recurse=true, filter='*.json', type='files', listInfo='path' );
```

### Ensure a Directory
```boxlang
if( !CSDirectoryExists( 's3:///data/cache' ) ) CSDirectoryCreate( 's3:///data/cache' );
```

### Get Object Bytes and Metadata
```boxlang
var bytes = CSFileGet( 's3:///exports/report.csv' );
fileWrite( '/tmp/report.csv', bytes );

var info = CSFileInfo( 's3:///exports/report.csv' );
writeDump( info ); // { size=..., contentType=..., eTag=..., lastModified=... }

// Directory / prefix metadata (when key is not a real object)
var dirInfo = CSFileInfo( 's3:///exports/' );
writeDump( dirInfo ); // { isDirectory=true, objectCount=..., totalSize=..., lastModifiedLatest=... }
```

### Presign URLs
```boxlang
// GET (URL string shortcut)
var url = CSFilePresign( file='s3:///exports/report.csv' )?.url;

// GET with options (struct)
var presignedGet = CSFilePresign( file='s3:///exports/report.csv', method='GET', expiresSeconds=300, responseHeaders={} );

// PUT with content type and metadata (struct + headers)
var presignedPut = CSFilePresign( file='s3:///uploads/new.csv', method='PUT', contentType='text/csv', metadata={ source='app' } );
```

## Testing
Specs are provided under `tests/specs/` using TestBox BDD patterns. Many specs are env‑driven integration tests.

Run the test suite:
```bash
box testbox run
```
Watch mode:
```bash
# If using the TestBox runner:
box testbox run watch
# Or execute the BoxLang runner directly
box boxlang run testbox/bx/runner/index.bxm
```
### Test configuration (env‑driven)
Set these environment variables to exercise integration tests:

```bash
export S3_ACCESS_KEY=...
export S3_SECRET_KEY=...
export S3_REGION=us-east-1

export S3_TEST_PATH_S3="s3:///my-test-bucket/test-dir/"
export S3_TEST_LOCAL_PATH="/tmp/s3test/"
export S3_TEST_LOCAL_FILE_SMALL="/tmp/s3test/small.txt"
export S3_TEST_LOCAL_FILE_LARGE="/tmp/s3test/large.bin"
```


## Utilities & Extensibility
- `CloudStorageService` exposes helpers like `isCloudPath`, `resolvePath`, `getStorageForPath`, and provides vendor strategies.
- Strategies: AWS Java SDK v2, aws‑cfml (and s3sdk where present). Java strategy reuses a single credentials provider and region.
- Override or decorate `CloudStorageService` for custom credential resolution.
- Adjust thresholds in settings for different performance profiles.
- Extend `CSFileCopy` for advanced multipart copy of very large objects.

## Error Handling
Each BIF returns a struct or boolean:
- Success path contains `error=false`, `success=true`, or boolean true.
- Failures include descriptive `message` field; batch operations aggregate an `errors` array.

## Performance Notes
- Large downloads use asynchronous range reads to avoid loading entire objects in memory.
- Multipart uploads keep memory bounded by `UploadPartSize` buffers.
- Batch concurrency should be tuned to instance capacity (default 5).

## Roadmap / Ideas
- Optional progress callback hooks for async and batch operations.
- Integrate checksum / integrity validation post transfer and BIF.
- Additional cloud storage providers.

## Contributing
1. Fork & clone.
2. Create feature branch.
3. Add/update specs for new behavior.
4. Run `box testbox run` to ensure green.
5. Submit PR with concise description and rationale.

## License
Apache 2.0. See `LICENSE` or <https://www.apache.org/licenses/LICENSE-2.0>.

## Support / Issues
Report issues at: https://ortussolutions.atlassian.net/browse/BLMODULES