# High-Level System Design: Case Study Invoice Intelligence

## Core Architecture Overview

This system processes vendor invoices through AI-powered extraction and provides business intelligence for procurement decisions. Built with containerized services and dual database support (Docker for development, Neon for production) for optimal developer experience and production scalability.

### Database Architecture Strategy

**Development Environment:**

- Docker PostgreSQL container with pgvector extension (port 5433)
- Local Redis for caching and background jobs (port 6379)
- Full monitoring stack (Prometheus, Grafana, AlertManager)
- Complete isolation for development and testing

**Production Environment:**

- Neon Database: Serverless PostgreSQL with auto-scaling and database branching
- Built-in pgvector support for vector operations
- Azure Cache for Redis for production workloads
- Both Flask API and Vercel frontend connect to the same Neon instance

**Configuration Management:**

- Automatic database type detection and connection optimization
- Environment-specific configurations (.env.development, .env.production)
- Smart connection pooling based on database provider (smaller pools for Neon serverless)
- SSL optimization for Neon, local optimization for Docker

## System Components & Responsibilities

### Flask API v0 Core Endpoints

**Authentication & User Management:**

- `POST /api/auth/login` - Google OAuth login with JWT token generation
- `POST /api/auth/logout` - Session termination and token invalidation
- `GET /api/auth/me` - Current user profile and permissions
- `POST /api/auth/refresh` - Refresh JWT token before expiration
- `GET /api/users` - User management dashboard (admin only)
- `POST /api/users/invite` - Invite new users with access codes (admin only)
- `PATCH /api/users/{id}/role` - Change user roles between 'user' and 'admin' (admin only)
- `PATCH /api/users/{id}/status` - Activate/deactivate user accounts (admin only)
- `DELETE /api/users/{id}` - Soft delete user accounts (admin only)
- `GET /api/audit/user-activity` - User activity audit trail (admin only)

**Document Processing Pipeline:**

- `POST /api/invoices/upload` - Accept invoice files (PDF/images), store in Azure Blob, trigger processing [user/admin]
- `GET /api/invoices/{id}/status` - Real-time processing status via WebSocket events [user/admin - own files only, admin - all files]
- `POST /api/invoices/{id}/process` - Manual trigger for reprocessing with different parameters [user/admin - own files only, admin - all files]

**AI-Powered Extraction:**

- `POST /api/extract/analyze-document` - Send document to GPT-4V for field extraction [user/admin]
- `POST /api/extract/match-products` - Use vector embeddings to match extracted items to existing products [user/admin]
- `POST /api/extract/validate-extraction` - LLM tool-calling to validate and insert data into database [user/admin]

**Business Intelligence:**

- `GET /api/analytics/price-variance` - Compare vendor prices against internal pricing using `products.list_price` vs `invoice_line_items.unit_price` [user/admin]
- `GET /api/analytics/vendor-performance` - Track vendor pricing trends using `companies` and `invoices` data [user/admin]
- `GET /api/analytics/sales-territory-performance` - Territory-based sales analysis using `sales_territories` and `invoices` [user/admin]
- `GET /api/analytics/product-category-analysis` - Category performance using `product_categories` and `invoice_line_items` [user/admin]
- `GET /api/search/natural-language` - Natural language queries over processed invoice data using vector embeddings [user/admin]

**Data Management & Administration:**

- `GET /api/invoices` - Paginated list with filtering and search using `invoice_details` view [user - own invoices only, admin - all invoices]
- `PATCH /api/invoices/{id}` - Manual corrections and overrides for invoice data [user - own invoices only, admin - all invoices]
- `GET /api/companies` - Customer/company management with address handling [user - read only, admin - full access]
- `POST /api/companies` - Create/update company records [admin only]
- `GET /api/products` - Product catalog management with category hierarchy [user - read only, admin - full access]
- `POST /api/products` - Create/update product catalog entries [admin only]
- `POST /api/products/bulk-import` - Load Case Study product catalog into `products`, `product_categories`, `product_subcategories` [admin only]
- `GET /api/salespersons` - Sales team management with territory assignments [user - read only, admin - full access]
- `POST /api/salespersons` - Create/update salesperson records [admin only]
- `GET /api/audit-log` - Audit trail access for compliance [admin only]
- `GET /api/health` - System health checks for load balancer [public]

### Database Schema (PostgreSQL + pgvector)

**Multi-Environment Support:**

- Schema compatibility across Docker PostgreSQL (development) and Neon (production)
- Automated migration scripts for environment synchronization
- Vector operations optimized for both local and serverless environments

**User Management & Security Tables:**

- `users` - User authentication with Google OAuth, role-based permissions (user/admin), access codes for invitations
- `user_sessions` - JWT token management with expiration tracking and session invalidation
- `file_storage` - Vercel Blob metadata with user ownership, file deduplication, and access control
- `processing_jobs` - Async processing job queue with user context and progress tracking
- `file_access_log` - Complete audit trail of all file operations with user attribution

**Business Data Tables:**

- `invoices` - Invoice metadata and processing status with financial totals, linked to user who uploaded
- `invoice_line_items` - LLM-extracted products with confidence scores and pricing
- `products` - Master product catalog with vector embeddings for semantic matching
- `companies` - Customer/company information with address embeddings
- `salespersons` - Sales team data with territory assignments
- `sales_territories` - Geographic organization structure
- `product_categories` & `product_subcategories` - Product hierarchy (Bikes, Components, Clothing, Accessories)
- `ship_methods` - Shipping options and rates
- `payments` - Payment tracking and history
- `document_processing_log` - LLM interaction logs for compliance and audit
- `extraction_rules` - Configurable extraction patterns and validation criteria
- `audit_log` - Complete audit trail for all data changes with user attribution

**Vector Operations:**

- Product name and description embeddings for semantic matching
- Company name and address embeddings for fuzzy customer identification
- Salesperson name embeddings for territory assignment
- Invoice content embeddings for full-text search and analysis
- Real-time similarity search during extraction and matching processes

**Key Database Features:**

- **Role-Based Access Control**: Two-tier user system (user/admin) with granular permissions on all data operations
- **Session Security**: JWT token management with automatic expiration and session invalidation capabilities
- **File Ownership**: All uploaded files linked to users with proper access control and deduplication via SHA-256 hashing
- **Audit Trail**: Complete `audit_log` and `file_access_log` tables track all data changes and file operations with user attribution
- **Financial Validation**: Built-in `validate_invoice_totals_enhanced()` function ensures calculated totals match stored values
- **Vector Indexes**: Optimized ivfflat indexes for fast similarity search across all embedding fields
- **Data Integrity**: Foreign key constraints ensure referential integrity between invoices, companies, products, and line items
- **Flexible Addresses**: `company_addresses` table supports multiple address types (billing, shipping, mailing)
- **Product Hierarchy**: Structured categories and subcategories for organized product management
- **Processing Metadata**: `document_processing_log` tracks LLM interactions and confidence scores with user context

## Security & Session Management

### Authentication Strategy

**Google OAuth Integration**: Users authenticate via Google OAuth with automatic account creation on first login. Admin users can generate invitation access codes for new team members.

**Two-Tier Permission System**:

- **User Role**: Standard business team members can upload invoices, view their own data, and access analytics dashboards
- **Admin Role**: Managers/IT personnel have full system access including user management, system configuration, and all data visibility

### Session Management

**JWT Token Strategy**:

- Access tokens with 1-hour expiration for security
- Refresh tokens with 30-day expiration stored in `user_sessions` table
- Automatic token refresh on API requests within 10 minutes of expiration
- Session invalidation on logout or admin action

**Security Features**:

- Rate limiting: 1000 requests/hour per user, 100 requests/hour per IP for auth endpoints
- File deduplication via SHA-256 hashing prevents duplicate uploads
- All file operations logged with user attribution in `file_access_log`
- Admin audit trail tracks all user management actions
- Session cleanup job removes expired tokens daily

### Data Access Control

**File Ownership Model**:

- Users can only access files they uploaded unless admin
- Admins have visibility into all system data for management purposes
- Processing jobs inherit user permissions from the file owner
- File sharing requires explicit admin assignment

**API Authorization**:

- All endpoints require valid JWT token except `/api/health` and auth endpoints
- Middleware validates user permissions before processing requests
- Resource ownership checked at database level using user_id foreign keys
- Admin bypass available for system management operations

## LLM Integration Strategy

### Prompt Engineering for Invoice Processing

The system uses a multi-stage LLM approach:

1. **Document Analysis**: GPT-4V analyzes invoice layout and extracts structured data
2. **Product Matching**: Vector search + LLM reasoning to match vendor items to internal catalog
3. **Tool Calling**: LLM directly inserts validated data into database using function calls
4. **Business Logic**: LLM applies bike industry knowledge for better matching accuracy

### Example LLM Tool Integration

```python
# LLM will have access to database insertion tools
def insert_extracted_invoice_data(invoice_data: dict) -> str:
    """Tool for LLM to insert validated invoice data directly into database"""
    # LLM decides what data is ready for insertion
    # Handles confidence thresholds and business rule validation
    # Returns success/failure status back to LLM for decision making

def match_company_by_embedding(company_name: str, address: str) -> UUID
    """Use vector similarity to find existing company or create new one"""
    # Searches companies.name_embedding and company_addresses.full_address_embedding
    # Returns company UUID for invoice association

def match_product_by_embedding(description: str) -> UUID
    """Use vector similarity to match extracted items to products catalog"""
    # Searches products.name_embedding and products.description_embedding
    # Returns product UUID for line item association

def validate_invoice_totals(invoice_uuid: UUID) -> bool
    """Validate calculated totals match stored values using database function"""
    # Uses validate_invoice_totals_enhanced() function from schema
    # Ensures data integrity before finalizing invoice
```

## Scaling Strategy: Prototype → Production

### Environment Progression Strategy

**Local Development (Docker):**

- Complete development stack with PostgreSQL, Redis, monitoring
- Full feature parity with production
- Offline development capability
- Rich debugging and monitoring tools

**Production Deployment:**

- Neon Database: Serverless PostgreSQL with auto-scaling
- Azure Container Apps: Auto-scaling Flask API and Celery workers
- Vercel Frontend: Edge deployment with same database connection
- Unified data layer across all production services

**Database Migration Strategy:**

```bash
# Development to Production Migration
1. Export from Docker PostgreSQL
2. Apply schema to Neon using migration scripts
3. Validate data integrity and vector operations
4. Switch environment configuration
5. Deploy with production database connection
```

## Key Technical Decisions

### Database Connection Management

**Development**: Docker PostgreSQL container with direct connections and connection pooling optimized for local development
**Production**: Neon serverless PostgreSQL with optimized connection pooling:

- Smaller pool sizes (5 connections) for serverless efficiency
- No overflow connections for predictable resource usage
- Shorter timeouts optimized for serverless cold starts
- SSL connections with proper certificate handling
- Automatic scaling based on usage patterns

### Database Provider Selection Rationale

**Why Neon for Production:**

- Serverless auto-scaling: Database scales to zero when inactive
- Database branching: Create staging/development branches instantly
- Built-in pgvector: No extension management required
- Cost optimization: Pay only for actual usage
- Global edge distribution for low latency

**Why Docker for Development:**

- Complete control over development environment
- Consistent experience across team members
- Offline development capability
- Full monitoring and debugging stack
- No external dependencies for development work

### Real-time Communication Strategy

**WebSocket Architecture**: Redis pub/sub handles real-time processing updates across multiple API instances
**Event Types**: `processing_started`, `extraction_complete`, `matching_complete`, `validation_complete`, `processing_error`

### AI Processing Pipeline

**Synchronous**: Simple uploads with immediate processing feedback
**Asynchronous**: Background processing for bulk uploads with WebSocket status updates
**Tool Integration**: LLM uses function calling to directly insert validated data into database

## Demonstration Flow

### Live Demo Scenarios (20 minutes total)

1. **Smart Extraction** (5 min): Upload business invoice image → Watch real-time processing → Show extracted structured data
2. **Intelligent Matching** (5 min): Demonstrate semantic product matching with Case Study catalog
3. **Business Intelligence** (5 min): Show price variance analysis and vendor performance insights
4. **Scaling Architecture** (5 min): Walk through containerized deployment and auto-scaling capabilities

### Technical Deep Dive Points

- **Multi-environment database strategy**: Seamless development with Docker, production with Neon
- **Smart configuration management**: Automatic database type detection and optimization
- **LLM prompt engineering abstractions** with 4 different industries as examples
- **Vector similarity search** for fuzzy product matching (pgvector in both environments)
- **Real-time WebSocket updates** during processing with Redis pub/sub
- **Container orchestration** for production scalability with Azure Container Apps
- **Database tool calling by LLM** for intelligent data insertion
- **Unified data layer**: Both Flask API and Vercel frontend use same Neon database
- **Cost optimization**: Serverless database scaling and development environment efficiency

### Current API Implementation Status

**Implemented Endpoints:**

- Health monitoring: `/health`, `/health/detailed`, `/health/database`
- Invoice management: Full CRUD with `/api/invoices/` endpoints
- Document processing: `/api/processing/` and `/api/webhook/` async processing
- Admin monitoring: `/api/admin/` comprehensive system management
- WebSocket real-time updates for processing status
- Prometheus metrics endpoint at `/metrics`

**Database Integration:**

- Automatic Neon DB detection and optimization in `config.py`
- Dual environment support with `.env.development` and `.env.production`
- Migration scripts: `migrate_to_neon.py` and `verify_neon.py`
- Docker profiles for both local (`full`) and production (`production`) modes

**Deployment Architecture:**

```
Development: Docker Stack → Local PostgreSQL + Redis
                ↓
Production: Azure Container Apps → Neon PostgreSQL + Azure Redis
              Vercel App → Same Neon Database
```

This architecture demonstrates both immediate prototype capability and clear production scaling path, with emphasis on developer experience optimization and AI-powered intelligence rather than just document extraction.
