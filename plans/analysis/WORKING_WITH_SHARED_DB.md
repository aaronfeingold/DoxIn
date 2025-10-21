# Working With the Shared Database Architecture

## Quick Reference Guide

### When to Use Which Tool

#### Use Prisma (Frontend) When:

- Adding/modifying user authentication features
- Working with access codes or access requests
- Managing file upload metadata
- Tracking processing job status from the UI

#### Use SQLAlchemy (Backend) When:

- Processing invoices
- Managing companies, products, territories
- Generating reports
- Running analytics
- Doing batch operations

---

## Common Workflows

### Adding a New Auth-Related Feature

**Example:** Adding a "reason" field to access requests

1. Update Prisma schema:

```prisma
// frontend/prisma/schema.prisma
model AccessRequest {
  // ... existing fields
  reason String? @db.Text  // ADD THIS
}
```

2. Run Prisma migration:

```bash
cd frontend
npx prisma migrate dev --name add_access_request_reason
```

3. Update SQLAlchemy model:

```python
# backend/api/app/models/access_request.py
class AccessRequest(BaseModel):
    # ... existing fields
    reason = Column(Text)  # ADD THIS
```

4. Deploy both services together

---

### Adding a New Invoice Domain Feature

**Example:** Adding a "discount_percentage" field to invoices

1. Update SQLAlchemy model:

```python
# backend/api/app/models/invoice.py
class Invoice(BaseModel):
    # ... existing fields
    discount_percentage = Column(DECIMAL(5, 2), default=0)
```

2. Restart Flask (it will auto-create columns via `db.create_all()`)

3. Frontend doesn't need changes (it doesn't access invoice tables directly)

---

## Reading Data From Both Services

### Frontend Reading User Data

```typescript
// Uses Prisma - direct database access
const user = await prisma.user.findUnique({
  where: { email: userEmail },
  include: {
    generatedAccessCodes: true,
    reviewedRequests: true,
  },
});
```

### Backend Reading Same User Data

```python
# Uses SQLAlchemy - direct database access to same table
user = User.query.filter_by(email=user_email).first()
access_codes = user.generated_access_codes
requests = user.reviewed_requests
```

Both are reading the SAME rows from the SAME `users` table.

---

## Writing Data From Both Services

### Frontend Creating Access Request

```typescript
// Prisma creates record
await prisma.accessRequest.create({
  data: {
    email: "user@example.com",
    name: "John Doe",
    message: "I need access to process invoices",
    status: "pending",
    requestedAt: new Date(),
  },
});
```

### Backend Approving Access Request

```python
# SQLAlchemy reads same record and updates it
request = AccessRequest.query.filter_by(id=request_id).first()
request.approve(reviewer_user_id=current_user.id)

# Create access code
code = AccessCode(
    code=generate_unique_code(),
    expires_at=datetime.utcnow() + timedelta(days=7),
    generated_by=current_user.id,
    generation_type='user_request',
    access_request_id=request.id
)
db.session.add(code)
db.session.commit()
```

### Frontend Checking Code Status

```typescript
// Prisma reads the access code that Flask created
const code = await prisma.accessCode.findUnique({
  where: { code: userEnteredCode },
  include: {
    accessRequest: true,
  },
});
```

---

## Important Rules

### DO:

- Use Prisma migrations for auth/file tables
- Use SQLAlchemy models in backend to read/write auth/file tables
- Keep invoice domain logic in backend only
- Deploy frontend/backend together when changing shared tables

### DON'T:

- Don't run `db.create_all()` for auth/file tables from Flask (Prisma manages those)
- Don't try to access invoice tables from Prisma (they're backend-only)
- Don't make incompatible schema changes without coordinating both sides
- Don't forget to update both Prisma schema AND SQLAlchemy models for shared tables

---

## Troubleshooting

### "Table already exists" error in Flask

Flask is trying to create a table that Prisma already created. This is OK - Flask will skip it. The error should be non-fatal.

### "Relation does not exist" error in Next.js

Prisma schema is out of sync with database. Run:

```bash
npx prisma db pull  # Pull current schema
npx prisma generate # Regenerate client
```

### "Column doesn't exist" error in Flask

SQLAlchemy model is out of sync with Prisma migrations. Check the Prisma schema and update your SQLAlchemy model to match.

### Foreign key constraint violations

Make sure relationships are defined correctly in BOTH Prisma and SQLAlchemy. They must match exactly.

---

## Schema Change Checklist

When modifying a shared table:

- [ ] Update Prisma schema (`frontend/prisma/schema.prisma`)
- [ ] Run Prisma migration (`npx prisma migrate dev`)
- [ ] Update SQLAlchemy model (`backend/api/app/models/*.py`)
- [ ] Test in both frontend and backend
- [ ] Deploy both services together
- [ ] Verify no foreign key or constraint issues

---

## Architecture Benefits Recap

1. **Single Database** - One source of truth, no sync issues
2. **Clear Ownership** - Prisma owns auth, Flask owns invoices
3. **Both Can Read/Write** - Backend can manage users, frontend can track jobs
4. **No Duplication** - Each table has one owner, one migration path
5. **Independent Services** - Backend can be used standalone via API

This is a pragmatic, maintainable approach that leverages the strengths of both frameworks.
