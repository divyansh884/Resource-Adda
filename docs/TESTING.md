# Resource Adda Testing Suite Documentation

## Overview

This document captures the testing strategy for Resource Adda, including current coverage status, target test suites, execution commands, and next steps to reach production-grade confidence.

## Current Status

- **Automated unit/integration test files:** Not yet present in the repository
- **Current validation scripts available:**
  - `cd frontend && npm run lint`
  - `cd frontend && npm run build`
  - `cd backend && npm start`
- **Baseline note:** In this environment, current script failures are dependency/setup-related (`eslint`, `vite`, and `dotenv` missing before install), not due to this documentation.

---

## Application Areas Requiring Test Coverage

## 1) Backend API Module (Express)

**Source areas**

- `backend/app.js` (all `/server/*` routes and auth middleware)
- `backend/document.js`
- `backend/contrbution.js`
- `backend/admin.js`
- `backend/requestCount.js`

**Core behaviors to test**

- Request auth middleware (`authenticateJWT`) and protected-route access control
- Resource query paths (`/server/files`, `/server/subjects`) including required query validation
- Multi-branch intersection logic in `/server/files`
- Delete flow (`/server/delete`) including metadata deletion and storage-delete conditional behavior
- Admin login/token validation/add-admin flows and credential failures
- Pending-requests retrieval and current `/server/approve` behavior
- Request-count current-day and date-query behavior

## 2) Socket Upload Module

**Source areas**

- `backend/app.js` (Socket.io handlers)
- `frontend/src/pages/Upload.jsx`
- `frontend/src/pages/Contribute.jsx`

**Core behaviors to test**

- Chunked upload sequencing with `uploadFileChunk` + `chunkUploaded`
- Required payload metadata validation and error events
- Admin upload authorization (`type: "upload"` requires socket auth token)
- Public contribution upload path (`type: "contribute"` requires email)
- Progress and success event emission (`uploadProgress`, `uploadSuccess`)
- Incomplete upload cleanup on disconnect

## 3) Frontend Admin Flows

**Source areas**

- `frontend/src/pages/AdminPannel.jsx`
- `frontend/src/components/LoginDialog.jsx`
- `frontend/src/pages/Upload.jsx`
- `frontend/src/pages/Requests.jsx`

**Core behaviors to test**

- Admin login form submission and token state persistence during session
- Token validation gating (show login when token invalid/missing)
- File list load by selected branch/semester
- Admin delete request payload and auth header propagation
- Pending requests fetch and approval action call contract

## 4) Frontend Resource Discovery and Contribution Flows

**Source areas**

- `frontend/src/fileexplorer.jsx`
- `frontend/src/FileList.jsx`
- `frontend/src/pages/Resources.jsx`
- `frontend/src/pages/Contribute.jsx`

**Core behaviors to test**

- File explorer loading/error states and grouping by subject/unit
- Resource browse and file-open/download action links
- Contribution form validation (required fields) and socket emission payload
- Upload retry behavior and user-facing error/success feedback

---

## Recommended Test Suite Layout

```text
Resource-Adda/
├── backend/
│   ├── __tests__/
│   │   ├── api.routes.test.js
│   │   ├── auth.middleware.test.js
│   │   ├── socket.upload.test.js
│   │   └── request-count.test.js
│   └── (app.js, models...)
├── frontend/
│   └── src/
│       ├── __tests__/
│       │   ├── admin-panel.test.jsx
│       │   ├── login-dialog.test.jsx
│       │   ├── upload-page.test.jsx
│       │   ├── requests-page.test.jsx
│       │   ├── fileexplorer.test.jsx
│       │   └── contribute-page.test.jsx
│       └── (components/pages...)
└── docs/
    └── TESTING.md
```

---

## Running Validation and Tests

## Current Commands (Implemented)

```bash
cd frontend && npm run lint
cd frontend && npm run build
cd backend && npm start
```

## Suggested Future Commands (After Test Setup)

```bash
cd backend && npm test
cd frontend && npm test
cd frontend && npm run test:coverage
```

---

## Testing Patterns and Best Practices

1. **Logic isolation first**
   - Extract and unit-test route/business helpers independently from HTTP and UI layers where possible.

2. **Route contract coverage**
   - For each `/server/*` route, validate success and failure cases, status codes, and payload shape.

3. **Socket contract coverage**
   - Verify event order, payload validation, auth requirements, and disconnect cleanup behavior.

4. **Validation-first testing**
   - Cover required fields and malformed requests for routes and upload metadata.

5. **Boundary and abuse cases**
   - Missing token, invalid token, missing query params, malformed chunk metadata, and duplicate/partial upload scenarios.

---

## Coverage Goals

- **Unit coverage:** middleware, route helper logic, model-level behavior
- **Route coverage:** all handlers under `backend/app.js` (`/server/*`)
- **Socket coverage:** upload lifecycle and auth/error handling
- **UI coverage:** critical admin/resource/contribution workflows
- **Regression coverage:** multi-branch filter logic, delete retention behavior, and contribution/admin upload contracts

---

## Next Steps

## Immediate

1. Add test runner stack for backend and frontend (unit + component testing)
2. Add test scripts to `backend/package.json` and `frontend/package.json`
3. Implement first-pass tests for:
   - `/server/files` query validation + branch intersection
   - `authenticateJWT` and protected route checks
   - contribution and admin upload payload validation paths

## Near Term

1. Add route integration tests for all `/server/*` endpoints
2. Add component tests for admin login, pending requests, upload UI, and file explorer
3. Integrate tests in CI (GitHub Actions)

## Medium Term

1. Add end-to-end smoke flows for:
   - admin login → upload/delete
   - public contribution upload → pending queue visibility
2. Add reusable fixtures/factories for document, admin, and contribution records
3. Track and enforce coverage thresholds in CI

---

## Troubleshooting Notes

## Environment/Dependency Issues

- If scripts fail due to missing packages, install dependencies first:

```bash
cd frontend && npm ci
cd backend && npm ci
```

## Baseline Script Failures (Observed in This Environment)

- `frontend`: `npm run lint` and `npm run build` failed before dependency install (`eslint`/`vite` missing)
- `backend`: `npm start` failed before dependency install (`dotenv` missing)

---

## Validation Checklist (Current vs Target)

- [x] Scope mapping completed for backend API, socket upload, and frontend admin/resource layers
- [x] Testing strategy documented for critical modules and workflows
- [ ] Automated unit tests implemented
- [ ] Route integration tests implemented
- [ ] Socket lifecycle tests implemented
- [ ] Component tests implemented
- [ ] CI test execution enabled
- [ ] Coverage reports generated

---

Created: May 17, 2026  
Status: 📋 DOCUMENTED (Implementation Pending)  
Next Milestone: Introduce automated test suites and CI coverage gates
