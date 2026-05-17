# Resource Adda

<img src="https://img.shields.io/badge/PRs-welcome-brightgreen.svg"> <img src="https://img.shields.io/badge/Live-Google%20Cloud%20Run-blue.svg">

The academic resource-sharing platform — designed to help students discover study materials, contribute notes, and connect with communities organised by branch and semester.

🌐 **Live:** https://resource-adda-1030059749120.asia-south1.run.app/

## 📖 About

Resource Adda is a full-stack platform built with React and Node.js for:

- **Browsing study materials** filtered by branch, semester, subject, and unit
- **Contributing resources** — students submit notes/files for admin review and approval
- **Admin operations** — managing uploads, reviewing contributions, and controlling access
- **Community groups** — connecting students around shared academic interests

It replaces scattered, manual note-sharing with a unified system including JWT-based admin auth, Google Cloud Storage for files, real-time upload progress via Socket.io, and a contribution approval workflow.

## Key Problems Solved

- **Fragmented resource discovery** — Hierarchical filter (branch → semester → subject → unit) for fast lookup
- **Unverified contributions** — Admin-gated approval workflow before resources are published
- **Large file handling** — Chunked upload over Socket.io with real-time progress feedback
- **Access control** — JWT-authenticated admin routes; super-admin-only admin provisioning
- **Cross-branch content** — Query files common across multiple branches in one request

## Who Is This For?

- **Students** searching for notes and past papers by their branch and semester
- **Top performers / contributors** submitting their notes for community benefit
- **Department admins** reviewing, approving, or rejecting submitted resources
- **System maintainers** extending upload, auth, and contribution workflows

## ✨ Features

### Resource Discovery ✅

- Filter resources by branch, semester, subject, and unit
- Cross-branch query support (returns files common to multiple branches)
- In-browser PDF viewer
- One-click file download via proxied backend endpoint

### Contribution System ✅

- Students upload files with metadata (branch, sem, subject, unit, email)
- Contributions enter a `pending` queue for admin review
- Admin can approve or reject with optional comments
- Lifecycle: `pending` → `approved` / `rejected`

### Admin Operations ✅

- JWT-based admin login (1-hour token expiry)
- Admin panel for browsing, deleting, and managing uploaded resources
- Pending contribution queue with approval/rejection actions
- Super-admin-only endpoint to provision new admins (password-protected)

### Real-time Upload ✅

- Chunked file streaming via Socket.io (up to 200 MB per file)
- Per-chunk progress events back to the client
- Authenticated admin uploads vs. public contribution uploads handled on the same socket

### Analytics ✅

- Daily request counter with midnight reset
- Historical request-count lookup by date stored in MongoDB

## 🚀 Quick Start

### Prerequisites

- Node.js 22+
- npm / yarn
- Google Cloud SDK (authenticated) with a GCS bucket
- MongoDB instance

### Installation

```bash
# Frontend
cd frontend
npm install

# Backend
cd backend
npm install
```

### Configure Environment

Create a `.env` file in the `backend/` directory:

```env
# MongoDB
MONGO_URI=mongodb://localhost:27017/resource-adda

# Super-admin password (bcrypt hash stored in DB manually or via /server/addAdmin)
PASSWORD=your-bcrypt-hashed-super-admin-password

# JWT
JWT_SECRET=your-jwt-secret

# Google Cloud Storage bucket name
BUCKET_NAME=your-gcs-bucket-name
```

> **Note:** Ensure your environment is authenticated with `gcloud auth application-default login` so the backend can access Google Cloud Storage.

### Configure Frontend

In `frontend/src/constants.jsx`, set the backend server URL before running:

```js
export const SERVER_URL = "http://localhost:3333";
```

### Run the Project

```bash
# Start the backend (from /backend)
npm run dev

# Start the frontend (from /frontend)
npm run dev
```

Frontend: http://localhost:5173  
Backend: http://localhost:3333

### Build Frontend into Backend

To serve the frontend through the Express backend:

```bash
cd frontend
npm run build   # outputs to ../backend/dist
```

### Docker Compose

```bash
docker-compose up --build
```

This starts the backend on port `3333` and the frontend on port `80`.

## 🔐 Access Model

| Route | Access |
|---|---|
| `GET /server/files` | Public |
| `GET /server/subjects` | Public |
| `GET /server/download` | Public |
| `GET /server/request-count` | Public |
| `GET /server/validate-token` | Token holder |
| `DELETE /server/delete` | Admin (JWT) |
| `GET /server/pending-requests` | Admin (JWT) |
| `POST /server/approve` | Admin (JWT) |
| `POST /server/admin_login` | Credential-based |
| `POST /server/addAdmin` | Super-admin password |

Socket.io upload events distinguish admin uploads (`type: "upload"`, requires JWT) from public contributions (`type: "contribute"`, requires email).

## 🛠️ Tech Stack

- **Frontend**: React 18, Vite, React Router DOM v6, Framer Motion, React PDF, Axios, Socket.io-client
- **Backend**: Node.js 22, Express, Mongoose (MongoDB), JWT, bcrypt, Multer, Socket.io, Axios
- **Storage**: Google Cloud Storage (GCS)
- **Deployment**: Docker, Google Cloud Run
- **Analytics**: MongoDB-backed daily request counting

## 🏗️ Architecture Highlights

### Layered Design

- **API Layer**: Express route handlers in `backend/app.js`
- **Data Layer**: Mongoose models in `backend/document.js`, `backend/admin.js`, `backend/contrbution.js`, `backend/requestCount.js`
- **Auth Layer**: JWT middleware (`authenticateJWT`) protecting admin-only routes
- **Upload Layer**: Socket.io chunked streaming to Google Cloud Storage with per-socket upload state

### Core Modules

- **Resource Module** — Fetch, filter (branch/sem/subject/unit), download, and delete documents
- **Contribution Module** — Student submission flow, pending queue, admin approval/rejection
- **Admin Module** — JWT login, token validation, super-admin-controlled admin provisioning
- **Upload Module** — Chunked Socket.io streaming, GCS write streams, upload progress events

## 📁 Project Structure

```text
backend/
  app.js              # Express server, Socket.io, all route handlers
  document.js         # Mongoose model: uploaded resource documents
  admin.js            # Mongoose model: admin accounts
  contrbution.js      # Mongoose model: student contribution requests
  requestCount.js     # Mongoose model: daily request analytics
  Dockerfile

frontend/
  src/
    pages/
      Home.jsx        # Landing page
      Resources.jsx   # Branch/semester selector
      Contribute.jsx  # Student contribution upload form
      AdminPannel.jsx # Admin dashboard
      SuperAdmin.jsx  # Add-admin page (super admin only)
      About.jsx
      View.jsx        # In-browser PDF viewer
      Requests.jsx    # Pending contribution requests (admin)
      Upload.jsx      # Admin file upload page
    components/
      Navbar.jsx
      LoginDialog.jsx
      ProgressMenu.jsx
      AdminNavbar.jsx
      ...
    fileexplorer.jsx  # Subject/unit/file hierarchy browser
    groups.jsx        # Community groups
    constants.jsx     # SERVER_URL and other config
  Dockerfile

docker-compose.yml
```

## ✅ Validation Commands

Run from each sub-directory:

```bash
# Frontend lint
cd frontend && npm run lint

# Frontend build
cd frontend && npm run build

# Backend start (no dedicated test suite yet)
cd backend && npm start
```

## 📚 Documentation

- [MongoDB Setup Guide](docs/MONGODB_SETUP.md)
- [MongoDB Migration Guide](docs/MONGODB_MIGRATION.md)
- [Implementation Summary](docs/IMPLEMENTATION_SUMMARY.md)
- [API Reference](docs/API_REFERENCE.md)
- [Testing Suite Documentation](docs/TESTING.md)
- [Testing Status Report](docs/TEST_RESULTS.md)

## 🎯 Roadmap Snapshot

- ✅ Resource discovery with multi-filter and cross-branch queries
- ✅ Student contribution flow with admin approval queue
- ✅ JWT-authenticated admin panel and super-admin provisioning
- ✅ Chunked Socket.io uploads to Google Cloud Storage
- ✅ Docker and Google Cloud Run deployment
- 🟡 Automated unit/integration test suite
- 📋 Rate limiting, input sanitisation hardening, and email notifications on contribution status changes

## 🤝 Contributing

Contributions are welcome. Please keep changes aligned with:

- Existing module boundaries (`resources`, `contributions`, `admin`, `upload`)
- Mongoose schema conventions in the backend model files
- Root validation checks (`npm run lint`, `npm run build` in `frontend/`)
- Documentation updates for any behaviour changes
