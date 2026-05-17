# MongoDB Migration Guide (Resource Adda)

## Purpose

Document the backend migration context for Resource Adda data handling to MongoDB-backed persistence and clarify current operational behavior.

## Audience

Developers and maintainers who need to understand MongoDB-backed modules, route integration points, and extension patterns.

## Status

MongoDB-backed persistence is implemented for core backend domains used by the app server.
There is no global in-memory datastore fallback for core entities; request analytics uses an in-process counter and optionally persists snapshots to MongoDB.

---

## Completed Setup

### Database Infrastructure

- [x] Mongoose-based MongoDB connection initialized at server startup (`backend/app.js`)
- [x] Schema/model-based data access modules for active backend domains
- [x] Async route handlers performing MongoDB I/O with `await`
- [x] Environment-driven database configuration (`MONGO_URI`)
- [x] Setup and operations guide (`docs/MONGODB_SETUP.md`)

### Collections in Use

| Domain | File(s) | Collection(s) |
|---|---|---|
| Resources | `backend/document.js` | `documents` |
| Contributions | `backend/contrbution.js` | `contributions` |
| Admin | `backend/admin.js` | `admins` |
| Request Analytics | `backend/requestCount.js` | `requestcounts` |

---

## Migration Summary by Module

### 1) Resource Metadata Persistence

**Primary file**: `backend/document.js`

**What changed / current behavior**
- Resource metadata is persisted in MongoDB through `Document` model records.
- Route-level read/write flows use MongoDB queries and updates:
  - `GET /server/files` (filter + multi-branch intersection handling)
  - `DELETE /server/delete` (document deletion with branch-aware behavior)
  - `GET /server/subjects` (distinct subject extraction)

**API integration**
- `backend/app.js` resource routes are async and `await` MongoDB operations on `Document`.

### 2) Contribution Persistence and Moderation Queue

**Primary file**: `backend/contrbution.js`

**What changed / current behavior**
- Contribution submissions are persisted with status lifecycle:
  - `pending` → `approved` / `rejected`
- Submission metadata (branch/sem/subject/unit/email/file) is stored in MongoDB.
- Pending moderation queue is read from MongoDB.

**API integration**
- `GET /server/pending-requests` uses MongoDB query on `Contribution`.
- Socket upload flow stores contribution records via `Contribution` model in `backend/app.js`.

### 3) Admin Access Persistence

**Primary file**: `backend/admin.js`

**What changed / current behavior**
- Admin credential records are stored in MongoDB.
- Login path verifies MongoDB-stored bcrypt hashes and issues JWTs.
- Additional admin creation persists new records in MongoDB (guarded by super-admin password check).

**API integration**
- `POST /server/admin_login`
- `POST /server/addAdmin`

### 4) Request Count Persistence (Optional Snapshot Mode)

**Primary file**: `backend/requestCount.js`

**What changed / current behavior**
- Runtime request counting is in-memory (`requestCount` variable in `backend/app.js`).
- Historical daily snapshots are optionally persisted to MongoDB based on `SAVE_USER_CNT`.
- Date-filtered lookup reads persisted snapshots from MongoDB.

**API integration**
- `GET /server/request-count`

---

## Controller/Route Pattern

Current backend route pattern in migrated paths:

1. Validate payload/query/auth constraints.
2. Execute MongoDB operations via Mongoose models.
3. Return structured JSON/status responses.
4. Handle runtime and DB errors with guarded responses.

---

## Environment Setup

Use `docs/MONGODB_SETUP.md` for full setup details.

Minimum backend configuration:

```env
MONGO_URI=mongodb://localhost:27017/resource-adda
```

Commonly required alongside MongoDB in this app:

```env
JWT_SECRET=your-jwt-secret
PASSWORD=your-bcrypt-hashed-super-admin-password
BUCKET_NAME=your-gcs-bucket-name
```

---

## Error Handling Notes

Current implementation handles MongoDB failures at route level with `try/catch` and status responses.

Recommended hardening:
- Normalize DB/runtime errors to a shared response shape.
- Add explicit validation handling for malformed identifiers and payload fields.
- Reduce verbose debug logs in production code paths.

---

## Testing and Validation

Current repository validation commands:

```bash
cd frontend && npm run lint
cd frontend && npm run build
cd backend && npm start
```

No dedicated automated MongoDB test suite is present today.
Recommended next step: add integration tests for key MongoDB-backed routes (`/server/files`, `/server/admin_login`, `/server/pending-requests`).

---

## Performance and Operational Follow-ups

Recommended indexes (if not yet applied operationally):
- `documents`: `fileUrl`, `{ branch, sem, subject, unit }`, `uploadedAt`
- `contributions`: `status`, `email`, `submitted_at`
- `admins`: `username` (unique)
- `requestcounts`: `date`

---

## Rollback Strategy

If runtime issues occur after deployment:

1. Roll back the application image/version to the last known stable release.
2. Keep MongoDB data intact and restore from backup if needed.
3. For request analytics continuity, rely on in-process counter behavior while fixing snapshot persistence logic.
4. Re-deploy after verification in a staging environment.

---

## Related Docs

- `docs/MONGODB_SETUP.md`
- `README.md` (project overview and documentation links)
