## Overview

This project demonstrates a fully serverless, event-driven architecture built on AWS designed to process asynchronous workflows in real time. Using a pizza ordering scenario, the system decouples API requests from background processing workers, updating order states sequentially while broadcasting real-time status changes back to the client application via WebSockets.

---

## Why Use Each Tool?

- **Amazon API Gateway (HTTP API)**: Provides a low-latency, cost-effective endpoint for RESTful request ingestion. Integrates directly with EventBridge without requiring intermediate compute code.
- **Amazon API Gateway (WebSocket API)**: Enables persistent, bi-directional communication, allowing the server to push real-time order updates to the client without polling.
- **Amazon EventBridge**: Serves as the central event bus, decoupling producers from consumers and routing events dynamically using declarative JSON pattern matching.
- **AWS Lambda**: Executes lightweight, stateless compute logic for event transformation and status management, scaling automatically per invocation.
- **Amazon DynamoDB**: Stores active WebSocket session mappings (`order_id` to `connection_id`) with single-digit millisecond latency.

---


## Event Schema Structure

In Amazon EventBridge, all envelope fields are standardized. Custom payload properties are wrapped inside the `detail` object. Below is the envelope structure generated when an order event is published to `lab_event_bus`:

```json
{
  "version": "0",
  "id": "e3f12a84-7c30-4b82-bc10-983dfa2890ac",
  "detail-type": "eventtype",
  "source": "lab_http_api",
  "account": "123456789012",
  "time": "2026-07-28T22:15:00Z",
  "region": "us-west-2",
  "resources": [],
  "detail": {
    "item": {
      "order_id": "3AB",
      "eventtype": "make_pizza"
    }
  }
}
```

Envelope Fields Breakdown:
- source: Identifies the service or application that generated the event (e.g., lab_http_api, make_pizza, cook_pizza).
- detail-type: Categorizes the event payload for pattern matching (configured as eventtype).
- detail: The custom JSON payload containing order specifics (order_id, eventtype).
- time: ISO-8601 timestamp representing when the event occurred.
- resources: Optional array of AWS ARNs involved in the event context.

---

## Architecture Diagram

![AWS Event-Driven Architecture](https://github.com/GiovaniSerra/aws-labs/blob/main/aws-event-driven-architecture/images/ars.jfif)

---

## Key Objectives

- Implement an event-driven pattern using EventBridge custom buses and rules.
- Integrate API Gateway directly with AWS services (EventBridge) without proxy Lambdas.
- Manage persistent WebSocket connections for real-time frontend updates.
- Demonstrate both sequential event chaining and concurrent fan-out patterns.
- Store and query session metadata using DynamoDB.

---

## Step-by-Step Implementation

### Step 1: Compute Layer Setup (AWS Lambda)
1. Created five Lambda functions using Python 3.12 (`make_pizza`, `cook_pizza`, `deliver_pizza`, `websocket_connect`, and `receive_events`).
2. Configured IAM execution roles for least-privilege access to EventBridge, DynamoDB, and API Gateway Management API.
3. Defined environment variables (`EVENT_BUS`, `TABLENAME`, `APIGW_ENDPOINT`) to remove hardcoded configuration from source code.

### Step 2: Event Bus & Routing Rules (Amazon EventBridge)
1. Provisioned custom event bus `lab_event_bus`.
2. Created pattern-matching rules for sequential state updates:
   - `lab_make_pizza_rule` -> Targets `make_pizza` Lambda.
   - `lab_cook_pizza_rule` -> Targets `cook_pizza` Lambda.
   - `lab_deliver_pizza_rule` -> Targets `deliver_pizza` Lambda.
3. Created `lab_receive_events_rule` matching all event types to trigger `receive_events` concurrently (fan-out pattern).

### Step 3: API Gateway & Frontend Integration
1. Configured HTTP API (`lab_http_api`) with a `POST` route mapped directly to EventBridge `PutEvents`.
2. Enabled CORS configuration (`*` origins, `POST` method) to allow web client execution.
3. Created WebSocket API (`lab_websocket_api`) with `$connect` route pointing to `websocket_connect` Lambda.
4. Deployed WebSocket API and updated `receive_events` Lambda with the `@connections` management endpoint URL.

---

---

## Deployment & Verification Screenshots

### 1. Compute Layer (AWS Lambda)
- **Functions Overview**: All 5 microservices deployed using Python 3.12 runtime.
  
  ![Lambda Functions](./images/functions.png)

- **Sequential Processing Lambdas**: Configuration and environment variables (`EVENT_BUS=lab_event_bus`) for `make_pizza`, `cook_pizza`, and `deliver_pizza`.
  
  | `make_pizza` | `cook_pizza` | `deliver_pizza` |
  | :---: | :---: | :---: |
  | ![make_pizza](./images/make_pizza.png) | ![cook_pizza](./images/cook_pizza.png) | ![deliver_pizza](./images/deliver_pizza.png) |

- **WebSocket Session Management Lambdas**: `websocket_connect` storing sessions in DynamoDB, and `receive_events` configured with `APIGW_ENDPOINT` to broadcast events back to clients.
  
  | `websocket_connect` | `receive_events` |
  | :---: | :---: |
  | ![websocket_connect](./images/websocket_connect.png) | ![receive_events](./images/receive_events.png) |

---

### 2. Event Routing (Amazon EventBridge)
- **Rules on Custom Event Bus**: Active rules matching pattern attributes on `lab_event_bus`.
  
  ![EventBridge Rules](https://github.com/GiovaniSerra/aws-labs/blob/main/aws-event-driven-architecture/images/event%20buses%20-%20rules.png)

- **Event Bus Metrics & Monitoring**: Metric graphs demonstrating successful event publishing, latency, and function invocations.
  
  ![EventBridge Monitoring](https://github.com/GiovaniSerra/aws-labs/blob/main/aws-event-driven-architecture/images/event%20buses%20-%20monitoring.png)

---

### 3. API Ingestion & Real-Time Gateway (Amazon API Gateway)
- **APIs Overview**: HTTP API for RESTful order entry and WebSocket API for persistent client connections.
  
  ![API Gateway APIs](./images/api%20gatew%20-%20apis.png)

- **Direct EventBridge Integration**: Route `POST /` mapped directly to EventBridge `PutEvents` action without intermediary compute code.
  
  ![API Gateway Integrations](./images/integrations.png)

- **WebSocket Route Setup**: Route `$connect` mapped directly to the `websocket_connect` Lambda function.
  
  ![WebSocket Connect Route](./images/websocket_connect.png)

- **CORS Configuration**: Cross-Origin Resource Sharing enabled for frontend client access.
  
  ![API Gateway CORS](./images/lab_http_api%20-%20cors.png)

  ---

### 4. End-to-End Application Testing
- **Real-Time Order Flow**: Web application connecting to the WebSocket API, submitting an HTTP POST order, and receiving live status updates (`make_pizza` -> `cook_pizza` -> `deliver_pizza`) as events are published and processed asynchronously.

  ![End-to-End Web App Test](./images/test.png)

  ---

## Cost Analysis & FinOps Considerations

### Estimated Cost to Reproduce: **~$0.00 USD**

This architecture runs 100% on a serverless, pay-per-use model and fits well within the **AWS Free Tier**:

- **Amazon API Gateway**:
  - *HTTP API*: $1.00 per million requests (First 1M free/month).
  - *WebSocket API*: $1.00 per million messages + connection minutes.
- **Amazon EventBridge**: $1.00 per million custom events published (First 1M free/month).
- **AWS Lambda**: Charged per request volume and duration (GB-seconds), covered by the 1M free requests/month.
- **Amazon DynamoDB**: On-Demand capacity model costing $0.00 for idle storage and minimal lab reads/writes.

---

## Infrastructure as Code with AWS CloudFormation

To deploy this architecture programmatically, use the template snippet below:

```yaml

```

---

## Key Financial Takeaways

- **Zero Idle Cost**: No servers, load balancers, or containers running when no orders are being placed.
- **Direct Service Integration**: Integrating API Gateway directly with EventBridge eliminates an intermediate Lambda function, cutting request volume costs by up to 50% on API ingestion.
- **Scale-to-Zero Efficiency**: Ideal for applications with unpredictable burst traffic, automatically scaling compute capacity during peak hours without over-provisioning.

---

---

## Error Handling & Resiliency Patterns

While this lab demonstrates a happy-path workflow, production event-driven architectures require fault-tolerant design:

1. **Dead-Letter Queues (DLQ)**: Attach an Amazon SQS queue to EventBridge target rules. If a Lambda function fails to process an event after retry attempts, the payload is directed to the DLQ for investigation instead of being lost.
2. **Lambda Retry Behavior**: EventBridge retries failed asynchronous invocations up to 2 times by default, with exponential backoff.
3. **Idempotency**: Downstream workers (e.g., `cook_pizza`) should handle duplicate event delivery safely by validating current state before processing.
4. **DynamoDB Connection Cleanup**: If a client abruptly closes a WebSocket connection, API Gateway returns a `410 GoneException`. In production, the `receive_events` Lambda should catch this exception and automatically delete stale `connection_id` records from DynamoDB.

---

## Lessons Learned & Best Practices

- **Loose Coupling**: EventBridge acts as a buffer, ensuring producers don't need awareness of consumer implementation or availability.
- **Environment Variable Isolation**: Decoupling configuration data from code simplifies multi-environment deployments (Dev/Staging/Prod).
- **Fan-Out Power**: A single published event can trigger background processing and real-time UI updates simultaneously without complex orchestration code.
- **Session Cleanup**: In production, stale WebSocket connection IDs in DynamoDB should be cleaned up upon disconnect events or via DynamoDB TTL.

