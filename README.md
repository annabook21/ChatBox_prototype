# AWS Contextual Chatbot with Amazon Bedrock Knowledge Bases

## Overview

The AWS Contextual Chatbot is a production-ready, enterprise-grade Retrieval-Augmented Generation (RAG) solution built on AWS serverless infrastructure. The system enables organizations to query their document repositories using natural language, receiving accurate, contextual answers with source citations.

This solution demonstrates best practices for building secure, scalable, and cost-effective AI-powered applications using Amazon Bedrock Knowledge Bases, Lambda functions, and multi-region disaster recovery.

⚠️ **Important**
- Running this code might result in charges to your AWS account.
- We recommend that you grant your code least privilege. At most, grant only the minimum permissions required to perform the task.
- This code is not tested in every AWS Region. For more information, see [AWS Regional Services](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/).

## Prerequisites

Before deploying this solution, ensure you have the following:

- AWS CLI installed and configured with appropriate permissions
- Node.js ≥ 22.9.0 and npm
- AWS CDK CLI v2
- Docker Desktop installed and running
- Access to Amazon Bedrock foundation models in your target regions

### Required AWS Permissions

Your AWS credentials must have permissions to:
- Create and manage CloudFormation stacks
- Deploy Lambda functions and API Gateway
- Create S3 buckets and configure CloudFront
- Access Amazon Bedrock services
- Create IAM roles and policies

### Bedrock Model Access

**CRITICAL**: Before deployment, enable access to these foundation models in the Amazon Bedrock console:

1. Navigate to the [Amazon Bedrock console](https://console.aws.amazon.com/bedrock/home)
2. Click **Model access** in the bottom-left corner
3. Click **Manage model access** in the top-right
4. Enable access for:
   - **Titan Embeddings G1 - Text**: `amazon.titan-embed-text-v1` (for Knowledge Base)
   - **Anthropic Claude 3 Sonnet**: `anthropic.claude-3-sonnet-20240229-v1:0` (for answer generation)

**Note**: Enable model access in **both** us-west-2 and us-east-1 regions for disaster recovery.

## Assumptions & Design Decisions

### Business Context Assumptions

**Document Volume & Types**
- Expected document volume: 100-10,000 documents per knowledge base
- Document types: PDF, TXT, DOCX, MD, HTML (common enterprise formats)
- Average document size: 1-50 pages per document
- Update frequency: Weekly to monthly document refreshes

**Usage Patterns**
- Concurrent users: 10-100 simultaneous users
- Query volume: 100-1,000 queries per day
- Query complexity: Multi-sentence questions requiring document synthesis
- Response time requirement: < 10 seconds for complex queries

**Data & Compliance**
- Data sensitivity: Internal enterprise documents (not public data)
- Compliance requirements: SOC 2, GDPR-ready architecture
- Data retention: 7 years for audit purposes
- Access control: Role-based access (future enhancement)

### Technical Design Decisions

**Architecture Pattern: Serverless-First**
- **Decision**: Fully serverless over containerized architecture
- **Rationale**: 
  - Zero infrastructure management overhead
  - Automatic scaling without capacity planning
  - Pay-per-use cost model aligns with variable workloads
  - Faster time-to-market for MVP deployment

**RAG Implementation: Manual Two-Step Process**
- **Decision**: Separate Retrieve + InvokeModel calls vs RetrieveAndGenerate API
- **Rationale**: 
  - RetrieveAndGenerate doesn't support Bedrock Guardrails (critical for enterprise)
  - Provides finer control over the RAG pipeline
  - Enables custom prompt engineering and context manipulation
  - Better error handling and retry logic

**Document Chunking Strategy: 500 Tokens, 20% Overlap**
- **Decision**: Fixed-size chunks with significant overlap
- **Rationale**:
  - 500 tokens ≈ 1-2 paragraphs (optimal semantic unit)
  - 20% overlap prevents context loss at boundaries
  - Balances precision vs. recall in retrieval
  - Compatible with Claude 3 Sonnet's context window

**Vector Store: Bedrock-Managed OpenSearch Serverless**
- **Decision**: Managed service over self-hosted alternatives
- **Rationale**:
  - Zero operational overhead (no cluster management)
  - Automatic scaling and cost optimization
  - Tight integration with Bedrock Knowledge Bases
  - No additional infrastructure to monitor or maintain

**Disaster Recovery: Manual Backend Failover**
- **Decision**: Manual config.json update vs automatic DNS failover
- **Rationale**:
  - Simpler implementation for MVP (no custom domain required)
  - Manual control over failover timing and validation
  - Cost-effective (no Route 53 health checks needed)
  - Can be enhanced with automatic DNS failover post-MVP

**Frontend Architecture: Static SPA on S3 + CloudFront**
- **Decision**: Static hosting over Amplify or EC2
- **Rationale**:
  - Simplest deployment model with global CDN
  - Automatic HTTPS and edge caching
  - Minimal cost and operational overhead
  - Easy to implement origin failover for DR

### Operational Assumptions

**Monitoring & Alerting**
- CloudWatch native monitoring sufficient for initial deployment
- SNS email notifications for critical alerts
- No third-party monitoring tools required initially
- 24/7 monitoring not required (business hours support acceptable)

**Maintenance Windows**
- Monthly maintenance windows acceptable
- Zero-downtime deployments via blue-green approach
- Emergency patches during business hours with 2-hour advance notice

**Support Model**
- AWS Support Center for infrastructure issues
- Internal team handles application-level support
- No dedicated DevOps team required initially
- Documentation and runbooks sufficient for L1 support

**Cost Optimization**
- Pay-per-use model acceptable for variable workloads
- No upfront capacity reservations required
- Monthly cost reviews and optimization cycles
- Right-sizing recommendations based on usage patterns

### Constraints & Limitations

**Current Limitations**
- No user authentication (planned for Phase 2)
- No multi-tenant support (single knowledge base per deployment)
- Manual document upload only (no API-based ingestion)
- English language only (no multi-language support)

**Future Enhancements**
- Cognito integration for user management
- Multi-knowledge base support
- API-based document ingestion
- Multi-language document processing
- Custom domain with automatic DNS failover

## Architecture

The solution implements a fully serverless, multi-region architecture with automatic failover capabilities:

[<img src="images/contextual_chat_bot architecture.png">]

## Architecture

The solution implements a fully serverless, multi-region architecture with automatic failover capabilities:

```
┌─────────────────────────────────────────────────────────┐
│                    Global CDN Layer                     │
│  CloudFront Distribution (Origin Group Failover)        │
│  ├─ Primary Origin: S3 Frontend (us-west-2)            │
│  └─ Failover Origin: S3 Frontend (us-east-1)           │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│                  Primary Region (us-west-2)             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │API Gateway  │  │Lambda       │  │Bedrock      │     │
│  │(REST API)   │─▶│Functions    │─▶│Knowledge    │     │
│  │             │  │(5 functions)│  │Base         │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                              │                         │
│                              ▼                         │
│                    ┌─────────────┐                     │
│                    │S3 Documents │                     │
│                    │(Versioned)  │                     │
│                    └─────────────┘                     │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼ (Cross-Region Replication)
┌─────────────────────────────────────────────────────────┐
│                 Failover Region (us-east-1)             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │API Gateway  │  │Lambda       │  │Bedrock      │     │
│  │(Standby)    │  │Functions    │  │Knowledge    │     │
│  │             │  │(Standby)    │  │Base         │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### Core Components

#### Frontend Layer
- **CloudFront Distribution**: Global CDN with origin failover (automatic < 1 second RTO)
- **S3 Frontend Buckets**: Private buckets with Origin Access Control (OAC) in both regions
- **React Application**: Modern web interface with drag-and-drop file uploads and real-time chat

#### API Layer
- **API Gateway**: RESTful API with CORS, throttling, and usage plans
- **Endpoints**:
  - `POST /docs`: Submit queries to the chatbot
  - `POST /upload`: Generate pre-signed URLs for file uploads
  - `GET /ingestion-status`: Check document processing status
  - `GET /health`: Health check for monitoring and failover

#### Compute Layer (5 Lambda Functions)
- **Query Lambda**: Core RAG orchestration (Retrieve + Generate with Claude 3 Sonnet)
- **Upload Lambda**: Generate pre-signed S3 URLs for direct file uploads
- **Ingestion Lambda**: Triggered by S3 events to start Bedrock ingestion jobs
- **Status Lambda**: Poll Bedrock for ingestion job status
- **Health Lambda**: Test Bedrock connectivity for monitoring

#### AI/ML Services
- **Bedrock Knowledge Base**: Automated document chunking (500 tokens, 20% overlap)
- **Titan Embeddings**: Vectorization of document chunks
- **Claude 3 Sonnet**: Answer generation with context from retrieved chunks
- **Bedrock Guardrails**: Content filtering for harmful content

#### Storage
- **S3 Documents Bucket**: Versioned, encrypted storage with lifecycle policies
- **OpenSearch Serverless**: Vector store for semantic search (managed by Bedrock)

#### Monitoring & Observability
- **CloudWatch Dashboard**: Real-time metrics for API Gateway, Lambda, and Bedrock
- **CloudWatch Alarms**: Automated alerts for errors and system health
- **X-Ray Tracing**: Distributed tracing across all services
- **SNS Notifications**: Alert delivery for operational teams

## Security Architecture

### Data Protection
- **Encryption at Rest**: All S3 buckets use S3-managed encryption
- **Encryption in Transit**: HTTPS enforcement via CloudFront and API Gateway
- **Versioning**: S3 versioning enabled for point-in-time recovery
- **Access Controls**: Private S3 buckets with OAC, no public access

### Content Safety
- **Bedrock Guardrails**: Multi-category content filtering (sexual, violence, hate, insults)
- **Input Validation**: API Gateway request validation and throttling
- **Least Privilege**: IAM roles with minimal required permissions

### Network Security
- **Private S3 Origins**: No direct S3 access, only via CloudFront OAC
- **CORS Configuration**: Restricted cross-origin requests
- **VPC Integration**: Ready for VPC deployment if required

## Disaster Recovery & Business Continuity

### Frontend Failover (Automatic)
- **Method**: CloudFront origin group with automatic failover
- **Detection**: Instant (5xx errors from primary S3 bucket)
- **RTO**: < 1 second (automatic)
- **User Impact**: None (same CloudFront URL)

### Backend API Failover (Manual)
- **Method**: Runtime configuration via config.json update
- **Detection**: Manual monitoring or custom alerting
- **RTO**: Manual (minutes to hours depending on response time)
- **Process**: Update config.json in both S3 frontend buckets, invalidate CloudFront cache

### Data Synchronization
- **Method**: S3 Cross-Region Replication (CRR) - manual setup required
- **RPO**: ~15 minutes (automatic background sync)
- **Scope**: Documents bucket from us-west-2 → us-east-1

## Performance & Scalability

### Auto-scaling Capabilities
- **Lambda**: Automatic scaling based on request volume
- **API Gateway**: Handles up to 10,000 requests per second
- **CloudFront**: Global edge caching for static content
- **Bedrock**: Managed service with automatic scaling

### Performance Optimizations
- **Edge Caching**: Static assets cached at 450+ CloudFront edge locations
- **Connection Pooling**: Lambda functions optimized for AWS service connections
- **Chunking Strategy**: 500-token chunks with 20% overlap for optimal retrieval

## Cost Analysis

### Monthly Cost Estimates (US East/West)
Based on moderate usage (1,000 queries/month, 10GB documents):

| Service | Primary Region | Failover Region | Monthly Cost |
|---------|---------------|-----------------|--------------|
| Lambda (Compute) | $2-5 | $1-3 | $3-8 |
| API Gateway | $1-3 | $0.50-1.50 | $1.50-4.50 |
| S3 Storage | $0.25-0.50 | $0.25-0.50 | $0.50-1.00 |
| CloudFront | $1-2 | - | $1-2 |
| Bedrock (Titan) | $0.50-1.50 | $0.50-1.50 | $1-3 |
| Bedrock (Claude) | $5-15 | $5-15 | $10-30 |
| **Total** | | | **$17-49** |

*Note: Failover region costs only apply during active failover scenarios.*

### Cost Optimization Recommendations
- Enable S3 lifecycle policies for document retention
- Monitor Bedrock usage and optimize chunk sizes
- Use CloudWatch to track and optimize Lambda cold starts
- Consider Reserved Capacity for predictable workloads

## Deployment

### Multi-Region Deployment

```bash
# Navigate to backend directory
cd backend

# Bootstrap both regions (one-time setup)
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
cdk bootstrap aws://$ACCOUNT/us-west-2
cdk bootstrap aws://$ACCOUNT/us-east-1

# Deploy to both regions
cdk deploy --all
```

This creates:
- `BackendStack-Primary` in us-west-2 with CloudFront distribution
- `BackendStack-Failover` in us-east-1 with standby resources

### Single-Region Deployment (Development)

```bash
# Set target region
export AWS_DEFAULT_REGION=us-west-2

# Bootstrap (one-time)
cdk bootstrap aws://$ACCOUNT/us-west-2

# Deploy
cdk deploy BackendStack-Primary
```

## Usage

### Initial Setup
1. Navigate to the CloudFront URL provided in deployment outputs
2. Upload documents using the file upload interface
3. Wait for ingestion status to show "Complete"
4. Begin querying your documents

### API Usage Examples

#### Submit a Query
```bash
curl -X POST https://your-api-gateway-url/prod/docs \
  -H "Content-Type: application/json" \
  -d '{"query": "What are the key features of this product?"}'
```

#### Check Ingestion Status
```bash
curl https://your-api-gateway-url/prod/ingestion-status
```

#### Health Check
```bash
curl https://your-api-gateway-url/prod/health
```

## Monitoring & Operations

### CloudWatch Dashboard
Access the dashboard named `contextual-chatbot-metrics-{region}` to monitor:
- API Gateway request counts and error rates
- Lambda function performance and errors
- Bedrock service health
- Dead Letter Queue message counts

### Key Metrics to Monitor
- **API Gateway**: 4xx/5xx error rates, request latency
- **Lambda**: Error rates, duration, cold starts
- **Bedrock**: Retrieval latency, generation latency
- **S3**: Request counts, error rates

### Alerting
Configured alarms trigger SNS notifications for:
- Query Lambda errors (>5 in 5 minutes)
- Ingestion Lambda errors (>3 in 5 minutes)
- Dead Letter Queue messages (any messages)

## Troubleshooting

### Common Issues

#### Deployment Failures
- **Bedrock Model Access**: Ensure model access is enabled in both regions
- **Docker Issues**: Verify Docker Desktop is running
- **Permissions**: Check IAM permissions for CDK deployment

#### Runtime Issues
- **"Model not accessible"**: Verify Bedrock model access in target region
- **Upload failures**: Check S3 bucket permissions and CORS configuration
- **Query timeouts**: Monitor Lambda duration and Bedrock service limits

### Debugging Commands
```bash
# Check Lambda logs
aws logs tail /aws/lambda/query-bedrock-llm-{region} --follow

# Test Bedrock connectivity
aws bedrock list-foundation-models --region us-west-2

# Verify S3 bucket access
aws s3 ls s3://your-docs-bucket-name
```

## Additional Resources

- [Amazon Bedrock Developer Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/)
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/)
- [Amazon S3 Developer Guide](https://docs.aws.amazon.com/s3/latest/userguide/)
- [AWS CDK Developer Guide](https://docs.aws.amazon.com/cdk/latest/guide/)
- [Amazon Bedrock API Reference](https://docs.aws.amazon.com/bedrock/latest/APIReference/)

## Support & Maintenance

### Operational Procedures
- Monitor CloudWatch dashboards daily
- Review SNS alerts promptly
- Test failover procedures quarterly
- Update Bedrock models as new versions become available

### Maintenance Windows
- **Scheduled**: Monthly during maintenance windows
- **Emergency**: As needed for critical issues
- **Updates**: Follow AWS service announcements for new features

### Support Contacts
- **AWS Support**: Use AWS Support Center for service issues
- **Documentation**: Refer to AWS documentation for service-specific guidance
- **Community**: AWS re:Post for community support and best practices



# AWS Architectural Diagram - Official Style Guide

## AWS Icon Library & Resources

### Official AWS Icons
- **AWS Architecture Icons**: https://aws.amazon.com/architecture/icons/
- **Download**: AWS-Architecture_Icons_AWSServicelight-bg.zip
- **Format**: SVG icons with light backgrounds
- **Usage**: Free for AWS diagrams (check license for commercial use)

### Recommended Tools for AWS-Style Diagrams

1. **Lucidchart** (Recommended)
   - Has built-in AWS icon library
   - Official AWS template available
   - Professional output quality
   - Export to PNG/PDF for presentations

2. **Draw.io (diagrams.net)**
   - Free tool with AWS icon library
   - Good for quick diagrams
   - Can import AWS icons

3. **Visio**
   - Microsoft's professional diagramming tool
   - AWS stencil available
   - Enterprise-standard output

## AWS Diagram Style Guidelines

### Color Scheme
- **AWS Services**: Light blue backgrounds (#F7F7F7 or white)
- **User/External**: Light green (#D5E8D4)
- **Data Flow**: Blue arrows (#1F77B4)
- **Failover/Standby**: Light gray (#E1E1E1)
- **Monitoring**: Light yellow (#FFF2CC)

### Icon Specifications
- **Size**: 64x64 pixels standard
- **Style**: Rounded rectangles with AWS service icons
- **Labels**: Service name below icon
- **Connections**: Clean lines with arrowheads

## Complete AWS-Style Architecture Diagram

### Diagram Title
**"AWS Contextual Chatbot with Amazon Bedrock Knowledge Bases - Architecture Overview"**

### Layout Structure (Recommended)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              USER                                       │
└─────────────────────┬───────────────────────────────────────────────────┘
                      │ HTTPS
┌─────────────────────▼───────────────────────────────────────────────────┐
│                    CLOUDFRONT                                         │
│              Global CDN Distribution                                   │
│            (Origin Group Failover)                                     │
└─────┬─────────────────────────────┬─────────────────────────────────────┘
      │ Primary Origin              │ Failover Origin
      │ (us-west-2)                 │ (us-east-1)
┌─────▼─────┐                ┌─────▼─────┐
│ S3 BUCKET │                │ S3 BUCKET │
│ Frontend  │                │ Frontend  │
│ Primary   │                │ Failover  │
└───────────┘                └───────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                      PRIMARY REGION (us-west-2)                        │
│                                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│  │ API GATEWAY │    │   LAMBDA    │    │   LAMBDA    │                │
│  │             │───▶│   QUERY     │    │   UPLOAD    │                │
│  │ REST API    │    │             │    │             │                │
│  └─────────────┘    └─────────────┘    └─────────────┘                │
│       │                     │                   │                      │
│       │                     ▼                   ▼                      │
│       │              ┌─────────────┐    ┌─────────────┐                │
│       │              │   LAMBDA    │    │   LAMBDA    │                │
│       │              │  INGEST     │    │   STATUS    │                │
│       │              │             │    │             │                │
│       │              └─────────────┘    └─────────────┘                │
│       │                     │                   │                      │
│       │                     ▼                   │                      │
│       │              ┌─────────────┐            │                      │
│       │              │   LAMBDA    │            │                      │
│       │              │   HEALTH    │            │                      │
│       │              │             │            │                      │
│       │              └─────────────┘            │                      │
│       │                     │                   │                      │
│       ▼                     ▼                   ▼                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│  │   BEDROCK   │    │   BEDROCK   │    │   BEDROCK   │                │
│  │ KNOWLEDGE   │    │  GUARDRAILS │    │   CLAUDE    │                │
│  │    BASE     │    │             │    │   SONNET    │                │
│  └─────────────┘    └─────────────┘    └─────────────┘                │
│       │                                                               │
│       ▼                                                               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│  │ OPENSEARCH  │    │ S3 BUCKET   │    │ CLOUDWATCH  │                │
│  │ SERVERLESS  │    │ DOCUMENTS   │    │ DASHBOARD   │                │
│  └─────────────┘    └─────────────┘    └─────────────┘                │
│                              │                   │                      │
│                              ▼                   ▼                      │
│                       ┌─────────────┐    ┌─────────────┐                │
│                       │ SNS TOPIC   │    │ DEAD LETTER │                │
│                       │  ALERTS     │    │    QUEUE    │                │
│                       └─────────────┘    └─────────────┘                │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              │ Cross-Region Replication
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    FAILOVER REGION (us-east-1)                         │
│                                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│  │ API GATEWAY │    │   LAMBDA    │    │   LAMBDA    │                │
│  │  (STANDBY)  │    │   QUERY     │    │   UPLOAD    │                │
│  │             │    │ (STANDBY)   │    │ (STANDBY)   │                │
│  └─────────────┘    └─────────────┘    └─────────────┘                │
│                                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│  │   BEDROCK   │    │   BEDROCK   │    │   BEDROCK   │                │
│  │ KNOWLEDGE   │    │  GUARDRAILS │    │   CLAUDE    │                │
│  │    BASE     │    │             │    │   SONNET    │                │
│  │ (STANDBY)   │    │ (STANDBY)   │    │ (STANDBY)   │                │
│  └─────────────┘    └─────────────┘    └─────────────┘                │
│                                                                         │
│  ┌─────────────┐    ┌─────────────┐                                   │
│  │ OPENSEARCH  │    │ S3 BUCKET   │                                   │
│  │ SERVERLESS  │    │ DOCUMENTS   │                                   │
│  │ (STANDBY)   │    │ (REPLICA)   │                                   │
│  └─────────────┘    └─────────────┘                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

## AWS Icons to Use

### User & External
- **User**: `user-icon.svg`
- **Internet**: `internet-gateway.svg`

### Frontend & CDN
- **CloudFront**: `cloudfront.svg`
- **S3**: `amazon-s3.svg`

### Compute
- **API Gateway**: `api-gateway.svg`
- **Lambda**: `aws-lambda.svg`

### AI/ML Services
- **Bedrock**: `amazon-bedrock.svg`
- **Bedrock Knowledge Base**: `bedrock-knowledge-base.svg`
- **Bedrock Guardrails**: `bedrock-guardrails.svg`

### Storage
- **S3**: `amazon-s3.svg`
- **OpenSearch**: `amazon-opensearch.svg`

### Monitoring
- **CloudWatch**: `amazon-cloudwatch.svg`
- **SNS**: `amazon-sns.svg`
- **SQS**: `amazon-sqs.svg`

## Step-by-Step Creation Instructions

### Using Lucidchart (Recommended)

1. **Create New Diagram**
   - Go to Lucidchart.com
   - Select "AWS Architecture" template
   - Choose blank canvas

2. **Add AWS Icons**
   - Click "Shapes" panel
   - Search for "AWS" to access icon library
   - Drag icons onto canvas

3. **Layout Components**
   - Arrange services according to the layout above
   - Use alignment tools for clean positioning
   - Group related services with rectangles

4. **Add Connections**
   - Use connector tool to draw arrows
   - Label connections where needed
   - Use different line styles for different flows

5. **Apply Styling**
   - Use AWS color scheme
   - Add service labels below icons
   - Apply consistent font (Arial or similar)

6. **Export**
   - Export as PNG (high resolution)
   - Or PDF for presentations

### Using Draw.io

1. **Setup**
   - Go to diagrams.net
   - Create new diagram
   - Add AWS icon library

2. **Import Icons**
   - Download AWS icons from official site
   - Import as custom shapes
   - Or use built-in AWS shapes

3. **Build Diagram**
   - Follow same layout structure
   - Use grid for alignment
   - Group components logically

## Professional Presentation Tips

### For Client Presentations
- **High Resolution**: Export at 300 DPI minimum
- **Clean Layout**: Plenty of white space
- **Consistent Styling**: Same icon sizes and colors
- **Clear Labels**: Service names and purposes
- **Flow Indicators**: Clear data flow arrows

### Color Coding Recommendations
- **Primary Services**: Light blue (#E6F3FF)
- **User/External**: Light green (#E6F7E6)
- **Failover/Standby**: Light gray (#F5F5F5)
- **Monitoring**: Light yellow (#FFFACD)
- **Data Flow**: Blue arrows (#0066CC)

## Alternative: Use AWS Architecture Center

AWS provides official reference architectures:
- **AWS Architecture Center**: https://aws.amazon.com/architecture/
- **Search for**: "RAG", "Bedrock", "Serverless"
- **Use as**: Starting templates for your diagram

## Final Output Specifications

### For README/Documentation
- **Format**: PNG or SVG
- **Size**: 1200x800 pixels minimum
- **Resolution**: 300 DPI for print quality

### For Client Presentations
- **Format**: PNG or PDF
- **Size**: 1920x1080 for full HD
- **Background**: White or light gray
- **Text**: Black, Arial, 12pt minimum






