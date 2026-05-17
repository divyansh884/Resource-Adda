# Resource Adda API Reference

## Purpose

Provide endpoint-level documentation for Resource Adda APIs used by resource discovery, contributions, admin operations, and analytics workflows.

## Audience

Backend and frontend developers integrating with Resource Adda services.

## Status

Reference documentation for the current Express API surface implemented in `backend/app.js`.

## Base Path

All HTTP routes are served under:

`/server/...`

## Overview

Resource Adda currently exposes these API groups:

- **Resource Discovery**: list files, subjects, and downloads
- **Admin Auth & Management**: admin login, token validation, admin creation
- **Contribution Moderation**: pending contribution queue and approve action endpoint
- **File Management**: admin-protected delete flow
- **Analytics**: request-count read endpoint

---

## Resource Discovery Module

### List Files

**GET** `/server/files`
- Query:
  - `branch` (required, can be repeated for multi-branch mode)
  - `sem` (required)
  - `subject` (optional)
  - `unit` (optional)
- Behavior:
  - Increments in-memory request counter
  - With one branch: returns matching documents
  - With multiple branches: returns files common across all requested branches
- Returns: `{ message, files }`

### List Subjects

**GET** `/server/subjects`
- Query: `{ branch, sem }` (both required)
- Behavior: returns distinct subjects from matching documents
- Returns: `{ subjects }`

### Download File

**GET** `/server/download`
- Query: `{ fileUrl, fileName }`
- Behavior:
  - Streams file from remote URL (`fileUrl`) via backend proxy
  - Sets `Content-Disposition` with provided `fileName`
- Returns: streamed file response

---

## Admin Auth & Management Module

### Admin Login

**POST** `/server/admin_login`
- Body: `{ username, password }`
- Behavior:
  - Validates credentials against `Admin` collection
  - Issues JWT token with 1-hour expiry
- Returns: `{ token }`

### Validate Token

**GET** `/server/validate-token`
- Requires: `Authorization: Bearer <token>`
- Behavior: validates JWT signature and expiry
- Returns:
  - Success: `"Token is valid"`
  - Failure: `401/403` with error text

### Add Admin (Super Admin Gate)

**POST** `/server/addAdmin`
- Body: `{ admin_password, username, password }`
- Rules:
  - `admin_password` required
  - `admin_password` must pass bcrypt compare against `process.env.PASSWORD`
- Behavior:
  - Hashes new admin password
  - Creates admin document
- Returns:
  - Success: `201`
  - Failure: `400`, `403`, or `500`

---

## Contribution Moderation Module

### List Pending Contribution Requests

**GET** `/server/pending-requests`
- Requires: `Authorization: Bearer <token>`
- Behavior: returns contributions where `status = "pending"`
- Returns: `Contribution[]`

### Approve Endpoint (Current Implementation)

**POST** `/server/approve`
- Requires: `Authorization: Bearer <token>`
- Body: currently frontend sends `{ id }`
- Behavior:
  - Logs request body
  - Returns success without state mutation in current implementation
- Returns: `200`

---

## File Management Module

### Delete File Metadata (and possibly Storage Object)

**DELETE** `/server/delete`
- Requires: `Authorization: Bearer <token>`
- Body: `{ fileUrl, branch }`
  - `branch` can be string or string[]
- Behavior:
  - Deletes matching `Document` records for requested branches
  - Deletes underlying GCS object only if no remaining document references `fileUrl`
- Returns:
  - Success: `{ message }`
  - Failure: `400`, `404`, or `500`

---

## Analytics Module

### Request Count

**GET** `/server/request-count`
- Query (optional): `date` (ISO-like date string)
- Behavior:
  - Without `date`: returns current in-memory request counter
  - With `date`: returns persisted count for that day if available, else `0`
- Returns: `{ message, count }`

---

## Socket.io Upload API

Resource Adda also exposes real-time upload handling over Socket.io on the backend server root.

### Client Event: `uploadFileChunk`

Payload fields:
- Required for both modes:
  - `fileBuffer`, `branch`, `sem`, `subject`, `unit`, `fileName`, `offset`, `fileSize`, `id`, `type`
- Additional for contribution mode:
  - `email` required when `type = "contribute"`

### Upload Modes

- `type: "upload"` (admin upload)
  - Requires valid JWT in Socket handshake `auth.token`
  - Writes to `Document` collection
- `type: "contribute"` (public contribution)
  - No JWT required
  - Requires `email`
  - Writes to `Contribution` collection with default `pending` status

### Server Events

- `uploadProgress` — numeric upload progress percentage
- `chunkUploaded` — acknowledgement for chunk sequencing
- `uploadSuccess` — upload and metadata-save completion
- `error` — validation/auth/write failure

---

## Authentication and Access Rules

### Public Access

- `GET /server/files`
- `GET /server/subjects`
- `GET /server/download`
- `GET /server/request-count`
- Socket upload with `type="contribute"` (email required)

### JWT Required

- `GET /server/validate-token`
- `DELETE /server/delete`
- `GET /server/pending-requests`
- `POST /server/approve`
- Socket upload with `type="upload"` (token in handshake)

### Super Admin Gate

- `POST /server/addAdmin` requires `admin_password` validated via env-backed bcrypt comparison.

---

## Example Workflows

### Admin Login + Protected Request

```bash
# 1) Login
POST /server/admin_login
{ "username": "admin1", "password": "secret123" }

# 2) Validate token
GET /server/validate-token
Authorization: Bearer <token>

# 3) Read pending requests
GET /server/pending-requests
Authorization: Bearer <token>
```

### Public Resource Discovery

```bash
# 1) List files
GET /server/files?branch=CSE&sem=5

# 2) Multi-branch common files
GET /server/files?branch=CSE&branch=IT&sem=5&subject=DBMS

# 3) Download
GET /server/download?fileUrl=<encoded-url>&fileName=notes.pdf
```

### Admin Delete Flow

```bash
DELETE /server/delete
Authorization: Bearer <token>
Content-Type: application/json

{
  "fileUrl": "https://storage.googleapis.com/<bucket>/<object>",
  "branch": ["CSE", "IT"]
}
```

---

## Related Docs

- `docs/IMPLEMENTATION_SUMMARY.md`
- `docs/MONGODB_SETUP.md`
- `docs/MONGODB_MIGRATION.md`
