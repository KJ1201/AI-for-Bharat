# Design Document: SarkariSaathi
## Voice AI for Navigating Government Schemes

> **Technical Design Specification**

---

## 1. Overview

### 1.1 Purpose
This document provides the technical design specification for SarkariSaathi, a voice-first AI assistant that enables users to discover and apply for government benefits through natural conversations in their native language.

### 1.2 Scope
This design covers:
- System architecture and component design
- AWS services integration
- Data models and schemas
- API specifications
- Voice processing pipeline
- RAG (Retrieval-Augmented Generation) implementation
- Security and compliance architecture
- Deployment and infrastructure

### 1.3 Design Principles
- **Voice-First**: Natural conversation as primary interface
- **Accuracy**: RAG-powered responses with zero hallucinations
- **Accessibility**: Multi-language support for digital inclusion
- **Scalability**: Serverless architecture for elastic scaling
- **Security**: DPDP Act 2023 compliant with data localization
- **Performance**: Sub-3-second response latency

---

## 2. System Architecture

### 2.1 High-Level Architecture

The system follows a serverless, event-driven architecture with the following layers:

**User Interface Layer**
- Mobile applications (Android, iOS)
- Progressive web application
- WhatsApp bot integration

**Edge & API Layer**
- Amazon CloudFront (CDN)
- Amazon API Gateway (REST/WebSocket)
- Amazon Cognito (Authentication)

**AI/ML Processing Layer**
- AWS Transcribe (Speech-to-Text)
- Amazon Lex (Intent Recognition)
- Amazon Bedrock (LLM & Embeddings)
- AWS Polly (Text-to-Speech)
- Amazon Comprehend (Entity Extraction)

**Business Logic Layer**
- AWS Lambda (Serverless Functions)
- AWS Step Functions (Workflow Orchestration)

**Data Layer**
- Amazon DynamoDB (User Profiles, Sessions)
- Amazon RDS PostgreSQL (Scheme Database)
- Amazon OpenSearch (Vector Search)
- Amazon S3 (Document Storage)

**Monitoring & Security**
- AWS CloudWatch (Logging & Metrics)
- AWS X-Ray (Distributed Tracing)
- AWS Secrets Manager (Credential Management)
- AWS IAM (Access Control)

### 2.2 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                      User Interface Layer                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │  Mobile  │  │   Web    │  │ WhatsApp │  │   IVR    │       │
│  │   App    │  │   App    │  │   Bot    │  │  System  │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Edge & API Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  CloudFront  │  │ API Gateway  │  │   Cognito    │         │
│  │     CDN      │  │   REST/WS    │  │     Auth     │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   AI/ML Processing Layer                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │Transcribe│  │   Lex    │  │ Bedrock  │  │  Polly   │       │
│  │ Speech2T │  │  Intent  │  │LLM+Embed │  │ Text2Sp  │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Business Logic Layer                           │
│  ┌──────────────────────┐  ┌──────────────────────┐            │
│  │   AWS Lambda         │  │  Step Functions      │            │
│  │  (API Handlers)      │  │  (RAG Orchestration) │            │
│  └──────────────────────┘  └──────────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Data Layer                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ DynamoDB │  │   RDS    │  │OpenSearch│  │    S3    │       │
│  │ Profiles │  │ Schemes  │  │  Vector  │  │Documents │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Data Flow

**Voice Query Flow:**
1. User speaks query → Mobile/Web app captures audio
2. Audio sent to API Gateway → Lambda function
3. Lambda invokes AWS Transcribe → Text output
4. Text sent to Amazon Lex → Intent + entities extracted
5. Lambda builds context from DynamoDB session
6. Step Function orchestrates RAG pipeline:
   - Generate embedding (Bedrock Titan)
   - Vector search (OpenSearch)
   - Retrieve scheme details (RDS + S3)
   - Apply eligibility filters
7. Lambda constructs prompt with context
8. Bedrock (Claude 3.5) generates response
9. AWS Polly converts to speech
10. Audio response returned to user

---

## 3. Component Design

### 3.1 Voice Processing Pipeline

**3.1.1 Speech-to-Text (AWS Transcribe)**

**Configuration:**
```json
{
  "languageCode": "hi-IN",
  "mediaSampleRateHertz": 16000,
  "enableAutomaticPunctuation": true,
  "vocabularyName": "government-schemes-vocab",
  "customVocabulary": [
    "Mukhyamantri", "Kanyadan", "Yojana", "Pradhan Mantri",
    "Aadhaar", "DigiLocker", "E-Mitra", "Jan Seva Kendra"
  ]
}
```

**Implementation:**
- Use streaming API for real-time transcription
- Support 10+ Indian languages (Hindi, English, Tamil, Telugu, Bengali, Marathi, Gujarati, Kannada, Malayalam, Punjabi)
- Automatic language detection
- Custom vocabulary for government terminology
- Confidence scoring for quality control

**3.1.2 Intent Recognition (Amazon Lex)**

**Bot Configuration:**
- Bot Name: SarkariSaathiBot
- Languages: Multi-language support
- Session timeout: 15 minutes

**Intents:**
1. **SearchScheme**: User looking for schemes
   - Utterances: "scheme chahiye", "benefits milenge", "yojana batao"
   - Slots: category, lifeEvent, location, occupation

2. **CheckEligibility**: Check if eligible for scheme
   - Utterances: "eligible hoon", "qualify karunga", "milega mujhe"
   - Slots: schemeId, age, income, caste

3. **FindOffice**: Locate application centers
   - Utterances: "office kahan hai", "center batao", "apply kahan karein"
   - Slots: schemeId, location

4. **TrackApplication**: Check application status
   - Utterances: "status check", "application kahan hai", "approved hua"
   - Slots: applicationId

5. **GetDocuments**: List required documents
   - Utterances: "documents chahiye", "papers kya lagenge"
   - Slots: schemeId

**3.1.3 Text-to-Speech (AWS Polly)**

**Configuration:**
```json
{
  "voiceId": "Aditi",
  "engine": "neural",
  "languageCode": "hi-IN",
  "outputFormat": "mp3",
  "sampleRate": "24000",
  "textType": "ssml"
}
```

**Voice Selection:**
- Hindi: Aditi (Neural)
- English (India): Raveena (Neural)
- Tamil: Custom voice (if available)
- Use SSML for natural prosody and emphasis

### 3.2 RAG Pipeline Implementation

**3.2.1 Embedding Generation**

**Service:** Amazon Bedrock Titan Embeddings

**Model:** amazon.titan-embed-text-v1
**Dimensions:** 1536

**Implementation:**
```python
import boto3

bedrock = boto3.client('bedrock-runtime', region_name='ap-south-1')

def generate_embedding(text):
    response = bedrock.invoke_model(
        modelId='amazon.titan-embed-text-v1',
        body=json.dumps({
            "inputText": text
        })
    )
    return json.loads(response['body'].read())['embedding']
```

**3.2.2 Vector Search (Amazon OpenSearch)**

**Index Configuration:**
```json
{
  "settings": {
    "index": {
      "knn": true,
      "knn.algo_param.ef_search": 512
    }
  },
  "mappings": {
    "properties": {
      "scheme_id": { "type": "keyword" },
      "scheme_name": { "type": "text" },
      "description": { "type": "text" },
      "embedding": {
        "type": "knn_vector",
        "dimension": 1536,
        "method": {
          "name": "hnsw",
          "space_type": "cosinesimil",
          "engine": "nmslib"
        }
      },
      "category": { "type": "keyword" },
      "ministry": { "type": "keyword" },
      "state": { "type": "keyword" }
    }
  }
}
```

**Search Query:**
```python
def search_schemes(query_embedding, filters, k=10):
    query = {
        "size": k,
        "query": {
            "bool": {
                "must": [
                    {
                        "knn": {
                            "embedding": {
                                "vector": query_embedding,
                                "k": k
                            }
                        }
                    }
                ],
                "filter": filters
            }
        }
    }
    return opensearch_client.search(index="schemes", body=query)
```

**3.2.3 Response Generation (Amazon Bedrock)**

**Model:** anthropic.claude-3-5-sonnet-20241022-v2:0

**Prompt Template:**
```python
SYSTEM_PROMPT = """You are SarkariSaathi, a helpful AI assistant that helps Indian citizens discover and apply for government schemes. 

Guidelines:
- Provide accurate information based only on the retrieved scheme data
- Use simple, conversational language
- Respond in the user's preferred language (Hindi/English/Regional)
- Be empathetic and supportive
- Never make up information - if unsure, say so
- Keep responses concise for voice output (2-3 sentences max)
"""

USER_PROMPT_TEMPLATE = """
User Query: {user_query}

Retrieved Schemes:
{retrieved_schemes}

User Context:
- Age: {age}
- Location: {location}
- Occupation: {occupation}
- Previous conversation: {conversation_history}

Based on the retrieved schemes and user context, provide a helpful response. Focus on the most relevant schemes and explain eligibility clearly.
"""
```

**Implementation:**
```python
def generate_response(user_query, retrieved_schemes, user_context):
    prompt = USER_PROMPT_TEMPLATE.format(
        user_query=user_query,
        retrieved_schemes=format_schemes(retrieved_schemes),
        **user_context
    )
    
    response = bedrock.invoke_model(
        modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
        body=json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 500,
            "system": SYSTEM_PROMPT,
            "messages": [
                {"role": "user", "content": prompt}
            ],
            "temperature": 0.3
        })
    )
    return json.loads(response['body'].read())['content'][0]['text']
```

### 3.3 WhatsApp Integration

**3.3.1 Architecture**

```
WhatsApp User → WhatsApp Business API → Webhook (API Gateway)
                                              ↓
                                        Lambda Function
                                              ↓
                                    Process Message Type
                                    /        |        \
                              Text      Voice      Media
                                \        |        /
                                  Core AI Pipeline
                                        ↓
                                  Format Response
                                        ↓
                              WhatsApp Business API
                                        ↓
                                  User Receives
```

**3.3.2 Message Types**

**Incoming:**
- Text messages
- Voice messages (audio files)
- Location sharing
- Button responses
- List selections

**Outgoing:**
- Text responses
- Voice messages
- Rich media (images, PDFs)
- Interactive buttons
- Lists
- Template messages (notifications)

**3.3.3 Session Management**

```python
# DynamoDB schema for WhatsApp sessions
{
    "session_id": "whatsapp_+919876543210",
    "user_id": "user_12345",
    "phone_number": "+919876543210",
    "conversation_history": [
        {
            "timestamp": "2026-02-15T10:30:00Z",
            "role": "user",
            "content": "Mujhe farming scheme chahiye",
            "message_type": "text"
        },
        {
            "timestamp": "2026-02-15T10:30:03Z",
            "role": "assistant",
            "content": "Aapke liye 3 farming schemes hain...",
            "message_type": "text"
        }
    ],
    "context": {
        "age": 45,
        "location": "Jaipur, Rajasthan",
        "occupation": "farmer"
    },
    "ttl": 1708000000  # 15 minutes from last activity
}
```

---

## 4. Data Models

### 4.1 DynamoDB Tables

**4.1.1 Users Table**

```json
{
  "TableName": "SarkariSaathi-Users",
  "KeySchema": [
    { "AttributeName": "user_id", "KeyType": "HASH" }
  ],
  "AttributeDefinitions": [
    { "AttributeName": "user_id", "AttributeType": "S" },
    { "AttributeName": "phone_number", "AttributeType": "S" }
  ],
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "phone-index",
      "KeySchema": [
        { "AttributeName": "phone_number", "KeyType": "HASH" }
      ]
    }
  ],
  "BillingMode": "PAY_PER_REQUEST"
}
```

**Schema:**
```python
{
    "user_id": "uuid",
    "phone_number": "+919876543210",
    "email": "user@example.com",
    "name": "Ramesh Kumar",
    "demographics": {
        "age": 45,
        "gender": "male",
        "occupation": "farmer",
        "annual_income": 200000,
        "caste_category": "general",
        "disability_status": false
    },
    "location": {
        "state": "Rajasthan",
        "district": "Jaipur",
        "pincode": "302001",
        "coordinates": {
            "lat": 26.9124,
            "lng": 75.7873
        }
    },
    "preferences": {
        "language": "hi",
        "voice_speed": "normal",
        "notifications_enabled": true
    },
    "created_at": "2026-02-15T10:00:00Z",
    "updated_at": "2026-02-15T10:30:00Z"
}
```

**4.1.2 Conversations Table**

```python
{
    "session_id": "uuid",
    "user_id": "uuid",
    "channel": "mobile_app",  # mobile_app, web, whatsapp, ivr
    "messages": [
        {
            "timestamp": "2026-02-15T10:30:00Z",
            "role": "user",
            "content": "Mujhe farming scheme chahiye",
            "audio_url": "s3://bucket/audio/uuid.mp3",
            "intent": "SearchScheme",
            "entities": {
                "category": "agriculture",
                "occupation": "farmer"
            }
        },
        {
            "timestamp": "2026-02-15T10:30:03Z",
            "role": "assistant",
            "content": "Aapke liye 3 farming schemes hain...",
            "audio_url": "s3://bucket/audio/uuid.mp3",
            "schemes_mentioned": ["PM-KISAN", "PMFBY", "KCC"]
        }
    ],
    "context": {
        "current_intent": "SearchScheme",
        "extracted_entities": {},
        "schemes_discussed": []
    },
    "started_at": "2026-02-15T10:30:00Z",
    "last_activity": "2026-02-15T10:35:00Z",
    "ttl": 1708000000
}
```

**4.1.3 Applications Table**

```python
{
    "application_id": "uuid",
    "user_id": "uuid",
    "scheme_id": "PM-KISAN",
    "status": "submitted",  # draft, submitted, under_review, approved, rejected
    "submitted_at": "2026-02-15T11:00:00Z",
    "documents": [
        {
            "type": "aadhaar",
            "s3_url": "s3://bucket/docs/uuid.pdf",
            "uploaded_at": "2026-02-15T10:55:00Z",
            "verified": true
        }
    ],
    "timeline": [
        {
            "status": "submitted",
            "timestamp": "2026-02-15T11:00:00Z",
            "notes": "Application submitted successfully"
        }
    ],
    "created_at": "2026-02-15T10:50:00Z",
    "updated_at": "2026-02-15T11:00:00Z"
}
```

**4.1.4 Bookmarks Table**

```python
{
    "bookmark_id": "uuid",
    "user_id": "uuid",
    "scheme_id": "PM-KISAN",
    "bookmarked_at": "2026-02-15T10:35:00Z",
    "notes": "For next farming season"
}
```

### 4.2 RDS PostgreSQL Schema

**4.2.1 Schemes Table**

```sql
CREATE TABLE schemes (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(500) NOT NULL,
    name_hindi VARCHAR(500),
    colloquial_names TEXT[],
    ministry VARCHAR(200),
    category VARCHAR(100),
    description TEXT,
    description_hindi TEXT,
    eligibility_rules JSONB,
    benefits JSONB,
    documents_required JSONB,
    application_process TEXT,
    application_url VARCHAR(500),
    helpline_number VARCHAR(20),
    state VARCHAR(50),
    is_central BOOLEAN DEFAULT true,
    launch_date DATE,
    deadline DATE,
    processing_time_days INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_schemes_category ON schemes(category);
CREATE INDEX idx_schemes_state ON schemes(state);
CREATE INDEX idx_schemes_ministry ON schemes(ministry);
```

**Eligibility Rules JSON Structure:**
```json
{
  "age": {
    "min": 18,
    "max": 60
  },
  "income": {
    "max": 250000,
    "currency": "INR"
  },
  "occupation": ["farmer", "agricultural_worker"],
  "caste_category": ["general", "obc", "sc", "st"],
  "gender": ["male", "female", "other"],
  "state": ["Rajasthan", "all"],
  "custom_conditions": [
    "Must own agricultural land",
    "Bank account mandatory"
  ]
}
```

**4.2.2 Offices Table**

```sql
CREATE TABLE offices (
    id SERIAL PRIMARY KEY,
    office_type VARCHAR(100),  -- Tehsil, Block, District, E-Mitra, CSC
    name VARCHAR(300) NOT NULL,
    address TEXT,
    state VARCHAR(50),
    district VARCHAR(100),
    pincode VARCHAR(10),
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    contact_numbers TEXT[],
    email VARCHAR(200),
    operating_hours JSONB,
    services_offered TEXT[],
    accessibility_features TEXT[],
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_offices_location ON offices USING GIST (
    ll_to_earth(latitude, longitude)
);
CREATE INDEX idx_offices_state_district ON offices(state, district);
```

**Operating Hours JSON:**
```json
{
  "monday": { "open": "09:00", "close": "17:00" },
  "tuesday": { "open": "09:00", "close": "17:00" },
  "wednesday": { "open": "09:00", "close": "17:00" },
  "thursday": { "open": "09:00", "close": "17:00" },
  "friday": { "open": "09:00", "close": "17:00" },
  "saturday": { "open": "09:00", "close": "13:00" },
  "sunday": { "closed": true }
}
```

**4.2.3 Documents Table**

```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    document_type VARCHAR(100) NOT NULL,
    name VARCHAR(300),
    name_hindi VARCHAR(300),
    description TEXT,
    description_hindi TEXT,
    alternatives JSONB,
    how_to_obtain TEXT,
    issuing_authority VARCHAR(200),
    validity_period VARCHAR(100),
    sample_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Alternatives JSON:**
```json
{
  "alternatives": [
    {
      "name": "Voter ID Card",
      "description": "Can be used instead of Aadhaar for identity proof"
    },
    {
      "name": "Passport",
      "description": "Valid identity and address proof"
    }
  ]
}
```

### 4.3 S3 Bucket Structure

```
sarkari-saathi-documents/
├── schemes/
│   ├── PM-KISAN/
│   │   ├── guidelines.pdf
│   │   ├── application-form.pdf
│   │   └── faq.pdf
│   └── PMFBY/
│       └── ...
├── user-uploads/
│   └── {user_id}/
│       └── {application_id}/
│           ├── aadhaar.pdf
│           └── income-certificate.pdf
├── audio-cache/
│   └── {hash}/
│       └── response.mp3
└── sample-documents/
    ├── aadhaar-sample.pdf
    └── income-certificate-sample.pdf
```

---

## 5. API Specifications

### 5.1 REST API Endpoints

**Base URL:** `https://api.sarkarisaathi.in/v1`

**Authentication:** Bearer token (JWT from Cognito)

**5.1.1 Voice Query API**

```
POST /voice/query
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "audio": "base64_encoded_audio",
  "language": "hi-IN",
  "session_id": "uuid",
  "user_id": "uuid"
}

Response:
{
  "session_id": "uuid",
  "text_query": "Mujhe farming scheme chahiye",
  "text_response": "Aapke liye 3 farming schemes hain...",
  "audio_response": "base64_encoded_audio",
  "audio_url": "https://cdn.sarkarisaathi.in/audio/uuid.mp3",
  "schemes": [
    {
      "id": "PM-KISAN",
      "name": "Pradhan Mantri Kisan Samman Nidhi",
      "relevance_score": 0.95
    }
  ],
  "intent": "SearchScheme",
  "entities": {
    "category": "agriculture",
    "occupation": "farmer"
  }
}
```

**5.1.2 Text Query API**

```
POST /text/query
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "query": "Mujhe farming scheme chahiye",
  "language": "hi",
  "session_id": "uuid",
  "user_id": "uuid"
}

Response:
{
  "session_id": "uuid",
  "response": "Aapke liye 3 farming schemes hain...",
  "schemes": [...],
  "suggestions": [
    "Tell me about PM-KISAN",
    "Check eligibility",
    "Find nearest office"
  ]
}
```

**5.1.3 Scheme Search API**

```
GET /schemes/search?q={query}&category={category}&state={state}&page={page}&limit={limit}
Authorization: Bearer {token}

Response:
{
  "total": 150,
  "page": 1,
  "limit": 10,
  "schemes": [
    {
      "id": "PM-KISAN",
      "name": "Pradhan Mantri Kisan Samman Nidhi",
      "name_hindi": "प्रधानमंत्री किसान सम्मान निधि",
      "category": "agriculture",
      "ministry": "Ministry of Agriculture",
      "description": "Financial support to farmers",
      "benefit_amount": "₹6,000 per year",
      "is_central": true,
      "state": null
    }
  ]
}
```

**5.1.4 Scheme Details API**

```
GET /schemes/{scheme_id}
Authorization: Bearer {token}

Response:
{
  "id": "PM-KISAN",
  "name": "Pradhan Mantri Kisan Samman Nidhi",
  "name_hindi": "प्रधानमंत्री किसान सम्मान निधि",
  "colloquial_names": ["PM Kisan", "Kisan Samman"],
  "ministry": "Ministry of Agriculture",
  "category": "agriculture",
  "description": "...",
  "eligibility": {
    "age": { "min": 18 },
    "occupation": ["farmer"],
    "land_ownership": true
  },
  "benefits": {
    "amount": 6000,
    "currency": "INR",
    "frequency": "annual",
    "installments": 3
  },
  "documents_required": [
    {
      "type": "aadhaar",
      "name": "Aadhaar Card",
      "mandatory": true,
      "alternatives": ["Voter ID", "Passport"]
    }
  ],
  "application_process": "...",
  "application_url": "https://pmkisan.gov.in",
  "helpline": "155261",
  "processing_time_days": 30
}
```

**5.1.5 Eligibility Check API**

```
POST /schemes/{scheme_id}/eligibility
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "user_profile": {
    "age": 45,
    "occupation": "farmer",
    "annual_income": 200000,
    "state": "Rajasthan",
    "owns_land": true
  }
}

Response:
{
  "eligible": true,
  "confidence": 0.95,
  "matched_criteria": [
    "Age requirement met (18+)",
    "Occupation matches (farmer)",
    "Income below threshold (₹2.5L)"
  ],
  "missing_criteria": [],
  "recommendations": [
    "Ensure Aadhaar is linked to bank account",
    "Keep land ownership documents ready"
  ]
}
```

**5.1.6 Nearby Offices API**

```
GET /offices/nearby?lat={latitude}&lng={longitude}&type={office_type}&radius={km}&limit={limit}
Authorization: Bearer {token}

Response:
{
  "offices": [
    {
      "id": 123,
      "name": "E-Mitra Center, Jaipur Road",
      "type": "E-Mitra",
      "address": "Shop No. 45, Jaipur Road, Jaipur",
      "distance_km": 2.3,
      "contact_numbers": ["+91-141-XXXXXXX"],
      "operating_hours": {...},
      "services": ["PM-KISAN", "PMFBY", "KCC"],
      "coordinates": {
        "lat": 26.9124,
        "lng": 75.7873
      },
      "directions_url": "https://maps.google.com/?q=26.9124,75.7873"
    }
  ]
}
```

**5.1.7 User Profile API**

```
GET /users/profile
Authorization: Bearer {token}

Response:
{
  "user_id": "uuid",
  "name": "Ramesh Kumar",
  "phone_number": "+919876543210",
  "demographics": {...},
  "location": {...},
  "preferences": {...}
}

PUT /users/profile
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "demographics": {
    "age": 45,
    "occupation": "farmer"
  },
  "preferences": {
    "language": "hi",
    "voice_speed": "slow"
  }
}

Response:
{
  "success": true,
  "updated_at": "2026-02-15T10:30:00Z"
}
```

**5.1.8 Application Tracking API**

```
POST /applications
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "scheme_id": "PM-KISAN",
  "documents": [
    {
      "type": "aadhaar",
      "file": "base64_encoded_file"
    }
  ]
}

Response:
{
  "application_id": "uuid",
  "status": "draft",
  "created_at": "2026-02-15T10:30:00Z"
}

GET /applications/{application_id}
Authorization: Bearer {token}

Response:
{
  "application_id": "uuid",
  "scheme_id": "PM-KISAN",
  "status": "under_review",
  "timeline": [
    {
      "status": "submitted",
      "timestamp": "2026-02-15T11:00:00Z"
    },
    {
      "status": "under_review",
      "timestamp": "2026-02-16T09:00:00Z"
    }
  ]
}
```

### 5.2 WhatsApp Webhook API

```
POST /whatsapp/webhook
Content-Type: application/json

Request (from WhatsApp):
{
  "object": "whatsapp_business_account",
  "entry": [{
    "changes": [{
      "value": {
        "messages": [{
          "from": "919876543210",
          "id": "wamid.xxx",
          "timestamp": "1708000000",
          "type": "text",
          "text": {
            "body": "Mujhe farming scheme chahiye"
          }
        }]
      }
    }]
  }]
}

Response:
{
  "status": "received"
}

GET /whatsapp/webhook?hub.mode=subscribe&hub.verify_token={token}&hub.challenge={challenge}

Response:
{hub.challenge}
```

---

## 6. Security Architecture

### 6.1 Authentication & Authorization

**6.1.1 Amazon Cognito Configuration**

```json
{
  "UserPoolId": "ap-south-1_xxxxx",
  "ClientId": "xxxxx",
  "IdentityPoolId": "ap-south-1:xxxxx",
  "MfaConfiguration": "OPTIONAL",
  "PasswordPolicy": {
    "MinimumLength": 8,
    "RequireUppercase": true,
    "RequireLowercase": true,
    "RequireNumbers": true,
    "RequireSymbols": false
  }
}
```

**Authentication Flow:**
1. User signs up with phone number
2. OTP sent via SMS
3. User verifies OTP
4. Cognito issues JWT tokens (ID token, Access token, Refresh token)
5. Client includes access token in API requests
6. API Gateway validates token with Cognito

**6.1.2 IAM Roles & Policies**

**Lambda Execution Role:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "transcribe:StartStreamTranscription",
        "polly:SynthesizeSpeech",
        "bedrock:InvokeModel",
        "lex:PostText",
        "lex:PostContent"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:ap-south-1:*:table/SarkariSaathi-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::sarkari-saathi-*/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:ap-south-1:*:secret:sarkari-saathi/*"
    }
  ]
}
```

### 6.2 Data Encryption

**6.2.1 Encryption at Rest**
- DynamoDB: AWS managed keys (KMS)
- RDS: AES-256 encryption enabled
- S3: Server-side encryption (SSE-S3)
- OpenSearch: Encryption at rest enabled
- Secrets Manager: KMS encryption

**6.2.2 Encryption in Transit**
- TLS 1.3 for all API communications
- CloudFront with HTTPS only
- Certificate from AWS Certificate Manager

**6.2.3 Field-Level Encryption**

Sensitive PII fields encrypted before storage:
```python
from cryptography.fernet import Fernet
import boto3

kms = boto3.client('kms', region_name='ap-south-1')

def encrypt_pii(plaintext, key_id):
    response = kms.encrypt(
        KeyId=key_id,
        Plaintext=plaintext.encode()
    )
    return base64.b64encode(response['CiphertextBlob']).decode()

def decrypt_pii(ciphertext, key_id):
    response = kms.decrypt(
        CiphertextBlob=base64.b64decode(ciphertext)
    )
    return response['Plaintext'].decode()
```

### 6.3 DPDP Act 2023 Compliance

**6.3.1 Data Localization**
- Primary region: AWS Mumbai (ap-south-1)
- Backup region: AWS Hyderabad (ap-south-2)
- No data stored outside India

**6.3.2 User Consent Management**

```python
{
    "user_id": "uuid",
    "consents": [
        {
            "type": "data_collection",
            "granted": true,
            "timestamp": "2026-02-15T10:00:00Z",
            "version": "1.0"
        },
        {
            "type": "location_access",
            "granted": true,
            "timestamp": "2026-02-15T10:00:00Z",
            "version": "1.0"
        },
        {
            "type": "voice_recording",
            "granted": true,
            "timestamp": "2026-02-15T10:00:00Z",
            "version": "1.0"
        }
    ]
}
```

**6.3.3 Data Retention & Deletion**

```python
# DynamoDB TTL for automatic deletion
{
    "session_id": "uuid",
    "data": {...},
    "ttl": 1708000000  # Unix timestamp for deletion
}

# User data deletion API
DELETE /users/profile
Authorization: Bearer {token}

# Deletes:
# - User profile from DynamoDB
# - Conversation history
# - Uploaded documents from S3
# - Application records (anonymized)
# - Audit trail retained for 180 days
```

### 6.4 Security Monitoring

**6.4.1 AWS GuardDuty**
- Threat detection enabled
- Alerts for suspicious activity
- Integration with SNS for notifications

**6.4.2 AWS WAF**
- Rate limiting: 100 requests/minute per IP
- SQL injection protection
- XSS protection
- Geo-blocking (if needed)

**6.4.3 CloudWatch Alarms**
```python
alarms = [
    {
        "name": "HighErrorRate",
        "metric": "Errors",
        "threshold": 50,
        "period": 300,
        "evaluation_periods": 2
    },
    {
        "name": "UnauthorizedAccess",
        "metric": "4XXError",
        "threshold": 100,
        "period": 60
    }
]
```

---

## 7. Performance Optimization

### 7.1 Caching Strategy

**7.1.1 CloudFront Caching**
- Static assets: 1 year TTL
- API responses: 5 minutes TTL (for public data)
- Audio responses: 1 hour TTL

**7.1.2 ElastiCache (Redis)**

```python
# Cache frequently accessed schemes
cache_key = f"scheme:{scheme_id}"
ttl = 3600  # 1 hour

# Cache user sessions
session_key = f"session:{session_id}"
ttl = 900  # 15 minutes

# Cache search results
search_key = f"search:{hash(query)}"
ttl = 300  # 5 minutes
```

**7.1.3 DynamoDB DAX**
- Microsecond latency for hot data
- Cache user profiles
- Cache conversation context

### 7.2 Lambda Optimization

**7.2.1 Configuration**
```json
{
  "MemorySize": 1024,
  "Timeout": 30,
  "ReservedConcurrentExecutions": 100,
  "Environment": {
    "Variables": {
      "POWERTOOLS_SERVICE_NAME": "sarkari-saathi",
      "LOG_LEVEL": "INFO"
    }
  }
}
```

**7.2.2 Cold Start Mitigation**
- Provisioned concurrency for critical functions
- Lambda SnapStart for Java functions
- Connection pooling for database connections

### 7.3 Database Optimization

**7.3.1 RDS Performance**
- Read replicas for read-heavy workloads
- Connection pooling (RDS Proxy)
- Query optimization with indexes
- Partitioning for large tables

**7.3.2 DynamoDB Optimization**
- On-demand billing for variable workload
- Global secondary indexes for query patterns
- DynamoDB Streams for real-time processing
- Batch operations for bulk writes

**7.3.3 OpenSearch Optimization**
- Index sharding based on data volume
- Replica shards for high availability
- Index lifecycle management
- Query result caching

---

## 8. Deployment Architecture

### 8.1 Infrastructure as Code

**8.1.1 AWS CDK Stack**

```python
from aws_cdk import (
    Stack,
    aws_lambda as lambda_,
    aws_apigateway as apigw,
    aws_dynamodb as dynamodb,
    aws_s3 as s3,
    aws_cognito as cognito
)

class SarkariSaathiStack(Stack):
    def __init__(self, scope, id, **kwargs):
        super().__init__(scope, id, **kwargs)
        
        # DynamoDB Tables
        users_table = dynamodb.Table(
            self, "UsersTable",
            partition_key=dynamodb.Attribute(
                name="user_id",
                type=dynamodb.AttributeType.STRING
            ),
            billing_mode=dynamodb.BillingMode.PAY_PER_REQUEST
        )
        
        # Lambda Functions
        voice_query_fn = lambda_.Function(
            self, "VoiceQueryFunction",
            runtime=lambda_.Runtime.PYTHON_3_11,
            handler="voice_query.handler",
            code=lambda_.Code.from_asset("lambda"),
            memory_size=1024,
            timeout=Duration.seconds(30)
        )
        
        # API Gateway
        api = apigw.RestApi(
            self, "SarkariSaathiAPI",
            rest_api_name="SarkariSaathi API"
        )
```

### 8.2 CI/CD Pipeline

**8.2.1 GitHub Actions Workflow**

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      
      - name: Run Tests
        run: |
          npm test
          pytest tests/
      
      - name: Deploy CDK Stack
        run: |
          npm install -g aws-cdk
          cdk deploy --require-approval never
      
      - name: Run Integration Tests
        run: |
          pytest tests/integration/
```

### 8.3 Multi-Region Deployment

**Primary Region:** ap-south-1 (Mumbai)
**Secondary Region:** ap-south-2 (Hyderabad)

**Replication:**
- DynamoDB Global Tables
- S3 Cross-Region Replication
- RDS Read Replica in secondary region
- Route 53 health checks for failover

---

## 9. Monitoring & Observability

### 9.1 CloudWatch Dashboards

**9.1.1 Operations Dashboard**

Metrics:
- API request count
- Error rate
- Response latency (p50, p95, p99)
- Lambda invocations
- DynamoDB consumed capacity
- OpenSearch query latency

**9.1.2 Business Metrics Dashboard**

Metrics:
- Active users (DAU, MAU)
- Queries per user
- Scheme discovery rate
- Application initiation rate
- User satisfaction (ratings)

### 9.2 Distributed Tracing

**AWS X-Ray Integration:**
```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

patch_all()

@xray_recorder.capture('process_voice_query')
def process_voice_query(audio, user_id):
    # Transcribe
    with xray_recorder.capture('transcribe'):
        text = transcribe_audio(audio)
    
    # RAG Pipeline
    with xray_recorder.capture('rag_pipeline'):
        schemes = retrieve_schemes(text)
    
    # Generate Response
    with xray_recorder.capture('generate_response'):
        response = generate_llm_response(text, schemes)
    
    return response
```

### 9.3 Logging Strategy

**9.3.1 Structured Logging**

```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def log_event(event_type, data):
    log_entry = {
        "timestamp": datetime.utcnow().isoformat(),
        "event_type": event_type,
        "data": data,
        "service": "sarkari-saathi"
    }
    logger.info(json.dumps(log_entry))

# Usage
log_event("voice_query", {
    "user_id": "uuid",
    "query": "farming scheme",
    "latency_ms": 2500,
    "schemes_returned": 3
})
```

**9.3.2 Log Retention**
- Application logs: 30 days
- Audit logs: 180 days
- Error logs: 90 days

---

## 10. Testing Strategy

### 10.1 Unit Testing

```python
import pytest
from voice_processor import transcribe_audio

def test_transcribe_audio_hindi():
    audio = load_test_audio("hindi_sample.mp3")
    result = transcribe_audio(audio, language="hi-IN")
    assert result["text"] == "मुझे खेती के लिए योजना चाहिए"
    assert result["confidence"] > 0.9

def test_eligibility_check():
    user_profile = {
        "age": 45,
        "occupation": "farmer",
        "income": 200000
    }
    scheme = load_scheme("PM-KISAN")
    result = check_eligibility(user_profile, scheme)
    assert result["eligible"] == True
```

### 10.2 Integration Testing

```python
def test_end_to_end_voice_query():
    # Setup
    user_id = create_test_user()
    audio = load_test_audio("query.mp3")
    
    # Execute
    response = api_client.post("/voice/query", {
        "audio": base64.b64encode(audio),
        "user_id": user_id
    })
    
    # Assert
    assert response.status_code == 200
    assert "schemes" in response.json()
    assert len(response.json()["schemes"]) > 0
```

### 10.3 Load Testing

```python
# locustfile.py
from locust import HttpUser, task, between

class SarkariSaathiUser(HttpUser):
    wait_time = between(1, 3)
    
    @task
    def search_schemes(self):
        self.client.get("/schemes/search?q=farming")
    
    @task(3)
    def voice_query(self):
        audio = load_test_audio()
        self.client.post("/voice/query", json={
            "audio": audio,
            "language": "hi-IN"
        })
```

**Load Test Targets:**
- 10,000 concurrent users
- 100,000 requests per minute
- <3 second response time at p95
- <2% error rate

---

## 11. Conclusion

This design document provides a comprehensive technical specification for SarkariSaathi. The architecture leverages AWS's AI/ML services to create a scalable, secure, and performant voice-first platform that democratizes access to government welfare schemes.

**Key Design Decisions:**
- Serverless architecture for cost-effective scaling
- RAG pipeline for accurate, hallucination-free responses
- Multi-region deployment for high availability
- DPDP Act 2023 compliant data architecture
- Voice-first interface for digital inclusion

**Next Steps:**
1. Review and approve design document
2. Set up AWS infrastructure
3. Implement core components
4. Conduct testing and validation
5. Deploy MVP to production

---

**Document Version:** 1.0  
**Last Updated:** February 15, 2026  
**Status:** Final  
**Author:** Technical Team - Suicide Squad
