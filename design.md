# CiviGuard AI - System Design Document

## 1. System Architecture Overview

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │   iOS App    │  │  Android     │  │   Web App    │  │    SMS API   │        │
│  │   (React      │  │  (React      │  │  (Next.js)   │  │  (Twilio)    │        │
│  │    Native)   │  │   Native)    │  │              │  │              │        │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘        │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           API GATEWAY LAYER                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                    AWS API Gateway (REST + WebSocket)                    │  │
│  │  - Authentication (Cognito)                                              │  │
│  │  - Rate Limiting                                                         │  │
│  │  - Request Routing                                                       │  │
│  │  - API Versioning                                                        │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         APPLICATION LAYER (Microservices)                       │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐             │
│  │   Report Service │  │   AI Processing  │  │   Routing &      │             │
│  │   (API)          │  │   Service        │  │   Escalation     │             │
│  │  - Report CRUD   │  │  - CV Analysis   │  │   Service        │             │
│  │  - User Mgmt     │  │  - Severity      │  │  - Department    │             │
│  │  - Status Update │  │    Prediction    │  │    Routing       │             │
│  └──────────────────┘  │  - Classification│  │  - Escalation    │             │
│                        └──────────────────┘  └──────────────────┘             │
│                                        │                                        │
│                                        ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                    Duplicate Detection Service                           │  │
│  │  - Spatial Deduplication                                                 │  │
│  │  - Temporal Deduplication                                                │  │
│  │  - Content Similarity (Embeddings)                                       │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           DATA LAYER                                            │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐             │
│  │   Amazon         │  │   Amazon         │  │   Amazon         │             │
│  │   DynamoDB       │  │   RDS (PostgreSQL)│  │   S3             │             │
│  │  - Reports       │  │  - Master Data   │  │  - Images        │             │
│  │  - User Sessions │  │  - Departments   │  │  - Videos        │             │
│  │  - Audit Logs    │  │  - Configuration │  │  - Attachments   │             │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘             │
│  ┌──────────────────┐  ┌──────────────────┐                                    │
│  │   Amazon         │  │   Amazon         │                                    │
│  │   OpenSearch     │  │   ElastiCache    │                                    │
│  │  - Full-text     │  │  - Session Cache   │                                   │
│  │    Search        │  │  - Embedding Cache │                                 │
│  └──────────────────┘  └────────���─────────┘                                    │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         EXTERNAL SERVICES                                       │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐             │
│  │   AWS            │  │   Twilio         │  │   Google Maps    │             │
│  │   Rekognition    │  │   (SMS)          │  │   API            │             │
│  │  - Image         │  │                  │  │  - Routing       │             │
│  │    Analysis      │  │                  │  │  - Geocoding     │             │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘             │
│  ┌──────────────────┐  ┌──────────────────┐                                    │
│  │   AWS            │  │   OpenAI/        │                                    │
│  │   Bedrock        │  │   HuggingFace    │                                    │
│  │  - LLM (Claude)  │  │   (Translation)  │                                    │
│  │  - TTS/STT       │  │                  │                                    │
│  └──────────────────┘  └──────────────────┘                                    │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Component Interaction Flow

1. **Report Submission**: User submits report via mobile/web app → API Gateway → Report Service
2. **AI Processing**: Report Service triggers AI Processing Service for image analysis
3. **Duplicate Check**: AI Processing Service queries Duplicate Detection Service
4. **Routing Decision**: Routing & Escalation Service determines responsible department
5. **Notification**: All services publish events to SNS for downstream processing

---

## 2. Layered Architecture

### 2.1 Frontend Layer

#### Mobile Applications (iOS/Android)
- **Framework**: React Native with TypeScript
- **Key Features**:
  - Camera integration with image optimization
  - GPS location capture
  - Offline mode with local SQLite storage
  - Push notifications via Firebase Cloud Messaging
  - Biometric authentication

#### Web Application
- **Framework**: Next.js 14 (App Router) with TypeScript
- **Key Features**:
  - Server-side rendering for SEO
  - Responsive design (mobile-first)
  - Admin dashboard with React Query
  - Real-time updates via WebSocket

#### API Gateway
- **AWS API Gateway** (REST + WebSocket)
- **Features**:
  - JWT validation via Cognito User Pools
  - Rate limiting (1000 requests/minute per user)
  - Request/response transformation
  - API versioning (`/api/v1/`)

### 2.2 AI Layer

#### Computer Vision Service
- **Image Processing Pipeline**:
  ```
  Upload → Preprocessing → Object Detection → Classification → Embedding Generation
  ```
- **Technologies**:
  - **Preprocessing**: OpenCV (Python)
  - **Object Detection**: YOLOv11 (Ultralytics) fine-tuned on civic hazard dataset
  - **Classification**: ResNet-50 with transfer learning
  - **Embeddings**: CLIP ViT-B/32 for semantic similarity

#### AI Processing Service
- **AWS Lambda Functions** (Python 3.12)
- **Functions**:
  - `process-image`: Main image analysis pipeline
  - `generate-severity`: Predicts severity level using XGBoost model
  - `generate-report`: Creates formal report using LLM
  - `translate-content`: Multilingual translation via Bedrock

#### LLM Integration
- **AWS Bedrock** (Anthropic Claude 3.5 Sonnet)
- **Use Cases**:
  - Report narrative generation
  - Formal language translation
  - Summarization of long descriptions
  - Sentiment analysis for escalation

### 2.3 Backend Layer

#### Microservices (AWS Fargate)
| Service | Port | Description |
|---------|------|-------------|
| report-service | 8080 | Report CRUD, user management |
| ai-service | 8081 | AI processing orchestration |
| routing-service | 8082 | Department routing, escalation |
| duplicate-service | 8083 | Deduplication logic |

#### Event-Driven Architecture
- **Amazon SNS** for pub/sub messaging
- **Amazon SQS** for reliable message delivery
- **Dead Letter Queues** for failed messages

### 2.4 Database Layer

#### Primary Database: Amazon RDS (PostgreSQL)
- **Instance**: db.r6g.large (2 vCPU, 16 GB RAM)
- **Storage**: 100 GB GP3 (3000 IOPS)
- **Multi-AZ**: Enabled
- **Tables**:
  - `users`, `reports`, `departments`, `report_history`, `audit_logs`

#### NoSQL Database: Amazon DynamoDB
- **Tables**:
  - `reports_by_user`: GSI on `userId, createdAt`
  - `reports_by_status`: GSI on `status, createdAt`
  - `sessions`: TTL-based session management
  - `audit_logs`: Time-series logging

#### Search: Amazon OpenSearch Service
- **Indices**: `reports`, `departments`
- **Features**: Full-text search, geospatial queries

#### Cache: Amazon ElastiCache (Redis)
- **Use Cases**:
  - Session storage
  - Embedding cache (90-day TTL)
  - Rate limiting counters

### 2.5 Cloud Infrastructure (AWS)

#### Regions & Availability
- **Primary Region**: US-East-1 (N. Virginia)
- **Secondary Region**: US-West-2 (Oregon) for DR
- **Compliance**: HIPAA-eligible, GDPR-compliant

#### Infrastructure as Code
- **Terraform** for AWS provisioning
- **CDK** for application deployment

---

## 3. Data Flow Explanation

### 3.1 Report Submission Flow

```
User → Mobile/Web App → API Gateway → Report Service → DynamoDB (temp)
                                      ↓
                              SNS Topic: report-created
                                      ↓
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
            AI Processing     Duplicate Check    Audit Log
                    ↓                 ↓
            S3 (Image)      DynamoDB (Final)
                    ↓
            LLM Report Gen
                    ↓
            Routing Service
                    ↓
            SNS: report-assigned
                    ↓
            Department System
```

### 3.2 Image Analysis Flow

```
Image Upload → S3 Bucket: civic-images-raw
                    ↓
            Lambda Trigger: process-image
                    ↓
            Preprocessing (OpenCV)
                    ↓
            YOLOv11 Detection → Bounding Boxes
                    ↓
            ResNet-50 Classification → Hazard Type
                    ↓
            CLIP Embedding → Semantic Vector
                    ↓
            XGBoost Model → Severity Score
                    ↓
            Bedrock LLM → Report Narrative
                    ↓
            S3 Bucket: civic-images-processed
            DynamoDB: report_data
```

### 3.3 Duplicate Detection Flow

```
New Report → Embedding Generated
                    ↓
            Redis: Check recent embeddings (1km, 1hr)
                    ↓
            OpenSearch: Full-text similarity
                    ↓
            DynamoDB: Spatial query (geohash)
                    ↓
            Similarity Score ≥ 0.85 → Duplicate
                    ↓
            SNS: duplicate-detected
                    ↓
            User Notification
```

### 3.4 Routing Flow

```
Report Created → Hazard Type + Location
                    ↓
            Department Mapping Table
                    ↓
            Google Maps API → Nearest Office
                    ↓
            Workload Balancing Algorithm
                    ↓
            Assign to Department
                    ↓
            SNS: report-assigned
                    ↓
            Work Order System Integration
```

---

## 4. Database Schema

### 4.1 PostgreSQL Schema (RDS)

```sql
-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20),
    name VARCHAR(255),
    language VARCHAR(10) DEFAULT 'en',
    role VARCHAR(20) DEFAULT 'citizen' CHECK (role IN ('citizen', 'department', 'admin')),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Departments
CREATE TABLE departments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) UNIQUE NOT NULL,
    address TEXT,
    phone VARCHAR(20),
    email VARCHAR(255),
    service_types JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Reports
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    department_id UUID REFERENCES departments(id),
    hazard_type VARCHAR(100) NOT NULL,
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('low', 'medium', 'high', 'critical')),
    status VARCHAR(20) NOT NULL DEFAULT 'reported' CHECK (status IN (
        'reported', 'verified', 'assigned', 'in_progress', 'resolved', 'closed', 'duplicate'
    )),
    location GEOGRAPHY(POINT, 4326) NOT NULL,
    location_description TEXT,
    ai_confidence DECIMAL(3,2),
    severity_score DECIMAL(3,2),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    resolved_at TIMESTAMPTZ,
    assigned_at TIMESTAMPTZ
);

-- Report Images
CREATE TABLE report_images (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID REFERENCES reports(id) ON DELETE CASCADE,
    s3_key VARCHAR(500) NOT NULL,
    original_filename VARCHAR(255) NOT NULL,
    file_size BIGINT,
    mime_type VARCHAR(100),
    processed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Report History (Audit Trail)
CREATE TABLE report_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID REFERENCES reports(id),
    status_from VARCHAR(20),
    status_to VARCHAR(20),
    user_id UUID REFERENCES users(id),
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Departments-Hazards Mapping
CREATE TABLE department_hazards (
    department_id UUID REFERENCES departments(id),
    hazard_type VARCHAR(100),
    PRIMARY KEY (department_id, hazard_type)
);

-- Audit Logs
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID,
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50),
    resource_id UUID,
    old_value JSONB,
    new_value JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 4.2 DynamoDB Tables

#### Table: `reports_by_user`
| Partition Key | Sort Key | Attributes |
|---------------|----------|------------|
| userId (UUID) | createdAt (ISO8601) | reportId, status, hazardType, severity |

#### Table: `reports_by_status`
| Partition Key | Sort Key | Attributes |
|---------------|----------|------------|
| status | severity#createdAt | reportId, userId, hazardType |

#### Table: `sessions`
| Partition Key | TTL | Attributes |
|---------------|-----|------------|
| sessionId | expiresAt | userId, token, ipAddress |

#### Table: `embeddings_cache`
| Partition Key | Sort Key | Attributes |
|---------------|----------|------------|
| reportId | embeddingType | vector (JSONB), createdAt |

---

## 5. API Structure

### 5.1 REST API Endpoints

#### Authentication
```
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password
```

#### Reports
```
GET    /api/v1/reports                    # List reports (paginated)
POST   /api/v1/reports                    # Create report
GET    /api/v1/reports/{id}               # Get report details
PUT    /api/v1/reports/{id}               # Update report
DELETE /api/v1/reports/{id}               # Delete report (citizen only)
GET    /api/v1/reports/{id}/images        # Get report images
POST   /api/v1/reports/{id}/images        # Add images to report
```

#### Departments
```
GET    /api/v1/departments                # List all departments
GET    /api/v1/departments/{id}           # Get department details
GET    /api/v1/departments/hazards        # Get departments by hazard type
```

#### AI Processing
```
POST   /api/v1/ai/process-image           # Submit image for analysis
GET    /api/v1/ai/severity/{reportId}     # Get severity prediction
GET    /api/v1/ai/classification/{id}     # Get classification details
```

#### Routing
```
GET    /api/v1/routing/nearest-department # Get nearest department
GET    /api/v1/routing/estimated-time     # Get estimated resolution time
```

#### Duplicate Detection
```
POST   /api/v1/duplicates/check           # Check for duplicates
GET    /api/v1/duplicates/{reportId}      # Get duplicate suggestions
```

### 5.2 Request/Response Examples

#### Create Report Request
```json
POST /api/v1/reports
Content-Type: application/json
Authorization: Bearer {token}

{
  "hazardType": "pothole",
  "location": {
    "type": "Point",
    "coordinates": [-73.9857, 40.7484]
  },
  "locationDescription": "Main St & 5th Ave intersection",
  "severity": "medium",
  "images": ["base64_encoded_image_data..."]
}
```

#### Create Report Response
```json
HTTP 201 Created

{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "123e4567-e89b-12d3-a456-426614174000",
  "hazardType": "pothole",
  "severity": "medium",
  "status": "reported",
  "aiConfidence": 0.92,
  "severityScore": 6.5,
  "createdAt": "2026-02-14T10:30:00Z",
  "assignedDepartment": {
    "id": "789e1234-e89b-12d3-a456-426614174000",
    "name": "Department of Public Works",
    "code": "DPW"
  }
}
```

#### AI Process Image Response
```json
HTTP 200 OK

{
  "reportId": "550e8400-e29b-41d4-a716-446655440000",
  "detections": [
    {
      "class": "pothole",
      "confidence": 0.95,
      "boundingBox": {
        "x": 120,
        "y": 80,
        "width": 200,
        "height": 150
      }
    }
  ],
  "severityPrediction": {
    "level": "medium",
    "score": 6.5,
    "factors": ["size: 15cm", "depth: 5cm", "location: high-traffic"]
  },
  "reportNarrative": "A medium-sized pothole approximately 15cm in diameter and 5cm deep is visible in the intersection area. The surrounding pavement shows signs of wear.",
  "embedding": "base64_encoded_vector..."
}
```

### 5.3 WebSocket Events

```
report-created: { reportId, userId, hazardType, status }
report-updated: { reportId, status, previousStatus, timestamp }
duplicate-detected: { reportId, duplicateOf, confidence }
department-assigned: { reportId, departmentId, assignedAt }
report-resolved: { reportId, resolvedBy, resolutionNotes }
```

---

## 6. Security Considerations

### 6.1 Authentication & Authorization

| Component | Implementation |
|-----------|----------------|
| User Auth | Cognito User Pools with MFA |
| API Auth | JWT tokens (HS256) with 1-hour expiry |
| Service Auth | IAM roles + SigV4 signing |
| Admin Auth | Cognito Admin Groups + SAML |

### 6.2 Data Protection

| Data Type | Encryption | Storage |
|-----------|------------|---------|
| At Rest | AES-256 | KMS-managed keys |
| In Transit | TLS 1.3 | AWS Load Balancer |
| Images | S3 SSE-S3 | Encrypted buckets |
| Databases | RDS Encryption | KMS integration |

### 6.3 Security Controls

| Control | Implementation |
|---------|----------------|
| Rate Limiting | API Gateway + Redis counters |
| Input Validation | OWASP ZAP + Custom validators |
| SQL Injection | Parameterized queries + Prisma |
| XSS | Content Security Policy + Helmet.js |
| CSRF | SameSite cookies + Token validation |
| Secrets | AWS Secrets Manager + Parameter Store |
| Logging | CloudWatch + GuardDuty integration |

### 6.4 Compliance

- **GDPR**: Data minimization, right to erasure, consent management
- **HIPAA**: BAA with AWS, PHI handling protocols
- **Section 508**: WCAG 2.1 AA compliance
- **SOC 2**: Audit-ready logging and monitoring

### 6.5 Security Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    WAF & Shield                             │
│  - DDoS Protection                                          │
│  - SQL Injection Blocking                                   │
│  - XSS Protection                                           │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              API Gateway + Lambda Authorizer                │
│  - JWT Validation                                           │
│  - RBAC Enforcement                                         │
│  - Rate Limiting                                            │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  VPC Security Groups                        │
│  - Private Subnets for Databases                            │
│  - Egress-only Internet Gateway                             │
│  - Security Groups (deny all by default)                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Scalability Strategy

### 7.1 Horizontal Scaling

| Component | Scaling Strategy | Trigger |
|-----------|------------------|---------|
| API Gateway | Auto-scaling | Request count |
| Lambda | Concurrent executions | Event queue depth |
| Fargate Services | ECS Auto Scaling | CPU/Memory > 70% |
| RDS | Read Replicas | Read latency > 100ms |
| S3 | Automatic | Object count |
| OpenSearch | Add nodes | Index size > 50GB |

### 7.2 Database Scaling

#### Read Scaling
- **Read Replicas**: 2 PostgreSQL read replicas
- **Caching**: Redis with 95% hit rate target
- **Connection Pooling**: PgBouncer

#### Write Scaling
- **Sharding Strategy**: By user ID hash
- **Async Writes**: SQS for non-critical operations
- **Batch Processing**: Kinesis for analytics

### 7.3 AI Processing Scaling

| Component | Scaling Approach |
|-----------|------------------|
| Image Processing | Lambda concurrency (1000 max) |
| Model Inference | SageMaker auto-scaling |
| Embedding Generation | Batch processing with SQS |
| LLM Calls | Rate limiting + Queue |

### 7.4 Performance Targets

| Metric | Target | Scaling Action |
|--------|--------|----------------|
| API Latency | <200ms p95 | Add Fargate tasks |
| Image Processing | <5s | Add Lambda concurrency |
| Search Latency | <100ms | Add OpenSearch nodes |
| Database Connections | <500 | Add read replicas |
| Concurrent Users | 10,000 | Auto-scale all layers |

### 7.5 Load Testing Strategy

- **Tool**: k6 + AWS Load Runner
- **Scenarios**:
  - 1000 concurrent users submitting reports
  - 5000 images uploaded in 1 hour
  - Peak traffic (5x normal) for 1 hour
- **Frequency**: Weekly in staging, monthly in production

---

## 8. Deployment Plan on AWS

### 8.1 Infrastructure as Code

```hcl
# Terraform Structure
terraform/
├── main.tf                    # Root module
├── variables.tf               # Input variables
├── outputs.tf                 # Output values
├── modules/
│   ├── vpc/                   # VPC, subnets, security groups
│   ├── database/              # RDS, DynamoDB, OpenSearch
│   ├── compute/               # Lambda, Fargate, ECS
│   ├── storage/               # S3 buckets
│   ├── networking/            # API Gateway, CloudFront
│   └── monitoring/            # CloudWatch, SNS, SQS
└── environments/
    ├── dev/
    ├── staging/
    └── production/
```

### 8.2 AWS Resource Breakdown

#### Compute
| Service | Count | Size | Purpose |
|---------|-------|------|---------|
| Lambda | 8 | 1024 MB | Image processing, report creation |
| Fargate Tasks | 4 | 2 vCPU, 4 GB | Microservices |
| ECS Services | 4 | 2x tasks | Report, AI, Routing, Duplicate |

#### Storage
| Service | Capacity | Purpose |
|---------|----------|---------|
| S3 | 1 TB | Images, videos, attachments |
| RDS | 100 GB | Primary database |
| DynamoDB | 50 GB | Sessions, caching |
| OpenSearch | 200 GB | Search index |

#### Networking
| Service | Configuration |
|---------|---------------|
| API Gateway | Regional, private endpoints |
| CloudFront | 2 distributions (web, static) |
| Load Balancer | 2 ALBs (public, private) |
| VPC | 2 AZs, 4 subnets |

### 8.3 CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: CiviGuard AI CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run unit tests
        run: npm test
      - name: Run integration tests
        run: npm run test:integration

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker images
        run: docker build -t civiguard/api .
      - name: Push to ECR
        run: aws ecr get-login-password | docker login --username AWS --password-stdin ...

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to AWS
        run: |
          terraform init -backend-config=backend.hcl
          terraform apply -auto-approve -var-file=../production.tfvars
```

### 8.4 Deployment Strategy

| Environment | Region | Instance Count | Purpose |
|-------------|--------|----------------|---------|
| Development | us-east-1a | 1 vCPU, 2 GB | Feature testing |
| Staging | us-east-1b | 2 vCPU, 4 GB | UAT, performance testing |
| Production | us-east-1 (Multi-AZ) | 4 vCPU, 8 GB | Live traffic |

### 8.5 Monitoring & Alerting

| Service | Metrics | Thresholds |
|---------|---------|------------|
| CloudWatch | CPU, Memory, Latency | >80% for 5 min |
| X-Ray | Request traces | Error rate >1% |
| CloudWatch Logs | Error patterns | 10+ errors/min |
| SNS | Alerts | Immediate notification |

### 8.6 Backup & Disaster Recovery

| Component | Frequency | Retention | Recovery Time |
|-----------|-----------|-----------|---------------|
| RDS | Every 6 hours | 30 days | <15 minutes |
| S3 | Versioning enabled | Indefinite | <1 hour |
| DynamoDB | On-demand backup | 30 days | <5 minutes |
| Full DR | Daily | 7 days | <4 hours |

### 8.7 Cost Optimization

| Strategy | Implementation |
|----------|----------------|
| Reserved Instances | 1-year RIs for RDS, OpenSearch |
| S3 Lifecycle | Move images to Glacier after 90 days |
| Lambda Concurrency | Auto-scale down to 0 during off-hours |
| Spot Instances | Batch processing jobs |
| CloudWatch Alarms | Auto-stop dev environments after hours |

---

## 9. Technical Stack Summary

| Layer | Technology |
|-------|------------|
| Mobile | React Native, TypeScript, Firebase |
| Web | Next.js 14, TypeScript, Tailwind CSS |
| API | AWS API Gateway, Lambda, Node.js |
| Backend | AWS Fargate, Express.js, TypeScript |
| AI/ML | Python, OpenCV, YOLOv11, CLIP, XGBoost |
| LLM | AWS Bedrock (Claude 3.5 Sonnet) |
| Database | PostgreSQL (RDS), DynamoDB, OpenSearch, Redis |
| Storage | S3, EBS |
| Infrastructure | Terraform, AWS CDK |
| CI/CD | GitHub Actions, ECR, ECS |
| Monitoring | CloudWatch, X-Ray, Datadog |
| Security | Cognito, KMS, WAF, Shield |

---

## 10. Future Enhancements

### Phase 2 (Post-Hackathon)
- Real-time video stream processing
- Predictive maintenance recommendations
- Citizen reward system
- Advanced analytics dashboard

### Phase 3
- IoT sensor integration
- Blockchain audit trail
- Predictive routing for response teams
- Public safety alert system
