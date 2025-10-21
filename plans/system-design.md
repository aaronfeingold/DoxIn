# DoxIn System Design Specification

## Executive Summary

DoxIn is a cloud-native invoice processing system with AI-powered extraction and intelligent chat assistance. Built for Google Cloud Platform (GCP) with serverless Neon PostgreSQL database, the system provides document processing, business intelligence, and self-service AI capabilities through a conversational interface.

### Core Capabilities

1. **AI-Powered Invoice Processing** - GPT-4 Vision extracts structured data from invoice documents
2. **Intelligent Chat Assistant** - RAG-powered chatbot with Text-to-SQL and tool execution (NEW)
3. **Business Intelligence** - Analytics dashboards with price variance and vendor performance
4. **Real-time Updates** - WebSocket streaming for processing status and chat responses

---

## System Architecture Overview

### Target Platform: Google Cloud Platform (GCP)

**Cloud Provider**: Google Cloud Platform
**Database**: Neon PostgreSQL (serverless, AWS-hosted with GCP connectivity)
**Compute**: Cloud Run (serverless containers)
**Cache**: Memorystore Redis (GCP managed)
**Storage**: Cloud Storage (GCS)
**Vector DB**: Qdrant (self-hosted on Cloud Run) - NEW
**Frontend**: Vercel (edge network)

### Multi-Environment Strategy

**Development Environment:**
- Docker Compose stack with PostgreSQL, Redis, Qdrant
- Local filesystem storage
- Full monitoring stack (Prometheus, Grafana)
- Hot reload for rapid development

**Production Environment:**
- Neon Database: Serverless PostgreSQL with auto-scaling
- Cloud Run: Auto-scaling containerized services
- Memorystore Redis: Managed cache and session store
- Cloud Storage: Document storage with lifecycle management
- Qdrant on Cloud Run: Vector database for AI chat
- Cloud Operations: Logging, monitoring, tracing

**Configuration Management:**
- Automatic database type detection and optimization
- Environment-specific configs (.env.development, .env.production)
- Smart connection pooling (smaller pools for Neon serverless)
- SSL optimization for Neon, local optimization for Docker

---

## System Components & API Specification

### Flask API Endpoints

#### Authentication & User Management

**Base Path**: `/api/v1/auth`

- `POST /login` - Google OAuth login with JWT token generation
  - **Public endpoint**
  - Request: `{oauth_code: string}`
  - Response: `{token: string, user: UserProfile, expires_in: number}`

- `POST /logout` - Session termination and token invalidation
  - **Authenticated**
  - Response: `{success: boolean}`

- `GET /me` - Current user profile and permissions
  - **Authenticated**
  - Response: `{user: UserProfile, permissions: string[]}`

- `POST /refresh` - Refresh JWT token before expiration
  - **Authenticated**
  - Request: `{refresh_token: string}`
  - Response: `{token: string, expires_in: number}`

**User Management** (Admin only):
- `GET /api/v1/users` - User management dashboard
- `POST /api/v1/users/invite` - Invite new users with access codes
- `PATCH /api/v1/users/{id}/role` - Change user roles (user/admin)
- `PATCH /api/v1/users/{id}/status` - Activate/deactivate accounts
- `DELETE /api/v1/users/{id}` - Soft delete user accounts
- `GET /api/v1/audit/user-activity` - User activity audit trail

#### Document Processing Pipeline

**Base Path**: `/api/v1/invoices`

- `POST /upload` - Accept invoice files, store in Cloud Storage, trigger processing
  - **Authenticated** (user/admin)
  - Request: `multipart/form-data` with file
  - Response: `{job_id: uuid, status: string, websocket_room: string}`

- `GET /{id}/status` - Real-time processing status via WebSocket
  - **Authenticated** (users: own files, admins: all files)
  - Response: `{status: string, progress: number, current_stage: string}`

- `POST /{id}/process` - Manual trigger for reprocessing
  - **Authenticated** (users: own files, admins: all files)
  - Request: `{parameters: ProcessingOptions}`
  - Response: `{job_id: uuid}`

- `POST /process-batch` - Batch upload and processing
  - **Authenticated**
  - Request: `{files: FileInfo[], options: ProcessingOptions}`
  - Response: `{task_ids: uuid[], message: string}`

- `GET /` - Paginated list with filtering
  - **Authenticated** (users: own invoices, admins: all with filters)
  - Query params: `page, per_page, status, date_from, date_to, customer_id`
  - Response: `{invoices: Invoice[], pagination: PaginationInfo}`

- `GET /{id}` - Single invoice with line items and blob URL
  - **Authenticated**
  - Response: `{invoice: Invoice, line_items: LineItem[], blob_url: string}`

- `PUT /{id}` - Update invoice and line items
  - **Authenticated** (users: own invoices, admins: all)
  - Request: `{invoice: Partial<Invoice>, line_items: LineItem[]}`
  - Response: `{success: boolean, invoice: Invoice}`

#### AI Chat Assistant (NEW)

**Base Path**: `/api/v1/chat`

- `POST /sessions` - Create new chat session
  - **Authenticated**
  - Request: `{title?: string}`
  - Response: `{session_id: uuid, created_at: timestamp}`

- `POST /sessions/{id}/messages` - Send message and receive streaming response
  - **Authenticated**
  - Headers: `Accept: text/event-stream`
  - Request: `{message: string, context?: object}`
  - Response: SSE stream with events:
    - `{type: "intent", data: string}` - Classified intent
    - `{type: "context", data: object}` - Retrieved context (RAG)
    - `{type: "data", data: object}` - SQL query results
    - `{type: "token", data: string}` - Streamed response token
    - `{type: "tool_call", data: {tool: string, args: object}}` - Tool execution
    - `{type: "complete", data: {message_id: uuid, tokens: number}}` - Stream complete

- `GET /sessions/{id}/messages` - Retrieve chat history
  - **Authenticated**
  - Query params: `limit, offset`
  - Response: `{messages: Message[], pagination: PaginationInfo}`

- `GET /sessions` - List user's chat sessions
  - **Authenticated**
  - Response: `{sessions: Session[]}`

- `DELETE /sessions/{id}` - Delete chat session
  - **Authenticated**
  - Response: `{success: boolean}`

- `POST /feedback` - User feedback on chat responses
  - **Authenticated**
  - Request: `{message_id: uuid, rating: number, comment?: string}`
  - Response: `{success: boolean}`

#### Business Intelligence & Analytics

**Base Path**: `/api/v1/analytics`

- `GET /price-variance` - Compare vendor prices vs internal pricing
  - **Authenticated**
  - Uses: `products.list_price` vs `invoice_line_items.unit_price`
  - Response: `{variances: PriceVariance[], summary: object}`

- `GET /vendor-performance` - Vendor pricing trends
  - **Authenticated**
  - Uses: `companies` and `invoices` data
  - Response: `{vendors: VendorPerformance[], trends: object}`

- `GET /sales-territory-performance` - Territory-based sales analysis
  - **Authenticated**
  - Uses: `sales_territories` and `invoices`
  - Response: `{territories: TerritoryStats[], summary: object}`

- `GET /product-category-analysis` - Category performance
  - **Authenticated**
  - Uses: `product_categories` and `invoice_line_items`
  - Response: `{categories: CategoryStats[], trends: object}`

#### Data Management & Administration

**Base Path**: `/api/v1`

- `GET /companies` - Customer/company management
  - **Authenticated** (users: read only, admins: full access)
  - Response: `{companies: Company[]}`

- `POST /companies` - Create/update company records
  - **Admin only**
  - Request: `{company: Company}`
  - Response: `{success: boolean, company: Company}`

- `GET /products` - Product catalog with category hierarchy
  - **Authenticated** (users: read only, admins: full access)
  - Response: `{products: Product[], categories: Category[]}`

- `POST /products` - Create/update product catalog
  - **Admin only**
  - Request: `{product: Product}`
  - Response: `{success: boolean, product: Product}`

- `POST /products/bulk-import` - Bulk load product catalog
  - **Admin only**
  - Request: `{source: string, data: Product[]}`
  - Response: `{imported: number, failed: number, errors: string[]}`

- `GET /salespersons` - Sales team management
  - **Authenticated** (users: read only, admins: full access)
  - Response: `{salespersons: Salesperson[]}`

- `POST /salespersons` - Create/update salesperson records
  - **Admin only**

- `GET /audit-log` - Audit trail for compliance
  - **Admin only**
  - Query params: `user_id, table_name, action, date_from, date_to`
  - Response: `{logs: AuditLog[], pagination: PaginationInfo}`

- `GET /health` - System health checks
  - **Public**
  - Response: `{status: string, service: string, version: string}`

- `GET /health/detailed` - Detailed health with dependencies
  - **Public**
  - Response: `{status: string, checks: {database: boolean, redis: boolean, qdrant: boolean}}`

#### Job Management

**Base Path**: `/api/v1/jobs`

- `GET /my-jobs` - User's processing jobs with pagination
  - **Authenticated**
  - Query params: `page, per_page, status`
  - Response: `{jobs: Job[], statistics: JobStats, pagination: PaginationInfo}`

- `GET /my-jobs/{id}` - Detailed job information
  - **Authenticated**
  - Response: `{job: Job, file_info: FileInfo, logs: string[]}`

- `GET /my-jobs/unread-count` - Count of unread completed jobs
  - **Authenticated**
  - Response: `{unread_count: number}`

- `POST /my-jobs/mark-as-read` - Mark jobs as viewed
  - **Authenticated**
  - Request: `{job_id?: uuid}` (omit for all)
  - Response: `{success: boolean, count: number}`

---

## Database Schema (PostgreSQL + pgvector)

### Multi-Environment Support

**Compatibility**:
- Schema works across Docker PostgreSQL (development) and Neon (production)
- Automated migration scripts for environment synchronization
- Vector operations optimized for both local and serverless

### Core Tables

#### User Management & Security

**`users`** - User authentication and authorization
```sql
- id (UUID, PK)
- email (VARCHAR, UNIQUE)
- name (VARCHAR)
- role (ENUM: 'user', 'admin')
- google_id (VARCHAR, UNIQUE)
- access_code (VARCHAR) -- invitation codes
- is_active (BOOLEAN)
- created_at (TIMESTAMP)
- last_login (TIMESTAMP)
```

**`user_sessions`** - JWT token management
```sql
- id (UUID, PK)
- user_id (UUID, FK → users)
- token_hash (VARCHAR)
- refresh_token_hash (VARCHAR)
- expires_at (TIMESTAMP)
- created_at (TIMESTAMP)
- last_accessed (TIMESTAMP)
```

**`file_storage`** - Cloud Storage metadata
```sql
- id (UUID, PK)
- user_id (UUID, FK → users)
- file_name (VARCHAR)
- file_size (INTEGER)
- mime_type (VARCHAR)
- blob_url (TEXT) -- Cloud Storage URL
- blob_path (TEXT)
- file_hash (VARCHAR) -- SHA-256 for deduplication
- upload_source (VARCHAR)
- processing_status (VARCHAR)
- created_at (TIMESTAMP)
```

**`processing_jobs`** - Async job queue with progress tracking
```sql
- id (UUID, PK)
- user_id (UUID, FK → users)
- file_storage_id (UUID, FK → file_storage)
- job_type (VARCHAR)
- status (ENUM: 'pending', 'running', 'completed', 'failed')
- progress (INTEGER)
- current_stage (VARCHAR)
- result_data (JSONB)
- error_message (TEXT)
- auto_save (BOOLEAN)
- cleanup (BOOLEAN)
- viewed_at (TIMESTAMP)
- created_at (TIMESTAMP)
- completed_at (TIMESTAMP)
```

**`file_access_log`** - Complete audit trail
```sql
- id (UUID, PK)
- user_id (UUID, FK → users)
- file_storage_id (UUID, FK → file_storage)
- action (VARCHAR) -- 'upload', 'download', 'delete'
- ip_address (VARCHAR)
- user_agent (TEXT)
- created_at (TIMESTAMP)
```

**`audit_log`** - Data change tracking
```sql
- id (UUID, PK)
- user_id (UUID, FK → users)
- table_name (VARCHAR)
- record_id (UUID)
- action (ENUM: 'CREATE', 'UPDATE', 'DELETE')
- old_values (JSONB)
- new_values (JSONB)
- reason (TEXT)
- created_at (TIMESTAMP)
```

#### Business Data

**`invoices`** - Invoice metadata and financials
```sql
- id (UUID, PK)
- uploaded_by_user_id (UUID, FK → users)
- invoice_number (VARCHAR)
- invoice_date (DATE)
- due_date (DATE)
- ship_date (DATE)
- customer_id (UUID, FK → companies)
- salesperson_id (UUID, FK → salespersons)
- territory_id (INTEGER, FK → sales_territories)
- account_number (VARCHAR)
- po_number (VARCHAR)
- subtotal (NUMERIC(10,2))
- tax_rate (NUMERIC(5,4))
- tax_amount (NUMERIC(10,2))
- freight (NUMERIC(10,2))
- total_amount (NUMERIC(10,2))
- order_status (INTEGER)
- payment_status (VARCHAR)
- confidence_score (NUMERIC(3,2))
- original_filename (VARCHAR)
- created_at (TIMESTAMP)
- updated_at (TIMESTAMP)
```

**`invoice_line_items`** - LLM-extracted products
```sql
- id (UUID, PK)
- invoice_id (UUID, FK → invoices)
- line_number (INTEGER)
- product_id (UUID, FK → products)
- item_number (VARCHAR)
- description (TEXT)
- quantity (INTEGER)
- unit_price (NUMERIC(10,2))
- unit_price_discount (NUMERIC(10,2))
- line_total (NUMERIC(10,2))
- confidence_score (NUMERIC(3,2))
- matched_by_embedding (BOOLEAN)
```

**`products`** - Master product catalog with embeddings
```sql
- id (UUID, PK)
- item_number (VARCHAR, UNIQUE)
- name (VARCHAR)
- description (TEXT)
- subcategory_id (INTEGER, FK → product_subcategories)
- color (VARCHAR)
- size (VARCHAR)
- standard_cost (NUMERIC(10,2))
- list_price (NUMERIC(10,2))
- name_embedding (VECTOR(1536)) -- for semantic matching
- description_embedding (VECTOR(1536))
```

**`companies`** - Customer/vendor information
```sql
- id (UUID, PK)
- customer_id (INTEGER, UNIQUE)
- company_type (VARCHAR)
- company_name (VARCHAR)
- street_address (TEXT)
- city (VARCHAR)
- state_province (VARCHAR)
- postal_code (VARCHAR)
- country_region (VARCHAR)
- phone (VARCHAR)
- territory_id (INTEGER, FK → sales_territories)
- name_embedding (VECTOR(1536))
- address_embedding (VECTOR(1536))
```

**`salespersons`** - Sales team data
```sql
- id (UUID, PK)
- salesperson_id (INTEGER, UNIQUE)
- name (VARCHAR)
- email (VARCHAR)
- employee_id (VARCHAR)
- territory_id (INTEGER, FK → sales_territories)
- name_embedding (VECTOR(1536))
```

**`sales_territories`** - Geographic organization
```sql
- territory_id (INTEGER, PK)
- name (VARCHAR)
- country_region_code (VARCHAR)
- territory_group (VARCHAR)
```

**`product_categories`** & **`product_subcategories`** - Product hierarchy
```sql
-- Categories: Bikes, Components, Clothing, Accessories
-- Subcategories: Mountain Bikes, Road Bikes, etc.
```

#### AI Chat Agent Tables (NEW)

**`chat_sessions`** - Chat conversation sessions
```sql
- id (UUID, PK)
- user_id (UUID, FK → users)
- title (VARCHAR)
- created_at (TIMESTAMP)
- updated_at (TIMESTAMP)
```

**`chat_messages`** - Individual messages
```sql
- id (UUID, PK)
- session_id (UUID, FK → chat_sessions)
- role (ENUM: 'user', 'assistant', 'system')
- content (TEXT)
- metadata (JSONB) -- tool calls, citations, etc.
- tokens_used (INTEGER)
- created_at (TIMESTAMP)
```

**`chat_feedback`** - User ratings and feedback
```sql
- id (UUID, PK)
- message_id (UUID, FK → chat_messages)
- user_id (UUID, FK → users)
- rating (INTEGER) -- 1-5 stars
- comment (TEXT)
- created_at (TIMESTAMP)
```

### Vector Operations & Indexes

**Embedding Dimensions**: 1536 (OpenAI text-embedding-3-small)

**Vector Indexes**:
```sql
CREATE INDEX idx_products_name_embedding ON products
  USING ivfflat (name_embedding vector_cosine_ops) WITH (lists = 100);

CREATE INDEX idx_products_description_embedding ON products
  USING ivfflat (description_embedding vector_cosine_ops) WITH (lists = 100);

CREATE INDEX idx_companies_name_embedding ON companies
  USING ivfflat (name_embedding vector_cosine_ops) WITH (lists = 50);
```

**Vector Search Functions**:
```sql
-- Semantic product matching
SELECT id, name,
  1 - (name_embedding <=> query_vector) AS similarity
FROM products
ORDER BY name_embedding <=> query_vector
LIMIT 5;
```

---

## AI & ML Integration

### LLM Integration Strategy

**Multi-Provider Architecture**:
1. **OpenAI GPT-4** - Primary LLM for vision, chat, text-to-SQL
2. **Anthropic Claude** - Complex reasoning and long-context tasks
3. **Hugging Face** - Cost-effective fallback (Llama/Mistral)

**Embedding Model**: OpenAI text-embedding-3-small (1536 dimensions)

### Invoice Processing Pipeline

**Multi-Stage LLM Approach**:

1. **Document Analysis** - GPT-4 Vision analyzes invoice layout
2. **Product Matching** - Vector search + LLM reasoning
3. **Tool Calling** - LLM directly inserts validated data
4. **Business Logic** - Domain knowledge for accuracy

**Example LLM Tools**:

```python
def insert_extracted_invoice_data(invoice_data: dict) -> str:
    """Tool for LLM to insert validated invoice data"""
    # LLM decides readiness for insertion
    # Handles confidence thresholds and validation
    # Returns success/failure for LLM decision making

def match_company_by_embedding(company_name: str, address: str) -> UUID:
    """Vector similarity to find/create company"""
    # Searches companies.name_embedding and address_embedding
    # Returns company UUID for association

def match_product_by_embedding(description: str) -> UUID:
    """Vector similarity product matching"""
    # Searches products.name_embedding and description_embedding
    # Returns product UUID for line item

def validate_invoice_totals(invoice_uuid: UUID) -> bool:
    """Validate calculated totals"""
    # Uses validate_invoice_totals_enhanced() DB function
    # Ensures data integrity
```

### AI Chat Agent Architecture (NEW)

**Chat Service Components**:

1. **Intent Router** - Classifies user intent
   - Documentation query → RAG Service
   - Data query → Text-to-SQL Service
   - Action request → Tool Executor
   - Status check → Job Service

2. **RAG Service** - Document retrieval
   - Searches Qdrant vector DB
   - Retrieves frontend/docs + backend/docs
   - Re-ranks results by relevance
   - Generates answer with citations

3. **Text-to-SQL Service** - Natural language queries
   - Converts user question to SQL
   - Validates query permissions
   - Executes safe, scoped queries
   - Formats results as table/JSON

4. **Stream Manager** - SSE event streaming
   - Progressive token display
   - Real-time user feedback
   - Error handling

**RAG Pipeline**:

```python
# Document Indexing (Offline)
def index_documentation():
    """Index all markdown docs to Qdrant"""
    for doc in scan_markdown_files(['frontend/docs', 'backend/docs']):
        chunks = chunk_document(doc, chunk_size=500, overlap=100)
        for chunk in chunks:
            embedding = embed_model.encode(chunk.content)
            qdrant.upsert(
                collection="documentation",
                points=[{
                    "vector": embedding,
                    "payload": {
                        "source": chunk.source,
                        "content": chunk.content,
                        "title": chunk.title,
                        "section": chunk.section
                    }
                }]
            )

# Query Time
def search_documentation(query: str, top_k: int = 5):
    """Search docs using vector similarity"""
    query_vector = embed_model.encode(query)
    results = qdrant.search(
        collection="documentation",
        query_vector=query_vector,
        limit=top_k,
        score_threshold=0.7
    )
    return results
```

**Text-to-SQL Pipeline**:

```python
def text_to_sql(user_query: str, user_id: UUID) -> dict:
    """Convert natural language to SQL"""
    # 1. Generate SQL with LLM
    sql = llm.generate_sql(
        query=user_query,
        schema=get_database_schema(),
        examples=get_sql_examples()
    )

    # 2. Validate and inject user scope
    validated_sql = validate_and_scope_sql(sql, user_id)

    # 3. Execute safely
    results = execute_safe_query(validated_sql)

    # 4. Format for display
    return format_as_table(results)
```

---

## GCP Deployment Architecture

### Production Infrastructure (GCP)

**Compute Layer**:
- **Cloud Run API Service**: 2-20 instances, 2vCPU, 4GB RAM
- **Cloud Run Worker Service**: 1-10 instances, 1vCPU, 2GB RAM
- **Cloud Run Qdrant Service**: 1-3 instances, 2vCPU, 8GB RAM + 50GB SSD

**Data Layer**:
- **Neon PostgreSQL**: Serverless database (AWS us-east-1, VPC peering to GCP)
- **Memorystore Redis**: 2GB HA for cache and sessions
- **Cloud Storage**: Document storage with lifecycle policies
- **Qdrant**: Vector database on persistent SSD

**Networking**:
- **Cloud Load Balancer**: Global HTTPS with Cloud Armor WAF
- **VPC Network**: Private IPs with Serverless VPC Connector
- **Cloud NAT**: Egress gateway for external services

**Managed Services**:
- **Cloud Tasks**: Job queue with retry and DLQ
- **Secret Manager**: API keys and credentials
- **Cloud Operations**: Logging, Monitoring, Trace, Error Reporting

### Development Infrastructure (Docker)

**Local Stack**:
```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16
    ports: ["5433:5432"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  qdrant:
    image: qdrant/qdrant:latest
    ports: ["6333:6333", "6334:6334"]

  api:
    build: ./backend/api
    ports: ["5000:5000"]

  frontend:
    build: ./frontend
    ports: ["3000:3000"]
```

### Deployment Strategy

**CI/CD Pipeline** (GitHub Actions → Cloud Build):

1. **Build Phase**:
   - Run tests (pytest, jest)
   - Build Docker images
   - Push to Artifact Registry

2. **Deploy Phase**:
   - Deploy to Cloud Run (blue-green)
   - Run migrations (Alembic)
   - Smoke tests
   - Traffic shift (10% → 100%)

3. **Monitor Phase**:
   - Cloud Monitoring dashboards
   - Error rate alerts
   - Performance tracking

**Database Migration Strategy**:

```bash
# Development → Production
1. Export from Docker PostgreSQL
2. Apply schema to Neon via migrations
3. Validate data integrity and vector ops
4. Update environment config
5. Deploy with production DB connection
```

---

## Security & Access Control

### Authentication Strategy

**Google OAuth Integration**:
- Users authenticate via Google OAuth
- Automatic account creation on first login
- Admin users generate invitation access codes

**Two-Tier Permission System**:
- **User Role**: Upload invoices, view own data, access analytics
- **Admin Role**: Full system access, user management, all data

### Session Management

**JWT Token Strategy**:
- Access tokens: 1-hour expiration
- Refresh tokens: 30-day expiration in `user_sessions`
- Automatic refresh within 10 minutes of expiration
- Session invalidation on logout or admin action

**Security Features**:
- Rate limiting: 1000 req/hour per user, 100 req/hour per IP (auth)
- File deduplication via SHA-256 hashing
- All file operations logged in `file_access_log`
- Admin audit trail for user management
- Daily session cleanup job

### Data Access Control

**File Ownership Model**:
- Users access only their uploaded files (unless admin)
- Admins have visibility into all system data
- Processing jobs inherit file owner permissions
- File sharing requires explicit admin assignment

**API Authorization**:
- All endpoints require valid JWT (except `/health` and auth)
- Middleware validates permissions before processing
- Resource ownership checked at database level
- Admin bypass for system management

**SQL Query Security** (NEW):
- All Text-to-SQL queries scoped to user
- Automatic injection of `WHERE user_id = :current_user`
- Query validation before execution
- No admin bypass in SQL generation

---

## Scaling & Performance Strategy

### Horizontal Scaling

**Auto-Scaling Policies**:

**API Service**:
- Min instances: 2 (high availability)
- Max instances: 20
- Scale on: CPU > 70%, Concurrency > 100 requests
- Scale down delay: 5 minutes

**Worker Service**:
- Min instances: 1
- Max instances: 10
- Scale on: CPU > 80%, Queue depth > 50 jobs

**Qdrant Service**:
- Min instances: 1 (0 for dev)
- Max instances: 3
- Scale on: Memory > 75%

### Performance Optimizations

**Database**:
- Connection pooling optimized for Neon
- Smaller pool sizes (5 connections) for serverless
- Query optimization with EXPLAIN ANALYZE
- Vector index tuning (ivfflat lists)

**Caching Strategy**:
- Redis for session data (TTL: 30 days)
- SQL query result caching (TTL: 5 minutes)
- Chat context caching (TTL: 1 hour)
- CDN for static assets

**Background Processing**:
- Async job processing via Cloud Tasks
- Queue prioritization (user-triggered > batch)
- Batch processing optimization

---

## Cost Optimization

### Monthly Cost Estimate (Production)

| Service | Usage | Monthly Cost |
|---------|-------|--------------|
| Cloud Run (API) | 2-20 instances, 2vCPU, 4GB | $150-$800 |
| Cloud Run (Worker) | 1-10 instances, 1vCPU, 2GB | $50-$300 |
| Cloud Run (Qdrant) | 1-3 instances, 2vCPU, 8GB | $100-$250 |
| Memorystore Redis | 2GB, HA | $50 |
| Neon Database | 100GB, 200 compute hours | $69 |
| Cloud Storage | 1TB, 1M operations | $26 |
| Cloud Load Balancer | 1TB egress | $25 |
| Cloud Logging | 50GB | $25 |
| Cloud Monitoring | Custom metrics | $10 |
| Vercel | Pro Plan | $20 |
| OpenAI API | 10M tokens | $100-$200 |
| **Total** | | **$625-$1,905/month** |

### Cost Optimization Strategies

1. **Serverless Scaling**: Cloud Run scales to zero when idle
2. **Neon Auto-Pause**: Database pauses after inactivity
3. **Smart Caching**: Reduce database and LLM calls
4. **Lifecycle Policies**: Move old files to Nearline/Coldline storage
5. **Reserved Capacity**: Commit to baseline usage for discounts

---

## Monitoring & Observability

### Key Metrics

**Application Metrics**:
- Request rate, latency (P50, P95, P99)
- Error rate (4xx, 5xx)
- WebSocket connection count
- Chat response latency
- SQL query performance

**Infrastructure Metrics**:
- CPU, memory, disk usage
- Database connection pool utilization
- Redis cache hit ratio
- Qdrant vector search latency

**Business Metrics**:
- Document processing throughput
- AI extraction accuracy
- Chat session duration
- User satisfaction (feedback ratings)

### Alert Policies

**Critical Alerts** (PagerDuty):
- API Service Down (3+ health check failures)
- Database connection pool exhausted (>90%)
- Error rate > 10%

**Warning Alerts** (Slack):
- High error rate (>5%)
- Slow response time (P95 > 1s)
- Low cache hit ratio (<70%)

---

## Implementation Roadmap

### Phase 1: Core Infrastructure (Weeks 1-2)
- ✅ Docker development environment
- ✅ PostgreSQL + pgvector schema
- ✅ Flask API with auth and invoice processing
- ✅ Next.js frontend with upload UI

### Phase 2: AI Chat Agent (Weeks 3-4) - NEW
- [ ] Deploy Qdrant vector database
- [ ] Implement RAG pipeline (indexing + search)
- [ ] Build chat service with intent routing
- [ ] Implement Text-to-SQL service
- [ ] Create chat UI with SSE streaming

### Phase 3: GCP Migration (Weeks 5-6)
- [ ] Set up GCP project and networking
- [ ] Deploy to Cloud Run
- [ ] Configure Neon DB connection
- [ ] Set up Memorystore Redis
- [ ] Cloud Operations monitoring

### Phase 4: Production Launch (Weeks 7-8)
- [ ] Load testing and optimization
- [ ] Security audit
- [ ] Documentation completion
- [ ] Production deployment
- [ ] User training

---

## Technical Decisions & Rationale

### Why GCP over Azure?

1. **Serverless Excellence**: Cloud Run has superior cold starts and pricing
2. **Cost Predictability**: Better pricing for variable workloads
3. **Global Network**: Best CDN integration with Vercel
4. **Developer Experience**: Simpler deployment, better CLI
5. **AI/ML Support**: Native support for streaming and AI workloads

### Why Neon for Production?

1. **Serverless Auto-Scaling**: Scales to zero when inactive
2. **Database Branching**: Instant staging/dev branches
3. **Built-in pgvector**: No extension management
4. **Cost Optimization**: Pay only for usage
5. **Global Distribution**: Low latency edge locations

### Why Docker for Development?

1. **Complete Control**: Full environment control
2. **Consistency**: Same across team members
3. **Offline Capability**: No external dependencies
4. **Full Monitoring**: Prometheus + Grafana stack
5. **Feature Parity**: Matches production exactly

### Why Qdrant for Vectors?

1. **Open Source**: No vendor lock-in
2. **Performance**: Excellent for our scale
3. **Cost-Effective**: Self-hosted on Cloud Run
4. **Rich Features**: Hybrid search, filtering
5. **Easy Backup**: Simple volume snapshots

---

## Appendix: Data Models

### TypeScript Interfaces

```typescript
// User & Auth
interface User {
  id: string;
  email: string;
  name: string;
  role: 'user' | 'admin';
  google_id: string;
  is_active: boolean;
  created_at: Date;
  last_login?: Date;
}

// Invoice Processing
interface Invoice {
  id: string;
  uploaded_by_user_id: string;
  invoice_number: string;
  invoice_date: Date;
  total_amount: number;
  confidence_score: number;
  // ... additional fields
}

interface ProcessingJob {
  id: string;
  user_id: string;
  status: 'pending' | 'running' | 'completed' | 'failed';
  progress: number;
  current_stage?: string;
  result_data?: object;
}

// AI Chat (NEW)
interface ChatSession {
  id: string;
  user_id: string;
  title?: string;
  created_at: Date;
}

interface ChatMessage {
  id: string;
  session_id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  metadata?: {
    tool_calls?: ToolCall[];
    citations?: Citation[];
    sql_query?: string;
  };
  tokens_used?: number;
  created_at: Date;
}

interface ToolCall {
  tool: string;
  args: object;
  result?: any;
}

interface Citation {
  source: string;
  title: string;
  url?: string;
}
```

---

**Document Version**: 2.0
**Last Updated**: 2025-10-20
**Platform**: Google Cloud Platform
**Database**: Neon PostgreSQL
**Status**: Design Complete → Implementation Ready
