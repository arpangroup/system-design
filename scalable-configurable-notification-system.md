# Scalable, Configurable & Extensible Notification System
## Step-by-Step System Design Guide

> **Goal:** Design a production-grade notification platform that can send notifications through multiple channels such as Push, SMS, Email, WhatsApp, In-App, and WebSocket, while remaining scalable, configurable, fault-tolerant, and easy to extend.
>
> The design should follow **SOLID principles**, clean architecture, separation of concerns, and an event-driven approach so that adding a new channel, provider, retry strategy, template engine, preference rule, or business feature does not require rewriting the existing system.

---

# 1. Interview / System Design Question

## Primary Question

> **Design a scalable notification system for a large application.**
>
> The system should allow different business services to send notifications to users through multiple channels such as:
>
> - Push notification
> - Email
> - SMS
> - WhatsApp
> - In-app notification
> - WebSocket
>
> The system must support:
>
> 1. Multiple notification channels.
> 2. Multiple third-party providers per channel.
> 3. User notification preferences.
> 4. Notification templates.
> 5. Immediate and scheduled notifications.
> 6. Retry and failure handling.
> 7. Priority-based delivery.
> 8. Rate limiting.
> 9. Deduplication / idempotency.
> 10. Delivery status tracking.
> 11. Provider failover.
> 12. Large traffic spikes.
> 13. Multi-tenant configuration.
> 14. Localization.
> 15. Extensibility without modifying existing business logic.
>
> **Design the architecture, APIs, database model, event flow, class design, configuration model, failure strategy, scaling strategy, and future enhancements.**

---

# 2. Clarify Requirements

Before designing, divide requirements into **functional** and **non-functional** requirements.

## 2.1 Functional Requirements

The system should support:

```text
Business Event
      |
      v
Notification Request
      |
      +----> Push
      +----> Email
      +----> SMS
      +----> WhatsApp
      +----> In-App
      +----> WebSocket
```

### Notification Operations

- Send notification immediately.
- Schedule notification.
- Cancel scheduled notification.
- Retry failed notification.
- Query notification status.
- Query notification history.
- Mark in-app notification as read.
- Send to one user.
- Send to multiple users.
- Send to a group/segment.
- Broadcast notification.

---

# 3. Non-Functional Requirements

The system should be:

### Scalable

It should handle:

```text
100 notifications/sec
        |
        v
10,000 notifications/sec
        |
        v
1,000,000+ notifications/sec
```

without redesigning the entire application.

### Highly Available

A failure of:

- Email provider
- SMS provider
- Push provider
- One worker
- One application instance

should not bring down the entire notification system.

### Fault Tolerant

Temporary failures should be retried.

Permanent failures should eventually move to a DLQ.

### Extensible

Adding:

```text
Telegram
Slack
Teams
Voice Call
New SMS Provider
New Email Provider
```

should require minimal changes.

### Configurable

Business behavior should not be hardcoded.

For example:

```yaml
notification:
  channels:
    email:
      enabled: true

    sms:
      enabled: true

    push:
      enabled: true

  retry:
    max-attempts: 5

  provider:
    sms:
      primary: twilio
      fallback: aws-sns
```

---

# 4. High-Level Architecture

A good scalable architecture is event-driven.

```text
                         +----------------------+
                         | Business Services    |
                         | Order / Payment etc. |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | Notification API     |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | Notification         |
                         | Orchestrator         |
                         +----------+-----------+
                                    |
                         +----------+-----------+
                         |                      |
                         v                      v
                 Preference Service       Template Service
                         |                      |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | Message Broker       |
                         | Kafka / RabbitMQ     |
                         +----------+-----------+
                                    |
              +---------------------+---------------------+
              |                     |                     |
              v                     v                     v
       +-------------+       +-------------+       +-------------+
       | Push Worker |       | Email Worker|       | SMS Worker  |
       +------+------+       +------+------+       +------+------+
              |                     |                     |
              v                     v                     v
          FCM/APNs             SES/SendGrid           Twilio/SNS

              +-------------------------------------------+
              |
              v
       +----------------+
       | Delivery Store |
       +----------------+
```

---

# 5. Why Event-Driven Architecture?

Avoid this design:

```text
Order Service
     |
     +---- call Email API
     |
     +---- call SMS API
     |
     +---- call Push API
```

This creates tight coupling.

If SMS provider takes 10 seconds:

```text
Order API
   |
   +---- SMS -> 10 sec
   |
   +---- Push
   |
   +---- Email
```

The business API becomes slow.

Instead:

```text
Order Service
      |
      | publish event
      v
 Notification Topic
      |
      +---- Email Worker
      +---- SMS Worker
      +---- Push Worker
```

The business operation becomes independent of notification delivery.

---

# 6. Step 1 — Notification API

Expose a generic API.

## Send Notification

```http
POST /api/v1/notifications
```

Request:

```json
{
  "userId": "U123",
  "eventType": "ORDER_CONFIRMED",
  "channels": ["PUSH", "EMAIL"],
  "templateId": "order-confirmed",
  "data": {
    "orderId": "ORD123",
    "amount": "999"
  },
  "priority": "HIGH",
  "idempotencyKey": "ORDER-ORD123-CONFIRMED"
}
```

Response:

```json
{
  "notificationId": "NTF123",
  "status": "ACCEPTED"
}
```

Important:

The API should normally return:

```text
ACCEPTED
```

instead of waiting for:

```text
Email delivered
SMS delivered
Push delivered
```

---

# 7. Step 2 — Notification Domain Model

Do not make the domain depend directly on FCM, Twilio, SES, etc.

Use a generic model.

```java
public class Notification {

    private String id;

    private String userId;

    private String eventType;

    private String templateId;

    private NotificationPriority priority;

    private NotificationStatus status;

    private Map<String, Object> data;

    private Instant scheduledAt;

    private String idempotencyKey;
}
```

---

# 8. Step 3 — Notification Channel

Create an abstraction.

```java
public interface NotificationChannel {

    ChannelType getType();

    DeliveryResult send(NotificationMessage message);
}
```

Examples:

```text
PushChannel
EmailChannel
SmsChannel
WhatsAppChannel
InAppChannel
WebSocketChannel
```

This follows:

## Open/Closed Principle

New channels can be added without modifying the core orchestration logic.

---

# 9. Step 4 — Channel Implementations

```java
public class EmailChannel implements NotificationChannel {

    private final EmailProvider provider;

    @Override
    public ChannelType getType() {
        return ChannelType.EMAIL;
    }

    @Override
    public DeliveryResult send(NotificationMessage message) {
        return provider.send(message);
    }
}
```

Similarly:

```java
public class SmsChannel implements NotificationChannel {

    private final SmsProvider provider;

    @Override
    public ChannelType getType() {
        return ChannelType.SMS;
    }
}
```

---

# 10. Step 5 — Provider Abstraction

Do not couple `SmsChannel` directly to Twilio.

Create:

```java
public interface SmsProvider {

    ProviderResponse send(SmsMessage message);

    boolean isAvailable();
}
```

Implementations:

```text
TwilioSmsProvider
AwsSnsSmsProvider
VonageSmsProvider
```

Architecture:

```text
             SmsChannel
                 |
                 v
          SmsProvider
           /       \
          /         \
     Twilio        AWS SNS
```

Now changing provider does not change the channel.

---

# 11. Step 6 — Provider Factory / Registry

Avoid:

```java
if (provider.equals("TWILIO")) {
   ...
} else if (provider.equals("AWS")) {
   ...
}
```

Instead:

```java
public interface ProviderRegistry {

    <T> T getProvider(
        ChannelType channel,
        String providerName
    );
}
```

Example:

```text
ProviderRegistry

EMAIL
 ├── SES
 ├── SENDGRID
 └── SMTP

SMS
 ├── TWILIO
 ├── AWS_SNS
 └── VONAGE

PUSH
 ├── FCM
 └── APNS
```

---

# 12. Step 7 — Notification Orchestrator

The orchestrator coordinates the workflow.

```java
public interface NotificationOrchestrator {

    NotificationResult process(
        NotificationRequest request
    );
}
```

Conceptually:

```text
Notification Request
       |
       v
Validate
       |
       v
Load User Preferences
       |
       v
Resolve Channels
       |
       v
Resolve Template
       |
       v
Create Notification
       |
       v
Publish Events
```

The orchestrator should NOT know:

```text
how Twilio works
how FCM works
how SES works
```

It only knows abstractions.

---

# 13. Step 8 — Notification Preferences

Users should control notification preferences.

Example:

```text
User: U123

ORDER_CONFIRMED
    PUSH   = ON
    EMAIL  = ON
    SMS    = OFF

PROMOTIONAL
    PUSH   = OFF
    EMAIL  = OFF
    SMS    = OFF
```

Database:

```text
user_notification_preferences

id
user_id
event_type
channel
enabled
quiet_hours_start
quiet_hours_end
updated_at
```

---

# 14. Preference Resolution

Do not simply check:

```java
if (userPreference == true)
```

Use a policy abstraction.

```java
public interface NotificationPolicy {

    PolicyDecision evaluate(
        NotificationContext context
    );
}
```

Possible policies:

```text
UserPreferencePolicy
QuietHoursPolicy
DoNotDisturbPolicy
ConsentPolicy
FrequencyPolicy
TenantPolicy
ChannelPolicy
```

Then:

```text
Notification
     |
     v
Policy Engine
     |
     +---- User Preference
     +---- Quiet Hours
     +---- Consent
     +---- Frequency Limit
     +---- Tenant Rules
     |
     v
Allowed Channels
```

---

# 15. Step 9 — Templates

Never hardcode notification messages.

Bad:

```java
"Your order " + orderId + " has been confirmed";
```

Instead:

```text
Template:

Order {{orderId}} has been confirmed.
Amount: {{amount}}
```

Template database:

```text
notification_templates

id
template_code
event_type
channel
language
subject
body
version
status
created_at
updated_at
```

Example:

```text
template_code:
ORDER_CONFIRMED

channel:
EMAIL

language:
en-IN
```

---

# 16. Step 10 — Template Resolver

```java
public interface TemplateResolver {

    NotificationTemplate resolve(
        String templateId,
        ChannelType channel,
        Locale locale
    );
}
```

This allows:

```text
ORDER_CONFIRMED
       |
       +---- EMAIL / en-IN
       +---- EMAIL / hi-IN
       +---- PUSH  / en-IN
       +---- SMS   / en-IN
```

---

# 17. Step 11 — Localization

Support:

```text
English
Hindi
Bengali
Telugu
Tamil
```

Template selection:

```text
templateId
+
channel
+
locale
+
version
```

Example:

```text
ORDER_CONFIRMED
EMAIL
hi-IN
v3
```

---

# 18. Step 12 — Message Broker

Use a broker between the API and workers.

Possible technologies:

```text
Kafka
RabbitMQ
AWS SQS
Google Pub/Sub
Azure Service Bus
```

For very high throughput and event streaming:

```text
Kafka
```

is a common choice.

---

# 19. Topic Design

A simple model:

```text
notification.events
```

Or channel-specific:

```text
notification.push
notification.email
notification.sms
notification.whatsapp
```

A useful scalable architecture is:

```text
notification.events
       |
       v
 Notification Router
       |
       +---- notification.push
       +---- notification.email
       +---- notification.sms
       +---- notification.whatsapp
```

---

# 20. Step 13 — Notification Router

The router determines where the notification goes.

```java
public interface NotificationRouter {

    List<ChannelType> route(
        NotificationContext context
    );
}
```

Example:

```text
ORDER_CONFIRMED
     |
     +---- PUSH
     +---- EMAIL

PASSWORD_RESET
     |
     +---- EMAIL
     +---- SMS
```

This mapping should ideally be configuration-driven.

---

# 21. Step 14 — Configuration-Driven Rules

Instead of:

```java
if (eventType == ORDER_CONFIRMED) {
    sendPush();
    sendEmail();
}
```

Store configuration.

Example:

```yaml
notifications:

  ORDER_CONFIRMED:
    channels:
      - PUSH
      - EMAIL

  PAYMENT_FAILED:
    channels:
      - PUSH
      - SMS
      - EMAIL

  PASSWORD_RESET:
    channels:
      - EMAIL
      - SMS
```

Better still, store rules in a database/config service when runtime changes are required.

---

# 22. Configuration Hierarchy

A production system may have:

```text
Global Configuration
        |
        v
Tenant Configuration
        |
        v
Application Configuration
        |
        v
Event Configuration
        |
        v
User Preferences
        |
        v
Final Delivery Decision
```

Example:

```text
Global:
SMS enabled

Tenant:
SMS disabled

Event:
SMS enabled

User:
SMS enabled

Final:
SMS disabled
```

Tenant restrictions should win.

---

# 23. Step 15 — Priority

Support:

```text
CRITICAL
HIGH
NORMAL
LOW
```

Example:

```text
OTP             -> CRITICAL
Payment Failed  -> HIGH
Order Confirmed -> NORMAL
Marketing       -> LOW
```

Kafka/RabbitMQ can use separate queues/topics:

```text
notification.high
notification.normal
notification.low
```

This prevents promotional traffic from delaying OTP or security notifications.

---

# 24. Step 16 — Worker Architecture

Workers consume messages.

```text
              Message Broker
                    |
       +------------+------------+
       |            |            |
       v            v            v
  Push Workers  SMS Workers  Email Workers
       |            |            |
       v            v            v
      FCM          Twilio       SES
```

Workers should be stateless.

That allows horizontal scaling:

```text
Email Worker x 1
      |
      v
Email Worker x 10
      |
      v
Email Worker x 100
```

---

# 25. Step 17 — Retry Strategy

Not every failure should be retried.

## Transient Error

Examples:

```text
Timeout
Connection reset
HTTP 429
Provider temporarily unavailable
HTTP 503
```

Retry.

## Permanent Error

Examples:

```text
Invalid email
Invalid phone number
Invalid template
User blocked
Invalid token
```

Do not retry indefinitely.

---

# 26. Exponential Backoff

Example:

```text
Attempt 1 -> immediately
Attempt 2 -> 1 sec
Attempt 3 -> 5 sec
Attempt 4 -> 30 sec
Attempt 5 -> 5 min
```

Formula:

```text
delay = min(maxDelay, baseDelay * 2^attempt)
```

Add jitter:

```text
delay = exponentialDelay + randomJitter
```

This prevents a thundering herd.

---

# 27. Step 18 — Dead Letter Queue

After maximum attempts:

```text
Main Queue
    |
    v
Worker
    |
    +---- success
    |
    +---- retry
            |
            v
        Retry Queue
            |
            v
        Worker
            |
            +---- success
            |
            +---- max retries
                    |
                    v
                   DLQ
```

DLQ messages can later be:

```text
inspected
replayed
manually resolved
automatically replayed
```

---

# 28. Step 19 — Idempotency

A distributed system may deliver the same message more than once.

Example:

```text
Notification Event
       |
       v
Worker sends SMS
       |
       v
Provider responds slowly
       |
       v
Worker times out
       |
       v
Retry
```

The SMS may already have been sent.

Use:

```text
idempotency_key
```

Example:

```text
ORDER-123-PAYMENT-SUCCESS
```

Database:

```text
notification_delivery

id
notification_id
channel
provider
idempotency_key
status
attempt_count
```

Unique constraint:

```text
UNIQUE(
    notification_id,
    channel
)
```

For provider-level idempotency, use the provider's supported idempotency mechanism where available.

---

# 29. Step 20 — Delivery State Machine

Use explicit states.

```text
CREATED
   |
   v
QUEUED
   |
   v
PROCESSING
   |
   +------> SENT
   |          |
   |          v
   |       DELIVERED
   |
   +------> FAILED
              |
              v
            RETRY
              |
              v
          PROCESSING

FAILED -> DLQ
```

For each channel maintain independent state.

Example:

```text
Notification: N123

PUSH:
  DELIVERED

EMAIL:
  SENT

SMS:
  FAILED
```

---

# 30. Step 21 — Database Design

## notification

```sql
CREATE TABLE notification (
    id              UUID PRIMARY KEY,
    user_id         VARCHAR(100) NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    template_id     VARCHAR(100),
    priority        VARCHAR(20),
    status          VARCHAR(30),
    idempotency_key VARCHAR(200),
    scheduled_at    TIMESTAMP,
    created_at      TIMESTAMP NOT NULL,
    updated_at      TIMESTAMP NOT NULL
);
```

## notification_delivery

```sql
CREATE TABLE notification_delivery (
    id              UUID PRIMARY KEY,
    notification_id UUID NOT NULL,
    channel         VARCHAR(30) NOT NULL,
    provider        VARCHAR(50),
    status          VARCHAR(30),
    attempt_count   INT DEFAULT 0,
    provider_id     VARCHAR(200),
    error_code      VARCHAR(100),
    error_message   TEXT,
    sent_at         TIMESTAMP,
    delivered_at    TIMESTAMP,
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP
);
```

---

# 31. Preference Tables

```sql
CREATE TABLE notification_preference (
    id          UUID PRIMARY KEY,
    user_id     VARCHAR(100) NOT NULL,
    event_type  VARCHAR(100) NOT NULL,
    channel     VARCHAR(30) NOT NULL,
    enabled     BOOLEAN NOT NULL,
    created_at  TIMESTAMP,
    updated_at  TIMESTAMP,

    UNIQUE(user_id, event_type, channel)
);
```

---

# 32. Template Tables

```sql
CREATE TABLE notification_template (
    id              UUID PRIMARY KEY,
    template_code   VARCHAR(100) NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    channel         VARCHAR(30) NOT NULL,
    locale          VARCHAR(20) NOT NULL,
    subject         TEXT,
    body            TEXT,
    version         INT NOT NULL,
    status          VARCHAR(20),
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP
);
```

---

# 33. Provider Configuration

Do NOT store secrets directly in ordinary configuration tables.

Store references to:

```text
AWS Secrets Manager
HashiCorp Vault
Azure Key Vault
Kubernetes Secrets
```

Configuration:

```yaml
providers:

  sms:
    primary: twilio

    fallback:
      - aws-sns
      - vonage

  email:
    primary: ses

    fallback:
      - sendgrid

  push:
    primary: fcm
```

Secrets:

```text
twilio.account.sid
twilio.auth.token
```

should be stored in a secrets manager.

---

# 34. Step 22 — Provider Failover

Suppose:

```text
SMS
 |
 +---- Twilio -> DOWN
 |
 +---- AWS SNS -> UP
```

Provider selection:

```java
public interface ProviderSelector {

    NotificationProvider select(
        ChannelType channel,
        NotificationContext context
    );
}
```

Configuration:

```yaml
sms:
  providers:
    - name: twilio
      priority: 1
    - name: aws-sns
      priority: 2
```

The selector should consider:

```text
enabled
health
quota
cost
region
tenant
priority
```

Do not hardcode provider decisions into workers.

---

# 35. Step 23 — Circuit Breaker

If a provider is continuously failing:

```text
Worker
  |
  v
Twilio
  |
  X
500
  |
  X
500
  |
  X
500
```

Open a circuit:

```text
CLOSED
   |
   v
FAILURES
   |
   v
OPEN
   |
   v
HALF_OPEN
   |
   +---- success -> CLOSED
   |
   +---- failure -> OPEN
```

This protects the notification system from repeatedly calling an unhealthy provider.

Libraries:

```text
Resilience4j
Spring Cloud Circuit Breaker
```

---

# 36. Step 24 — Rate Limiting

Rate limit at multiple levels.

```text
Global
Tenant
User
Channel
Provider
API
```

Example:

```text
User:
Maximum 5 OTP requests / 10 minutes

Tenant:
Maximum 100,000 SMS / hour

Provider:
Maximum 10,000 SMS / minute
```

Redis is commonly useful for distributed counters/token buckets.

---

# 37. Step 25 — Scheduled Notifications

Support:

```http
POST /api/v1/notifications/schedule
```

Request:

```json
{
  "userId": "U123",
  "templateId": "REMINDER",
  "scheduledAt": "2026-10-01T09:00:00Z"
}
```

Architecture:

```text
API
 |
 v
Scheduler Store
 |
 v
Scheduler
 |
 v
Message Broker
 |
 v
Workers
```

For large scale, avoid scanning millions of database rows every second.

Possible approaches:

```text
Delay Queue
Kafka + scheduler
Redis Sorted Set
Dedicated scheduler service
Cloud Scheduler
```

---

# 38. Step 26 — Batch Notifications

Do not create an HTTP request for every recipient.

Bad:

```text
1 million users
    |
1 million API requests
```

Better:

```text
Campaign
   |
   v
Audience
   |
   v
Batch Generator
   |
   +---- 10,000
   +---- 10,000
   +---- 10,000
   ...
```

Workers process batches.

---

# 39. Step 27 — Broadcast / Campaign Architecture

Separate transactional notifications from marketing campaigns.

```text
Transactional
    |
    +---- OTP
    +---- Payment
    +---- Order

Campaign
    |
    +---- Promotion
    +---- Newsletter
    +---- Announcement
```

Why?

Because campaign traffic can be huge.

Example:

```text
1 million promotional push notifications
```

should not block:

```text
OTP
Payment failure
Security alerts
```

Use separate queues, quotas, and worker pools.

---

# 40. Step 28 — Multi-Tenant Architecture

If this is a SaaS platform:

```text
Tenant A
Tenant B
Tenant C
```

Each tenant can configure:

```text
channels
providers
templates
limits
branding
sender IDs
retry policy
quiet hours
```

Example:

```yaml
tenant:
  id: TENANT_A

  notification:
    sms:
      enabled: true

    email:
      provider: ses

    limits:
      smsPerMinute: 1000
```

Tenant isolation must exist at:

```text
API
Database
Kafka topics/partitions
Configuration
Metrics
Authorization
```

---

# 41. Step 29 — SOLID Design

## Single Responsibility Principle

Each class should have one responsibility.

Bad:

```text
NotificationService
    |
    +-- validate
    +-- template
    +-- preference
    +-- FCM
    +-- SMS
    +-- Email
    +-- retry
    +-- database
```

Better:

```text
NotificationValidator
TemplateResolver
PreferenceService
NotificationRouter
ProviderSelector
RetryPolicy
DeliveryService
NotificationRepository
```

---

# 42. Open/Closed Principle

Adding:

```text
WhatsApp
```

should not require changing:

```text
NotificationOrchestrator
```

Add:

```java
class WhatsAppChannel
```

and register it.

---

# 43. Liskov Substitution Principle

Every provider implementation should behave according to the provider abstraction.

```java
SmsProvider
   |
   +-- TwilioSmsProvider
   +-- AwsSmsProvider
   +-- VonageSmsProvider
```

The channel should work with any valid implementation.

---

# 44. Interface Segregation Principle

Avoid one huge interface:

```java
interface NotificationProvider {

    sendSms();
    sendEmail();
    sendPush();
    sendWhatsApp();
}
```

Instead:

```java
interface SmsProvider
interface EmailProvider
interface PushProvider
interface WhatsAppProvider
```

---

# 45. Dependency Inversion Principle

High-level business logic depends on abstractions.

```text
NotificationOrchestrator
          |
          v
     NotificationChannel
          |
          v
       Provider
```

Not:

```text
NotificationOrchestrator
          |
          v
       Twilio SDK
```

---

# 46. Recommended Package Structure

For Spring Boot:

```text
notification-service
│
├── api
│   ├── NotificationController
│   └── dto
│
├── application
│   ├── NotificationOrchestrator
│   ├── NotificationCommandService
│   └── NotificationQueryService
│
├── domain
│   ├── model
│   ├── event
│   ├── policy
│   └── repository
│
├── channel
│   ├── NotificationChannel
│   ├── push
│   ├── email
│   ├── sms
│   └── whatsapp
│
├── provider
│   ├── ProviderSelector
│   ├── sms
│   ├── email
│   └── push
│
├── template
│
├── preference
│
├── scheduler
│
├── retry
│
├── infrastructure
│   ├── kafka
│   ├── redis
│   ├── database
│   └── external
│
└── configuration
```

---

# 47. Step 30 — End-to-End Flow

Suppose an order is confirmed.

```text
Order Service
     |
     | OrderConfirmedEvent
     v
Notification API / Event Consumer
     |
     v
Notification Orchestrator
     |
     +---- Validate
     |
     +---- Load Preferences
     |
     +---- Resolve Template
     |
     +---- Resolve Locale
     |
     +---- Resolve Channels
     |
     v
Notification Created
     |
     v
Kafka
     |
     +------------+------------+
     |            |            |
     v            v            v
 Push Queue   Email Queue   SMS Queue
     |            |            |
     v            v            v
Push Worker   Email Worker   SMS Worker
     |            |            |
     v            v            v
FCM/APNs       SES/SMTP       Twilio
     |            |            |
     +------------+------------+
                  |
                  v
          Delivery Status
                  |
                  v
          Notification DB
                  |
                  v
            Metrics/Event
```

---

# 48. Step 31 — Event-Driven Business Integration

Prefer business events:

```text
OrderConfirmed
PaymentSuccessful
PaymentFailed
UserRegistered
PasswordChanged
DriverAssigned
TripStarted
TripCompleted
```

Business services publish events.

Notification service subscribes.

This means:

```text
Order Service
```

does not need to know:

```text
Email
SMS
Push
WhatsApp
```

---

# 49. Transactional Outbox Pattern

A major production problem:

```text
Database transaction succeeds
       |
       X
Kafka publish fails
```

Now the business event is lost.

Use:

```text
Business DB
    |
    +---- Business Data
    |
    +---- Outbox Event
             |
             v
       Outbox Publisher
             |
             v
           Kafka
```

Both business data and outbox event are committed in the same DB transaction.

Then the publisher reliably sends the event to Kafka.

---

# 50. Step 32 — Exactly Once vs At Least Once

Do not assume exactly-once delivery across external providers.

A realistic design is:

```text
At-Least-Once Processing
+
Idempotent Consumers
+
Idempotency Keys
```

This gives practical duplicate protection.

---

# 51. Step 33 — Observability

Every notification should have:

```text
notificationId
correlationId
traceId
userId
tenantId
eventType
channel
provider
attempt
status
latency
errorCode
```

Example:

```text
notificationId = N123
channel        = SMS
provider       = TWILIO
attempt        = 2
status         = DELIVERED
latency        = 420ms
```

---

# 52. Metrics

Important metrics:

```text
notifications.created
notifications.sent
notifications.delivered
notifications.failed
notifications.retried
notifications.dlq

notification.latency
provider.latency

provider.error.rate
provider.success.rate

queue.depth
queue.lag

rate.limit.exceeded
```

Track metrics by:

```text
tenant
channel
provider
eventType
region
```

---

# 53. Distributed Tracing

Use:

```text
OpenTelemetry
```

Trace:

```text
Order Service
   |
   v
Kafka
   |
   v
Notification Service
   |
   v
SMS Worker
   |
   v
Twilio
```

This makes debugging distributed failures much easier.

---

# 54. Step 34 — Security

Protect:

```text
Notification APIs
Provider credentials
User contact information
Templates
Tenant configuration
```

Use:

```text
OAuth2 / JWT
RBAC
API authentication
Encryption at rest
TLS
Secrets Manager
Audit logs
```

Never log:

```text
OTP
password reset token
provider secret
full sensitive payload
```

---

# 55. Step 35 — API Idempotency

Client sends:

```http
POST /notifications

Idempotency-Key: ORDER-123-CONFIRMED
```

If request is repeated:

```text
Request 1 -> N123
Request 2 -> N123
Request 3 -> N123
```

Do not create three notifications.

---

# 56. Step 36 — Notification API Versioning

Use:

```text
/api/v1/notifications
/api/v2/notifications
```

Do not break existing clients when request structures evolve.

---

# 57. Step 37 — Configuration Service

As the platform grows, centralize dynamic configuration.

```text
              Config Service
                    |
       +------------+-------------+
       |            |             |
       v            v             v
 Notification   Provider       Template
   Rules         Rules           Rules
```

Possible technologies:

```text
Spring Cloud Config
Consul
AWS AppConfig
LaunchDarkly
Database-backed configuration
```

Use caching so notification delivery does not depend on a config service being available for every request.

---

# 58. Step 38 — Feature Flags

Use feature flags for gradual rollout.

Example:

```yaml
features:
  whatsapp:
    enabled: false

  new-email-provider:
    enabled: true

  smart-provider-routing:
    enabled: false
```

Roll out:

```text
0%
 |
5%
 |
25%
 |
50%
 |
100%
```

---

# 59. Step 39 — Smart Provider Routing

Initially:

```text
SMS -> Twilio
```

Later:

```text
Provider Selector
       |
       +---- Twilio
       +---- AWS
       +---- Vonage
```

Routing can consider:

```text
cost
latency
success rate
quota
region
tenant
provider health
message type
```

Keep the routing algorithm behind:

```java
ProviderSelectionStrategy
```

Possible strategies:

```java
RoundRobinStrategy
PriorityStrategy
CostBasedStrategy
LatencyBasedStrategy
HealthBasedStrategy
WeightedStrategy
```

---

# 60. Strategy Pattern

This is a good place for Strategy Pattern.

```java
public interface ProviderSelectionStrategy {

    SmsProvider select(
        List<SmsProvider> providers,
        NotificationContext context
    );
}
```

Then:

```text
PriorityStrategy
WeightedStrategy
HealthBasedStrategy
CostBasedStrategy
```

can be added independently.

---

# 61. Step 40 — Chain of Responsibility for Policies

Policies can be composed.

```text
Notification
      |
      v
Consent Policy
      |
      v
User Preference Policy
      |
      v
Quiet Hours Policy
      |
      v
Rate Limit Policy
      |
      v
Tenant Policy
      |
      v
Final Decision
```

Example:

```java
public interface NotificationPolicy {

    PolicyResult evaluate(NotificationContext context);
}
```

This avoids one massive `if/else` block.

---

# 62. Step 41 — Template Rendering Strategy

Different channels may require different renderers.

```java
public interface TemplateRenderer {

    RenderedMessage render(
        NotificationTemplate template,
        Map<String, Object> data
    );
}
```

Implement:

```text
EmailTemplateRenderer
SmsTemplateRenderer
PushTemplateRenderer
WhatsAppTemplateRenderer
```

For email:

```text
HTML
Plain Text
Subject
Attachments
```

For push:

```text
Title
Body
Image
Deep Link
Actions
```

---

# 63. Step 42 — User Device Management

Push notifications require device registration.

```text
user_device

id
user_id
device_id
platform
push_token
app_version
locale
timezone
active
last_seen_at
```

One user may have:

```text
Android phone
iPhone
Tablet
Web browser
```

Therefore:

```text
User
 |
 +---- Device 1
 +---- Device 2
 +---- Device 3
```

Push delivery should resolve active devices.

---

# 64. Step 43 — Web Push / PWA

For web applications:

```text
Browser
   |
   v
Service Worker
   |
   v
Push Subscription
   |
   v
Notification Service
```

Store the push subscription/token securely.

When sending:

```text
User
 |
 v
Active Web Devices
 |
 +---- Browser A
 +---- Browser B
```

---

# 65. Step 44 — In-App Notification

In-app notifications usually require persistence.

```text
notification_inbox

id
user_id
notification_id
title
body
deep_link
read
read_at
created_at
```

API:

```http
GET /api/v1/users/{userId}/notifications
```

```http
PATCH /api/v1/notifications/{id}/read
```

---

# 66. Step 45 — Real-Time Notification

For real-time delivery:

```text
Backend
   |
   v
WebSocket Gateway
   |
   v
Browser
```

Use Redis/Kafka or another shared event mechanism when multiple WebSocket nodes exist.

```text
              Kafka
             /     \
            v       v
       WS Node 1  WS Node 2
            |       |
            v       v
         Users    Users
```

---

# 67. Step 46 — Quiet Hours

Example:

```text
User:
Quiet hours = 10 PM - 7 AM
```

For a non-critical notification:

```text
10:30 PM
   |
   v
Schedule for 7:00 AM
```

But:

```text
OTP
Security alert
Critical incident
```

may use a policy that bypasses quiet hours, subject to the product's rules and user consent.

Do not hardcode this exception. Make it configuration-driven.

---

# 68. Step 47 — Digest Notifications

Instead of:

```text
20 notifications
```

send:

```text
Daily Digest
```

Example:

```text
You have:

5 new messages
3 order updates
2 recommendations
```

Architecture:

```text
Events
  |
  v
Digest Aggregator
  |
  v
Scheduled Job
  |
  v
Notification
```

---

# 69. Step 48 — Notification Deduplication

Suppose the same business event is emitted twice:

```text
PAYMENT_SUCCESS
PAYMENT_SUCCESS
```

Deduplicate using:

```text
eventId
idempotencyKey
businessReference
```

Example:

```text
eventId = EVT-123
```

Store processed event IDs.

---

# 70. Step 49 — Ordering

Some notifications have ordering requirements.

Example:

```text
Payment Initiated
Payment Successful
Payment Refunded
```

Do not deliver:

```text
Refunded
Successful
```

when order matters.

Kafka partitioning can help:

```text
partition key = userId
```

or:

```text
partition key = orderId
```

depending on the ordering boundary.

---

# 71. Step 50 — Backpressure

Suppose:

```text
Incoming = 100,000/sec
Processing = 20,000/sec
```

The queue grows.

Use:

```text
Queue
 |
 +---- autoscaling workers
 +---- rate limiting
 +---- priority queues
 +---- provider throttling
```

Do not allow the database or provider to become overloaded.

---

# 72. Step 51 — Horizontal Scaling

Notification API:

```text
Load Balancer
      |
 +----+----+----+
 |    |    |    |
API1 API2 API3 API4
```

Workers:

```text
Email Worker x 50
SMS Worker x 30
Push Worker x 100
```

Scale independently.

---

# 73. Step 52 — Partitioning

At very large scale:

```text
Kafka
 |
 +---- Partition 0
 +---- Partition 1
 +---- Partition 2
 +---- ...
```

Database:

```text
notification
notification_delivery
```

may eventually require:

```text
partitioning
sharding
archival
```

Partition candidates:

```text
tenant_id
user_id
created_at
```

Choose based on access patterns.

---

# 74. Step 53 — Data Retention

Notification data can become enormous.

Use:

```text
Hot Data
   |
   v
Recent notifications
```

and:

```text
Cold Storage
   |
   v
Old notification history
```

For example:

```text
0-90 days -> primary DB
90-365 days -> archive
>365 days -> delete/archive according to policy
```

Retention should be configurable and subject to legal/privacy requirements.

---

# 75. Step 54 — Disaster Recovery

Plan for:

```text
Database failure
Kafka failure
Redis failure
Provider outage
Region failure
```

Use:

```text
Replication
Backups
Multi-AZ
Multi-region where justified
Provider failover
Replayable events
```

---

# 76. Step 55 — Multi-Region

Large systems can use:

```text
                 Global Traffic
                       |
             +---------+---------+
             |                   |
             v                   v
          Region A             Region B
             |                   |
          Kafka A             Kafka B
             |                   |
          Workers             Workers
             |                   |
        Providers           Providers
```

Routing may depend on:

```text
user region
provider availability
data residency
tenant
latency
```

---

# 77. Step 56 — Failure Scenarios

## Scenario 1: Kafka is temporarily unavailable

Business service:

```text
DB transaction
     |
     v
Outbox
```

Outbox publisher retries later.

---

## Scenario 2: SMS provider is down

```text
Twilio
  X
  |
Provider Selector
  |
  v
AWS SNS
```

Use circuit breaker + failover.

---

## Scenario 3: Email worker crashes

Kafka message remains available for another consumer instance depending on consumer-group semantics.

---

## Scenario 4: Provider returns 429

Apply:

```text
backoff
retry
rate limiting
provider throttling
```

---

## Scenario 5: User has disabled SMS

Preference engine removes SMS before delivery.

---

## Scenario 6: Duplicate event

Idempotency layer prevents duplicate processing.

---

# 78. Step 57 — Security / OTP Example

OTP notifications should be treated differently from marketing.

```text
OTP
 |
 +---- HIGH/CRITICAL priority
 +---- strict rate limit
 +---- short TTL
 +---- no persistent plaintext OTP logging
 +---- idempotency
 +---- provider failover
```

Never log:

```text
OTP=123456
```

Instead:

```text
OTP notification generated
```

---

# 79. Step 58 — Complete Class Diagram

Conceptual model:

```text
                 NotificationOrchestrator
                           |
                           v
                  NotificationRouter
                           |
                           v
                  NotificationChannel
                    /      |       \
                   /       |        \
                Push     Email      SMS
                  |        |         |
                  v        v         v
              PushProvider EmailProvider SmsProvider
                  |        |         |
             +----+     +--+--+    +--+--+
             |          |     |    |     |
            FCM        SES SendGrid Twilio AWS
```

Supporting services:

```text
NotificationOrchestrator
        |
        +---- PreferenceService
        +---- TemplateResolver
        +---- PolicyEngine
        +---- ProviderSelector
        +---- RetryPolicy
        +---- Scheduler
        +---- DeliveryRepository
```

---

# 80. Step 59 — Recommended Design Patterns

## Strategy Pattern

Use for:

```text
Provider selection
Retry strategy
Routing strategy
Template rendering
Rate limiting
```

## Factory / Abstract Factory

Use for:

```text
Provider creation
Channel creation
```

## Chain of Responsibility

Use for:

```text
Notification policies
Validation
Filtering
```

## Observer / Event-Driven

Use for:

```text
Business events
Notification events
```

## State Pattern

Use for:

```text
Notification lifecycle
Delivery lifecycle
```

## Adapter Pattern

Use for:

```text
Twilio SDK
FCM SDK
SES SDK
WhatsApp SDK
```

Each provider adapter converts an external API into your internal abstraction.

---

# 81. Adapter Pattern Example

```java
public interface SmsProvider {

    ProviderResponse send(SmsMessage message);
}
```

Twilio:

```java
public class TwilioSmsProvider
        implements SmsProvider {

    private final TwilioClient client;

    @Override
    public ProviderResponse send(SmsMessage message) {
        // Translate internal message
        // to Twilio API
    }
}
```

Your domain never depends on Twilio's API model.

---

# 82. Step 60 — Configuration-First Design

A mature notification system should have configuration such as:

```yaml
notification:

  channels:
    push:
      enabled: true

    email:
      enabled: true

    sms:
      enabled: true

    whatsapp:
      enabled: false

  retry:
    default:
      maxAttempts: 5
      baseDelay: 1000
      maxDelay: 300000
      jitter: true

  providers:

    sms:
      primary: twilio
      fallback:
        - aws-sns

    email:
      primary: ses
      fallback:
        - sendgrid

  routing:

    ORDER_CONFIRMED:
      channels:
        - PUSH
        - EMAIL

    PAYMENT_FAILED:
      channels:
        - PUSH
        - SMS
        - EMAIL
```

---

# 83. Step 61 — Avoid Configuration Explosion

Do not put every business rule into YAML.

Use three levels:

```text
Static Configuration
        |
        +---- application.yaml

Dynamic Configuration
        |
        +---- Config Service / DB

Secrets
        |
        +---- Secret Manager
```

Examples:

### Static

```text
Kafka broker
Database URL
```

### Dynamic

```text
SMS enabled
Provider priority
Template selection
Rate limits
```

### Secrets

```text
API key
OAuth secret
Provider token
```

---

# 84. Step 62 — Event Schema

Use a versioned event.

```json
{
  "eventId": "EVT123",
  "eventType": "ORDER_CONFIRMED",
  "version": 1,
  "occurredAt": "2026-09-18T10:00:00Z",
  "tenantId": "TENANT1",
  "userId": "U123",
  "data": {
    "orderId": "ORD123",
    "amount": 999
  },
  "metadata": {
    "correlationId": "CORR123"
  }
}
```

Never casually change event contracts.

Use:

```text
versioning
schema validation
backward compatibility
```

---

# 85. Step 63 — API vs Event Consumption

There are two useful entry points.

## Direct API

Useful when:

```text
Another service explicitly requests notification.
```

```http
POST /notifications
```

## Event Driven

Useful when:

```text
Business event occurs.
```

```text
OrderConfirmed
PaymentFailed
UserRegistered
```

A mature system can support both.

---

# 86. Step 64 — Notification Commands vs Events

Keep the distinction clear.

### Command

```text
SendNotification
```

Means:

> Please perform this action.

### Event

```text
OrderConfirmed
```

Means:

> This thing already happened.

Notification service can react to events.

---

# 87. Step 65 — API Separation

Recommended APIs:

```text
POST   /notifications
GET    /notifications/{id}

POST   /notifications/schedule
DELETE /notifications/schedule/{id}

GET    /users/{id}/notifications

PATCH  /users/{id}/notifications/{notificationId}/read

GET    /users/{id}/notification-preferences
PUT    /users/{id}/notification-preferences

GET    /templates
POST   /templates
PUT    /templates/{id}
```

Admin APIs should be separated and protected.

---

# 88. Step 66 — Admin Console

A production notification platform benefits from an admin UI.

Admin should be able to:

```text
Manage templates
Manage providers
Enable/disable channels
Configure routing
Configure retry policy
View delivery status
Replay DLQ messages
View metrics
Configure tenant rules
Configure campaigns
```

Use RBAC:

```text
SUPER_ADMIN
TENANT_ADMIN
OPERATOR
READ_ONLY
```

---

# 89. Step 67 — Audit Trail

Track changes to configuration:

```text
Who changed?
What changed?
When?
Previous value?
New value?
```

Example:

```text
Admin A
changed

SMS provider:
Twilio -> AWS SNS

Timestamp:
2026-09-18T10:00:00Z
```

---

# 90. Step 68 — Testing Strategy

## Unit Tests

Test:

```text
Router
Policy
Template resolver
Retry strategy
Provider selector
```

## Integration Tests

Test:

```text
Kafka
Database
Redis
Provider adapters
```

## Contract Tests

Verify:

```text
Provider adapter
Event schema
API schema
```

## Load Tests

Test:

```text
1,000/sec
10,000/sec
100,000/sec
```

Measure:

```text
latency
throughput
queue lag
CPU
memory
database load
provider limits
```

---

# 91. Step 69 — Chaos Testing

Simulate:

```text
Kafka unavailable
Redis unavailable
Database slow
Provider timeout
Provider 500
Provider 429
Worker crash
Network partition
```

Verify the system still:

```text
does not lose events
does not duplicate excessively
retries correctly
fails over correctly
recovers automatically
```

---

# 92. Step 70 — Scaling Strategy

Start simple.

## Version 1

```text
Spring Boot
PostgreSQL
Kafka
Redis
FCM
Email Provider
SMS Provider
```

Architecture:

```text
              Spring Boot
                   |
             Notification
             Orchestrator
                   |
                 Kafka
              /    |    \
             /     |     \
          Push   Email   SMS
```

---

# 93. Version 2

Add:

```text
Provider failover
Retry
DLQ
Preferences
Templates
Scheduling
Metrics
Tracing
```

---

# 94. Version 3

Add:

```text
Multi-tenancy
Campaigns
Batch processing
Dynamic routing
Smart provider selection
Feature flags
Admin portal
```

---

# 95. Version 4 — Very Large Scale

Add:

```text
Multi-region
Dedicated queues
Advanced autoscaling
Partitioned database
Cold storage
Global provider routing
Advanced rate limiting
Dedicated campaign infrastructure
```

---

# 96. Final Architecture

```text
                              +----------------------+
                              |    Client / Admin    |
                              +----------+-----------+
                                         |
                                         v
                              +----------------------+
                              | API Gateway / Auth   |
                              +----------+-----------+
                                         |
                 +-----------------------+-----------------------+
                 |                                               |
                 v                                               v
       +--------------------+                         +---------------------+
       | Notification API   |                         | Event Consumers     |
       +---------+----------+                         +----------+----------+
                 |                                               |
                 +-----------------------+-----------------------+
                                         |
                                         v
                              +----------------------+
                              | Notification         |
                              | Orchestrator         |
                              +----------+-----------+
                                         |
             +---------------------------+---------------------------+
             |             |             |             |             |
             v             v             v             v             v
       Preferences     Templates     Policy Engine   Scheduler    Deduplication
             |             |             |             |             |
             +-------------+-------------+-------------+-------------+
                                         |
                                         v
                              +----------------------+
                              | Notification Router  |
                              +----------+-----------+
                                         |
                                         v
                              +----------------------+
                              | Message Broker       |
                              | Kafka                |
                              +----------+-----------+
                                         |
          +------------------------------+------------------------------+
          |               |              |              |               |
          v               v              v              v               v
      Push Queue      Email Queue     SMS Queue    WhatsApp Queue   In-App Queue
          |               |              |              |               |
          v               v              v              v               v
       Workers         Workers        Workers        Workers          Workers
          |               |              |              |               |
          v               v              v              v               v
       FCM/APNs       SES/SMTP       Twilio/SNS     WhatsApp API      DB/WS
          |               |              |              |               |
          +---------------+--------------+--------------+---------------+
                                         |
                                         v
                              +----------------------+
                              | Delivery Tracking    |
                              +----------+-----------+
                                         |
                   +---------------------+---------------------+
                   |                     |                     |
                   v                     v                     v
                Metrics              Audit Log              Analytics
```

---

# 97. Key Design Principles

The most important principles are:

```text
1. Business services should not know providers.

2. Channels should not know business rules.

3. Providers should be replaceable.

4. Configuration should drive behavior.

5. External APIs should be behind adapters.

6. Notifications should be asynchronous by default.

7. Use queues to absorb traffic spikes.

8. Use idempotency because duplicate delivery can happen.

9. Use retry + backoff + DLQ.

10. Separate transactional and campaign traffic.

11. Keep workers stateless.

12. Scale channels independently.

13. Treat provider failure as expected.

14. Keep secrets outside normal configuration.

15. Version event schemas and APIs.

16. Build observability from the beginning.

17. Prefer composition over giant services.

18. Follow SOLID boundaries.

19. Keep the domain independent of infrastructure.

20. Design extension points before adding complexity.
```

---

# 98. Enhancement Roadmap

Once the basic system works, enhancements can be added incrementally.

## Enhancement 1 — WhatsApp

```text
WhatsAppChannel
        |
        v
WhatsAppProvider
```

No core orchestration rewrite should be necessary.

---

## Enhancement 2 — Telegram / Slack / Teams

Add:

```text
TelegramChannel
SlackChannel
TeamsChannel
```

---

## Enhancement 3 — Smart Routing

Introduce:

```text
ProviderSelectionStrategy
```

with:

```text
HealthBased
CostBased
LatencyBased
Weighted
```

---

## Enhancement 4 — AI-Assisted Personalization

Possible future capability:

```text
User Profile
     |
     v
Personalization Engine
     |
     v
Template Variables
```

Examples:

```text
preferred language
preferred channel
preferred send time
content personalization
```

Keep this as a separate service so the core notification pipeline does not depend on AI.

---

## Enhancement 5 — Notification Digest

Aggregate low-priority notifications.

```text
20 events
   |
   v
Digest Aggregator
   |
   v
1 notification
```

---

## Enhancement 6 — Notification Frequency Optimization

Track:

```text
sent
opened
clicked
ignored
unsubscribed
```

Then provide configurable policies for frequency management.

---

## Enhancement 7 — Provider Cost Optimization

Track:

```text
provider cost
delivery success
latency
```

Then select providers according to tenant/business policy.

---

## Enhancement 8 — Multi-Region

Move from:

```text
Single Region
```

to:

```text
Global
 |
 +---- India
 +---- Europe
 +---- US
```

---

# 99. Interview Follow-Up Questions

After completing the basic design, an interviewer can progressively increase the difficulty.

### Level 1 — Basic

1. How would you send an email?
2. How would you add SMS?
3. How would you add Push?
4. Why use an interface for channels?

### Level 2 — SOLID

5. How would you add a new provider?
6. How would you avoid `if/else` provider selection?
7. Where would you use Strategy Pattern?
8. Where would you use Adapter Pattern?

### Level 3 — Reliability

9. What happens if the provider is down?
10. How would you retry?
11. What happens after retry exhaustion?
12. How do you prevent duplicate notifications?
13. How do you handle provider rate limits?

### Level 4 — Scale

14. How do you handle 100,000 notifications/sec?
15. How do you scale workers?
16. How do you isolate SMS traffic from marketing traffic?
17. How would you partition Kafka?
18. How would you partition the database?

### Level 5 — Advanced

19. How would you implement multi-region?
20. How would you implement provider failover?
21. How would you implement smart routing?
22. How would you implement campaign notifications?
23. How would you support scheduled notifications?
24. How would you guarantee event durability?
25. How would you implement transactional outbox?

### Level 6 — Production

26. How do you monitor delivery latency?
27. How do you debug a missing notification?
28. How do you replay DLQ messages?
29. How do you manage provider secrets?
30. How do you safely change notification configuration in production?

---

# 100. Final Mental Model

Remember the system as six layers:

```text
1. API / Events
       |
2. Orchestration
       |
3. Policy + Preference + Template
       |
4. Routing
       |
5. Queue + Workers
       |
6. Channel + Provider
```

The most important dependency direction is:

```text
Business
   |
   v
Notification Domain
   |
   v
Abstractions
   |
   v
Adapters
   |
   v
External Providers
```

NOT:

```text
Business
   |
   v
Twilio / FCM / SES
```

The final system should make the following change easy:

```text
Today:
SMS -> Twilio

Tomorrow:
SMS -> Twilio + AWS + Vonage
```

or:

```text
Today:
Push + Email

Tomorrow:
Push + Email + SMS + WhatsApp
```

without modifying the business services or rewriting the notification orchestration layer.

---

# 101. Suggested Implementation Stack

For a Java/Spring implementation:

```text
Language:
Java 21

Framework:
Spring Boot

API:
Spring Web

Messaging:
Kafka

Database:
PostgreSQL / MySQL

Cache:
Redis

Scheduling:
Quartz / Kafka-based scheduler / Redis Sorted Set

Resilience:
Resilience4j

Observability:
OpenTelemetry
Prometheus
Grafana

Logs:
ELK / OpenSearch

Secrets:
Vault / AWS Secrets Manager

Container:
Docker

Orchestration:
Kubernetes
```

The exact technology can change; the architectural abstractions should remain stable.

---

# 102. Recommended Development Sequence

Implement in this order:

```text
STEP 1
Define Notification domain model

STEP 2
Define NotificationChannel interface

STEP 3
Implement EmailChannel

STEP 4
Implement SmsChannel

STEP 5
Implement PushChannel

STEP 6
Define Provider abstractions

STEP 7
Implement provider adapters

STEP 8
Build NotificationOrchestrator

STEP 9
Add TemplateResolver

STEP 10
Add PreferenceService

STEP 11
Add PolicyEngine

STEP 12
Add Kafka

STEP 13
Create channel-specific workers

STEP 14
Add retry + exponential backoff

STEP 15
Add DLQ

STEP 16
Add idempotency

STEP 17
Add delivery tracking

STEP 18
Add provider failover

STEP 19
Add rate limiting

STEP 20
Add scheduling

STEP 21
Add observability

STEP 22
Add admin configuration

STEP 23
Add multi-tenancy

STEP 24
Add campaigns

STEP 25
Add multi-region
```

---

# 103. What Makes This Design Extensible?

The key is not simply using many interfaces.

The important separation is:

```text
WHAT should be sent?
        |
        v
Notification Domain

WHEN should it be sent?
        |
        v
Scheduler / Policy

WHICH channel?
        |
        v
Router

WHICH provider?
        |
        v
Provider Selector

HOW to call provider?
        |
        v
Provider Adapter
```

Each question has a separate responsibility.

That is the core design principle behind a scalable notification platform.

---

# 104. Final Interview Answer Structure

During a system-design interview, present the design in this sequence:

```text
1. Requirements
2. Scale assumptions
3. API
4. High-level architecture
5. Notification domain
6. Channel abstraction
7. Provider abstraction
8. Configuration
9. Preferences
10. Templates
11. Event-driven architecture
12. Kafka / queue
13. Workers
14. Retry
15. DLQ
16. Idempotency
17. Provider failover
18. Rate limiting
19. Scheduling
20. Database
21. Observability
22. Security
23. SOLID / Design Patterns
24. Scaling
25. Failure scenarios
26. Future enhancements
```

This gives a clean progression from:

```text
Simple Design
      |
      v
Extensible Design
      |
      v
Reliable Design
      |
      v
Scalable Production System
```

---

# End

## Core Principle

> **Build the notification system around stable domain abstractions and configuration-driven policies, while isolating channels, providers, infrastructure, and business logic from one another.**

That allows the platform to evolve from a simple:

```text
Email + SMS + Push
```

into a large-scale:

```text
Multi-tenant
Multi-channel
Multi-provider
Multi-region
Event-driven
Highly available
Configurable
Observable
Fault-tolerant
Notification Platform
```

without requiring a fundamental redesign.
