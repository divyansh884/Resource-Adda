# MongoDB Setup (Resource Adda)

## Purpose

Provide a single, consistent setup and operational guide for MongoDB in Resource Adda.

## Audience

Developers running Resource Adda locally or in CI who need a working MongoDB connection and baseline operational guidance.

## Status

MongoDB integration is implemented and required for backend data operations (resources, contributions, admins, and request analytics persistence when enabled).

---

## Prerequisites

```bash
# MongoDB installed locally or running via Docker
docker run -d -p 27017:27017 --name mongodb mongo:latest

# Or use MongoDB Atlas (cloud)
# Connection string: mongodb+srv://username:password@cluster.mongodb.net/
```

## Environment Setup

Create a `.env` file in `backend/`:

```env
MONGO_URI=mongodb://localhost:27017/resource-adda
JWT_SECRET=your-jwt-secret
PASSWORD=your-bcrypt-hashed-super-admin-password
BUCKET_NAME=your-gcs-bucket-name
PORT=3333
NODE_ENV=development
```

> Notes:
> - `MONGO_URI` is read from `backend/app.js`.
> - `PASSWORD` is compared in `/server/addAdmin` as the super-admin gate value.
> - `BUCKET_NAME` is required by Google Cloud Storage upload flows.

---

## Database Architecture

### Collections

Mongoose model names are pluralized to collection names by default.

| Collection (default) | Model | Purpose | Source |
|---|---|---|---|
| `documents` | `Document` | Store uploaded resource metadata | `backend/document.js` |
| `contributions` | `Contribution` | Store contributed files and moderation status | `backend/contrbution.js` |
| `admins` | `Admin` | Store admin login credentials | `backend/admin.js` |
| `requestcounts` | `RequestCount` | Store daily request count snapshots (when enabled) | `backend/requestCount.js` |

### Relationships

Document
└── Referenced by branch/semester/subject/unit filters

Contribution
└── Moderated by Admin via pending/approval workflow

Admin
└── Can create additional admins via `/server/addAdmin` (requires super-admin password env gate)

RequestCount
└── Daily counters read via `/server/request-count` (persistence controlled by `SAVE_USER_CNT`)

---

## Schema Details

### Document (documents)

```js
{
  fileUrl: String,
  branch: String,
  sem: String,
  fileName: String,
  subject: String,   // normalized to uppercase in pre-save hook
  unit: String,      // normalized to uppercase in pre-save hook
  uploadedAt: Date
}
```

### Contribution (contributions)

```js
{
  branch: String,
  sem: String,
  subject: String,   // uppercase
  unit: String,      // normalized to uppercase in pre-save hook
  filename: String,
  fileUrl: String,
  email: String,
  status: "pending" | "approved" | "rejected",
  admin_comments: String,
  submitted_at: Date,
  approved_at: Date | null
}
```

### Admin (admins)

```js
{
  username: String,
  password: String   // bcrypt hash
}
```

### RequestCount (requestcounts)

```js
{
  date: Date,
  count: Number
}
```

---

## Database Connection Options

### Local Development

```env
MONGO_URI=mongodb://localhost:27017/resource-adda
```

### Docker (with authentication)

```bash
docker run -d \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  --name mongodb \
  mongo:latest
```

If authentication is enabled:

```env
MONGO_URI=mongodb://admin:password@localhost:27017/resource-adda?authSource=admin
```

### MongoDB Atlas (Cloud)

```env
MONGO_URI=mongodb+srv://username:password@cluster.mongodb.net/resource-adda?retryWrites=true&w=majority
```

---

## Health Check Endpoint (Optional)

You can add a Mongo health endpoint in Express:

```js
app.get("/server/health/db", async (req, res) => {
  try {
    await mongoose.connection.db.admin().command({ ping: 1 });
    return res.status(200).json({ status: "healthy", database: "mongodb" });
  } catch {
    return res.status(503).json({ status: "unhealthy", database: "mongodb" });
  }
});
```

---

## Index Strategy

Resource Adda currently does not create MongoDB indexes explicitly in code. Recommended indexes:

| Collection | Recommended Indexes |
|---|---|
| `documents` | `fileUrl`, `branch`, `sem`, `subject`, `unit`, `uploadedAt` |
| `contributions` | `status`, `email`, `submitted_at`, `approved_at` |
| `admins` | `username` (unique) |
| `requestcounts` | `date` |

Example:

```js
await db.collection("documents").createIndex({ fileUrl: 1 });
await db.collection("documents").createIndex({ branch: 1, sem: 1, subject: 1, unit: 1 });
await db.collection("contributions").createIndex({ status: 1, submitted_at: -1 });
await db.collection("admins").createIndex({ username: 1 }, { unique: true });
await db.collection("requestcounts").createIndex({ date: 1 });
```

---

## Data Backup Strategy

```bash
# Backup
mongodump --uri="mongodb://localhost:27017" --db="resource-adda" --out=./backups

# Restore
mongorestore --uri="mongodb://localhost:27017" --db="resource-adda" ./backups/resource-adda
```

---

## Troubleshooting

### MongoDB Not Configured / Connection Fails

Typical causes:
- `MONGO_URI` missing or invalid
- MongoDB service not reachable
- Authentication mismatch in URI

Check:

```env
MONGO_URI=...
JWT_SECRET=...
PASSWORD=...
BUCKET_NAME=...
```

### Verify Connectivity in Code

```js
await mongoose.connect(process.env.MONGO_URI);
await mongoose.connection.db.admin().command({ ping: 1 });
```

### Admin Login Fails

- Ensure admin exists in `admins` collection.
- Ensure stored password is a bcrypt hash.
- Verify `JWT_SECRET` is set.

### Add Admin Fails

- `/server/addAdmin` requires `admin_password` that matches `PASSWORD` (bcrypt compare).
- Ensure `PASSWORD` in env is the correct bcrypt hash.

---

## Performance Considerations

- Add compound indexes for read-heavy resource filters (`branch`, `sem`, `subject`, `unit`).
- Add indexes for contribution moderation filters (`status`, `submitted_at`).
- Keep response payloads lean for list endpoints.
- Reuse the existing Mongoose connection (already done in startup flow).

---

## Next Steps

- Add explicit index initialization during startup.
- Add `/server/health/db` route for runtime monitoring.
- Add automated tests for MongoDB-backed routes (`/server/files`, `/server/admin_login`, `/server/pending-requests`).
- Consider schema-level validation tightening for enum-like fields (`branch`, `sem`, `unit` sets).

---

## Files Added/Updated

```text
docs/
└── MONGODB_SETUP.md (NEW)
README.md (UPDATED - added documentation link)
```
