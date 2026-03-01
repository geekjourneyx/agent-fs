---
name: afs
description: Agent-first cross-platform file operations CLI. Use when AI Agent needs local file operations (zip/unzip/info/read) or cloud storage operations (upload/download/list) with S3-compatible providers.
---

# afs - Agent-First File Operations CLI

CLI tool for AI Agents to perform local file operations and cloud storage synchronization with S3-compatible providers.

## Quick start

**Install**:
```bash
curl -fsSL https://raw.githubusercontent.com/geekjourneyx/agent-fs/main/scripts/install.sh | bash
```

**Basic usage**:
```bash
# Get file info
afs local info /path/to/file

# Read file with slicing (Token-aware)
afs local read /path/to/log --tail 50

# Create and extract archives
afs local zip /data --out backup.zip
afs local unzip backup.zip --dest /restore

# Cloud operations
afs cloud upload file.txt remote/path/
afs cloud list remote/ --limit 20
afs cloud url remote/file.txt --expires 3600  # Generate access URL
afs cloud providers  # List supported providers
```

## Commands

| Command | Purpose |
|---------|---------|
| `afs local zip` | Create zip archive from file/directory |
| `afs local unzip` | Extract zip archive to destination |
| `afs local info` | Get file/directory metadata |
| `afs local read` | Read file with slicing options |
| `afs cloud upload` | Upload to S3-compatible storage |
| `afs cloud download` | Download from cloud storage |
| `afs cloud list` | List objects in cloud storage |
| `afs cloud url` | Generate access URL (Presigned URL or Public URL) |
| `afs cloud providers` | List supported cloud storage providers |
| `afs config` | Manage configuration |

## Local operations

### Get file/directory info

```bash
# File info
afs local info /path/to/file

# Directory with details (file count, total bytes)
afs local info /path/to/dir --details
```

**Output**:
```json
{
  "success": true,
  "action": "local_info",
  "data": {
    "name": "file.txt",
    "path": "/path/to/file.txt",
    "type": "file",
    "size_bytes": 1024,
    "mode": "0644",
    "modified_time": "2024-01-01T00:00:00Z",
    "is_dir": false
  },
  "error": null
}
```

### Read file with slicing (Token-aware)

Designed for large files - avoid reading entire GB logs into context:

```bash
# Read last N lines (perfect for error logs)
afs local read /var/log/app.log --tail 100

# Read first N lines
afs local read /path/to/config.yaml --head 20

# Read first N bytes
afs local read /path/to/data.bin --bytes 4096

# Read entire file (limited to 10MB)
afs local read /path/to/small.txt
```

**Output**:
```json
{
  "success": true,
  "action": "local_read",
  "data": {
    "path": "/path/to/file.log",
    "content": "...",
    "line_count": 50,
    "byte_count": 2048,
    "truncated": true,
    "slice_type": "tail"
  },
  "error": null
}
```

### Create archive

```bash
# Zip a file or directory
afs local zip /data --out backup.zip

# Zip with nested paths
afs local zip ./project/src --out archive.zip
```

**Output**:
```json
{
  "success": true,
  "action": "local_zip",
  "data": {
    "source_path": "/data",
    "zip_path": "backup.zip",
    "size_bytes": 1048576
  },
  "error": null
}
```

### Extract archive

```bash
# Extract to directory
afs local unzip backup.zip --dest /restore
```

**Output**:
```json
{
  "success": true,
  "action": "local_unzip",
  "data": {
    "zip_path": "backup.zip",
    "destination": "/restore",
    "extracted_files": 42,
    "size_bytes": 1048576
  },
  "error": null
}
```

## Cloud operations

### Upload

**⚠️ Security requirement**: Remote path MUST follow `date/hash/` format for cloud uploads. This prevents conflicts and organizes files by time.

```bash
# Upload file (REQUIRED format: YYYYMMDD/hash/filename)
DATE=$(date +%Y%m%d)
HASH=$(md5sum /path/file | cut -c1-8)
afs cloud upload local_file.txt "${DATE}/${HASH}/file.txt" --provider s3

# Upload directory with automatic compression
afs cloud upload /data/logs/ "20260301/a3b4c5d6/logs.zip" --zip --provider r2

# Example: backup with timestamp
afs cloud upload backup.zip "backups/$(date +%Y%m%d)/$(date +%H%M%S).zip"
```

**Path format breakdown**:
- `YYYYMMDD` - Date for organization (e.g., `20260301`)
- `8-char-hash` - Short unique identifier (e.g., `a3b4c5d6`)
- `filename` - Actual filename

**Output**:
```json
{
  "success": true,
  "action": "cloud_upload",
  "data": {
    "provider": "s3",
    "local_path": "/local/file.txt",
    "remote_key": "remote/path/file.txt",
    "remote_url": "https://...",
    "size_bytes": 1024,
    "time_taken_ms": 250,
    "compressed": false
  },
  "error": null
}
```

### Download

```bash
# Download file
afs cloud download remote/file.txt /local/path/

# Download and auto-extract
afs cloud download remote/archive.zip /local/dir/ --unzip
```

**Output**:
```json
{
  "success": true,
  "action": "cloud_download",
  "data": {
    "provider": "s3",
    "remote_key": "remote/file.txt",
    "local_path": "/local/file.txt",
    "size_bytes": 1024,
    "time_taken_ms": 150,
    "decompressed": false
  },
  "error": null
}
```

### List objects

```bash
# List objects with prefix
afs cloud list backups/ --limit 50 --provider s3

# List all objects in bucket
afs cloud list --provider r2
```

**Output**:
```json
{
  "success": true,
  "action": "cloud_list",
  "data": {
    "provider": "s3",
    "objects": [
      {
        "key": "backups/file1.txt",
        "size_bytes": 1024,
        "last_modified": "2024-01-01T00:00:00Z",
        "etag": "\"d41d8cd98f00b204e9800998ecf8427e\""
      }
    ],
    "count": 1,
    "total_bytes": 1024,
    "prefix": "backups/",
    "is_truncated": false
  },
  "error": null
}
```

### Generate access URL

Generate presigned URL for temporary private access or public URL for publicly accessible objects:

```bash
# Generate presigned URL (default 15 minutes valid)
afs cloud url remote/file.txt

# Custom expiration time (in seconds)
afs cloud url remote/file.txt --expires 3600  # 1 hour
afs cloud url remote/file.txt --expires 60    # 1 minute

# Generate public URL (for objects with public read access)
afs cloud url remote/public.jpg --public

# Specify provider
afs cloud url remote/file.txt --provider r2 --expires 7200
```

**Output (Presigned URL)**:
```json
{
  "success": true,
  "action": "cloud_url",
  "data": {
    "provider": "s3",
    "remote_key": "remote/file.txt",
    "url": "https://my-bucket.s3.amazonaws.com/remote/file.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=...&X-Amz-Date=...&X-Amz-Expires=900&X-Amz-SignedHeaders=host&X-Amz-Signature=...",
    "expires_in": 900,
    "expires_at": "2024-03-01T18:15:30Z",
    "is_presigned": true
  }
}
```

**Output (Public URL)**:
```json
{
  "success": true,
  "action": "cloud_url",
  "data": {
    "provider": "s3",
    "remote_key": "remote/public.jpg",
    "url": "https://my-bucket.s3.amazonaws.com/remote/public.jpg",
    "is_presigned": false
  }
}
```

**URL flags**:
- `--expires N` - Expiration time in seconds (default: 900 = 15 minutes)
- `--public` - Generate public URL instead of presigned URL
- `--provider` - Cloud storage provider

### List supported providers

```bash
# List all supported cloud storage providers
afs cloud providers
```

**Output**:
```json
{
  "success": true,
  "action": "cloud_providers",
  "data": {
    "providers": [
      {
        "name": "s3",
        "description": "AWS S3",
        "endpoint_example": "https://s3.amazonaws.com"
      },
      {
        "name": "r2",
        "description": "Cloudflare R2",
        "endpoint_example": "https://{account_id}.r2.cloudflarestorage.com",
        "config_note": "Set account_id for auto-generated endpoint, or specify endpoint manually"
      },
      {
        "name": "minio",
        "description": "MinIO Self-Hosted Object Storage",
        "endpoint_example": "http://localhost:9000",
        "config_note": "Set path_style=true for non-virtual-hosted-style access"
      },
      {
        "name": "alioss",
        "description": "Alibaba Cloud Object Storage Service (OSS)",
        "endpoint_example": "https://oss-cn-hangzhou.aliyuncs.com",
        "config_note": "Replace region in endpoint: oss-{region}.aliyuncs.com"
      },
      {
        "name": "txcos",
        "description": "Tencent Cloud Object Storage (COS)",
        "endpoint_example": "https://cos.ap-guangzhou.myqcloud.com",
        "config_note": "Replace region in endpoint: cos.{region}.myqcloud.com"
      },
      {
        "name": "b2",
        "description": "Backblaze B2 (S3 Compatible)",
        "endpoint_example": "https://s3.us-west-004.backblazeb2.com"
      },
      {
        "name": "wasabi",
        "description": "Wasabi Hot Cloud Storage",
        "endpoint_example": "https://s3.wasabisys.com"
      }
    ],
    "note": "Any S3-compatible storage is supported. Configure custom endpoint via config."
  }
}
```

## Configuration

Config file locations (priority order):
1. `.agent-fs.yaml` (current directory)
2. `~/.agent-fs.yaml` (home directory)

**Environment variables**:
- `AFS_WORKSPACE` - Sandbox workspace root (security, prevents path traversal)
- `AFS_S3_ENDPOINT` - S3 endpoint URL
- `AFS_S3_BUCKET` - S3 bucket name
- `AFS_S3_ACCESS_KEY_ID` - S3 access key
- `AFS_S3_SECRET_ACCESS_KEY` - S3 secret key

**Config commands**:
```bash
# Set provider configuration
afs config set s3.endpoint https://xxx.r2.cloudflarestorage.com
afs config set s3.bucket my-bucket
afs config set s3.access_key_id AKIAIOSFODNN7EXAMPLE
afs config set s3.secret_access_key wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# View configuration
afs config get s3.endpoint

# Use global config
afs config set s3.endpoint https://xxx.r2.cloudflarestorage.com --global
```

**Provider-specific config**:
```yaml
# ~/.agent-fs.yaml
s3:
  endpoint: https://s3.amazonaws.com
  region: us-east-1
  bucket: my-bucket
  access_key_id: AKIAIOSFODNN7EXAMPLE
  secret_access_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

r2:
  account_id: your_account_id
  bucket: my-r2-bucket
  access_key_id: your_access_key
  secret_access_key: your_secret_key
  endpoint: https://your_account_id.r2.cloudflarestorage.com
```

## Supported providers

**Officially supported (with preset configuration)**:

| Provider | ID | Endpoint Example | Description |
|----------|----|-----------------|-------------|
| AWS S3 | `s3` | `https://s3.amazonaws.com` | Amazon S3 |
| Cloudflare R2 | `r2` | Auto-generated (set account_id) | Cloudflare R2 |
| MinIO | `minio` | `http://localhost:9000` | Self-hosted object storage |
| Aliyun OSS | `alioss` | `https://oss-cn-hangzhou.aliyuncs.com` | Aliyun Object Storage |
| Tencent COS | `txcos` | `https://cos.ap-guangzhou.myqcloud.com` | Tencent Cloud Object Storage |
| Backblaze B2 | `b2` | `https://s3.us-west-004.backblazeb2.com` | B2 S3 compatible |
| Wasabi | `wasabi` | `https://s3.wasabisys.com` | Wasabi Hot Cloud Storage |

**Other S3-compatible storage**:

Any S3-protocol compatible object storage is supported:
- DigitalOcean Spaces
- Google Cloud Storage (Interoperability)
- Oracle Cloud Infrastructure
- Custom MinIO deployments

**Query supported providers**:
```bash
afs cloud providers
```

## Output format

All commands return standardized JSON for AI Agent parsing:

**Success**:
```json
{
  "success": true,
  "action": "command_name",
  "data": {...},
  "error": null
}
```

**Failure**:
```json
{
  "success": false,
  "action": "command_name",
  "data": null,
  "error": {
    "code": "ERR_NOT_FOUND",
    "message": "File does not exist"
  }
}
```

## Security

### Cloud upload path requirement

**MANDATORY**: All cloud uploads MUST use `YYYYMMDD/hash/` path format:

```bash
# Correct format
afs cloud upload file.txt "20260301/a3b4c5d6/file.txt"

# Incorrect (will be rejected)
afs cloud upload file.txt "file.txt"
afs cloud upload file.txt "uploads/file.txt"
```

**Reasons**:
- Prevents path conflicts between concurrent operations
- Organizes files by date for easy management
- Hash provides uniqueness for same-day uploads
- Avoids accidental file overwrites

### Sandbox mode

Restrict operations to a specific directory:

```bash
export AFS_WORKSPACE=/safe/workspace

# Now all operations are restricted to /safe/workspace
afs local info /etc/passwd  # ERROR: path is outside AFS_WORKSPACE
```

### Path traversal protection

All paths are validated against `AFS_WORKSPACE`. The `../` sequences are blocked automatically.

## Common workflows

### Agent file backup workflow

```bash
# 1. Get info before processing
afs local info /data --details

# 2. Create compressed archive
afs local zip /data --out backup.zip

# 3. Upload to cloud (REQUIRED: date/hash/ format)
DATE=$(date +%Y%m%d)
HASH=$(md5sum backup.zip | cut -c1-8)
afs cloud upload backup.zip "backups/${DATE}/${HASH}/backup.zip"

# 4. Verify upload
afs cloud list "backups/${DATE}/" --limit 1
```

### Error log analysis workflow

```bash
# 1. Check log size
afs local info /var/log/app.log

# 2. Read last 100 lines for errors
afs local read /var/log/app.log --tail 100

# 3. If needed, get more context
afs local read /var/log/app.log --tail 500
```

### Cloud sync workflow

```bash
# 1. List remote files
afs cloud list projects/ --limit 100

# 2. Download needed file
afs cloud download projects/config.yaml ./

# 3. Extract if compressed
afs local unzip config.zip --dest ./
```

## Error codes

| Code | Description |
|------|-------------|
| `ERR_INVALID_ARGUMENT` | Invalid command argument |
| `ERR_PATH_TRAVERSAL` | Path outside workspace |
| `ERR_NOT_FOUND` | File/directory not found |
| `ERR_CONFLICT` | Destination already exists |
| `ERR_PROVIDER` | Unsupported/invalid provider |
| `ERR_UPLOAD` | Upload failed |
| `ERR_DOWNLOAD` | Download failed |
| `ERR_ARCHIVE` | Archive operation failed |
| `ERR_CONFIG` | Configuration error |
| `ERR_INTERNAL` | Internal error |

## Implementation notes

- **Go 1.24+** required
- **Cross-platform**: Linux, macOS, Windows
- **S3 protocol**: Compatible with any S3-compatible storage
- **Machine-readable output**: All commands return JSON
- **Token-aware**: Built-in file slicing for large files

## Project structure

```
agent-fs/
├── cmd/                    # CLI commands
│   ├── root.go             # Root command
│   ├── local.go            # Local operations
│   ├── cloud.go            # Cloud operations
│   └── config.go           # Config management
├── pkg/                    # Core logic
│   ├── local/              # Local file operations
│   │   ├── info.go         # File info
│   │   └── read.go         # File reading
│   ├── archive/            # Zip operations
│   ├── cloud/              # Cloud abstraction
│   ├── s3client/           # S3 client
│   ├── sandbox/            # Path security
│   ├── output/             # JSON output
│   └── apperr/             # Error handling
└── main.go                 # Entry point
```
