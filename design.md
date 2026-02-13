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

