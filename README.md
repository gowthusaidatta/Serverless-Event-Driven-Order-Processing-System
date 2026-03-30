# Serverless Event-Driven Order Processing System

A robust, scalable backend system for e-commerce order processing using AWS Lambda, SQS, SNS, and RDS. This project demonstrates advanced serverless architecture patterns, event-driven design principles, and asynchronous processing with eventual consistency.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [API Endpoints](#api-endpoints)
- [Components](#components)
- [Testing](#testing)
- [Local Development](#local-development)
- [Deployment](#deployment)
- [Best Practices Implemented](#best-practices-implemented)
- [Troubleshooting](#troubleshooting)

## Overview

This system showcases a modern, resilient backend for processing e-commerce orders asynchronously. By decoupling order submission from processing and notifications, the architecture achieves:

- **High Scalability**: Serverless compute that auto-scales with demand
- **Resilience**: Event-driven architecture with retry mechanisms and dead-letter queues
- **Eventual Consistency**: Asynchronous processing ensures data consistency across distributed components
- **Cost Efficiency**: Pay only for compute used, no idle infrastructure
- **Low Latency**: Immediate API response with background processing

### Key Features

✅ **POST /orders API Endpoint** - Create orders via API Gateway  
✅ **Asynchronous Processing** - SQS queue-based order processing  
✅ **Event Notifications** - SNS topic for status updates  
✅ **Data Persistence** - PostgreSQL/MySQL via RDS  
✅ **Idempotent Processing** - Safe duplicate message handling  
✅ **Error Handling** - Dead-Letter Queues for failed messages  
✅ **Structured Logging** - JSON-formatted logs for tracing  
✅ **Local Development** - Full emulation with LocalStack & Docker Compose  
✅ **Comprehensive Testing** - Unit and integration tests  
✅ **Production Deployment** - Serverless Framework configuration  

## Architecture

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Gateway                              │
│                       POST /orders                              │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 ▼
         ┌──────────────────┐
         │  Order Creator   │
         │     Lambda       │
         │  • Validates     │
         │  • Persists      │
         │  • Publishes SQS │
         └────────┬─────────┘
                  │
                  ▼ (SQS Message)
         ┌──────────────────┐
         │   Order Queue    │
         │      (SQS)       │
         │ • Durable        │
         │ • DLQ attached   │
         └────────┬─────────┘
                  │
                  ▼
         ┌──────────────────┐
         │ Order Processor  │
         │     Lambda       │
         │  • Retrieves     │
         │  • Processes     │
         │  • Updates DB    │
         │  • Publishes SNS │
         └────────┬─────────┘
                  │
                  ▼ (SNS Message)
         ┌──────────────────┐
         │  Status Topic    │
         │       (SNS)      │
         └────────┬─────────┘
                  │
                  ▼
         ┌──────────────────┐
         │  Notification    │
         │    Service       │
         │     Lambda       │
         │  • Logs status   │
         │  • Sends alerts  │
         └──────────────────┘

Database (PostgreSQL/MySQL):
┌──────────────────────────────────────────┐
│              Orders Table                │
│ • id (PK) • user_id • product_id        │
│ • quantity • status • created_at        │
│ • updated_at                            │
└──────────────────────────────────────────┘
```

### Component Interactions

1. **Client → API Gateway → OrderCreator Lambda**
   - Client submits order via POST /orders
   - Lambda validates input (product_id, quantity, user_id)
   - Returns 202 Accepted immediately

2. **OrderCreator Lambda → Database**
   - Inserts order with PENDING status
   - Generates unique order ID

3. **OrderCreator Lambda → SQS Queue**
   - Publishes message with order details
   - Message stored durably in queue

4. **SQS → OrderProcessor Lambda (Event Trigger)**
   - Lambda triggered by SQS messages (batch of up to 10)
   - Implements idempotency check

5. **OrderProcessor Lambda → Database**
   - Retrieves order details
   - Simulates processing (90% success rate)
   - Updates order status to CONFIRMED or FAILED

6. **OrderProcessor Lambda → SNS Topic**
   - Publishes status update message
   - Fans out to all subscribers

7. **SNS → NotificationService Lambda**
   - Lambda triggered by SNS messages
   - Logs notification (simulates email/SMS)

## Prerequisites

- **Docker** (version 20.10+) and **Docker Compose** (version 1.29+)
- **Python 3.9+**
- **Git**
- **AWS CLI** (for production deployment)
- **Node.js/npm** (for Serverless Framework deployment, optional)

## Quick Start

### 1. Clone and Setup

```bash
# Clone repository
git clone <repository-url>
cd serverless-order-processing

# Copy environment file
cp .env.example .env

# Install Python dependencies
pip install -r requirements.txt
```

### 2. Start LocalStack and Services

```bash
# Build and start all services
docker-compose up --build

# In another terminal, wait for services to be ready (10-15 seconds)
docker ps
```

### 3. Run Tests

```bash
# Unit tests
pytest tests/unit/ -v

# Integration tests (requires LocalStack running)
pytest tests/integration/ -v

# All tests with coverage
pytest tests/ -v --cov=src --cov-report=html
```

### 4. Test Order Creation API

```bash
# Check if LocalStack is responding
curl http://localhost:4566/_localstack/health

# Create a test order
curl -X POST http://localhost:4566/restapis/*/orders \
  -H "Content-Type: application/json" \
  -d '{
    "product_id": "PROD-123",
    "quantity": 5,
    "user_id": "USER-456"
  }'

# Expected response (202 Accepted):
# {
#   "statusCode": 202,
#   "body": {
#     "order_id": "ORD-...",
#     "message": "Order created successfully",
#     "status": "PENDING"
#   }
# }
```

## Project Structure

```
serverless-order-processing/
├── src/
│   ├── order_creator_lambda/
│   │   ├── app.py              # Order creation handler
│   │   └── Dockerfile          # Lambda container image
│   ├── order_processor_lambda/
│   │   ├── app.py              # Order processing handler
│   │   └── Dockerfile          # Lambda container image
│   ├── notification_service_lambda/
│   │   ├── app.py              # Notification handler
│   │   └── Dockerfile          # Lambda container image
│   └── shared/
│       ├── __init__.py
│       ├── database.py          # Database utilities
│       ├── aws_services.py      # AWS SQS/SNS clients
│       ├── validation.py        # Input validation
│       └── logging_config.py    # Structured logging
│
├── tests/
│   ├── unit/
│   │   ├── test_validation.py
│   │   ├── test_order_creator.py
│   │   ├── test_order_processor.py
│   │   └── test_notification_service.py
│   └── integration/
│       └── test_integration.py
│
├── infrastructure/
│   └── serverless.yml           # Serverless/SAM template
│
├── scripts/
│   ├── init-localstack.sh       # LocalStack setup
│   └── init-db.sql              # Database initialization
│
├── docker-compose.yml           # Local development orchestration
├── requirements.txt             # Python dependencies
├── .env.example                 # Environment variables template
├── conftest.py                  # Pytest configuration
└── README.md                    # This file
```

## API Endpoints

### Create Order (POST /orders)

**Request:**
```json
{
  "product_id": "PROD-001",
  "quantity": 5,
  "user_id": "USER-001"
}
```

**Response (202 Accepted):**
```json
{
  "statusCode": 202,
  "body": {
    "order_id": "ORD-ABC123",
    "message": "Order created successfully",
    "status": "PENDING"
  }
}
```

**Error Response (400 Bad Request):**
```json
{
  "statusCode": 400,
  "body": {
    "error": "Validation error message",
    "status_code": 400
  }
}
```

**Validation Rules:**
- `product_id`: Required, non-empty, max 255 characters
- `quantity`: Required, integer, > 0, ≤ 10000
- `user_id`: Required, non-empty, max 255 characters

## Components

### OrderCreator Lambda

**Responsibilities:**
- Validate incoming order payload
- Insert order into database with PENDING status
- Generate unique order ID
- Publish message to SQS queue
- Return 202 Accepted

**Key Features:**
- Input validation with comprehensive error messages
- Database transaction safety
- Idempotent ID generation
- Structured logging

### OrderProcessor Lambda

**Responsibilities:**
- Consume messages from SQS queue
- Retrieve order from database
- Simulate order processing (90% success rate)
- Update order status (CONFIRMED/FAILED)
- Publish notification to SNS
- Implement idempotent processing

**Key Fatures:**
- Idempotency check to prevent duplicate processing
- Database transaction management
- Automatic retry via SQS visibility timeout
- Error handling with DLQ support
- Structured logging with order context

### NotificationService Lambda

**Responsibilities:**
- Subscribe to SNS OrderStatusNotifications topic
- Process status update messages
- Log notifications (simulate email/SMS)
- Type-specific notification formatting

**Key Features:**
- Flexible notification type selection
- Message parsing and validation
- Structured logging
- Error resilience

## Testing

### Unit Tests

Test individual components in isolation:

```bash
# Run all unit tests
pytest tests/unit/ -v

# Run specific test file
pytest tests/unit/test_validation.py -v

# Test with coverage
pytest tests/unit/ --cov=src --cov-report=term-missing
```

**Coverage:**
- ✅ Validation: exact match, type checking, boundary conditions
- ✅ Lambda handlers: success/error paths, edge cases
- ✅ Database operations: insert, retrieve, update
- ✅ Message processing: idempotency, status updates

### Integration Tests

Test end-to-end workflows with LocalStack:

```bash
# Start LocalStack first
docker-compose up

# In another terminal, run integration tests
pytest tests/integration/ -v

# Specific integration test
pytest tests/integration/test_integration.py::TestOrderProcessingIntegration::test_sqs_queue_created -v
```

**Coverage:**
- ✅ SQS queue creation and configuration
- ✅ SNS topic creation
- ✅ Database connectivity and schema
- ✅ Message publishing and retrieval
- ✅ End-to-end order flow

### Running All Tests

```bash
# Full test suite with coverage report
pytest tests/ -v --cov=src --cov-report=html

# Open coverage report
open htmlcov/index.html
```

## Local Development

### Start Services

```bash
# Build images and start containers
docker-compose up --build

# Verify services
curl http://localhost:4566/_localstack/health  # LocalStack
curl http://localhost:5432/                     # PostgreSQL
```

### Environment Variables

Create `.env` file from template:

```bash
cp .env.example .env
```

Key variables:
- `AWS_ENDPOINT_URL`: LocalStack endpoint
- `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`: Database config
- `ORDER_PROCESSING_QUEUE_URL`: SQS queue endpoint
- `ORDER_STATUS_TOPIC_ARN`: SNS topic ARN

### Database Access

```bash
# Connect to PostgreSQL
docker exec -it postgres-db psql -U postgres -d orders_db

# List tables
\dt

# Query orders
SELECT * FROM orders;

# Check order audit trail
SELECT * FROM order_audit ORDER BY changed_at DESC;
```

### Lambda Invocation (Local)

```bash
# Invoke OrderCreator Lambda locally
docker exec order-creator-lambda python -m pytest tests/unit/test_order_creator.py

# Invoke OrderProcessor Lambda with test message
docker exec order-processor-lambda python -m pytest tests/unit/test_order_processor.py
```

## Deployment

### Prerequisites for AWS Deployment

1. **AWS Account** with appropriate IAM permissions
2. **AWS CLI** configured with credentials
3. **Node.js and npm** for Serverless Framework
4. **Configuration file** for RDS or use AWS managed database

### Deploy to AWS

```bash
# Install Serverless Framework
npm install -g serverless

# Install Serverless Python Requirements plugin
npm install --save-dev serverless-python-requirements

# Deploy to AWS
serverless deploy --stage prod --region us-east-1

# View deployed endpoints
serverless info --stage prod --region us-east-1

# Invoke Lambda function
serverless invoke --function orderCreator --stage prod --region us-east-1 --data '{"body":"{\"product_id\":\"PROD-001\",\"quantity\":5,\"user_id\":\"USER-001\"}"}'

# View logs
serverless logs -f orderCreator --stage prod --region us-east-1 --tail

# Remove deployment
serverless remove --stage prod --region us-east-1
```

### Environment Configuration for AWS

Update `.env` for AWS deployment:

```bash
AWS_ENDPOINT_URL=          # Leave empty for real AWS
DB_HOST=<RDS-endpoint>
DB_PORT=5432
DB_NAME=orders_db
DB_USER=<db-user>
DB_PASSWORD=<secure-password>
```

## Best Practices Implemented

### 1. **Event-Driven Architecture**
- Decoupled components communicate via SQS/SNS
- Orders flow through queues, not blocking calls
- Enables independent scaling of each component

### 2. **Asynchronous Processing**
- API returns 202 Accepted immediately
- Processing happens in background
- Improves user experience and system resilience

### 3. **Data Consistency**
- Individual database operations are transactional
- System ensures eventual consistency
- Status updates propagated via events

### 4. **Idempotency**
- SQS messages tracked to prevent duplicate processing
- Safe message redelivery without side effects
- Critical for exactly-once processing

### 5. **Error Handling**
- SQS Dead-Letter Queue for unprocessable messages
- Retry mechanism via visibility timeout
- Comprehensive error logging

### 6. **Structured Logging**
- JSON-formatted logs for easy parsing
- Includes timestamps, log levels, function names
- Order IDs for tracing requests through system

### 7. **Input Validation**
- Comprehensive validation at API entry point
- Type checking, boundary conditions
- Clear error messages for debugging

### 8. **Scalability Patterns**
- Serverless compute auto-scales to demand
- Queue-based buffering prevents cascade failures
- No infrastructure to manage

### 9. **Security**
- IAM policies for least privilege access
- Environment variables for sensitive data
- VPC support for private database connections (production)

### 10. **Testing**
- Unit tests for individual components
- Integration tests with LocalStack
- Coverage reporting for quality assurance

## Troubleshooting

### LocalStack Connection Issues

**Problem**: Cannot connect to LocalStack  
**Solution**:
```bash
# Verify LocalStack is running
docker ps | grep localstack

# Check health endpoint
curl http://localhost:4566/_localstack/health

# View LocalStack logs
docker logs localstack
```

### Database Connection Errors

**Problem**: Cannot connect to PostgreSQL  
**Solution**:
```bash
# Verify PostgreSQL is running
docker ps | grep postgres

# Check database credentials
docker exec postgres-db psql -U postgres -d orders_db -c "SELECT 1"

# View PostgreSQL logs
docker logs postgres-db
```

### SQS Queue Not Found

**Problem**: OrderProcessingQueue not created  
**Solution**:
```bash
# Reinitialize LocalStack
docker-compose restart localstack

# Manually create queue
docker exec localstack awslocal sqs create-queue --queue-name OrderProcessingQueue
```

### Tests Failing

**Problem**: Integration tests failing  
**Solution**:
```bash
# Ensure all services are healthy
docker-compose ps

# Wait for services to stabilize
sleep 10

# Run tests with verbose output
pytest tests/ -v --tb=short

# Check service logs
docker logs localstack
docker logs postgres-db
```

### High Database Connection Count

**Problem**: Too many open database connections  
**Solution**:
```bash
# Ensure connections are closed in Lambda handlers
# Use try/finally pattern:
try:
    db = get_db_connection()
    # ... operations ...
finally:
    if db:
        db.close()
```

## Performance Considerations

### Optimization Tips

1. **Database Indexing**: Already added on user_id, status, created_at
2. **Lambda Memory**: Set to 512MB for optimal cold start and CPU
3. **Batch Processing**: OrderProcessor handles up to 10 SQS messages per batch
4. **Connection Pooling**: Consider PgBouncer for production

### Monitoring Metrics

Track in production:
- Lambda execution time and errors
- SQS queue depth and age
- Database query performance
- DLQ message count

## Contributing

When contributing to this project:

1. Write tests for new features
2. Maintain code coverage above 80%
3. Follow PEP 8 style guidelines
4. Document architectural changes
5. Update README with new features

## License

This project is provided as-is for educational and demonstration purposes.

## Support

For issues and questions:

1. Check the Troubleshooting section above
2. Review test cases for usage examples
3. Check LocalStack logs for service issues
4. Consult AWS documentation for production deployment

---

**Version**: 1.0.0  
**Last Updated**: 2024  
**Author**: Development Team
#   S e r v e r l e s s - E v e n t - D r i v e n - O r d e r - P r o c e s s i n g - S y s t e m 
 
 