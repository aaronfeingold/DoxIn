# Schema Comparison: Frontend (Prisma) vs Backend (SQLAlchemy)

## Overview

The frontend uses Prisma with PostgreSQL for user authentication and file management, while the backend Flask API handles the core invoice processing domain logic.

## Models Comparison

### Models in Both Frontend and Backend

#### 1. User

**Status:** MISMATCH - Missing relationships in backend

**Frontend (Prisma):**

- All auth fields present
- Has relationships: `generatedAccessCodes`, `reviewedRequests`

**Backend (SQLAlchemy):**

- All auth fields present
- MISSING relationships to AccessCode and AccessRequest models

**Action:** Backend User model needs to add relationships for access code management

---

#### 2. Account (Better Auth)

**Status:** MATCH

**Frontend:** All Better Auth fields present
**Backend:** All Better Auth fields present

---

#### 3. Session (Better Auth)

**Status:** MINOR MISMATCH

**Frontend:** Uses UUID for id
**Backend:** Uses String for id (Better Auth compatibility)

**Note:** This is intentional for Better Auth token compatibility. Frontend should align.

---

#### 4. Verification (Better Auth)

**Status:** MATCH

**Frontend:** All Better Auth fields present
**Backend:** All Better Auth fields present

---

#### 5. FileStorage

**Status:** MATCH

All fields align correctly between frontend and backend.

---

#### 6. ProcessingJob

**Status:** MINOR MISMATCH

**Frontend:** Has `fileStorageId` foreign key
**Backend:** Has `file_storage_id` foreign key and additional field `viewed_at`

**Note:** Minor difference, functionally equivalent. Backend has extra tracking field.

---

#### 7. FileAccessLog

**Status:** MATCH

All fields align correctly.

---

#### 8. AccessCode

**Status:** MAJOR MISMATCH - Missing fields in backend

**Frontend (Prisma):**

```prisma
- code (String)
- isUsed (Boolean)
- usedByEmail (String?)
- usedAt (DateTime?)
- expiresAt (DateTime)
- generatedBy (String?) // UUID FK to User
- generationType (String) // "admin_invite" or "user_request"
- accessRequestId (String?) // UUID FK to AccessRequest
```

**Backend (SQLAlchemy):**

```python
- code (String)
- is_used (Boolean)
- used_by_email (String)
- used_at (DateTime)
- expires_at (DateTime)
# MISSING:
# - generated_by
# - generation_type
# - access_request_id
```

**Action:** Backend AccessCode model needs:

- `generated_by` field (UUID FK to User)
- `generation_type` field (String)
- `access_request_id` field (UUID FK to AccessRequest)
- Relationships to User and AccessRequest

---

### Models ONLY in Frontend

#### 9. AccessRequest

**Status:** MISSING IN BACKEND

**Frontend (Prisma):**

```prisma
model AccessRequest {
  id              String    @id @default(uuid()) @db.Uuid
  email           String    @db.VarChar(255)
  name            String    @db.VarChar(255)
  message         String?   @db.Text
  status          String    @default("pending") // pending, approved, rejected
  requestedAt     DateTime  @default(now())
  reviewedAt      DateTime?
  reviewedBy      String?   // UUID FK to User
  rejectionReason String?
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  reviewer   User?        @relation("ReviewedAccessRequests")
  accessCode AccessCode?
}
```

**Backend:** Does not exist

**Impact:** HIGH - This is a user-facing feature for access request management

**Action:** Create `backend/api/app/models/access_request.py` with full model definition

---

### Models ONLY in Backend

These are domain-specific models for invoice processing and don't need to be in the frontend:

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

**Status:** CORRECT - These belong only in the backend

---

## Summary of Issues

### Critical Issues

1. **AccessRequest model missing in backend** - Need to create this model for the access request system to work
2. **AccessCode missing fields in backend** - Need to add `generated_by`, `generation_type`, and `access_request_id`
3. **User model missing relationships in backend** - Need to add relationships for access code management

### Minor Issues

4. **Session.id type difference** - Frontend uses UUID, backend uses String (this is OK for Better Auth)
5. **ProcessingJob has extra field in backend** - `viewed_at` field not in frontend (this is OK)

---

## Recommendations

### For Backend (High Priority)

1. Create `AccessRequest` model
2. Update `AccessCode` model with missing fields and relationships
3. Update `User` model with missing relationships

### For Frontend (Low Priority)

1. Consider if Session.id should be String instead of UUID for Better Auth consistency
2. Consider adding `viewed_at` to ProcessingJob if needed for frontend display

---

## Frontend Schema Status

The frontend schema is generally well-aligned for its authentication and file management concerns. The main issue is that the backend is missing the access request management system that the frontend expects.

The division of concerns is good:

- Frontend: User auth, access codes, file metadata
- Backend: Invoice processing, companies, products, analytics

**Current Status:** Frontend schema is CORRECT, but backend is INCOMPLETE for access request features.
