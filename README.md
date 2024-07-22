# Assignment

## Context:
- XA bank has a marketing campaign on time deposit once weekly with a limited quota.
- There are multiple time deposit plans for customers with different periods and different interest rates but all plans are with a much more attractive interest rate than market.
- The bank promises to customers a first come first serve basis.
- XA bank currently has a client base of 3million customers. Every Monday at 9am, the XA customers can login to the mobile app and choose the corresponding time-deposit plans of their choice.
- XA bank is adopting a microservice architecture

## Business Requirements:
- Customers are able to browse a selection of time-deposit plans and select available time-deposit plans of their choice.
- Customers are able to choose the time-deposit plan and begin the time-deposit period immediately with their XA bank account (with enough money).
- The customer should be able to know which time-deposit plan is still available and which time-deposit plan is out of stock.
- The system should send notifications whenever the time-deposit plan is no longer available.
- The system should ensure that no time-deposit plan would be oversold.
- The managers would like to receive a live report regarding sales for the time-deposit plan.
- The managers would like to have a live dashboard where they can monitor the status of the system and the processing time for each time-deposit plan.

# Solution

The solution is based around two components (microservices) - account service and notification service. A mix of synchronous and asynchronous communication patterns is used to meet the stated business requirements while ensuring that peak load wouldn't cause downgrade in critical functionality that the bank system may need to serve.

## Components

### Account Service

Component is responsible for:
- ownership of time deposit products (entity `time_deposit_product`)
- providing a list of available products
- accepting and processing application for a specific product
- integrating with core banking system

#### Entity `time_deposit_product`

Entity that represents time deposit product configuration.

| Field           | Type      | Description                                          |
|-----------------|-----------|------------------------------------------------------|
| id              | UUID      | Product ID; serves as primary key                    |
| interest_rate   | Decimal   | Interest rate                                        |
| matures_on      | Date      | Maturity date                                        |
| count           | Integer   | Initial count; not updatable                         |
| available_count | Integer   | Available count                                      |
| denied_count    | Integer   | Denied application count                             |
| valid_from      | Timestamp | Moment in time before which the product is not valid |
| valid_until     | Timestamp | Moment in time after which the product is not valid  |

Example:

| id          | interest_rate | matures_on | count | available_count | denied_count | valid_from           | valid_until          |
|-------------|---------------|------------|-------|-----------------|--------------|----------------------|----------------------|
| _\<UUID\>_  | 6.25          | 2026-12-31 | 80    | 27              | 0            | 2024-07-01T09:00:00Z | 2024-09-01T09:00:00Z |
| _\<UUID\>_  | 6.75          | 2025-12-31 | 50    | 0               | 143          | 2024-07-01T09:00:00Z | 2024-09-01T09:00:00Z |

Notes:
- Responsibilities of this component could perhaps be split among multiple components - depends on the broader context and the overall system design
- Database would ideally have at least some read replicas to support the high load during the peak window 
- `time_deposit_product.valid_from` allows to honor one of the business requirements
- `time_deposit_product.valid_until` allows to avoid offering a product for a prolonged period of time

### Notification Service

Component is responsible for:
- processing requests to send push notification
- integrating with a notification delivery system

### Third-party components

- Kafka (message broker)
- Temporal (business workflows, idempotency)
- Grafana (observability dashboard)
- Firebase (push notifications)

![alt text 0](tf-apply-for-time-deposit%20-%20Page%203.svg)

## Communication

### List time deposits

REST API endpoint `GET /time-deposits` served by account service for getting a list of time deposits. Returns all valid time deposit products including those no longer available (having `available_count <= 0`). Example:

```
Request:
GET /time-deposits

Response:
200 OK
{
  "timeDeposits": [
    {
      "id": "a8ee09bf-731f-4b18-a378-ada3d2308be2",
      "interestRate": 6.25
      "maturesOn": "2026-12-31",
      "available": true
    },
    {
      "id": "2899cb16-84d4-4f53-93c5-44b2ebd786a2",
      "interestRate": 6.75
      "maturesOn": "2025-12-31",
      "available": false
    },
    ...
  ]
}
```

Notes:
- There is no need to pass exact `available_count` information to FE client - instead field `available` can be part of the response - it is `true` if `available_count > 0` else `false`.

### Apply for time deposit 

REST API endpoint `POST /time-deposits/{id}/application` served by account service for applying for a time deposit product with the given ID. Returns empty response. Example:

```
Request:
POST /time-deposits/a8ee09bf-731f-4b18-a378-ada3d2308be2/application

Response:
200 OK
```

Notes:
- A `200 OK` response does not imply account opening - only that application has been registered by the server.

### Open time deposit account

Asynchronous command `OpenTimeDepositAccountCommand` for instructing the consumer to open a time deposit account. Consumed by account service. Message key should be `timeDepositId` to ensure FIFO message order (honoring one of the business requirements). Example:

```
Key: "a8ee09bf-731f-4b18-a378-ada3d2308be2"
Value: {
  "customerId": "680ea96e-7f5b-4bff-9267-3b05efbe3526",
  "timeDepositId": "a8ee09bf-731f-4b18-a378-ada3d2308be2"
)
```

### Send notification

Asynchronous command `SendNotificationCommand` for instructing the consumer to send a notification. Consumed by notification service. Example:

```
Key: null
Value: {
  "customerId": "680ea96e-7f5b-4bff-9267-3b05efbe3526",
  "notificationType": "TIME_DEPOSIT_ACCOUNT_OPENED"
)
```

### Business flow for applying for a time deposit account

![alt text 1](tf-apply-for-time-deposit%20-%20sequence.svg)

## Observability

As mentioned in the Components section, Grafana will be used as an observability platform. Micrometer and Prometheus can be used as an observability facade and metrics scraper respectively.

Account service will expose custom metrics (per time deposit product) via actuator such as:
- available count
- denied application count

Metrics will be visualized in Grafana. Some widget ideas include:
- simple display of available count per product
- burndown chart of available count per product
- total applications per product
- denied applications per product
- time histogram of applications
- kafka lag per partition

Additionally, a live report of sales can be visualised by filtering logs related to successful openings of time deposit accounts. Logs matching a certain pattern can be aggregated and provide number of account openings for a relative time period (e.g. last 15 minutes).

# Open problems

- Customer can apply multiple times for a single product (there is no business requirement prohibiting that)
- One of the business requirement states that the account should be _immediately_ available - this solution does not guarantee that
