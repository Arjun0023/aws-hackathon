# RetailMind AI - Design Document

## Document Information

**Version**: 1.0  
**Last Updated**: February 13, 2026  
**Team**: RetailMind AI Crew  
**Team Leader**: Arjun

## Table of Contents

1. [System Architecture Overview](#system-architecture-overview)
2. [Component Design](#component-design)
3. [Data Architecture](#data-architecture)
4. [API Design](#api-design)
5. [Security Architecture](#security-architecture)
6. [Deployment Architecture](#deployment-architecture)
7. [Technology Stack](#technology-stack)
8. [Integration Patterns](#integration-patterns)
9. [Scalability & Performance](#scalability--performance)
10. [Monitoring & Observability](#monitoring--observability)

---

## 1. System Architecture Overview

### 1.1 High-Level Architecture

RetailMind AI follows a serverless, event-driven architecture built entirely on AWS services. The system is designed for scalability, cost-efficiency, and ease of maintenance.

**Architecture Diagram**: See `generated-diagrams/retailmind-ai-architecture.png.png`

### 1.2 Architecture Principles

- **Serverless-First**: Minimize operational overhead using Lambda, API Gateway, and managed services
- **Event-Driven**: Asynchronous processing using EventBridge and SNS for proactive notifications
- **Multi-Tenant**: Single deployment serving multiple SMB customers with data isolation
- **AI-Native**: Leverage Amazon Bedrock and Amazon Q for intelligent decision-making
- **Mobile-First**: WhatsApp as primary interface for maximum accessibility

### 1.3 System Layers

```
┌─────────────────────────────────────────────────────────┐
│  Presentation Layer: WhatsApp Business API              │
├─────────────────────────────────────────────────────────┤
│  Integration Layer: API Gateway + Lambda Authorizer     │
├─────────────────────────────────────────────────────────┤
│  Application Layer: Lambda Functions (Orchestrator,     │
│                     Processor, Notifier)                │
├─────────────────────────────────────────────────────────┤
│  AI/ML Layer: Bedrock, Amazon Q, SageMaker, Transcribe, │
│               Polly                                     │
├─────────────────────────────────────────────────────────┤
│  Data Layer: DynamoDB, S3, ElastiCache (optional)      │
├─────────────────────────────────────────────────────────┤
│  Infrastructure Layer: IAM, Secrets Manager, CloudWatch │
└─────────────────────────────────────────────────────────┘
```


## 2. Component Design

### 2.1 Frontend Interface

**WhatsApp Business API Integration**

- **Purpose**: Primary user interface for SMB owners
- **Capabilities**:
  - Receive text and voice messages
  - Send text responses and media (images, documents)
  - Support for interactive buttons and lists
  - Message templates for proactive notifications
- **Implementation**:
  - Webhook endpoint for incoming messages
  - WhatsApp Cloud API for sending messages
  - Message queue for handling high volume

### 2.2 API Gateway

**Amazon API Gateway (REST API)**

- **Endpoints**:
  - `POST /webhook` - Receive WhatsApp messages
  - `GET /webhook` - WhatsApp verification endpoint
  - `POST /api/v1/query` - Internal query processing
  - `GET /api/v1/health` - Health check
- **Features**:
  - Request validation and throttling
  - API key authentication for WhatsApp
  - CORS configuration
  - Request/response transformation
  - CloudWatch logging

### 2.3 Lambda Functions

**Orchestrator Lambda** (`retailmind-orchestrator`)

- **Purpose**: Main entry point for all WhatsApp messages
- **Responsibilities**:
  - Parse incoming WhatsApp messages
  - Identify user and session context
  - Route to appropriate processor based on intent
  - Manage conversation state in DynamoDB
  - Handle errors and fallback responses
- **Triggers**: API Gateway POST /webhook
- **Runtime**: Python 3.11
- **Memory**: 512 MB
- **Timeout**: 30 seconds

**Processor Lambda** (`retailmind-processor`)

- **Purpose**: Execute business logic for specific intents
- **Responsibilities**:
  - Inventory queries and updates
  - Demand forecasting requests
  - Pricing analysis
  - Customer query handling
  - Call Bedrock/Amazon Q for AI responses
  - Invoke Transcribe for voice processing
- **Triggers**: Invoked by Orchestrator Lambda
- **Runtime**: Python 3.11
- **Memory**: 1024 MB
- **Timeout**: 60 seconds

**Notifier Lambda** (`retailmind-notifier`)

- **Purpose**: Send proactive alerts and scheduled reports
- **Responsibilities**:
  - Check inventory thresholds
  - Generate daily/weekly summaries
  - Detect demand spikes
  - Send WhatsApp notifications
  - Log notification history
- **Triggers**: EventBridge scheduled rules
- **Runtime**: Python 3.11
- **Memory**: 512 MB
- **Timeout**: 300 seconds (5 minutes)

**Voice Processor Lambda** (`retailmind-voice`)

- **Purpose**: Handle voice message processing
- **Responsibilities**:
  - Download voice file from WhatsApp
  - Convert to format compatible with Transcribe
  - Invoke Amazon Transcribe
  - Return transcribed text to Orchestrator
  - Support Hindi, Marathi, English
- **Triggers**: Invoked by Orchestrator Lambda
- **Runtime**: Python 3.11
- **Memory**: 1024 MB
- **Timeout**: 60 seconds

### 2.4 AI/ML Components

**Amazon Bedrock Integration**

- **Models Used**:
  - Claude 3 Sonnet (primary) - General reasoning and conversation
  - Claude 3 Haiku (fallback) - Cost-effective for simple queries
- **Use Cases**:
  - Natural language understanding
  - Intent classification
  - Response generation
  - Business insights and recommendations
  - Agentic workflows for complex tasks
- **Configuration**:
  - Model ID: `anthropic.claude-3-sonnet-20240229-v1:0`
  - Max tokens: 2048
  - Temperature: 0.7
  - System prompts for SMB context

**Amazon Q Integration**

- **Purpose**: Business intelligence queries on structured data
- **Use Cases**:
  - "Show me top-selling products this month"
  - "What's my revenue trend?"
  - "Which items have low turnover?"
- **Data Sources**: DynamoDB tables via Q connector
- **Configuration**: Custom Q application with SMB-specific schema

**Amazon Transcribe**

- **Languages**: Hindi (hi-IN), Marathi (mr-IN), English (en-IN)
- **Features**:
  - Automatic language detection
  - Custom vocabulary for retail terms
  - Speaker identification (future)
- **Input**: Audio files from WhatsApp (OGG/MP3)
- **Output**: JSON with transcribed text and confidence scores

**Amazon Polly**

- **Languages**: Hindi (Aditi), Marathi (Aditi), English (Raveena)
- **Use Cases**: Voice responses for accessibility
- **Configuration**:
  - Neural voices for natural sound
  - SSML support for emphasis
  - Output format: MP3

**Amazon SageMaker (Optional for MVP)**

- **Purpose**: Advanced demand forecasting
- **Model**: DeepAR or Prophet for time-series forecasting
- **Training Data**: Historical sales transactions
- **Inference**: Real-time endpoint for predictions
- **Fallback**: Bedrock's built-in ML capabilities for MVP

### 2.5 Storage Components

**Amazon DynamoDB Tables**

See Section 3 (Data Architecture) for detailed schema.

**Amazon S3 Buckets**

- `retailmind-voice-files`: Store voice messages temporarily
- `retailmind-backups`: Daily DynamoDB backups
- `retailmind-reports`: Generated reports and analytics
- `retailmind-logs`: Application logs (if not using CloudWatch exclusively)

**Lifecycle Policies**:
- Voice files: Delete after 7 days
- Backups: Transition to Glacier after 30 days, delete after 90 days
- Reports: Transition to IA after 30 days

### 2.6 Notification Components

**Amazon EventBridge Rules**

- `daily-summary-rule`: Trigger at 8 AM daily for business summaries
- `inventory-check-rule`: Run every 4 hours to check stock levels
- `weekly-report-rule`: Trigger every Monday at 9 AM
- `demand-spike-rule`: Event pattern for sudden sales increases

**Amazon SNS Topics**

- `retailmind-alerts`: Critical system alerts
- `retailmind-notifications`: User notifications (backup to WhatsApp)
- `retailmind-admin`: Admin notifications for system issues


## 3. Data Architecture

### 3.1 DynamoDB Table Design

**Table 1: Users**

```
Table Name: retailmind-users
Partition Key: userId (String)
Attributes:
  - userId: String (UUID)
  - phoneNumber: String (E.164 format)
  - businessName: String
  - businessCategory: String (retail, cafe, online_seller, etc.)
  - location: String
  - languagePreference: String (hi, mr, en)
  - notificationSettings: Map
    - dailySummaryTime: String (HH:MM)
    - enableLowStockAlerts: Boolean
    - enableDemandAlerts: Boolean
  - subscriptionTier: String (free, basic, premium)
  - subscriptionStatus: String (active, trial, expired)
  - createdAt: String (ISO 8601)
  - lastActiveAt: String (ISO 8601)
  - onboardingCompleted: Boolean

GSI: phoneNumber-index (for lookup by phone)
```

**Table 2: Inventory**

```
Table Name: retailmind-inventory
Partition Key: userId (String)
Sort Key: itemId (String)
Attributes:
  - userId: String
  - itemId: String (UUID)
  - itemName: String
  - category: String
  - sku: String (optional)
  - currentStock: Number
  - minThreshold: Number
  - unitPrice: Number
  - costPrice: Number
  - unit: String (pieces, kg, liters, etc.)
  - lastUpdated: String (ISO 8601)
  - lastRestockDate: String (ISO 8601)
  - createdAt: String (ISO 8601)

GSI: userId-category-index (for category-based queries)
```

**Table 3: Transactions**

```
Table Name: retailmind-transactions
Partition Key: userId (String)
Sort Key: transactionId (String)
Attributes:
  - userId: String
  - transactionId: String (UUID)
  - transactionDate: String (ISO 8601)
  - transactionType: String (sale, purchase, adjustment)
  - itemId: String
  - itemName: String
  - quantity: Number
  - unitPrice: Number
  - totalAmount: Number
  - customerReference: String (optional)
  - notes: String (optional)
  - createdAt: String (ISO 8601)

GSI: userId-transactionDate-index (for time-based queries)
GSI: userId-itemId-index (for item-specific history)
```

**Table 4: Conversations**

```
Table Name: retailmind-conversations
Partition Key: userId (String)
Sort Key: messageId (String)
Attributes:
  - userId: String
  - messageId: String (UUID)
  - timestamp: String (ISO 8601)
  - messageType: String (text, voice, image)
  - userMessage: String
  - botResponse: String
  - intent: String (inventory_query, forecast_request, etc.)
  - confidence: Number
  - processingTime: Number (milliseconds)
  - sessionId: String

TTL: expiryTime (delete after 30 days)
```

**Table 5: Forecasts**

```
Table Name: retailmind-forecasts
Partition Key: userId (String)
Sort Key: forecastKey (String) # Format: itemId#forecastDate
Attributes:
  - userId: String
  - itemId: String
  - itemName: String
  - forecastDate: String (YYYY-MM-DD)
  - predictedDemand: Number
  - confidenceLevel: String (high, medium, low)
  - confidenceScore: Number (0-1)
  - factorsConsidered: List<String>
  - actualDemand: Number (filled after date passes)
  - generatedAt: String (ISO 8601)

GSI: userId-forecastDate-index
TTL: expiryTime (delete after 90 days)
```

**Table 6: Alerts**

```
Table Name: retailmind-alerts
Partition Key: userId (String)
Sort Key: alertId (String)
Attributes:
  - userId: String
  - alertId: String (UUID)
  - alertType: String (low_stock, demand_spike, pricing_opportunity)
  - severity: String (info, warning, critical)
  - title: String
  - message: String
  - relatedItemId: String (optional)
  - sentAt: String (ISO 8601)
  - acknowledged: Boolean
  - acknowledgedAt: String (ISO 8601, optional)

GSI: userId-sentAt-index
TTL: expiryTime (delete after 60 days)
```

### 3.2 Data Flow Diagrams

**Diagram Reference**: See `generated-diagrams/retailmind-data-flow.png.png`

**Inbound Message Flow**:
1. User sends WhatsApp message
2. WhatsApp API → API Gateway → Orchestrator Lambda
3. Orchestrator reads user context from DynamoDB (Users table)
4. Orchestrator reads conversation history (Conversations table)
5. Orchestrator invokes Processor Lambda with context
6. Processor queries relevant data (Inventory, Transactions, Forecasts)
7. Processor calls Bedrock/Amazon Q for AI response
8. Response saved to Conversations table
9. Response sent back through API Gateway → WhatsApp API

**Proactive Alert Flow**:
1. EventBridge triggers Notifier Lambda
2. Notifier queries all active users (Users table)
3. For each user, check inventory levels (Inventory table)
4. If threshold breached, create alert (Alerts table)
5. Generate notification message using Bedrock
6. Send via WhatsApp API
7. Log notification in Conversations table

### 3.3 Data Retention & Backup

**Retention Policies**:
- Conversations: 30 days (TTL enabled)
- Forecasts: 90 days (TTL enabled)
- Alerts: 60 days (TTL enabled)
- Users: Indefinite (until account deletion)
- Inventory: Indefinite
- Transactions: Indefinite (for historical analysis)

**Backup Strategy**:
- DynamoDB Point-in-Time Recovery (PITR) enabled
- Daily automated backups to S3 using AWS Backup
- Backup retention: 30 days in standard storage, 90 days in Glacier
- Cross-region backup replication for disaster recovery


## 4. API Design

### 4.1 WhatsApp Webhook API

**Endpoint**: `POST /webhook`

**Purpose**: Receive incoming messages from WhatsApp

**Request Headers**:
```
Content-Type: application/json
X-Hub-Signature-256: <signature> (for verification)
```

**Request Body** (WhatsApp format):
```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "WHATSAPP_BUSINESS_ACCOUNT_ID",
    "changes": [{
      "value": {
        "messaging_product": "whatsapp",
        "metadata": {
          "display_phone_number": "PHONE_NUMBER",
          "phone_number_id": "PHONE_NUMBER_ID"
        },
        "contacts": [{
          "profile": {
            "name": "NAME"
          },
          "wa_id": "WHATSAPP_ID"
        }],
        "messages": [{
          "from": "SENDER_PHONE_NUMBER",
          "id": "MESSAGE_ID",
          "timestamp": "TIMESTAMP",
          "type": "text|audio|image",
          "text": {
            "body": "MESSAGE_TEXT"
          }
        }]
      }
    }]
  }]
}
```

**Response**:
```json
{
  "status": "received"
}
```

**Endpoint**: `GET /webhook`

**Purpose**: WhatsApp webhook verification

**Query Parameters**:
- `hub.mode`: "subscribe"
- `hub.verify_token`: Verification token
- `hub.challenge`: Challenge string

**Response**: Echo back the `hub.challenge` value

### 4.2 Internal Processing API

**Endpoint**: `POST /api/v1/process-query`

**Purpose**: Internal API for processing user queries (invoked by Orchestrator)

**Request**:
```json
{
  "userId": "uuid",
  "sessionId": "uuid",
  "query": "How many shirts do I have?",
  "queryType": "text|voice",
  "language": "en|hi|mr",
  "context": {
    "previousIntent": "inventory_query",
    "conversationHistory": []
  }
}
```

**Response**:
```json
{
  "success": true,
  "response": {
    "text": "You currently have 45 shirts in stock.",
    "intent": "inventory_query",
    "confidence": 0.95,
    "data": {
      "itemName": "Shirts",
      "currentStock": 45,
      "minThreshold": 20
    },
    "suggestedActions": [
      "View detailed inventory",
      "Update stock level"
    ]
  },
  "processingTime": 1250
}
```

### 4.3 Inventory Management API

**Endpoint**: `GET /api/v1/inventory/{userId}`

**Purpose**: Retrieve user's inventory

**Query Parameters**:
- `category`: Filter by category (optional)
- `lowStock`: Boolean, show only low stock items (optional)

**Response**:
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "itemId": "uuid",
        "itemName": "Cotton Shirts",
        "category": "Apparel",
        "currentStock": 45,
        "minThreshold": 20,
        "unitPrice": 499,
        "lastUpdated": "2026-02-13T10:30:00Z"
      }
    ],
    "totalItems": 1,
    "lowStockCount": 0
  }
}
```

**Endpoint**: `POST /api/v1/inventory/{userId}`

**Purpose**: Update inventory

**Request**:
```json
{
  "itemId": "uuid",
  "operation": "add|subtract|set",
  "quantity": 10,
  "notes": "Sold 10 units"
}
```

**Response**:
```json
{
  "success": true,
  "data": {
    "itemId": "uuid",
    "previousStock": 45,
    "newStock": 35,
    "updatedAt": "2026-02-13T11:00:00Z"
  }
}
```

### 4.4 Forecasting API

**Endpoint**: `GET /api/v1/forecast/{userId}/{itemId}`

**Purpose**: Get demand forecast for specific item

**Query Parameters**:
- `days`: Number of days to forecast (7, 14, 30)

**Response**:
```json
{
  "success": true,
  "data": {
    "itemId": "uuid",
    "itemName": "Cotton Shirts",
    "forecasts": [
      {
        "date": "2026-02-14",
        "predictedDemand": 8,
        "confidenceLevel": "high",
        "confidenceScore": 0.87
      }
    ],
    "summary": {
      "totalPredictedDemand": 56,
      "averageDailyDemand": 8,
      "peakDemandDate": "2026-02-20",
      "recommendedReorderQuantity": 60
    },
    "generatedAt": "2026-02-13T11:00:00Z"
  }
}
```

### 4.5 Alert Management API

**Endpoint**: `GET /api/v1/alerts/{userId}`

**Purpose**: Retrieve user's alerts

**Query Parameters**:
- `unacknowledged`: Boolean, show only unacknowledged alerts
- `type`: Filter by alert type

**Response**:
```json
{
  "success": true,
  "data": {
    "alerts": [
      {
        "alertId": "uuid",
        "alertType": "low_stock",
        "severity": "warning",
        "title": "Low Stock Alert",
        "message": "Cotton Shirts stock is below threshold (15/20)",
        "relatedItemId": "uuid",
        "sentAt": "2026-02-13T09:00:00Z",
        "acknowledged": false
      }
    ],
    "totalAlerts": 1,
    "unacknowledgedCount": 1
  }
}
```

### 4.6 Error Responses

**Standard Error Format**:
```json
{
  "success": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "User-friendly error message",
    "details": "Technical details for debugging",
    "timestamp": "2026-02-13T11:00:00Z"
  }
}
```

**Error Codes**:
- `INVALID_REQUEST`: Malformed request
- `UNAUTHORIZED`: Authentication failed
- `USER_NOT_FOUND`: User doesn't exist
- `ITEM_NOT_FOUND`: Inventory item not found
- `RATE_LIMIT_EXCEEDED`: Too many requests
- `INTERNAL_ERROR`: Server error
- `SERVICE_UNAVAILABLE`: Downstream service unavailable


## 5. Security Architecture

### 5.1 Authentication & Authorization

**WhatsApp API Authentication**:
- API key stored in AWS Secrets Manager
- Webhook signature verification using X-Hub-Signature-256
- Token rotation every 90 days

**User Authentication**:
- Phone number-based identification (WhatsApp verified)
- Session management using DynamoDB with TTL
- No password required (WhatsApp handles authentication)

**Service-to-Service Authentication**:
- IAM roles for Lambda functions
- Least privilege access policies
- No hardcoded credentials

### 5.2 Data Encryption

**Encryption at Rest**:
- DynamoDB: AWS-managed encryption (KMS)
- S3: Server-side encryption (SSE-S3 or SSE-KMS)
- Secrets Manager: Automatic encryption with KMS
- Lambda environment variables: Encrypted with KMS

**Encryption in Transit**:
- HTTPS/TLS 1.2+ for all API calls
- WhatsApp Business API uses HTTPS
- Internal AWS service calls use AWS PrivateLink where possible

### 5.3 Data Privacy & Isolation

**Multi-Tenancy**:
- Partition key (userId) ensures data isolation in DynamoDB
- S3 bucket policies restrict access by user prefix
- Lambda functions validate userId in every request

**PII Handling**:
- Phone numbers hashed for logging
- Business data never shared across users
- Conversation history encrypted and TTL-enabled
- GDPR-compliant data deletion on user request

**Data Residency**:
- All data stored in AWS Asia Pacific (Mumbai) region
- No cross-region data transfer except for backups
- Compliance with Indian data protection regulations

### 5.4 Network Security

**API Gateway**:
- Rate limiting: 100 requests/minute per user
- Throttling: 1000 requests/second burst
- IP whitelisting for admin APIs (future)
- WAF rules for common attack patterns

**Lambda Functions**:
- VPC deployment (optional for enhanced security)
- Security groups restrict outbound traffic
- No direct internet access (use VPC endpoints)

**DynamoDB & S3**:
- VPC endpoints for private access
- Bucket policies deny public access
- Resource-based policies for fine-grained control

### 5.5 IAM Policies

**Orchestrator Lambda Role**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query"
      ],
      "Resource": [
        "arn:aws:dynamodb:*:*:table/retailmind-users",
        "arn:aws:dynamodb:*:*:table/retailmind-conversations"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:*:*:function:retailmind-processor"
    },
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:*:*:secret:retailmind/whatsapp-*"
    }
  ]
}
```

**Processor Lambda Role**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:UpdateItem"
      ],
      "Resource": [
        "arn:aws:dynamodb:*:*:table/retailmind-inventory",
        "arn:aws:dynamodb:*:*:table/retailmind-transactions",
        "arn:aws:dynamodb:*:*:table/retailmind-forecasts"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:aws:bedrock:*::foundation-model/anthropic.claude-*"
    },
    {
      "Effect": "Allow",
      "Action": "transcribe:StartTranscriptionJob",
      "Resource": "*"
    }
  ]
}
```

### 5.6 Secrets Management

**AWS Secrets Manager**:
- `retailmind/whatsapp-api-key`: WhatsApp Business API key
- `retailmind/whatsapp-verify-token`: Webhook verification token
- `retailmind/whatsapp-phone-number-id`: WhatsApp phone number ID
- `retailmind/encryption-key`: Application-level encryption key

**Rotation Policy**:
- Automatic rotation every 90 days
- Lambda function for rotation logic
- Zero-downtime rotation using versioning

### 5.7 Compliance & Auditing

**AWS CloudTrail**:
- All API calls logged
- Log retention: 90 days in CloudWatch, 1 year in S3
- Alerts for suspicious activities

**Compliance Standards**:
- GDPR-ready (data portability, right to deletion)
- SOC 2 Type II (AWS infrastructure)
- ISO 27001 (AWS infrastructure)
- PCI DSS (if payment processing added)

**Audit Logs**:
- User actions logged in Conversations table
- Admin actions logged in CloudTrail
- Data access patterns monitored via CloudWatch

