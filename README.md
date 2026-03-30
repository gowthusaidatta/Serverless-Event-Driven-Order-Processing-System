# Serverless Event-Driven Order Processing System

A serverless, event-driven backend for e-commerce order handling using AWS Lambda, API Gateway, SQS, SNS, and PostgreSQL.

## Overview

This project implements an asynchronous order pipeline:

1. Client sends `POST /orders`.
2. `OrderCreator` validates input and stores a `PENDING` order in PostgreSQL.
3. Order details are sent to SQS.
4. `OrderProcessor` consumes from SQS, updates status (`CONFIRMED` or `FAILED`), and publishes to SNS.
5. `NotificationService` consumes SNS notifications and logs delivery behavior.

The design demonstrates decoupling, eventual consistency, and resilient async processing.

## Architecture

```text
Client -> API Gateway -> OrderCreator Lambda -> PostgreSQL
                                 |
                                 v
                               SQS Queue -> OrderProcessor Lambda -> PostgreSQL
                                                            |
                                                            v
                                                          SNS Topic -> NotificationService Lambda
```

## Features

- `POST /orders` API endpoint
- Input validation with clear error responses
- SQS-based asynchronous order processing
- SNS-based status notification fan-out
- Idempotency check in processor logic (local simulation)
- PostgreSQL persistence with order lifecycle updates
- Local AWS emulation with LocalStack + Docker Compose
- Unit and integration test coverage

## Repository Structure

```text
.
|-- src/
|   |-- order_creator_lambda/
|   |-- order_processor_lambda/
|   |-- notification_service_lambda/
|   `-- shared/
|-- tests/
|   |-- unit/
|   `-- integration/
|-- scripts/
|-- infrastructure/
|-- docker-compose.yml
|-- requirements.txt
`-- README.md
```

## Prerequisites

- Docker + Docker Compose
- Python 3.9+
- Git
- Optional for cloud deployment:
  - AWS CLI
  - Node.js and npm (Serverless Framework)

## Quick Start

### 1. Clone and install dependencies

```bash
git clone <your-repository-url>
cd Serverless-Event-Driven-Order-Processing-System
pip install -r requirements.txt
```

### 2. Start local services

```bash
docker-compose up --build
```

### 3. Verify LocalStack and database

```bash
curl http://localhost:4566/_localstack/health
```

### 4. Run tests

```bash
py -m pytest tests/unit -v
py -m pytest tests/integration -v
```

## API

### Create Order

- Method: `POST`
- Path: `/orders`

Request body:

```json
{
  "product_id": "PROD-001",
  "quantity": 2,
  "user_id": "USER-001"
}
```

Success response:

```json
{
  "statusCode": 202,
  "body": "{\"order_id\":\"ORD-...\",\"message\":\"Order created successfully\",\"status\":\"PENDING\"}"
}
```

Validation rules:

- `product_id`: required, non-empty, max length 255
- `quantity`: required, integer, `> 0`
- `user_id`: required, non-empty, max length 255

## Local End-to-End Test Flow

After LocalStack initializes, the script wires:

- API Gateway `POST /orders` -> `OrderCreatorFunction`
- SQS queue -> `OrderProcessorFunction` event source mapping
- SNS topic -> `NotificationServiceFunction` subscription

Integration suite includes an end-to-end test that:

1. Calls API Gateway.
2. Waits for final DB status (`CONFIRMED` or `FAILED`).
3. Verifies NotificationService log evidence.

Run:

```bash
py -m pytest tests/integration/test_integration.py -v
```

## Deployment (AWS)

Example with Serverless Framework:

```bash
cd infrastructure
npm install
npx serverless deploy --stage prod --region us-east-1
```

To remove:

```bash
npx serverless remove --stage prod --region us-east-1
```

## Troubleshooting

### `pytest` not found

Use module invocation:

```bash
py -m pytest -v
```

### LocalStack resource errors

- Check container health:

```bash
docker ps
```

- Recreate stack:

```bash
docker-compose down -v
docker-compose up --build
```

### Database connection issues

- Ensure `postgres-db` container is healthy.
- Confirm values in `.env` match compose settings.

## Best Practices Included

- Event-driven decoupling (API, queue, topic, consumers)
- Asynchronous processing with eventual consistency
- Dead-letter queue support for failed messages
- Structured logging for traceability
- Clear separation of shared utilities and handlers
- Test pyramid with unit and integration coverage

## License

Provided for educational and demonstration purposes.
