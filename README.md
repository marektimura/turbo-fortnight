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

Component responsible for:
- ownership of time deposit products
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
| valid_from      | Timestamp | Moment in time before which the product is not valid |
| valid_until     | Timestamp | Moment in time after which the product is not valid  |

Example:

| id          | interest_rate | matures_on | count | available_count | valid_from          | valid_until         |
|-------------|---------------|------------|-------|-----------------|---------------------|---------------------|
| _\<UUID\>_  | 6.25          | 2026-12-31 | 80    | 27              | 2024-08-01T09:00:00 | 2024-09-01T09:00:00 |
| _\<UUID\>_  | 6.75          | 2025-12-31 | 50    | 0               | 2024-08-01T09:00:00 | 2024-09-01T09:00:00 |

Notes:
- Responsibilities of this component could perhaps be split among multiple components - depends on the broader context and the overall system design
- Database would ideally have at least some read replicas to support the high load during the peak window 
- `time_deposit_product.valid_from` allows to honor one of the business requirements
- `time_deposit_product.valid_until` allows to avoid offering a product for a prolonged period of time

### Notification Service

Responsible for:
- processing requests to send push notification
- integrating with a notification delivery system

### Third-party components

- Kafka (message broker)
- Temporal (business workflows, idempotency)
- Grafana (observability platform)
- Firebase (push notifications)

![alt text 0](tf-apply-for-time-deposit%20-%20Page%203.svg)

## Communication

### List time deposits

REST API endpoint `GET /time-deposits` for getting a list of time deposits. Returns all valid time deposit products including those no longer available (having `available_count <= 0`). Example:

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
- There is no need to pass exact `available_count` information to FE client - instead field `available` should be part of the response - it is `true` if `available_count > 0` else `false`.

### Apply for time deposit 

REST API endpoint `POST /time-deposits/{id}/application` for applying for a time deposit product with the given ID. Returns empty response. Example:

```
Request:
POST /time-deposits/a8ee09bf-731f-4b18-a378-ada3d2308be2/application

Response:
200 OK
```

Notes:
- A `200 OK` response does not imply account opening - only that application has been registered by the server.

### Open time deposit account

Asynchronous command `OpenTimeDepositAccountCommand` for instructing the consumer to open a time deposit account. Message key should be `timeDepositId` to ensure FIFO message order (honoring one of the business requirements). Example:

```
Key: "a8ee09bf-731f-4b18-a378-ada3d2308be2"
Value: {
  "customerId": "680ea96e-7f5b-4bff-9267-3b05efbe3526",
  "timeDepositId": "a8ee09bf-731f-4b18-a378-ada3d2308be2"
)
```

### Send notification

Asynchronous command `SendNotificationCommand` for instructing the consumer to send a notification. Example:

```
Key: null
Value: {
  "customerId": "680ea96e-7f5b-4bff-9267-3b05efbe3526",
  "notificationType": "TIME_DEPOSIT_ACCOUNT_OPENED"
)
```

![alt text 1](tf-apply-for-time-deposit%20-%20UML%20sequence_%20User%20login%20overview.svg)

## Observability

Explain how metrics are pushed to Grafana - custom actuator metrics, micrometer, etc.

Table of custom metrics

TODO: processing time

Live report can be generated from logs.

Widget ideas:
- available count simple display
- available count burndown chart
- kafka lag per partition
- denied applications per product
- time histogram of applications 

# Open problems

- How to configure time deposit plans - assumption is that for each week a new set of configurations is set up by business owner
- Customer can apply multiple times for a single product (there is no business requirement prohibiting that)
- One of the business requirement states that the account should be immediately available - this solution does not honor that
