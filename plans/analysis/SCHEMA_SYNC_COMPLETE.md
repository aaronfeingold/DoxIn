# Schema Synchronization Complete

## Architecture Decision: Shared Database, Dual Access

### The Setup

- **One PostgreSQL Database** - Single source of truth for all data
- **Frontend (Next.js + Prisma + Better Auth)** - Manages auth/file tables via migrations
- **Backend (Flask + SQLAlchemy)** - Mirrors auth/file tables + owns invoice domain models

### Division of Responsibilities

#### Frontend Owns (Prisma Migrations)

- User authentication tables (User, Account, Session, Verification)
- Access management (AccessCode, AccessRequest)
- File metadata (FileStorage, ProcessingJob, FileAccessLog)

**Frontend runs Prisma migrations for these tables**

#### Backend Mirrors + Extends (SQLAlchemy)

- Has SQLAlchemy models for ALL frontend tables (can read/write them)
- Owns invoice processing domain models (Invoice, Company, Product, etc.)
- Creates its own tables via `db.create_all()` for invoice domain

### Key Principle

**No schema duplication** - Prisma is the source of truth for auth/file tables, Flask just uses them. Flask is the source of truth for invoice domain tables.

---

## Changes Made to Backend Models

### 1. Created AccessRequest Model

**File:** `backend/api/app/models/access_request.py`

```python
class AccessRequest(BaseModel):
    """User-initiated access request model"""
    __tablename__ = 'access_requests'

    # Fields
    email, name, message, status
    requested_at, reviewed_at, reviewed_by
    rejection_reason

    # Relationships
    - reviewer -> User
    - access_code -> AccessCode (one-to-one)

    # Methods
    - approve(reviewer_user_id)
    - reject(reviewer_user_id, reason)
```

This model was completely missing from the backend, breaking any access request features.

---

### 2. Updated AccessCode Model

**File:** `backend/api/app/models/access_code.py`

**Added fields:**

- `generated_by` - UUID FK to User (who created this code)
- `generation_type` - String: "admin_invite" or "user_request"
- `access_request_id` - UUID FK to AccessRequest (if from request)

**Added relationships:**

- `generated_by_user` -> User who created the code
- `access_request` -> Associated AccessRequest if applicable

Now supports both admin-generated codes AND user request-generated codes.

---

### 3. Updated User Model

**File:** `backend/api/app/models/user.py`

**Added relationships:**

```python
generated_access_codes = relationship("AccessCode", ...)
reviewed_requests = relationship("AccessRequest", ...)
```

Now the backend can:

- Track which access codes a user has generated
- Track which access requests a user has reviewed

---

### 4. Updated Model Registry

**File:** `backend/api/app/models/__init__.py`

Added `AccessRequest` to imports and `__all__` list for proper model registration.

---

## How This Works in Practice

### Scenario 1: User Requests Access

1. **Frontend:** User fills out access request form
2. **Frontend:** Prisma creates record in `access_requests` table
3. **Backend:** Flask API can query `AccessRequest` model to list pending requests
4. **Backend:** Admin approves via Flask API, creates `AccessCode` record
5. **Frontend:** Prisma queries `access_codes` table to show generated code

### Scenario 2: Admin Creates Invite Code

1. **Backend:** Admin uses Flask API to create access code
2. **Backend:** SQLAlchemy creates record in `access_codes` table
3. **Frontend:** Prisma queries table to display code
4. **Frontend:** User registers using Better Auth, code gets marked as used

### Scenario 3: Invoice Upload

1. **Frontend:** User uploads invoice file
2. **Frontend:** Prisma creates `FileStorage` + `ProcessingJob` records
3. **Backend:** Flask polls `ProcessingJob` table, processes invoice
4. **Backend:** Creates `Invoice`, `InvoiceLineItem`, `Company` records (Flask-only tables)
5. **Frontend:** Polls `ProcessingJob` via Prisma to show status

---

## Database Migration Strategy

### For Auth/File Tables (Managed by Prisma)

1. Make changes in `frontend/prisma/schema.prisma`
2. Run `npx prisma migrate dev` from frontend
3. Update corresponding SQLAlchemy models in backend to match
4. Flask just uses the tables, doesn't try to create them

### For Invoice Domain Tables (Managed by Flask)

1. Make changes in `backend/api/app/models/`
2. Flask `db.create_all()` handles table creation
3. Frontend never touches these tables directly

### Migration Coordination

When changing shared tables:

1. Update Prisma schema first
2. Run Prisma migration
3. Update SQLAlchemy model to match
4. Deploy both services together

---

## Benefits of This Approach

1. **No Schema Duplication** - Each service owns certain tables, mirrors others
2. **Better Auth Works** - Prisma + Better Auth works as designed in Next.js
3. **Flask Independence** - Backend can be used as standalone API service
4. **Clear Boundaries** - Frontend: auth/files, Backend: invoice processing
5. **Single Database** - No data sync issues, single source of truth

---

## Models by Owner

### Prisma Owned (Frontend Manages)

- User
- Account
- Session
- Verification
- AccessCode
- AccessRequest
- FileStorage
- ProcessingJob
- FileAccessLog

### SQLAlchemy Owned (Backend Manages)

- Invoice, InvoiceLineItem
- Company, CompanyAddress
- Product, ProductCategory, ProductSubCategory
- SalesTerritory
- Salesperson
- ShipMethod
- DocumentProcessingLog
- Payment
- ExtractionRule
- AuditLog
- Report, SavedReportTemplate
- UsageAnalytics, PageViewSummary

### Mirrored (Both Can Access)

All Prisma-owned tables have SQLAlchemy models in the backend so Flask can interact with users, access codes, files, etc.

---

## Current Status

**Frontend Schema:** Complete and correct
**Backend Schema:** Now complete and synchronized
**Integration:** Backend can now fully interact with access request system

All models are aligned. No schema duplication. Both services can operate independently while sharing the same database.
