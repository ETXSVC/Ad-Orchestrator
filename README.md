# Creative Ad Generation Orchestrator

A cloud-native, microservice-based digital advertisement generation system that leverages AI to create high-quality, brand-compliant advertising content with human oversight.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Key Features](#key-features)
- [System Components](#system-components)
- [Data Flow](#data-flow)
- [Installation](#installation)
- [Configuration](#configuration)
- [API Reference](#api-reference)
- [Database Schema](#database-schema)
- [Security](#security)
- [Deployment](#deployment)
- [Usage](#usage)
- [Development](#development)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## Overview

The Creative Ad Generation Orchestrator represents a fundamental shift from passive image analysis to an active, dual-modality generative process. The system creates unique visual assets and corresponding structured advertising copy through a sophisticated AI-powered pipeline with mandatory human-in-the-loop approval.

### What It Does

1. **Text-to-Image Generation**: Transforms descriptive prompts into unique, high-quality visual advertisement images
2. **Structured Text Generation**: Creates compelling copy (headlines, descriptions, keywords) perfectly aligned with the generated image
3. **Human-in-the-Loop Approval**: Ensures quality control through mandatory review before deployment
4. **Asset Management**: Securely stores and tracks all creative assets with complete auditability

## Architecture

### Three-Tier Microservice Architecture

The system follows a cloud-native, three-tier architecture with clean separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                     Frontend Interface                       │
│          (Data Capture & Human Interaction)                  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Backend (FastAPI)                         │
│        (Orchestration, Validation, State Management)         │
└─────────────────────────────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
    ┌──────────────┐ ┌─────────────┐ ┌──────────────┐
    │  Cloud SQL   │ │     GCS     │ │  Gemini API  │
    │ (PostgreSQL) │ │  (Storage)  │ │    (AI)      │
    └──────────────┘ └─────────────┘ └──────────────┘
```

### Design Principles

1. **Reliability**: Structured output enforcement via Pydantic and JSON Schema ensuring all 17 required fields conform to constraints
2. **Security**: API key authentication for public endpoints and Google Cloud ADC for internal service communication
3. **Immutability**: UUID-based naming for committed assets ensuring auditability and compliance

## Key Features

### Generate-Approve-Commit Workflow

The system implements a stateful, three-phase pipeline:

1. **Generate Phase**: 
   - Creates unique advertisement ID
   - Generates visual asset via Gemini
   - Produces structured text content
   - Stores in temporary cache

2. **Review Phase**:
   - Human reviews generated content
   - Assets remain in PENDING status
   - No permanent storage commitment

3. **Commit Phase**:
   - Atomic transaction upon approval
   - Permanent GCS upload with UUID naming
   - Database status update to APPROVED
   - Immutable record creation

### Dual-Modality Content Generation

The system performs two sequential generative tasks:

```
Text Prompt → [Gemini Image Gen] → Visual Asset
                                      ↓
                           [Gemini Text Gen] → Structured Copy
                                                (Title, Description, 15 Keywords)
```

### Structured Output Requirements

Every generated advertisement must include:
- **Title**: Primary advertisement text
- **Description**: Supporting copy with call-to-action
- **15 SEO Keywords**: Individual, discrete keyword fields

## System Components

### 1. Frontend Interface

**Responsibilities**:
- Structured data capture aligned with Advertisement Creation Template
- Asynchronous polling for generated assets
- Display generated content for human review
- Approval/rejection workflow management

**Key Inputs**:
- Campaign Overview
- Final Advertisement Copy requirements
- Design Specifications (including hex codes)

### 2. Backend (FastAPI)

**Responsibilities**:
- Request validation
- State management in database
- Generative AI orchestration
- Secure storage coordination

**Technology Stack**:
- **Framework**: FastAPI (async support, Pydantic integration)
- **Language**: Python 3.9+
- **Authentication**: API Keys + Google Cloud ADC

### 3. Persistence Layer (Cloud SQL PostgreSQL)

**Why PostgreSQL**:
- Strong data typing
- Referential integrity
- Efficient querying for 15 distinct keyword columns
- Superior to NoSQL for structured, columnar analysis

**Key Features**:
- ACID transaction support
- Complex query capabilities
- Robust schema enforcement

### 4. Storage Layer (Google Cloud Storage)

**Bucket Strategy**:
- `ad-creatives-temp`: PENDING assets with 7-day TTL
- `ad-creatives-approved`: Permanent production assets

**Benefits**:
- Physical separation of draft/production assets
- Simplified security policy application
- Automatic lifecycle management

### 5. Generative Core (Gemini API)

**Model**: `gemini-2.5-flash`

**Capabilities**:
- High-fidelity text-to-image generation
- Structured text generation with schema enforcement
- Fast, multimodal processing

## Data Flow

### Complete Pipeline

```
1. POST /api/v1/generate
   ├─ Generate UUID (ad_id)
   ├─ Create DB record (status: PENDING)
   ├─ Call Gemini: Text → Image
   ├─ Call Gemini: Image → Structured Text
   ├─ Cache assets temporarily
   └─ Return ad_id + temp asset URLs

2. Human Review
   ├─ Frontend retrieves PENDING content
   ├─ Display for user review
   └─ User approves/rejects

3. POST /api/v1/commit/{ad_id}
   ├─ Validate approval
   ├─ Generate UUID-based filename
   ├─ Upload to GCS (permanent bucket)
   ├─ Update DB (status: APPROVED, image_storage_path)
   └─ Return success confirmation
```

### State Transitions

```
PENDING → APPROVED  (upon user approval + successful commit)
PENDING → REJECTED  (upon user rejection)
```

## Installation

### Prerequisites

- Python 3.9+
- Google Cloud Project with enabled APIs:
  - Cloud SQL
  - Cloud Storage
  - Gemini API
- Docker (for containerized deployment)

### Local Development Setup

```bash
# Clone the repository
git clone https://github.com/your-org/ad-generation-orchestrator.git
cd ad-generation-orchestrator

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
cp .env.example .env
# Edit .env with your configuration
```

### Required Dependencies

```txt
fastapi>=0.104.0
uvicorn[standard]>=0.24.0
google-genai>=0.3.0
google-cloud-storage>=2.10.0
psycopg2-binary>=2.9.9
pydantic>=2.5.0
python-multipart>=0.0.6
```

## Configuration

### Environment Variables

Create a `.env` file:

```env
# API Configuration
API_KEY=your-api-key-here
API_HOST=0.0.0.0
API_PORT=8000

# Google Cloud
GOOGLE_CLOUD_PROJECT=your-project-id
GCS_BUCKET_TEMP=ad-creatives-temp
GCS_BUCKET_APPROVED=ad-creatives-approved

# Database
DATABASE_HOST=your-cloud-sql-instance
DATABASE_NAME=ad_generation
DATABASE_USER=your-db-user
DATABASE_PASSWORD=your-db-password

# Gemini API
GEMINI_MODEL=gemini-2.5-flash
GEMINI_IMAGE_MODEL=gemini-2.5-flash-image

# Security
JWT_SECRET_KEY=your-secret-key-here
```

### Google Cloud Authentication

Set up Application Default Credentials:

```bash
gcloud auth application-default login
gcloud config set project YOUR_PROJECT_ID
```

## API Reference

### POST /api/v1/generate

Generate a new advertisement with AI.

**Request Body**:
```json
{
  "visual_description": "Modern smartphone with gradient background",
  "campaign_name": "Q4 Mobile Launch",
  "target_audience": "Tech enthusiasts aged 25-40",
  "brand_voice": "Professional yet approachable",
  "design_specs": {
    "hex_codes": ["#FF6B6B", "#4ECDC4", "#45B7D1"],
    "aspect_ratio": "1:1"
  }
}
```

**Response**:
```json
{
  "ad_id": "f8a4e9b2c1d045a78f30c6e1a4b5d9e0",
  "status": "PENDING",
  "temp_image_url": "https://storage.googleapis.com/temp/...",
  "generated_content": {
    "title": "Revolutionary Mobile Experience",
    "description": "Transform your digital life with cutting-edge technology...",
    "keywords": ["smartphone", "innovation", "technology", ...]
  },
  "generation_timestamp": "2024-01-15T10:30:00Z"
}
```

### POST /api/v1/commit/{ad_id}

Commit an approved advertisement to permanent storage.

**Path Parameters**:
- `ad_id` (UUID): The advertisement ID to commit

**Response**:
```json
{
  "ad_id": "f8a4e9b2c1d045a78f30c6e1a4b5d9e0",
  "status": "APPROVED",
  "image_storage_path": "gs://ad-creatives-approved/f8a4e9b2c1d045a78f30c6e1a4b5d9e0.jpg",
  "approval_timestamp": "2024-01-15T10:35:00Z",
  "approval_latency_seconds": 300
}
```

### GET /api/v1/ads/{ad_id}

Retrieve advertisement details and status.

**Response**:
```json
{
  "ad_id": "f8a4e9b2c1d045a78f30c6e1a4b5d9e0",
  "status": "APPROVED",
  "title": "Revolutionary Mobile Experience",
  "description": "Transform your digital life...",
  "keywords": ["smartphone", "innovation", ...],
  "image_url": "https://storage.googleapis.com/...",
  "generation_timestamp": "2024-01-15T10:30:00Z",
  "approval_timestamp": "2024-01-15T10:35:00Z"
}
```

## Database Schema

### Table: ad_creative_metadata

```sql
CREATE TABLE ad_creative_metadata (
    ad_id UUID PRIMARY KEY,
    
    -- Administrative Inputs
    campaign_name VARCHAR(255) NOT NULL,
    client_name VARCHAR(255),
    target_audience TEXT,
    brand_voice VARCHAR(100),
    visual_description TEXT NOT NULL,
    
    -- Generated Content
    title VARCHAR(150) NOT NULL,
    description TEXT NOT NULL,
    
    -- 15 Individual Keyword Columns
    keyword_1 VARCHAR(50) NOT NULL,
    keyword_2 VARCHAR(50) NOT NULL,
    keyword_3 VARCHAR(50) NOT NULL,
    keyword_4 VARCHAR(50) NOT NULL,
    keyword_5 VARCHAR(50) NOT NULL,
    keyword_6 VARCHAR(50) NOT NULL,
    keyword_7 VARCHAR(50) NOT NULL,
    keyword_8 VARCHAR(50) NOT NULL,
    keyword_9 VARCHAR(50) NOT NULL,
    keyword_10 VARCHAR(50) NOT NULL,
    keyword_11 VARCHAR(50) NOT NULL,
    keyword_12 VARCHAR(50) NOT NULL,
    keyword_13 VARCHAR(50) NOT NULL,
    keyword_14 VARCHAR(50) NOT NULL,
    keyword_15 VARCHAR(50) NOT NULL,
    
    -- Asset Management
    image_storage_path TEXT,
    temp_image_path TEXT,
    
    -- State Management
    status VARCHAR(20) NOT NULL CHECK (status IN ('PENDING', 'APPROVED', 'REJECTED')),
    
    -- Audit Trail
    generation_timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    approval_timestamp TIMESTAMP WITH TIME ZONE,
    approved_by VARCHAR(255),
    
    -- Metadata
    design_hex_codes TEXT[],
    aspect_ratio VARCHAR(10),
    
    -- Indexes for common queries
    INDEX idx_status (status),
    INDEX idx_campaign (campaign_name),
    INDEX idx_generation_time (generation_timestamp),
    INDEX idx_approval_time (approval_timestamp)
);
```

### Key Design Features

1. **UUID Primary Key**: Ensures global uniqueness and immutability
2. **Flattened Keywords**: 15 individual columns for efficient querying
3. **Dual Timestamps**: Tracks generation and approval for latency analysis
4. **Status Enforcement**: CHECK constraint ensures valid state transitions
5. **Audit Trail**: Complete history of who approved and when

## Security

### Authentication Layers

#### Layer 1: Frontend-to-Backend

**API Key Authentication**:
```python
from fastapi import Security, HTTPException
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key")

async def verify_api_key(api_key: str = Security(api_key_header)):
    if api_key != settings.API_KEY:
        raise HTTPException(status_code=403, detail="Invalid API Key")
    return api_key
```

**Benefits**:
- Protects expensive generative endpoints
- Prevents unauthorized access and DoS attacks
- Financial governance for per-request billing

#### Layer 2: Backend-to-Cloud

**Application Default Credentials (ADC)**:
```python
from google.cloud import storage
from google.oauth2 import service_account

# Automatic credential discovery
storage_client = storage.Client()  # Uses ADC
```

**Benefits**:
- No hardcoded credentials
- Environment-managed security
- Automatic credential rotation
- Principle of least privilege

### Additional Security Measures

1. **Rate Limiting**: Prevent abuse and DoS attacks
2. **Input Validation**: Strict Pydantic model validation
3. **CORS Configuration**: Controlled cross-origin requests
4. **Secure Storage**: Separate buckets with strict IAM policies

## Deployment

### Docker Deployment

**Dockerfile**:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    libpq-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Expose port
EXPOSE 8000

# Run application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Build and Run**:
```bash
# Build image
docker build -t ad-generation-api:latest .

# Run container
docker run -p 8000:8000 \
  -e GOOGLE_APPLICATION_CREDENTIALS=/secrets/key.json \
  -v /path/to/key.json:/secrets/key.json \
  ad-generation-api:latest
```

### Google Cloud Run Deployment

```bash
# Build and deploy
gcloud builds submit --tag gcr.io/PROJECT_ID/ad-generation-api

gcloud run deploy ad-generation-api \
  --image gcr.io/PROJECT_ID/ad-generation-api \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars "GCS_BUCKET_TEMP=ad-creatives-temp,GCS_BUCKET_APPROVED=ad-creatives-approved"
```

### Kubernetes Deployment

**deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ad-generation-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ad-generation-api
  template:
    metadata:
      labels:
        app: ad-generation-api
    spec:
      containers:
      - name: api
        image: gcr.io/PROJECT_ID/ad-generation-api:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_HOST
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: host
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

## Usage

### Basic Workflow Example

```python
import requests
import time

API_BASE = "https://your-api.com/api/v1"
API_KEY = "your-api-key"
HEADERS = {"X-API-Key": API_KEY}

# Step 1: Generate advertisement
generate_payload = {
    "visual_description": "Professional office workspace with laptop",
    "campaign_name": "Productivity Suite Launch",
    "target_audience": "Business professionals",
    "design_specs": {
        "hex_codes": ["#2C3E50", "#3498DB"],
        "aspect_ratio": "16:9"
    }
}

response = requests.post(
    f"{API_BASE}/generate",
    json=generate_payload,
    headers=HEADERS
)
ad_data = response.json()
ad_id = ad_data["ad_id"]

print(f"Generated ad: {ad_id}")
print(f"Title: {ad_data['generated_content']['title']}")
print(f"Temp Image: {ad_data['temp_image_url']}")

# Step 2: Human review (simulated)
print("\n--- Human Review Required ---")
print(f"Status: {ad_data['status']}")
user_approval = input("Approve this ad? (y/n): ")

# Step 3: Commit if approved
if user_approval.lower() == 'y':
    commit_response = requests.post(
        f"{API_BASE}/commit/{ad_id}",
        headers=HEADERS
    )
    commit_data = commit_response.json()
    
    print(f"\n✓ Ad committed successfully!")
    print(f"Status: {commit_data['status']}")
    print(f"Storage: {commit_data['image_storage_path']}")
    print(f"Approval latency: {commit_data['approval_latency_seconds']}s")
else:
    print("Ad rejected - will remain in PENDING status")
```

### Python Client Library Example

```python
from ad_generation_client import AdGenerationClient

client = AdGenerationClient(
    api_key="your-api-key",
    base_url="https://your-api.com/api/v1"
)

# Generate ad
ad = client.generate_ad(
    visual_description="Sunset beach scene",
    campaign_name="Summer Collection 2024",
    hex_codes=["#FF6B6B", "#FFE66D"]
)

# Review
print(f"Title: {ad.title}")
print(f"Keywords: {', '.join(ad.keywords)}")

# Commit
if ad.status == "PENDING":
    result = client.commit_ad(ad.id)
    print(f"Committed: {result.image_url}")
```

## Development

### Running Tests

```bash
# Install dev dependencies
pip install pytest pytest-asyncio pytest-cov

# Run all tests
pytest

# Run with coverage
pytest --cov=app --cov-report=html

# Run specific test file
pytest tests/test_api.py -v
```

### Code Quality

```bash
# Format code
black app/

# Lint
flake8 app/
pylint app/

# Type checking
mypy app/
```

### Local Development Server

```bash
# Run with auto-reload
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# With specific log level
uvicorn main:app --reload --log-level debug
```

## Troubleshooting

### Common Issues

#### 1. Validation Error: Keywords Count Mismatch

**Problem**: Gemini returns incorrect number of keywords

**Solution**:
```python
# Implement retry logic with explicit schema enforcement
for attempt in range(3):
    response = gemini_client.generate_content(
        model="gemini-2.5-flash",
        contents=prompt,
        generation_config={
            "response_schema": SocialMediaContent.model_json_schema()
        }
    )
    
    try:
        validated = SocialMediaContent.model_validate_json(response.text)
        break
    except ValidationError as e:
        if attempt == 2:
            raise
        continue
```

#### 2. GCS Upload Failure

**Problem**: Assets not uploading to permanent bucket

**Solution**:
- Verify bucket permissions
- Check ADC configuration
- Ensure bucket exists

```bash
# Verify ADC
gcloud auth application-default print-access-token

# Check bucket access
gsutil ls gs://ad-creatives-approved/
```

#### 3. Database Connection Timeouts

**Problem**: Cloud SQL connection pooling issues

**Solution**:
```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    database_url,
    poolclass=QueuePool,
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=3600
)
```

#### 4. High Approval Latency

**Problem**: Slow human review process

**Diagnostic**:
```sql
-- Query approval latency metrics
SELECT 
    AVG(EXTRACT(EPOCH FROM (approval_timestamp - generation_timestamp))) as avg_latency,
    MAX(EXTRACT(EPOCH FROM (approval_timestamp - generation_timestamp))) as max_latency,
    COUNT(*) as total_ads
FROM ad_creative_metadata
WHERE approval_timestamp IS NOT NULL
AND generation_timestamp > NOW() - INTERVAL '7 days';
```

## Performance Optimization

### Caching Strategy

```python
from functools import lru_cache
import redis

# Redis cache for temp assets
redis_client = redis.Redis(host='localhost', port=6379)

def cache_temp_asset(ad_id: str, image_bytes: bytes, ttl: int = 3600):
    redis_client.setex(
        f"temp:image:{ad_id}",
        ttl,
        image_bytes
    )

def get_temp_asset(ad_id: str) -> bytes:
    return redis_client.get(f"temp:image:{ad_id}")
```

### Database Query Optimization

```sql
-- Create indexes for common queries
CREATE INDEX idx_status_generation ON ad_creative_metadata(status, generation_timestamp DESC);
CREATE INDEX idx_campaign_status ON ad_creative_metadata(campaign_name, status);

-- Composite index for keyword analysis
CREATE INDEX idx_keywords_composite ON ad_creative_metadata(keyword_1, keyword_2, keyword_3);
```

### Async Processing

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

async def parallel_generation(requests: list):
    with ThreadPoolExecutor(max_workers=5) as executor:
        loop = asyncio.get_event_loop()
        tasks = [
            loop.run_in_executor(executor, generate_ad, req)
            for req in requests
        ]
        return await asyncio.gather(*tasks)
```

## Monitoring and Observability

### Key Metrics to Track

1. **Generation Latency**: Time to complete AI generation
2. **Approval Latency**: Time from generation to approval
3. **Validation Failure Rate**: Pydantic validation errors
4. **GCS Upload Success Rate**: Asset commitment success
5. **API Response Times**: P50, P95, P99 percentiles

### Logging Configuration

```python
import logging
from google.cloud import logging as cloud_logging

# Set up Cloud Logging
client = cloud_logging.Client()
client.setup_logging()

# Configure structured logging
logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

# Log with context
logger.info(
    "Ad generation completed",
    extra={
        "ad_id": ad_id,
        "generation_time_ms": elapsed_ms,
        "status": "PENDING"
    }
)
```

## Contributing

We welcome contributions! Please follow these guidelines:

### Development Process

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Write/update tests
5. Ensure all tests pass
6. Commit with conventional commits (`git commit -m 'feat: add amazing feature'`)
7. Push to your fork
8. Open a Pull Request

### Code Standards

- Follow PEP 8 style guide
- Write docstrings for all functions
- Maintain test coverage above 80%
- Update documentation for new features

### Commit Message Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types**: feat, fix, docs, style, refactor, test, chore

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Support

For issues and questions:

- GitHub Issues: [Create an issue](https://github.com/your-org/ad-generation-orchestrator/issues)
- Documentation: [Full docs](https://docs.your-org.com)
- Email: support@your-org.com

## Acknowledgments

- Google Cloud Platform for infrastructure
- Gemini API for generative AI capabilities
- FastAPI framework for efficient async API development
- PostgreSQL for robust data persistence

---

**Version**: 1.0.0  
**Last Updated**: January 2025  
**Maintained by**: Your Organization
