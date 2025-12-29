# MERN-Zombilet

> Production-Ready Event-Driven Microservices Ticketing Platform

[![GitHub](https://img.shields.io/badge/GitHub-chatonode%2Fmern--zombilet-blue)](https://github.com/chatonode/mern-zombilet)

## 📖 Table of Contents

- [🎯 Overview](#🎯-overview)
- [🏗️ Architecture](#️🏗️-architecture)
- [🎯 Microservices](#🎯-microservices)
- [🔄 Event-Driven Communication](#🔄-event-driven-communication)
- [🔒 Optimistic Concurrency Control (OCC)](#🔒-optimistic-concurrency-control-occ)
- [🛠️ Tech Stack](#️🛠️-tech-stack)
- [📦 Shared Common Library](#📦-shared-common-library)
- [☸️ Kubernetes Infrastructure](#️☸️-kubernetes-infrastructure)
- [🔧 Development Workflow](#🔧-development-workflow)
- [🚀 CI/CD Pipeline](#🚀-cicd-pipeline)
- [🧪 Testing Strategy](#🧪-testing-strategy)
- [🔐 Authentication & Security](#🔐-authentication--security)
- [📊 Data Models & Relationships](#📊-data-models--relationships)
- [⏰ Async Processing & Job Queues](#⏰-async-processing--job-queues)
- [💻 Client Application](#💻-client-application)
- [🚀 Getting Started](#🚀-getting-started)
- [📡 API Reference](#📡-api-reference)
- [🌐 Production Deployment](#🌐-production-deployment)
- [📐 Architecture Diagrams](#📐-architecture-diagrams)
- [📁 Project Structure](#📁-project-structure)
- [🎯 Design Decisions](#🎯-design-decisions)

---

## 🎯 Overview

**MERN-Zombilet** is a full-stack, production-grade ticket buying and selling marketplace built with a **microservices architecture**. The platform was successfully deployed and operated in production on **Digital Ocean Kubernetes**, demonstrating real-world scalability and reliability.

### Key Highlights

- 🎫 **Ticket Marketplace**: Users can list, purchase, and manage event tickets
- 🔄 **Event-Driven Architecture**: 6 independent microservices communicating via NATS Streaming
- 📊 **Database Per Service**: Each microservice owns its dedicated MongoDB instance
- 🔐 **JWT Authentication**: Secure authentication with shared secrets across services
- 💳 **Stripe Payments**: Integrated payment processing
- 📧 **Email Notifications**: SendGrid integration for transactional emails
- ⏰ **Async Job Processing**: Redis + Bull.js for time-based workflows
- 🚀 **Production Ready**: Deployed on Digital Ocean with full CI/CD pipeline

### Problem Domain

The ticketing platform addresses the challenge of building a distributed system where:

- Multiple services need to maintain their own data stores
- Services must communicate asynchronously without tight coupling
- Data consistency must be maintained without distributed transactions
- The system must handle concurrent operations (e.g., multiple users trying to purchase the same ticket)
- Order reservations expire after a defined time window

---

## 🏗️ Architecture

MERN-Zombilet follows a **microservices architecture** with the following core principles:

### 1. **Event-Driven Communication**

Services communicate asynchronously through **NATS Streaming Server** as the event bus. Each service publishes events when data changes and subscribes to relevant events from other services.

### 2. **Database Per Service Pattern**

Each microservice maintains its own MongoDB database, ensuring:

- **Service autonomy**: Services can be developed, deployed, and scaled independently
- **Technology flexibility**: Each service can choose its optimal data storage solution
- **Failure isolation**: Database issues in one service don't affect others

### 3. **Data Replication via Events**

Since services don't share databases, data is replicated across services through published events:

- The **Tickets** service creates a ticket → publishes `TicketCreated` event
- The **Orders** service listens to `TicketCreated` → replicates ticket data locally
- This enables the Orders service to create orders without directly querying the Tickets database

### 4. **Eventual Consistency**

The system embraces eventual consistency:

- Changes propagate through events asynchronously
- Services may have slightly stale data for brief periods
- **Optimistic Concurrency Control** (OCC) prevents conflicting updates

### 5. **Service Independence**

Each microservice:

- Has its own codebase, dependencies, and Dockerfile
- Runs in dedicated Kubernetes pods with dedicated MongoDB instances
- Can be tested, built, and deployed independently
- Scales independently based on load

### 6. **Production Deployment**

The entire system was deployed to **Digital Ocean Kubernetes** with:

- Ingress NGINX for load balancing and routing
- Kubernetes Secrets for sensitive configuration
- Rolling updates for zero-downtime deployments
- Horizontal pod autoscaling capabilities

---

## 🎯 Microservices

The platform consists of **6 independent microservices**, each with dedicated responsibilities:

### 1. **Auth Service** (`/auth`)

**Responsibility**: User authentication and authorization

**Key Features**:

- Email/password registration and login
- JWT token generation and validation
- Password reset flow with time-limited tokens
- Token validation middleware
- User session management via cookie-session

**Routes**:

- `POST /api/users/signup` - Register new user
- `POST /api/users/signin` - Authenticate user
- `POST /api/users/signout` - Clear user session
- `GET /api/users/currentuser` - Get current authenticated user
- `POST /api/users/forgot-password` - Request password reset
- `POST /api/users/validate-token` - Validate reset token
- `POST /api/users/reset-password` - Reset password with token

**Events Published**:

- `UserSignedUp` - New user registration
- `UserSignedIn` - User login
- `UserChangedPassword` - Password updated
- `UserForgotPassword` - Password reset requested

**Database**: MongoDB (`auth` database)

- Collections: `users`, `tokens`

---

### 2. **Tickets Service** (`/tickets`)

**Responsibility**: Ticket listing and management

**Key Features**:

- Create, read, update ticket listings
- Track ticket reservation status via `orderId` field
- Prevent editing of reserved tickets
- Optimistic concurrency control on ticket updates

**Routes**:

- `POST /api/tickets` - Create new ticket (requires auth)
- `GET /api/tickets` - List all available tickets
- `GET /api/tickets/:id` - Get ticket details
- `PUT /api/tickets/:id` - Update ticket (requires auth, only if not reserved)

**Events Published**:

- `TicketCreated` - New ticket listed
- `TicketUpdated` - Ticket modified (price/title change or reservation status)

**Events Consumed**:

- `OrderCreated` - Marks ticket as reserved by setting `orderId`
- `OrderCancelled` - Marks ticket as available by clearing `orderId`

**Database**: MongoDB (`tickets` database)

- Collections: `tickets`

**Business Logic**:

- Users can only update their own tickets
- Reserved tickets (with `orderId`) cannot be edited
- Ticket updates emit `TicketUpdated` event with version number

---

### 3. **Orders Service** (`/orders`)

**Responsibility**: Order creation and management

**Key Features**:

- Create orders with 1-minute expiration window
- Reserve tickets temporarily
- Handle order cancellation (manual or via expiration)
- Track order lifecycle (Created → AwaitingPayment → Completed/Cancelled)
- Maintain local replica of ticket data

**Routes**:

- `POST /api/orders` - Create new order (requires auth)
- `GET /api/orders` - List user's orders (requires auth)
- `GET /api/orders/:id` - Get order details (requires auth)
- `DELETE /api/orders/:id` - Cancel order (requires auth)

**Events Published**:

- `OrderCreated` - New order placed (1-minute reservation)
- `OrderCancelled` - Order cancelled by user or expiration

**Events Consumed**:

- `TicketCreated` - Replicates ticket data locally
- `TicketUpdated` - Updates local ticket replica
- `ExpirationCompleted` - Cancels expired unpaid orders
- `PaymentCreated` - Marks order as complete

**Database**: MongoDB (`orders` database)

- Collections: `orders`, `tickets` (replicated data)

**Business Logic**:

- Validates ticket exists and is not reserved
- Sets 1-minute expiration (`EXPIRATION_WINDOW_SECONDS = 60`)
- Creates order in `Created` status
- Enqueues expiration job in Redis queue
- Users can only view/cancel their own orders

---

### 4. **Payments Service** (`/payments`)

**Responsibility**: Payment processing via Stripe

**Key Features**:

- Process payments for orders
- Stripe integration for credit card charges
- Validate order ownership and status
- Prevent duplicate payments

**Routes**:

- `POST /api/payments` - Process payment (requires auth)

**Events Published**:

- `PaymentCreated` - Successful payment processed

**Events Consumed**:

- `OrderCreated` - Tracks orders available for payment
- `OrderCancelled` - Prevents payment on cancelled orders

**Database**: MongoDB (`payments` database)

- Collections: `payments`, `orders` (replicated data)

**Business Logic**:

- Validates order belongs to user
- Ensures order is in `Created` or `AwaitingPayment` status
- Prevents payment on cancelled/completed orders
- Stores Stripe charge ID

**External Integration**: Stripe API (v2020-08-27)

---

### 5. **Expiration Service** (`/expiration`)

**Responsibility**: Time-based order expiration

**Key Features**:

- Background job processing with Redis + Bull.js
- Monitors order expiration times
- Publishes expiration events after timeout

**Routes**: None (event-driven only)

**Events Published**:

- `ExpirationCompleted` - Order expiration timeout reached

**Events Consumed**:

- `OrderCreated` - Enqueues expiration job with delay

**Database**: None (uses Redis for job queue persistence)

**Technical Implementation**:

- Redis-backed Bull.js queue named `order-expiration`
- Jobs scheduled with delay based on order `expiresAt` timestamp
- Job processor publishes `ExpirationCompleted` event
- Graceful shutdown handling for in-flight jobs

---

### 6. **Contact Service** (`/contact`)

**Responsibility**: Email notifications via SendGrid

**Key Features**:

- Transactional email delivery
- Event-driven notification triggers
- User communication for all platform activities

**Routes**: None (event-driven only)

**Events Published**: None

**Events Consumed**:

- `UserSignedUp` - Welcome email
- `UserSignedIn` - Login notification
- `UserForgotPassword` - Password reset email with token
- `UserChangedPassword` - Password change confirmation
- `TicketCreated` - Ticket listing confirmation
- `TicketUpdated` - Ticket modification notification
- `OrderCreated` - Order confirmation
- `OrderCancelled` - Cancellation notification
- `OrderCompleted` - Purchase confirmation

**Database**: MongoDB (`contact` database)

- Collections: Stores email delivery logs

**External Integration**: SendGrid API

**Email Templates**:

- Custom HTML templates per event type
- Dynamic content injection (user names, ticket details, URLs)
- Sender address configured via `SENDGRID_EMAIL` environment variable

---

## 🔄 Event-Driven Communication

The platform uses **NATS Streaming Server** as the event bus for asynchronous, decoupled communication between microservices.

### NATS Streaming Configuration

**Deployment Details**:

- Image: `nats-streaming:0.17.0`
- Cluster ID: `zombilet`
- Client Port: `4222`
- Monitoring Port: `8222`
- Heartbeat configuration: 5s intervals, 2 failed attempts triggers disconnect

**Connection Pattern**:

```typescript
// Each service connects on startup
await natsWrapper.connect(
  process.env.NATS_CLUSTER_ID, // 'zombilet'
  process.env.NATS_CLIENT_ID, // Unique pod name
  { url: process.env.NATS_URL } // 'http://nats-streaming-srv:4222'
)
```

### Publisher/Listener Pattern

All event handling is abstracted through the **@chato-zombilet/common** library:

#### **Publishers**

Publishers extend the base `Publisher<T>` class:

```typescript
// Example: TicketCreatedPublisher
import { Publisher, Subjects, TicketCreatedEvent } from '@chato-zombilet/common'

export class TicketCreatedPublisher extends Publisher<TicketCreatedEvent> {
  public readonly subject = Subjects.TicketCreated
}

// Usage in service
await new TicketCreatedPublisher(natsWrapper.client).publish({
  id: ticket.id,
  version: ticket.version,
  title: ticket.title,
  price: ticket.price,
  userId: ticket.userId,
})
```

#### **Listeners**

Listeners extend the base `Listener<T>` class and implement `onMessage()`:

```typescript
// Example: TicketCreatedListener
import { Listener, Subjects, TicketCreatedEvent } from '@chato-zombilet/common'

export class TicketCreatedListener extends Listener<TicketCreatedEvent> {
  readonly subject = Subjects.TicketCreated
  queueGroupName = 'orders-service' // Load balancing

  async onMessage(
    data: TicketCreatedEvent['data'],
    msg: Message
  ): Promise<void> {
    // Replicate ticket data locally
    const ticket = Ticket.build({
      id: data.id,
      title: data.title,
      price: data.price,
    })
    await ticket.save()

    // Acknowledge message
    msg.ack()
  }
}

// Service startup
new TicketCreatedListener(natsWrapper.client).listen()
```

### Queue Groups

Services use **queue groups** to ensure:

- **Load balancing**: Only one instance of a service processes each event
- **Horizontal scaling**: Multiple pods of the same service share the workload
- **Fault tolerance**: If one pod fails, others continue processing

Each service defines its queue group:

```typescript
// orders/src/events/listeners/queue-group-name.ts
export const queueGroupName = 'orders-service'
```

### Event Subjects

All events are strongly typed through the `Subjects` enum:

| Subject               | Publisher  | Listeners                              |
| --------------------- | ---------- | -------------------------------------- |
| `TicketCreated`       | Tickets    | Orders, Contact                        |
| `TicketUpdated`       | Tickets    | Orders, Contact                        |
| `OrderCreated`        | Orders     | Tickets, Payments, Expiration, Contact |
| `OrderCancelled`      | Orders     | Tickets, Payments, Contact             |
| `ExpirationCompleted` | Expiration | Orders                                 |
| `PaymentCreated`      | Payments   | Orders, Contact                        |
| `UserSignedUp`        | Auth       | Contact                                |
| `UserSignedIn`        | Auth       | Contact                                |
| `UserForgotPassword`  | Auth       | Contact                                |
| `UserChangedPassword` | Auth       | Contact                                |

### Event Data Structures

Events carry versioned, strongly-typed data:

```typescript
// TicketCreatedEvent interface (from common library)
export interface TicketCreatedEvent {
  subject: Subjects.TicketCreated
  data: {
    id: string
    version: number // OCC version
    title: string
    price: number
    userId: string
  }
}

// OrderCreatedEvent with nested ticket data
export interface OrderCreatedEvent {
  subject: Subjects.OrderCreated
  data: {
    id: string
    version: number
    status: OrderStatus
    userId: string
    expiresAt: string // ISO timestamp
    ticket: {
      id: string
      price: number
    }
  }
}
```

### Graceful Shutdown

Each service implements graceful NATS shutdown:

```typescript
// Close NATS connection on process termination
natsWrapper.client.on('close', () => {
  console.log('Disconnected from STAN!')
  process.exit()
})

process.on('SIGINT', () => natsWrapper.client.close())
process.on('SIGTERM', () => natsWrapper.client.close())
```

This ensures:

- In-flight messages are acknowledged
- Queue group memberships are cleaned up
- No message loss during pod restarts

### Event Flow Example: Ticket Purchase

```
1. User creates order
   → Orders Service: POST /api/orders

2. Orders Service
   → Validates ticket availability
   → Creates order document
   → Publishes: OrderCreated event

3. Tickets Service listens to OrderCreated
   → Marks ticket as reserved (sets orderId)
   → Publishes: TicketUpdated event

4. Expiration Service listens to OrderCreated
   → Enqueues job in Redis (1-minute delay)

5. Payments Service listens to OrderCreated
   → Replicates order data locally

6. Contact Service listens to OrderCreated
   → Sends order confirmation email

7. After 1 minute (if no payment)
   → Expiration Service publishes: ExpirationCompleted

8. Orders Service listens to ExpirationCompleted
   → Cancels order
   → Publishes: OrderCancelled event

9. Tickets Service listens to OrderCancelled
   → Marks ticket as available (clears orderId)
   → Publishes: TicketUpdated event
```

### Benefits of This Pattern

✅ **Loose Coupling**: Services don't know about each other directly  
✅ **Scalability**: Services scale independently based on event load  
✅ **Resilience**: Service failures don't cascade  
✅ **Flexibility**: New services can subscribe to existing events  
✅ **Auditability**: All state changes are tracked via events

---

## 🔒 Optimistic Concurrency Control (OCC)

The platform implements **Optimistic Concurrency Control** to prevent race conditions and conflicting updates in the distributed system.

### The Problem

Without OCC, consider this scenario:

```
Time  | Tickets Service          | Orders Service (Listener)
------|--------------------------|---------------------------
T1    | Ticket: {price: $10}     |
T2    | UPDATE price = $20       |
T3    | Publish: TicketUpdated   |
T4    | UPDATE price = $30       |
T5    | Publish: TicketUpdated   |
T6    |                          | Receives: price = $30 (!)
T7    |                          | Receives: price = $20 (out of order!)
T8    |                          | Local ticket: {price: $20} ❌ WRONG!
```

Events can arrive out of order due to network delays, causing data inconsistency.

### The Solution: Version Numbers

Every entity includes a `version` field that increments on each update:

```typescript
// Ticket Model with versioning
import { updateIfCurrentPlugin } from 'mongoose-update-if-current'

const ticketSchema = new mongoose.Schema(
  {
    title: String,
    price: Number,
    userId: String,
    orderId: String,
  },
  {
    toJSON: {
      transform(doc, ret) {
        ret.id = ret._id
        delete ret._id
      },
    },
  }
)

// Enable versioning
ticketSchema.set('versionKey', 'version')
ticketSchema.plugin(updateIfCurrentPlugin)

const Ticket = mongoose.model('Ticket', ticketSchema)
```

### How It Works

#### 1. **Version Increments on Save**

```typescript
const ticket = await Ticket.findById(id)
ticket.set({ price: 20 })
await ticket.save() // version: 0 → 1

ticket.set({ price: 30 })
await ticket.save() // version: 1 → 2
```

#### 2. **Events Include Version Numbers**

```typescript
// Publishing TicketUpdated event
await new TicketUpdatedPublisher(natsWrapper.client).publish({
  id: ticket.id,
  version: ticket.version, // Critical!
  title: ticket.title,
  price: ticket.price,
  userId: ticket.userId,
  orderId: ticket.orderId,
})
```

#### 3. **Listeners Check Version Order**

```typescript
// TicketUpdatedListener in Orders Service
async onMessage(data: TicketUpdatedEvent['data'], msg: Message) {
    const { id, version, title, price } = data

    // Find ticket with PREVIOUS version
    const ticket = await Ticket.findPreviousVersion({
        id,
        version: version - 1  // Expects previous version
    })

    if (!ticket) {
        // Out-of-order event detected - DO NOT ACK
        throw new Error('Ticket not found or version mismatch')
    }

    // Update ticket (version will increment automatically)
    ticket.set({ title, price })
    await ticket.save()  // version: N → N+1

    // Acknowledge message
    msg.ack()
}
```

#### 4. **Finding Previous Version**

```typescript
// Custom static method on Ticket model
ticketSchema.statics.findPreviousVersion = (params: {
  id: string
  version: number
}) => {
  return Ticket.findOne({
    _id: params.id,
    version: params.version, // Must match expected previous version
  })
}
```

### Complete OCC Flow

```
1. Tickets Service: Ticket created with version 0
   → Publishes: {id: '123', version: 0, price: 10}

2. Orders Service receives event (version 0)
   → Expects: NO previous version (new ticket)
   → Creates local ticket with version 0 ✓

3. Tickets Service: Price updated
   → Ticket version: 0 → 1
   → Publishes: {id: '123', version: 1, price: 20}

4. Tickets Service: Price updated again
   → Ticket version: 1 → 2
   → Publishes: {id: '123', version: 2, price: 30}

5. Orders Service receives event (version 2) FIRST (out of order!)
   → Finds ticket with version 1: NOT FOUND ❌
   → Does NOT acknowledge message
   → NATS will redeliver later

6. Orders Service receives event (version 1)
   → Finds ticket with version 0: FOUND ✓
   → Updates to version 1
   → Acknowledges message

7. NATS redelivers version 2 event
   → Finds ticket with version 1: FOUND ✓
   → Updates to version 2
   → Acknowledges message
   → Data now consistent! ✅
```

### Preventing Concurrent Updates

The `mongoose-update-if-current` plugin also prevents concurrent updates in the same service:

```typescript
// Two concurrent requests try to update the same ticket

// Request A
const ticketA = await Ticket.findById('123') // version: 5
ticketA.set({ price: 20 })

// Request B
const ticketB = await Ticket.findById('123') // version: 5
ticketB.set({ price: 25 })

await ticketA.save() // ✓ Success! version: 5 → 6

await ticketB.save() // ❌ FAILS! Version mismatch
// Error: Document version (5) doesn't match current version (6)
```

### Models Using OCC

All core entities use versioning:

- ✅ **User** (auth service)
- ✅ **Ticket** (tickets service + replicated in orders)
- ✅ **Order** (orders service + replicated in payments)
- ✅ **Token** (auth service)

### Benefits of OCC

✅ **No Distributed Locks**: Avoids complex coordination between services  
✅ **Data Consistency**: Prevents out-of-order event processing  
✅ **Race Condition Prevention**: Detects and prevents conflicting updates  
✅ **Eventual Consistency**: Guarantees data converges to correct state  
✅ **Performance**: Optimistic approach doesn't block operations  
✅ **Simplicity**: Mongoose plugin handles version management automatically

### Trade-offs

⚠️ **Retry Logic**: Out-of-order events require redelivery (handled by NATS)  
⚠️ **Event Ordering**: Only works when events include version numbers  
⚠️ **Failed Updates**: Concurrent updates fail fast, requiring retry from client

---

## 🛠️ Tech Stack

### Backend Services

| Technology                     | Version | Purpose                             |
| ------------------------------ | ------- | ----------------------------------- |
| **TypeScript**                 | ^4.7.2  | Type-safe service development       |
| **Node.js**                    | 16.13   | Runtime environment                 |
| **Express**                    | ^4.17.2 | Web framework for REST APIs         |
| **MongoDB**                    | ^6.1.5  | Document database (one per service) |
| **Mongoose**                   | ^6.1.5  | MongoDB ODM with schema validation  |
| **mongoose-update-if-current** | ^1.4.0  | OCC versioning plugin               |

### Event Bus & Messaging

| Technology              | Version | Purpose                                   |
| ----------------------- | ------- | ----------------------------------------- |
| **NATS Streaming**      | 0.17.0  | Event bus for inter-service communication |
| **node-nats-streaming** | ^0.3.2  | NATS client library                       |

### Job Queue & Caching

| Technology | Version | Purpose               |
| ---------- | ------- | --------------------- |
| **Redis**  | Latest  | In-memory data store  |
| **Bull**   | ^4.8.1  | Redis-based job queue |

### Authentication & Security

| Technology            | Version | Purpose                         |
| --------------------- | ------- | ------------------------------- |
| **jsonwebtoken**      | ^8.5.1  | JWT generation and verification |
| **cookie-session**    | ^2.0.0  | Cookie-based session management |
| **express-validator** | ^6.14.0 | Request validation              |

### External Integrations

| Technology         | Version  | Purpose            |
| ------------------ | -------- | ------------------ |
| **Stripe**         | ^8.214.0 | Payment processing |
| **@sendgrid/mail** | ^7.7.0   | Email delivery     |

### Frontend

| Technology                | Version | Purpose                  |
| ------------------------- | ------- | ------------------------ |
| **Next.js**               | ^12.0.8 | React framework with SSR |
| **React**                 | ^17.0.2 | UI library               |
| **React DOM**             | ^17.0.2 | React rendering          |
| **Bootstrap**             | ^5.1.3  | CSS framework            |
| **Axios**                 | ^0.25.0 | HTTP client              |
| **react-stripe-checkout** | ^2.6.3  | Stripe payment UI        |

### Testing

| Technology                | Version | Purpose                          |
| ------------------------- | ------- | -------------------------------- |
| **Jest**                  | ^27.4.7 | Testing framework                |
| **ts-jest**               | ^27.1.3 | TypeScript preprocessor for Jest |
| **Supertest**             | ^6.2.2  | HTTP assertion library           |
| **mongodb-memory-server** | ^8.2.0  | In-memory MongoDB for tests      |

### DevOps & Infrastructure

| Technology         | Version  | Purpose                       |
| ------------------ | -------- | ----------------------------- |
| **Docker**         | Latest   | Containerization              |
| **Kubernetes**     | Latest   | Container orchestration       |
| **Skaffold**       | v2alpha3 | Local Kubernetes development  |
| **Ingress NGINX**  | Latest   | Load balancing and routing    |
| **GitHub Actions** | N/A      | CI/CD pipelines               |
| **Digital Ocean**  | N/A      | Production Kubernetes hosting |

### Shared Library

| Technology                 | Version | Purpose                          |
| -------------------------- | ------- | -------------------------------- |
| **@chato-zombilet/common** | ^1.0.48 | Shared types, errors, middleware |

### Development Tools

| Technology      | Version | Purpose                              |
| --------------- | ------- | ------------------------------------ |
| **ts-node-dev** | ^2.0.0  | TypeScript execution with hot-reload |
| **TypeScript**  | ^4.7.2  | Static type checking                 |

---

## 📦 Shared Common Library

The **@chato-zombilet/common** NPM package provides shared code across all microservices, ensuring consistency and reducing duplication.

### Installation

```bash
npm install @chato-zombilet/common
```

### Package Contents

#### 1. **Base Publisher Class**

Abstract class for publishing events to NATS:

```typescript
import { Stan } from 'node-nats-streaming'

export abstract class Publisher<T extends Event> {
  abstract subject: T['subject']
  private client: Stan

  constructor(client: Stan) {
    this.client = client
  }

  publish(data: T['data']): Promise<void> {
    return new Promise((resolve, reject) => {
      this.client.publish(this.subject, JSON.stringify(data), (err) => {
        if (err) return reject(err)
        console.log('Event published:', this.subject)
        resolve()
      })
    })
  }
}
```

#### 2. **Base Listener Class**

Abstract class for consuming events from NATS:

```typescript
import { Message, Stan } from 'node-nats-streaming'

export abstract class Listener<T extends Event> {
  abstract subject: T['subject']
  abstract queueGroupName: string
  abstract onMessage(data: T['data'], msg: Message): void

  private client: Stan
  protected ackWait = 5 * 1000 // 5 seconds

  constructor(client: Stan) {
    this.client = client
  }

  listen() {
    const subscription = this.client.subscribe(
      this.subject,
      this.queueGroupName,
      this.subscriptionOptions()
    )

    subscription.on('message', (msg: Message) => {
      console.log(`Message received: ${this.subject}`)

      const parsedData = this.parseMessage(msg)
      this.onMessage(parsedData, msg)
    })
  }

  parseMessage(msg: Message) {
    const data = msg.getData()
    return typeof data === 'string'
      ? JSON.parse(data)
      : JSON.parse(data.toString('utf8'))
  }

  subscriptionOptions() {
    return this.client
      .subscriptionOptions()
      .setDeliverAllAvailable()
      .setManualAckMode(true)
      .setAckWait(this.ackWait)
      .setDurableName(this.queueGroupName)
  }
}
```

#### 3. **Event Interfaces**

Strongly-typed event definitions:

```typescript
// Event Subjects Enum
export enum Subjects {
  TicketCreated = 'ticket:created',
  TicketUpdated = 'ticket:updated',
  OrderCreated = 'order:created',
  OrderCancelled = 'order:cancelled',
  ExpirationCompleted = 'expiration:completed',
  PaymentCreated = 'payment:created',
  UserSignedUp = 'user:signed-up',
  UserSignedIn = 'user:signed-in',
  UserForgotPassword = 'user:forgot-password',
  UserChangedPassword = 'user:changed-password',
}

// Event Type Definitions
export interface TicketCreatedEvent {
  subject: Subjects.TicketCreated
  data: {
    id: string
    version: number
    title: string
    price: number
    userId: string
  }
}

export interface OrderCreatedEvent {
  subject: Subjects.OrderCreated
  data: {
    id: string
    version: number
    status: OrderStatus
    userId: string
    expiresAt: string
    ticket: {
      id: string
      price: number
    }
  }
}

// ... more event interfaces
```

#### 4. **Custom Error Classes**

Standardized error handling:

```typescript
export abstract class CustomError extends Error {
  abstract statusCode: number

  constructor(message: string) {
    super(message)
    Object.setPrototypeOf(this, CustomError.prototype)
  }

  abstract serializeErrors(): { message: string; field?: string }[]
}

// NotFoundError (404)
export class NotFoundError extends CustomError {
  statusCode = 404

  constructor() {
    super('Route not found')
    Object.setPrototypeOf(this, NotFoundError.prototype)
  }

  serializeErrors() {
    return [{ message: 'Not Found' }]
  }
}

// BadRequestError (400)
export class BadRequestError extends CustomError {
  statusCode = 400

  constructor(public message: string) {
    super(message)
    Object.setPrototypeOf(this, BadRequestError.prototype)
  }

  serializeErrors() {
    return [{ message: this.message }]
  }
}

// NotAuthorizedError (401)
export class NotAuthorizedError extends CustomError {
  statusCode = 401

  constructor() {
    super('Not authorized')
    Object.setPrototypeOf(this, NotAuthorizedError.prototype)
  }

  serializeErrors() {
    return [{ message: 'Not authorized' }]
  }
}

// TokenExpiredError (401)
export class TokenExpiredError extends CustomError {
  statusCode = 401

  constructor() {
    super('Token expired')
    Object.setPrototypeOf(this, TokenExpiredError.prototype)
  }

  serializeErrors() {
    return [{ message: 'Token has expired' }]
  }
}
```

#### 5. **Express Middleware**

**Error Handler**:

```typescript
export const errorHandler = (
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  if (err instanceof CustomError) {
    return res.status(err.statusCode).send({ errors: err.serializeErrors() })
  }

  console.error(err)
  res.status(500).send({
    errors: [{ message: 'Something went wrong' }],
  })
}
```

**Current User**:

```typescript
export const currentUser = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  if (!req.session?.jwt) {
    return next()
  }

  try {
    const payload = jwt.verify(req.session.jwt, process.env.JWT_KEY!)
    req.currentUser = payload
  } catch (err) {
    // Invalid token - silently ignore
  }

  next()
}
```

**Require Auth**:

```typescript
export const requireAuth = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  if (!req.currentUser) {
    throw new NotAuthorizedError()
  }

  next()
}
```

**Validate Request**:

```typescript
import { validationResult } from 'express-validator'

export const validateRequest = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const errors = validationResult(req)

  if (!errors.isEmpty()) {
    throw new RequestValidationError(errors.array())
  }

  next()
}
```

#### 6. **Shared Enums**

**OrderStatus**:

```typescript
export enum OrderStatus {
  Created = 'created',
  Cancelled = 'cancelled',
  AwaitingPayment = 'awaiting:payment',
  Complete = 'complete',
}
```

**TokenType**:

```typescript
export enum TokenType {
  PasswordReset = 'password-reset',
  EmailVerification = 'email-verification',
}
```

### Benefits

✅ **Type Safety**: TypeScript ensures type consistency across services  
✅ **DRY Principle**: Eliminates code duplication  
✅ **Consistency**: All services handle errors and events identically  
✅ **Maintainability**: Update once, deploy everywhere  
✅ **Versioning**: Services can specify compatible common library versions

### Usage Example

```typescript
// In any microservice
import {
  Publisher,
  Listener,
  Subjects,
  TicketCreatedEvent,
  OrderStatus,
  requireAuth,
  validateRequest,
  NotFoundError,
  BadRequestError,
} from '@chato-zombilet/common'

// Use shared types and logic
router.post('/api/tickets', requireAuth, validateRequest, async (req, res) => {
  // Service-specific logic
})
```

### Version Management

Each service's `package.json` specifies the common library version:

```json
{
  "dependencies": {
    "@chato-zombilet/common": "^1.0.48"
  }
}
```

Updating the common library:

1. Make changes to common library
2. Publish new version to NPM
3. Update version in service `package.json`
4. Rebuild and deploy affected services

---

## ☸️ Kubernetes Infrastructure

All services run on Kubernetes with dedicated deployments and services.

### Cluster Configuration

**Location**: `infra/k8s/` (shared configs) and `infra/k8s-dev/` or `infra/k8s-prod/` (environment-specific)

### Deployment Pattern

Each microservice follows this pattern:

```yaml
# Example: auth-depl.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth
  template:
    metadata:
      labels:
        app: auth
    spec:
      containers:
        - name: auth
          image: chatonode/auth
          env:
            - name: MONGO_URI
              value: 'mongodb://auth-mongo-srv:27017/auth'
            - name: JWT_KEY
              valueFrom:
                secretKeyRef:
                  name: jwt-secret
                  key: JWT_KEY
            - name: NATS_CLUSTER_ID
              value: zombilet
            - name: NATS_CLIENT_ID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name # Unique pod name
            - name: NATS_URL
              value: 'http://nats-streaming-srv:4222'
---
apiVersion: v1
kind: Service
metadata:
  name: auth-srv
spec:
  selector:
    app: auth
  ports:
    - name: auth
      protocol: TCP
      port: 3000
      targetPort: 3000
```

### Core Components

#### 1. **Microservice Deployments**

| Deployment        | Image                | Database         | External Dependencies |
| ----------------- | -------------------- | ---------------- | --------------------- |
| `auth-depl`       | chatonode/auth       | auth-mongo       | NATS, SendGrid        |
| `tickets-depl`    | chatonode/tickets    | tickets-mongo    | NATS                  |
| `orders-depl`     | chatonode/orders     | orders-mongo     | NATS                  |
| `payments-depl`   | chatonode/payments   | payments-mongo   | NATS, Stripe          |
| `contact-depl`    | chatonode/contact    | contact-mongo    | NATS, SendGrid        |
| `expiration-depl` | chatonode/expiration | N/A (uses Redis) | NATS, Redis           |
| `client-depl`     | chatonode/client     | N/A              | N/A                   |

#### 2. **MongoDB Deployments**

Each service has a dedicated MongoDB instance:

```yaml
# Example: auth-mongo-depl.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-mongo-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-mongo
  template:
    metadata:
      labels:
        app: auth-mongo
    spec:
      containers:
        - name: auth-mongo
          image: mongo
---
apiVersion: v1
kind: Service
metadata:
  name: auth-mongo-srv
spec:
  selector:
    app: auth-mongo
  ports:
    - name: db
      protocol: TCP
      port: 27017
      targetPort: 27017
```

#### 3. **NATS Streaming Deployment**

```yaml
# nats-streaming-depl.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nats-streaming-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nats-streaming
  template:
    metadata:
      labels:
        app: nats-streaming
    spec:
      containers:
        - name: nats-streaming
          image: nats-streaming:0.17.0
          args: [
              '-p',
              '4222', # Client port
              '-m',
              '8222', # Monitoring port
              '-hbi',
              '5s', # Heartbeat interval
              '-hbt',
              '5s', # Heartbeat timeout
              '-hbf',
              '2', # Heartbeat fail count
              '-SD', # Enable streaming
              '-cid',
              'zombilet', # Cluster ID
            ]
---
apiVersion: v1
kind: Service
metadata:
  name: nats-streaming-srv
spec:
  selector:
    app: nats-streaming
  ports:
    - name: client
      protocol: TCP
      port: 4222
      targetPort: 4222
    - name: monitoring
      protocol: TCP
      port: 8222
      targetPort: 8222
```

#### 4. **Redis Deployment** (for Expiration Service)

```yaml
# expiration-redis-depl.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: expiration-redis-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: expiration-redis
  template:
    metadata:
      labels:
        app: expiration-redis
    spec:
      containers:
        - name: expiration-redis
          image: redis
---
apiVersion: v1
kind: Service
metadata:
  name: expiration-redis-srv
spec:
  selector:
    app: expiration-redis
  ports:
    - name: db
      protocol: TCP
      port: 6379
      targetPort: 6379
```

### Kubernetes Secrets

Sensitive configuration is stored as secrets:

```bash
# Create secrets
kubectl create secret generic jwt-secret --from-literal=JWT_KEY=your_jwt_key
kubectl create secret generic stripe-secret --from-literal=STRIPE_KEY=your_stripe_key
kubectl create secret generic sendgrid-secret --from-literal=SENDGRID_KEY=your_sendgrid_key
kubectl create secret generic sendgrid-email --from-literal=SENDGRID_EMAIL=your@email.com
```

Referenced in deployments:

```yaml
env:
  - name: JWT_KEY
    valueFrom:
      secretKeyRef:
        name: jwt-secret
        key: JWT_KEY
  - name: STRIPE_KEY
    valueFrom:
      secretKeyRef:
        name: stripe-secret
        key: STRIPE_KEY
```

### Ingress NGINX Configuration

#### Development (`infra/k8s-dev/ingress-srv.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-srv
  annotations:
    kubernetes.io/ingress.class: 'nginx'
    nginx.ingress.kubernetes.io/use-regex: 'true'
spec:
  rules:
    - host: zombilet.dev
      http:
        paths:
          - path: /api/users/?(.*)
            pathType: Prefix
            backend:
              service:
                name: auth-srv
                port:
                  number: 3000
          - path: /api/tickets/?(.*)
            pathType: Prefix
            backend:
              service:
                name: tickets-srv
                port:
                  number: 3000
          - path: /api/orders/?(.*)
            pathType: Prefix
            backend:
              service:
                name: orders-srv
                port:
                  number: 3000
          - path: /api/payments/?(.*)
            pathType: Prefix
            backend:
              service:
                name: payments-srv
                port:
                  number: 3000
          - path: /?(.*)
            pathType: Prefix
            backend:
              service:
                name: client-srv
                port:
                  number: 3000
```

**Routing Logic**:

- `/api/users/*` → Auth Service
- `/api/tickets/*` → Tickets Service
- `/api/orders/*` → Orders Service
- `/api/payments/*` → Payments Service
- All other paths → Client (Next.js)

#### Production (`infra/k8s-prod/ingress-srv.yaml`)

- Host: `www.zombilet.xyz`
- Includes Digital Ocean-specific Load Balancer configuration
- Proxy protocol enabled for proper client IP forwarding

### Service Communication

```
External Request
       ↓
Ingress NGINX (Load Balancer)
       ↓
Service Selection (based on path)
       ↓
ClusterIP Service (e.g., auth-srv:3000)
       ↓
Pod (e.g., auth-depl pod)
```

Internal service-to-service communication:

- Services communicate via NATS (async)
- No direct HTTP calls between services
- Each service only knows about NATS

### Environment-Specific Configs

**Development** (`k8s-dev/`):

- Domain: `zombilet.dev` (requires `/etc/hosts` modification)
- HTTPS disabled in cookie-session for local testing
- All services run with 1 replica

**Production** (`k8s-prod/`):

- Domain: `www.zombilet.xyz`
- HTTPS enforced
- Digital Ocean Load Balancer annotations
- Scalable replica counts

### Deployment Commands

```bash
# Apply all configurations
kubectl apply -f infra/k8s
kubectl apply -f infra/k8s-dev  # or k8s-prod

# Check deployments
kubectl get deployments
kubectl get pods
kubectl get services

# View logs
kubectl logs <pod-name>

# Access NATS monitoring
kubectl port-forward nats-streaming-depl-<hash> 8222:8222
# Visit: http://localhost:8222/streaming
```

---

## 🔧 Development Workflow

### Skaffold for Local Development

**Skaffold** enables simultaneous development across all microservices with hot-reload capabilities.

### Configuration (`skaffold.yaml`)

```yaml
apiVersion: skaffold/v2alpha3
kind: Config
deploy:
  kubectl:
    manifests:
      - ./infra/k8s/*
      - ./infra/k8s-dev/*
build:
  local:
    push: false # Don't push to registry (local development)
  artifacts:
    - image: chatonode/auth
      context: ./auth
      docker:
        dockerfile: Dockerfile
      sync:
        manual:
          - src: 'src/**/*.ts'
            dest: .
    - image: chatonode/tickets
      context: ./tickets
      docker:
        dockerfile: Dockerfile
      sync:
        manual:
          - src: 'src/**/*.ts'
            dest: .
    # ... (similar for orders, payments, expiration, contact, client)
```

### Hot-Reload Mechanism

**File Sync Without Rebuild**:

- TypeScript files (`src/**/*.ts`) sync directly to containers
- `ts-node-dev` watches for changes and restarts automatically
- No Docker image rebuild required for code changes
- Near-instant feedback loop

**When Rebuild IS Required**:

- Changes to `package.json` (new dependencies)
- Changes to `Dockerfile`
- Changes outside `src/` directory

### Development Setup

#### Prerequisites

1. **Docker Desktop** with Kubernetes enabled

   - macOS/Windows: Enable Kubernetes in Docker Desktop settings
   - Linux: Install Minikube or K3s

2. **kubectl** - Kubernetes CLI

   ```bash
   # Verify installation
   kubectl version
   ```

3. **Skaffold** - Development orchestrator

   ```bash
   # macOS
   brew install skaffold

   # Verify installation
   skaffold version
   ```

4. **Node.js 16.x**
   ```bash
   node --version  # Should be v16.x
   ```

#### Local Domain Configuration

Add entry to `/etc/hosts`:

```bash
# /etc/hosts
127.0.0.1 zombilet.dev
```

#### Kubernetes Secrets

Create required secrets before starting:

```bash
# JWT Secret
kubectl create secret generic jwt-secret --from-literal=JWT_KEY=your_secret_key_here

# Stripe Secret
kubectl create secret generic stripe-secret --from-literal=STRIPE_KEY=sk_test_your_stripe_key

# SendGrid Secret
kubectl create secret generic sendgrid-secret --from-literal=SENDGRID_KEY=SG.your_sendgrid_key

# SendGrid Email
kubectl create secret generic sendgrid-email --from-literal=SENDGRID_EMAIL=your@email.com
```

#### Starting Development Environment

```bash
# From project root
skaffold dev
```

**What happens**:

1. Builds Docker images for all 7 services
2. Deploys all Kubernetes manifests
3. Sets up file sync for hot-reload
4. Streams logs from all pods
5. Watches for file changes

**Access the application**:

- Frontend: `http://zombilet.dev`
- NATS Monitoring: `http://zombilet.dev:8222/streaming` (port-forward required)

#### Stopping Development Environment

```bash
# Press Ctrl+C in terminal running skaffold dev
# Skaffold will automatically clean up deployments
```

### Development Workflow Example

```bash
# 1. Start Skaffold
skaffold dev

# 2. Edit auth service
vim auth/src/routes/sign-up.ts

# 3. Save file
# → ts-node-dev detects change
# → Auth service restarts automatically
# → Changes visible in ~2 seconds

# 4. Check logs (streaming automatically)
# [auth] Listening on port 3000...
```

### Working with Individual Services

**Running tests**:

```bash
# In service directory
cd auth
npm run test          # Watch mode
npm run test:ci       # CI mode (single run)
```

**Installing dependencies**:

```bash
cd auth
npm install new-package

# Restart skaffold to rebuild
# (or just rebuild that service's image)
```

### Common Development Commands

```bash
# View all pods
kubectl get pods

# View logs for specific service
kubectl logs auth-depl-<hash>

# Exec into pod
kubectl exec -it auth-depl-<hash> sh

# Restart deployment
kubectl rollout restart deployment auth-depl

# Delete all resources
kubectl delete all --all

# Port-forward to NATS monitoring
kubectl port-forward nats-streaming-depl-<hash> 8222:8222
```

### Debugging Tips

**Service won't start?**

```bash
# Check pod status
kubectl get pods

# View pod events
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>
```

**Database connection issues?**

```bash
# Verify MongoDB service
kubectl get service auth-mongo-srv

# Test connection from pod
kubectl exec -it auth-depl-<hash> -- sh
nc -zv auth-mongo-srv 27017
```

**Ingress not routing?**

```bash
# Check ingress
kubectl get ingress

# Verify ingress-nginx is running
kubectl get pods -n ingress-nginx
```

### Monorepo Advantages

✅ **Unified Development**: All services in one repository  
✅ **Shared Tooling**: Common scripts, configs, and dependencies  
✅ **Atomic Changes**: Update multiple services in single PR  
✅ **Simplified Onboarding**: Clone once, develop anywhere  
✅ **Version Control**: Track inter-service changes together

---

## 🚀 CI/CD Pipeline

Automated testing and deployment via **GitHub Actions**.

### Test Workflows

Each microservice has a dedicated test workflow triggered on pull requests:

**File**: `.github/workflows/tests-auth.yml`

```yaml
name: tests-auth

on:
  pull_request:
    paths:
      - 'auth/**' # Only trigger when auth service changes

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        run: cd auth && npm install && npm run test:ci
```

**All Test Workflows**:

- `tests-auth.yml`
- `tests-tickets.yml`
- `tests-orders.yml`
- `tests-payments.yml`
- `tests-contact.yml`
- `tests-expiration.yml`

### Deployment Workflows

Separate deploy workflows for production (currently `.yaml.old` - disabled):

**File**: `.github/workflows/deploy-auth.yaml.old`

```yaml
name: deploy-auth

on:
  push:
    branches:
      - main
    paths:
      - 'auth/**' # Only deploy when auth service changes

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build Docker image
        run: |
          cd auth
          docker build -t chatonode/auth .

      - name: Log in to Docker
        run: docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
        env:
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}

      - name: Push image to Docker Hub
        run: docker push chatonode/auth

      - name: Install doctl
        uses: digitalocean/action-doctl@v2
        with:
          token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}

      - name: Save DigitalOcean kubeconfig
        run: doctl kubernetes cluster kubeconfig save ticketing

      - name: Deploy to Kubernetes
        run: kubectl rollout restart deployment auth-depl
```

### CI/CD Flow

```
Developer
    ↓
Pushes to feature branch
    ↓
Opens Pull Request
    ↓
GitHub Actions: Test workflows trigger
    ├─ tests-auth.yml (if auth/** changed)
    ├─ tests-tickets.yml (if tickets/** changed)
    └─ ... (only changed services)
    ↓
All tests pass ✓
    ↓
Merge to main branch
    ↓
GitHub Actions: Deploy workflows trigger
    ├─ deploy-auth.yaml (if auth/** changed)
    ├─ Deploy to Docker Hub
    └─ Deploy to Digital Ocean K8s
    ↓
Rolling update on production
```

### Optimized Builds

**Path-based triggers** ensure only changed services are tested/deployed:

```yaml
on:
  pull_request:
    paths:
      - 'auth/**' # Only this service
```

Benefits:

- ✅ Faster CI/CD (no unnecessary builds)
- ✅ Independent deployments
- ✅ Reduced costs
- ✅ Clear change tracking

### Required GitHub Secrets

```
DOCKER_USERNAME          # Docker Hub username
DOCKER_PASSWORD          # Docker Hub password/token
DIGITALOCEAN_ACCESS_TOKEN    # DO API token
```

---

## 🧪 Testing Strategy

Comprehensive test coverage across all layers using **Jest** and **Supertest**.

### Test Configuration

**File**: `auth/package.json` (example)

```json
{
  "jest": {
    "preset": "ts-jest",
    "testEnvironment": "node",
    "setupFilesAfterEnv": ["./src/test/setup.ts"],
    "testTimeout": 35000
  },
  "scripts": {
    "test": "jest --watchAll --no-cache",
    "test:ci": "jest"
  }
}
```

### Test Setup

**File**: `auth/src/test/setup.ts`

```typescript
import { MongoMemoryServer } from 'mongodb-memory-server'
import mongoose from 'mongoose'

// Mock NATS wrapper
jest.mock('../nats-wrapper')

let mongo: any

beforeAll(async () => {
  // Set test environment variable
  process.env.JWT_KEY = 'test-jwt-key'

  // Start in-memory MongoDB
  mongo = await MongoMemoryServer.create()
  const mongoUri = await mongo.getUri()

  await mongoose.connect(mongoUri)
})

beforeEach(async () => {
  // Clear all mocks
  jest.clearAllMocks()

  // Reset database between tests
  const collections = await mongoose.connection.db.collections()
  for (let collection of collections) {
    await collection.deleteMany({})
  }
})

afterAll(async () => {
  jest.restoreAllMocks()
  await mongo.stop()
  await mongoose.connection.close()
})
```

### NATS Mocking

**File**: `auth/src/__mocks__/nats-wrapper.ts`

```typescript
export const natsWrapper = {
  client: {
    publish: jest
      .fn()
      .mockImplementation(
        (subject: string, data: string, callback: () => void) => {
          callback()
        }
      ),
  },
}
```

### Test Examples

**Route Test**:

```typescript
// auth/src/routes/__test__/sign-up.test.ts
import request from 'supertest'
import { app } from '../../app'

it('returns 201 on successful signup', async () => {
  const response = await request(app)
    .post('/api/users/signup')
    .send({
      email: 'test@test.com',
      password: 'password',
    })
    .expect(201)
})

it('sets cookie after successful signup', async () => {
  const response = await request(app)
    .post('/api/users/signup')
    .send({
      email: 'test@test.com',
      password: 'password',
    })
    .expect(201)

  expect(response.get('Set-Cookie')).toBeDefined()
})
```

**Event Listener Test**:

```typescript
// tickets/src/events/listeners/__test__/order-created-listener.test.ts
import { OrderCreatedListener } from '../order-created-listener'
import { natsWrapper } from '../../../nats-wrapper'
import { Ticket } from '../../../models/ticket'
import { OrderCreatedEvent, OrderStatus } from '@chato-zombilet/common'

it('sets orderId of the ticket', async () => {
  const listener = new OrderCreatedListener(natsWrapper.client)

  const ticket = Ticket.build({
    title: 'concert',
    price: 20,
    userId: '123',
  })
  await ticket.save()

  const data: OrderCreatedEvent['data'] = {
    id: 'order-id',
    version: 0,
    status: OrderStatus.Created,
    userId: '123',
    expiresAt: '2025-12-31',
    ticket: {
      id: ticket.id,
      price: ticket.price,
    },
  }

  // @ts-ignore
  const msg: Message = {
    ack: jest.fn(),
  }

  await listener.onMessage(data, msg)

  const updatedTicket = await Ticket.findById(ticket.id)

  expect(updatedTicket!.orderId).toEqual('order-id')
  expect(msg.ack).toHaveBeenCalled()
})
```

### Test Coverage

✅ **Route handlers** - HTTP request/response testing  
✅ **Middleware** - Authentication, validation  
✅ **Models** - Database operations  
✅ **Event publishers** - NATS publishing (mocked)  
✅ **Event listeners** - Event processing logic  
✅ **Business logic** - Service-specific rules

### Running Tests

```bash
# Watch mode (development)
npm run test

# Single run (CI)
npm run test:ci

# Specific test file
npm test -- sign-up.test.ts

# With coverage
npm test -- --coverage
```

---

## 🔐 Authentication & Security

JWT-based authentication with shared secrets across services.

### Authentication Flow

```
1. User signs up/signs in
   → Auth Service validates credentials
   → Generates JWT containing {id, email}
   → Stores JWT in encrypted cookie

2. User makes authenticated request
   → Browser sends cookie automatically
   → currentUser middleware extracts JWT
   → req.currentUser = {id, email}

3. Protected route handler
   → requireAuth middleware checks req.currentUser
   → If present: continue
   → If missing: throw NotAuthorizedError (401)
```

### JWT Generation

```typescript
// Auth Service
import jwt from 'jsonwebtoken'

const userJwt = jwt.sign(
  {
    id: user.id,
    email: user.email,
  },
  process.env.JWT_KEY! // Shared secret
)

req.session = { jwt: userJwt }
```

### Cookie Configuration

```typescript
import cookieSession from 'cookie-session'

app.use(
  cookieSession({
    signed: false, // Disable encryption (JWT already signed)
    secure: process.env.NODE_ENV !== 'test', // HTTPS only (except tests)
  })
)
```

### Middleware Chain

```typescript
import { currentUser, requireAuth } from '@chato-zombilet/common'

router.post(
  '/api/tickets',
  currentUser, // Extract user from JWT
  requireAuth, // Ensure user exists
  async (req, res) => {
    const userId = req.currentUser!.id
    // Create ticket
  }
)
```

### Password Hashing

```typescript
// auth/src/services/password.ts
import { scrypt, randomBytes } from 'crypto'
import { promisify } from 'util'

const scryptAsync = promisify(scrypt)

export class Password {
  static async toHash(password: string) {
    const salt = randomBytes(8).toString('hex')
    const buf = (await scryptAsync(password, salt, 64)) as Buffer
    return `${buf.toString('hex')}.${salt}`
  }

  static async compare(storedPassword: string, suppliedPassword: string) {
    const [hashedPassword, salt] = storedPassword.split('.')
    const buf = (await scryptAsync(suppliedPassword, salt, 64)) as Buffer
    return buf.toString('hex') === hashedPassword
  }
}
```

### Password Reset Flow

```
1. User requests password reset
   → POST /api/users/forgot-password
   → Creates Token document with expiration
   → Sends email with token

2. User clicks email link with token
   → GET /api/users/validate-token?token=abc123
   → Validates token exists and not expired

3. User submits new password
   → POST /api/users/reset-password
   → Validates token
   → Updates password
   → Deletes token
   → Publishes UserChangedPassword event
```

### Shared JWT Secret

All services share the same JWT_KEY via Kubernetes Secret:

```bash
kubectl create secret generic jwt-secret \
    --from-literal=JWT_KEY=your_secret_key
```

This enables:

- Auth service to generate JWT
- All other services to verify JWT
- No service-to-service authentication calls needed

### Security Features

✅ **Password hashing** with scrypt  
✅ **Salted hashes** prevent rainbow table attacks  
✅ **HTTPS enforcement** in production  
✅ **HTTP-only cookies** prevent XSS  
✅ **Signed JWTs** prevent tampering  
✅ **Shared secrets** enable distributed auth  
✅ **Token expiration** for password resets  
✅ **Request validation** with express-validator

---

## 📊 Data Models & Relationships

Each service owns its data models with versioning for concurrency control.

### User Model (Auth Service)

```typescript
interface UserDoc {
  id: string
  email: string
  password: string // Hashed with scrypt
  version: number
}
```

**Features**:

- Email uniqueness enforced
- Password hashed before save (Mongoose pre-save hook)
- Version tracking for OCC

### Ticket Model (Tickets Service)

```typescript
interface TicketDoc {
  id: string
  title: string
  price: number
  userId: string // Owner of ticket
  orderId?: string // Reserved if set
  version: number
}
```

**Key Methods**:

- `isReserved()`: Checks if ticket has active order

### Ticket Model (Orders Service - Replicated)

```typescript
interface TicketDoc {
    id: string          // Same as Tickets service
    title: string
    price: number
    version: number
}

// Custom finder for OCC
static findPreviousVersion(params: {id: string, version: number})
```

**Data Replication**:

- Listens to `TicketCreated` → Creates local copy
- Listens to `TicketUpdated` → Updates local copy
- Uses OCC to prevent out-of-order updates

### Order Model (Orders Service)

```typescript
interface OrderDoc {
  id: string
  userId: string
  status: OrderStatus // Created | Cancelled | AwaitingPayment | Complete
  expiresAt: Date
  ticket: TicketDoc // Reference to local ticket replica
  version: number
}

enum OrderStatus {
  Created = 'created',
  Cancelled = 'cancelled',
  AwaitingPayment = 'awaiting:payment',
  Complete = 'complete',
}
```

**Relationships**:

- References local `Ticket` document (not cross-service!)
- Status transitions: Created → Complete (payment) or Cancelled (expiration/user)

### Order Model (Payments Service - Replicated)

```typescript
interface OrderDoc {
  id: string
  userId: string
  status: OrderStatus
  version: number
  price: number
}
```

**Data Replication**:

- Listens to `OrderCreated` → Creates local copy
- Listens to `OrderCancelled` → Updates status
- Listens to `PaymentCreated` → Prevents duplicate payments

### Payment Model (Payments Service)

```typescript
interface PaymentDoc {
  id: string
  orderId: string
  stripeId: string // Stripe charge ID
}
```

**Features**:

- Stores Stripe charge reference
- Linked to order for idempotency

### Token Model (Auth Service)

```typescript
interface TokenDoc {
  id: string
  userId: string
  type: TokenType // password-reset | email-verification
  expiresAt: Date
}
```

**Usage**:

- Password reset tokens with expiration
- One-time use (deleted after consumption)

### Data Replication Pattern

```
Tickets Service (Source of Truth)
    ↓ Publishes: TicketCreated
Orders Service (Consumer)
    ↓ Creates local replica
    ↓ Uses for order creation (no cross-service query)

Orders Service (Source of Truth)
    ↓ Publishes: OrderCreated
Payments Service (Consumer)
    ↓ Creates local replica
    ↓ Uses for payment validation
```

### Key Principles

✅ **No Foreign Keys**: Services don't share databases  
✅ **Data Replication**: Events replicate necessary data  
✅ **Version Control**: All entities use OCC  
✅ **Local Queries**: Services query their own database only  
✅ **Event Sourcing**: State changes tracked via events

---

## ⏰ Async Processing & Job Queues

Redis + Bull.js handle time-based workflows in the Expiration Service.

### Queue Setup

```typescript
// expiration/src/queues/expiration-queue.ts
import Queue from 'bull'

interface Payload {
  orderId: string
}

const expirationQueue = new Queue<Payload>('order-expiration', {
  redis: {
    host: process.env.REDIS_HOST, // expiration-redis-srv
    port: 6379,
  },
})

// Process jobs
expirationQueue.process(async (job) => {
  // Publish expiration event
  await new ExpirationCompletedPublisher(natsWrapper.client).publish({
    orderId: job.data.orderId,
  })
})
```

### Job Creation

```typescript
// When order is created
new OrderCreatedListener(natsWrapper.client).listen()

async onMessage(data: OrderCreatedEvent['data'], msg: Message) {
    const delay = new Date(data.expiresAt).getTime() - new Date().getTime()

    // Enqueue job with delay
    await expirationQueue.add(
        { orderId: data.id },
        { delay }  // 1 minute (60000ms)
    )

    msg.ack()
}
```

### Expiration Flow

```
1. User creates order
   → Orders Service publishes OrderCreated
   → expiresAt = now + 1 minute

2. Expiration Service receives event
   → Enqueues job in Redis with 1-minute delay
   → Job: {orderId: '123'}

3. After 1 minute
   → Bull.js processes job
   → Publishes ExpirationCompleted event

4. Orders Service receives ExpirationCompleted
   → Finds order by ID
   → If status still 'Created': Cancel order
   → Publishes OrderCancelled event

5. Tickets Service receives OrderCancelled
   → Clears ticket's orderId
   → Ticket becomes available again
```

### Configuration

**Expiration Window**:

```typescript
// orders/src/routes/new.ts
const EXPIRATION_WINDOW_SECONDS = 1 * 60 // 1 minute for testing
// Production could use: 15 * 60 (15 minutes)
```

### Benefits

✅ **Decoupled**: Expiration logic isolated in dedicated service  
✅ **Reliable**: Redis persistence ensures job execution  
✅ **Scalable**: Bull.js handles thousands of concurrent jobs  
✅ **Flexible**: Easy to adjust expiration windows

---

## 💻 Client Application

Next.js 12 application with server-side rendering and Kubernetes-aware routing.

### Architecture

**Directory Structure** (Pre-v13 Pages):

```
client/
├── pages/
│   ├── _app.js              # Global wrapper
│   ├── index.js             # Landing page (ticket list)
│   ├── auth/
│   │   ├── signin.js
│   │   ├── signup.js
│   │   └── signout.js
│   ├── tickets/
│   │   ├── [ticketId].js    # Ticket detail
│   │   └── new.js           # Create ticket
│   └── orders/
│       └── [orderId].js     # Order detail with payment
├── components/
│   └── header.js            # Navigation
├── hooks/
│   └── use-request.js       # Custom hook for API calls
└── api/
    └── build-client.js      # Axios client builder
```

### Build Client Utility

Handles server-side vs browser-side API requests:

```javascript
// api/build-client.js
import axios from 'axios'

const buildClient = ({ req }) => {
  if (typeof window === 'undefined') {
    // Server-side: Use Kubernetes internal service
    return axios.create({
      baseURL:
        'http://ingress-nginx-controller.ingress-nginx.svc.cluster.local',
      headers: req.headers, // Forward cookies!
    })
  } else {
    // Browser-side: Relative URLs
    return axios.create({})
  }
}
```

**Why This Pattern**:

- Server-side rendering needs full URL
- Must forward cookies for authentication
- Uses Kubernetes internal service name

### Server-Side Rendering

```javascript
// pages/index.js
const LandingPage = ({ currentUser, tickets }) => {
  // Render ticket list
}

LandingPage.getInitialProps = async (context, client, currentUser) => {
  // Runs on server during initial load
  const { data: tickets } = await client.get('/api/tickets')
  return { tickets }
}
```

### Custom useRequest Hook

```javascript
// hooks/use-request.js
import axios from 'axios'
import { useState } from 'react'

export default ({ url, method, body, onSuccess }) => {
  const [errors, setErrors] = useState(null)

  const doRequest = async () => {
    try {
      setErrors(null)
      const response = await axios[method](url, body)

      if (onSuccess) {
        onSuccess(response.data)
      }

      return response.data
    } catch (err) {
      setErrors(
        <div className="alert alert-danger">
          <ul className="my-0">
            {err.response.data.errors.map((err) => (
              <li key={err.message}>{err.message}</li>
            ))}
          </ul>
        </div>
      )
    }
  }

  return { doRequest, errors }
}
```

**Usage**:

```javascript
const { doRequest, errors } = useRequest({
  url: '/api/users/signup',
  method: 'post',
  body: { email, password },
  onSuccess: () => Router.push('/'),
})

await doRequest()
```

### Stripe Integration

```javascript
// pages/orders/[orderId].js
import StripeCheckout from 'react-stripe-checkout'
;<StripeCheckout
  token={(token) => doRequest({ token: token.id })}
  stripeKey="pk_test_your_key"
  amount={order.ticket.price * 100} // cents
  email={currentUser.email}
/>
```

### Authentication State

```javascript
// pages/_app.js
const AppComponent = ({ Component, pageProps, currentUser }) => {
  return (
    <div>
      <Header currentUser={currentUser} />
      <Component currentUser={currentUser} {...pageProps} />
    </div>
  )
}

AppComponent.getInitialProps = async (appContext) => {
  const client = buildClient(appContext.ctx)
  const { data } = await client.get('/api/users/currentuser')

  // Fetch page-specific props
  let pageProps = {}
  if (appContext.Component.getInitialProps) {
    pageProps = await appContext.Component.getInitialProps(
      appContext.ctx,
      client,
      data.currentUser
    )
  }

  return {
    pageProps,
    currentUser: data.currentUser,
  }
}
```

### Styling

- **Bootstrap 5.1.3** for responsive design
- Global styles in `_app.js`
- Navbar with authentication state
- Form validation with error display

---

## 🚀 Getting Started

### Prerequisites

Ensure you have the following installed:

- **Docker Desktop** (with Kubernetes enabled) or Minikube
- **Node.js** 16.x
- **kubectl** CLI
- **Skaffold** CLI
- **Git**

### Step-by-Step Setup

#### 1. Clone Repository

```bash
git clone https://github.com/chatonode/mern-zombilet.git
cd mern-zombilet
```

#### 2. Configure Local Domain

Add entry to your hosts file:

```bash
# macOS/Linux
sudo nano /etc/hosts

# Add this line:
127.0.0.1 zombilet.dev

# Windows
# Edit C:\Windows\System32\drivers\etc\hosts
```

#### 3. Enable Kubernetes

**Docker Desktop**:

- Open Docker Desktop → Settings → Kubernetes
- Check "Enable Kubernetes"
- Apply & Restart

**Verify**:

```bash
kubectl version
kubectl get nodes  # Should show 1 node ready
```

#### 4. Install Ingress NGINX

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/cloud/deploy.yaml
```

#### 5. Create Kubernetes Secrets

```bash
# JWT Secret (required)
kubectl create secret generic jwt-secret --from-literal=JWT_KEY=your_secret_here

# Stripe Secret (required for payments)
kubectl create secret generic stripe-secret --from-literal=STRIPE_KEY=sk_test_your_stripe_key

# SendGrid Secrets (required for emails)
kubectl create secret generic sendgrid-secret --from-literal=SENDGRID_KEY=SG.your_sendgrid_api_key
kubectl create secret generic sendgrid-email --from-literal=SENDGRID_EMAIL=noreply@yourdom

ain.com
```

#### 6. Install Skaffold

```bash
# macOS
brew install skaffold

# Linux
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
sudo install skaffold /usr/local/bin/

# Windows
choco install skaffold

# Verify
skaffold version
```

#### 7. Start Development Environment

```bash
# From project root
skaffold dev
```

**What to expect**:

- Docker images will build (first time: 5-10 minutes)
- All services deploy to Kubernetes
- Logs stream from all pods
- File changes trigger automatic reloads

#### 8. Access Application

- **Frontend**: http://zombilet.dev
- **NATS Monitoring**: Port-forward first, then http://localhost:8222/streaming

```bash
# Port-forward NATS monitoring
kubectl port-forward svc/nats-streaming-srv 8222:8222
```

### Troubleshooting

**"zombilet.dev" not resolving?**

- Check `/etc/hosts` file contains `127.0.0.1 zombilet.dev`
- Flush DNS cache: `sudo dscacheutil -flushcache` (macOS)

**Pods not starting?**

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

**Ingress not working?**

```bash
kubectl get ingress
kubectl get svc -n ingress-nginx
```

**Port conflicts?**

- Ensure ports 80, 443 are free
- Stop other local web servers

### Development Tips

**Running individual service tests**:

```bash
cd auth
npm install
npm test
```

**Rebuilding specific service**:

```bash
# Make changes to package.json or Dockerfile
# Stop skaffold (Ctrl+C)
skaffold dev  # Restart
```

**Viewing service logs**:

```bash
kubectl logs -f auth-depl-<pod-hash>
```

---

## 📡 API Reference

### Authentication Endpoints

#### Sign Up

```
POST /api/users/signup
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}

Response 201:
{
  "id": "507f1f77bcf86cd799439011",
  "email": "user@example.com"
}
```

#### Sign In

```
POST /api/users/signin

{
  "email": "user@example.com",
  "password": "password123"
}

Response 200:
{
  "id": "507f1f77bcf86cd799439011",
  "email": "user@example.com"
}
```

#### Sign Out

```
POST /api/users/signout

Response 200: {}
```

#### Current User

```
GET /api/users/currentuser

Response 200:
{
  "currentUser": {
    "id": "507f1f77bcf86cd799439011",
    "email": "user@example.com"
  }
}
```

#### Forgot Password

```
POST /api/users/forgot-password

{
  "email": "user@example.com"
}

Response 200: { "message": "Reset email sent" }
```

### Tickets Endpoints

#### Create Ticket

```
POST /api/tickets
Authorization: Required

{
  "title": "Concert Ticket",
  "price": 20
}

Response 201:
{
  "id": "507f1f77bcf86cd799439011",
  "title": "Concert Ticket",
  "price": 20,
  "userId": "507f1f77bcf86cd799439012",
  "version": 0
}
```

#### List Tickets

```
GET /api/tickets

Response 200:
[
  {
    "id": "507f1f77bcf86cd799439011",
    "title": "Concert Ticket",
    "price": 20,
    "userId": "507f1f77bcf86cd799439012",
    "orderId": null,
    "version": 0
  }
]
```

#### Get Ticket

```
GET /api/tickets/:id

Response 200:
{
  "id": "507f1f77bcf86cd799439011",
  "title": "Concert Ticket",
  "price": 20,
  "userId": "507f1f77bcf86cd799439012"
}
```

#### Update Ticket

```
PUT /api/tickets/:id
Authorization: Required

{
  "title": "Updated Concert",
  "price": 25
}

Response 200:
{
  "id": "507f1f77bcf86cd799439011",
  "title": "Updated Concert",
  "price": 25,
  "version": 1
}
```

### Orders Endpoints

#### Create Order

```
POST /api/orders
Authorization: Required

{
  "ticketId": "507f1f77bcf86cd799439011"
}

Response 201:
{
  "id": "507f1f77bcf86cd799439020",
  "status": "created",
  "userId": "507f1f77bcf86cd799439012",
  "expiresAt": "2025-12-29T12:01:00.000Z",
  "ticket": {
    "id": "507f1f77bcf86cd799439011",
    "title": "Concert Ticket",
    "price": 20
  }
}
```

#### List Orders

```
GET /api/orders
Authorization: Required

Response 200:
[
  {
    "id": "507f1f77bcf86cd799439020",
    "status": "created",
    "expiresAt": "2025-12-29T12:01:00.000Z",
    "ticket": { ... }
  }
]
```

#### Get Order

```
GET /api/orders/:id
Authorization: Required

Response 200:
{
  "id": "507f1f77bcf86cd799439020",
  "status": "created",
  "ticket": { ... }
}
```

#### Cancel Order

```
DELETE /api/orders/:id
Authorization: Required

Response 204: No Content
```

### Payments Endpoints

#### Create Payment

```
POST /api/payments
Authorization: Required

{
  "token": "tok_visa",  // Stripe token
  "orderId": "507f1f77bcf86cd799439020"
}

Response 201:
{
  "id": "507f1f77bcf86cd799439030",
  "orderId": "507f1f77bcf86cd799439020",
  "stripeId": "ch_1234567890"
}
```

### Error Responses

All endpoints return errors in this format:

```json
{
  "errors": [
    {
      "message": "Error description",
      "field": "fieldName" // Optional
    }
  ]
}
```

Common status codes:

- `400` - Bad Request (validation errors)
- `401` - Not Authorized
- `404` - Not Found
- `500` - Internal Server Error

---

## 🌐 Production Deployment

### Digital Ocean Kubernetes

The platform was deployed to **Digital Ocean Kubernetes** with the domain **www.zombilet.xyz**.

#### Production Configuration

**Ingress** (`infra/k8s-prod/ingress-srv.yaml`):

```yaml
spec:
  rules:
    - host: www.zombilet.xyz
      http:
        paths:
          - path: /api/users/?(.*)
            backend:
              service:
                name: auth-srv
                port: 3000
          # ... other services
```

**Load Balancer Configuration**:

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/do-loadbalancer-enable-proxy-protocol: 'true'
    service.beta.kubernetes.io/do-loadbalancer-hostname: 'www.zombilet.xyz'
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
```

#### Deployment Process

1. **Cluster Creation** on Digital Ocean
2. **DNS Configuration**: Point `www.zombilet.xyz` to Load Balancer IP
3. **Secrets Management**: Create all K8s secrets
4. **Deploy Infrastructure**:
   ```bash
   kubectl apply -f infra/k8s
   kubectl apply -f infra/k8s-prod
   ```
5. **CI/CD**: GitHub Actions automate deployments on push to main

#### Scaling

**Horizontal Pod Autoscaling**:

```bash
kubectl autoscale deployment auth-depl --cpu-percent=80 --min=1 --max=5
```

**Manual Scaling**:

```bash
kubectl scale deployment auth-depl --replicas=3
```

#### Monitoring

- **NATS Monitoring**: Port-forward to access streaming dashboard
- **Kubernetes Dashboard**: Standard K8s monitoring
- **Application Logs**: `kubectl logs <pod>`

#### Rolling Updates

```bash
# Update deployment
kubectl set image deployment/auth-depl auth=chatonode/auth:v2

# Check rollout status
kubectl rollout status deployment/auth-depl

# Rollback if needed
kubectl rollout undo deployment/auth-depl
```

---

## 📐 Architecture Diagrams

### 1. Overall System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Client Browser                            │
│                        (Next.js React App)                          │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             │ HTTPS
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Ingress NGINX (Load Balancer)                  │
│                      Routes: /api/users, /api/tickets, etc.         │
└─┬───────┬────────┬────────┬──────────┬────────────────────────┬────┘
  │       │        │        │          │                        │
  ▼       ▼        ▼        ▼          ▼                        ▼
┌───────┐┌────────┐┌───────┐┌─────────┐┌──────────┐┌───────────────┐
│ Auth  ││Tickets ││Orders ││Payments ││Contact   ││  Expiration   │
│Service││Service ││Service││Service  ││Service   ││   Service     │
│:3000  ││:3000   ││:3000  ││:3000    ││:3000     ││   :3000       │
└───┬───┘└───┬────┘└───┬───┘└────┬────┘└────┬─────┘└───────┬───────┘
    │        │          │         │          │              │
    ▼        ▼          ▼         ▼          ▼              ▼
┌────────┐┌────────┐┌────────┐┌────────┐┌────────┐    ┌─────────┐
│Auth DB ││Tickets ││Orders  ││Payments││Contact │    │ Redis   │
│MongoDB ││MongoDB ││MongoDB ││MongoDB ││MongoDB │    │ Queue   │
└────────┘└────────┘└────────┘└────────┘└────────┘    └─────────┘
    │        │          │         │          │              │
    └────────┴──────────┴─────────┴──────────┴──────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │   NATS Streaming Server      │
              │   Event Bus (Port 4222)      │
              │   Monitoring (Port 8222)     │
              └──────────────────────────────┘
```

### 2. Event Flow - Ticket Purchase Workflow

```
┌────────┐     ┌────────┐     ┌────────┐     ┌──────────┐     ┌─────────┐
│ Client │     │Tickets │     │ Orders │     │Expiration│     │Payments │
│        │     │Service │     │ Service│     │ Service  │     │ Service │
└───┬────┘     └───┬────┘     └───┬────┘     └────┬─────┘     └────┬────┘
    │              │              │               │                │
    │ 1. Create    │              │               │                │
    │   Ticket     │              │               │                │
    ├─────────────>│              │               │                │
    │              │              │               │                │
    │              │ 2. TicketCreated Event       │                │
    │              ├──────────────>│               │                │
    │              │              │               │                │
    │ 3. Create    │              │               │                │
    │    Order     │              │               │                │
    ├──────────────┼──────────────>│               │                │
    │              │              │               │                │
    │              │ 4. OrderCreated Event         │                │
    │              │<─────────────┤               │                │
    │              │              ├───────────────>│                │
    │              │              │               │                │
    │              │ 5. TicketUpdated Event        │                │
    │              │   (orderId set)               │                │
    │              ├──────────────>│               │                │
    │              │              │               │                │
    │              │              │ 6. Queue Expiration Job         │
    │              │              │   (60 seconds)                  │
    │              │              ├───────────────>│                │
    │              │              │               │                │
    │ 7. Create    │              │               │                │
    │   Payment    │              │               │                │
    ├──────────────┼──────────────┼───────────────┼───────────────>│
    │              │              │               │                │
    │              │              │ 8. PaymentCreated Event         │
    │              │              │<──────────────┼────────────────┤
    │              │              │               │                │
    │              │              │ 9. Order Status → Complete      │
    │              │              │               │                │
    │              │ 10. OrderCompleted Event      │                │
    │              │<─────────────┤               │                │
    │              │              │               │                │
    │              │ 11. TicketUpdated Event       │                │
    │              │   (orderId cleared)           │                │
    │              ├──────────────>│               │                │
    │              │              │               │                │

Alternative Flow - Order Expires:
    │              │              │               │                │
    │              │              │ 12. ExpirationComplete Event   │
    │              │              │<──────────────┤                │
    │              │              │               │                │
    │              │              │ 13. Order Status → Cancelled   │
    │              │              │               │                │
    │              │ 14. OrderCancelled Event      │                │
    │              │<─────────────┤               │                │
    │              │              │               │                │
    │              │ 15. TicketUpdated Event       │                │
    │              │   (orderId cleared)           │                │
    │              ├──────────────>│               │                │
```

### 3. Service Communication Patterns

```
                    ┌────────────────────────┐
                    │   NATS Streaming       │
                    │     Event Bus          │
                    └───────────┬────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        │ Publishes             │ Publishes             │ Publishes
        │ - TicketCreated       │ - OrderCreated        │ - PaymentCreated
        │ - TicketUpdated       │ - OrderCancelled      │
        ▼                       ▼                       ▼
┌───────────────┐       ┌───────────────┐       ┌──────────────┐
│    Tickets    │       │    Orders     │       │   Payments   │
│    Service    │       │    Service    │       │   Service    │
└───────┬───────┘       └───────┬───────┘       └──────────────┘
        │                       │
        │ Listens to:           │ Listens to:
        │ - OrderCreated        │ - TicketCreated
        │ - OrderCancelled      │ - TicketUpdated
        │                       │ - ExpirationComplete
        │                       │ - PaymentCreated
        │                       │
        └───────────────────────┘

Key Patterns:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
► Data Replication: Services replicate needed data via events
► Eventual Consistency: Updates propagate asynchronously
► Queue Groups: Multiple instances share event processing
► Optimistic Concurrency: Version numbers prevent conflicts
► Graceful Shutdown: Services close NATS before terminating
```

### 4. Kubernetes Deployment Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    Namespace: default                      │  │
│  │                                                            │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │  │
│  │  │ Auth Pod     │  │ Auth Pod     │  │ Auth Pod     │   │  │
│  │  │ (Replica 1)  │  │ (Replica 2)  │  │ (Replica 3)  │   │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │  │
│  │         │                 │                 │            │  │
│  │         └─────────────────┴─────────────────┘            │  │
│  │                           │                               │  │
│  │                    ┌──────▼──────┐                       │  │
│  │                    │  Auth-Srv   │ ClusterIP Service     │  │
│  │                    └──────┬──────┘                       │  │
│  │                           │                               │  │
│  │  Similar for: Tickets, Orders, Payments, Contact, Exp    │  │
│  │                                                            │  │
│  │  ┌──────────────────────────────────────────────────┐   │  │
│  │  │           MongoDB Deployments                    │   │  │
│  │  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐   │   │  │
│  │  │  │Auth-DB │ │Ticket  │ │Orders  │ │Payment │   │   │  │
│  │  │  │Pod     │ │DB Pod  │ │DB Pod  │ │DB Pod  │   │   │  │
│  │  │  └────────┘ └────────┘ └────────┘ └────────┘   │   │  │
│  │  │  + Contact-DB + Redis Pod                        │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  │                                                            │  │
│  │  ┌──────────────────────────────────────────────────┐   │  │
│  │  │         NATS Streaming Deployment                │   │  │
│  │  │  ┌─────────────────────────────────────┐        │   │  │
│  │  │  │ NATS Pod                            │        │   │  │
│  │  │  │ - Client Port: 4222                 │        │   │  │
│  │  │  │ - Monitoring Port: 8222             │        │   │  │
│  │  │  │ - Store: Memory                     │        │   │  │
│  │  │  └─────────────────────────────────────┘        │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  │                                                            │  │
│  │  ┌──────────────────────────────────────────────────┐   │  │
│  │  │          Configuration & Secrets                 │   │  │
│  │  │  - JWT_KEY (Secret)                              │   │  │
│  │  │  - STRIPE_KEY (Secret)                           │   │  │
│  │  │  - SENDGRID_API_KEY (Secret)                     │   │  │
│  │  │  - NATS_URL (ConfigMap)                          │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              Namespace: ingress-nginx                  │  │
│  │  ┌──────────────────────────────────────────────┐     │  │
│  │  │   Ingress NGINX Controller                   │     │  │
│  │  │   - LoadBalancer Service                     │     │  │
│  │  │   - Routes traffic to backend services       │     │  │
│  │  │   - SSL/TLS termination                      │     │  │
│  │  └──────────────────────────────────────────────┘     │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### 5. CI/CD Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         GitHub Repository                       │
│                     github.com/chatonode/mern-zombilet          │
└────────────────┬──────────────────────────┬─────────────────────┘
                 │                          │
    ┌────────────▼───────────┐   ┌──────────▼────────────┐
    │   Pull Request Event   │   │  Push to Main Event   │
    │   (Modified files in   │   │  (Modified files in   │
    │    auth/**, etc.)      │   │   auth/**, etc.)      │
    └────────────┬───────────┘   └──────────┬────────────┘
                 │                          │
                 ▼                          ▼
    ┌──────────────────────────┐  ┌─────────────────────────────┐
    │  Test Workflow           │  │  Deploy Workflow            │
    │  (.github/workflows/     │  │  (.github/workflows/        │
    │   tests-auth.yml)        │  │   deploy-auth.yaml.old)     │
    └────────┬─────────────────┘  └──────────┬──────────────────┘
             │                               │
             ▼                               ▼
    ┌─────────────────────┐        ┌──────────────────────────┐
    │ 1. Checkout Code    │        │ 1. Checkout Code         │
    │ 2. Setup Node.js 16 │        │ 2. Build Docker Image    │
    │ 3. Install deps     │        │    - Tag: latest & SHA   │
    │ 4. Run Jest Tests   │        │ 3. Push to Docker Hub    │
    │    (35s timeout)    │        │    chatonode/auth:latest │
    └─────────┬───────────┘        └──────────┬───────────────┘
              │                               │
              ▼                               ▼
    ┌─────────────────────┐        ┌──────────────────────────┐
    │  ✅ Tests Pass      │        │ 4. Install doctl CLI     │
    │  ✅ PR Ready        │        │ 5. Auth to DO K8s        │
    │  ❌ Tests Fail      │        │ 6. kubectl set image     │
    │  ❌ Block PR        │        │    deployment/auth-depl  │
    └─────────────────────┘        │    auth=chatonode/auth   │
                                   │ 7. Rollout Restart       │
                                   └──────────┬───────────────┘
                                              │
                                              ▼
                                   ┌──────────────────────────┐
                                   │   Digital Ocean K8s      │
                                   │   - Rolling Update       │
                                   │   - Zero Downtime        │
                                   │   - Health Checks        │
                                   │   ✅ Deployment Success  │
                                   └──────────────────────────┘

Separate workflows for each service:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
├── tests-auth.yml       → Triggers on: auth/**
├── tests-tickets.yml    → Triggers on: tickets/**
├── tests-orders.yml     → Triggers on: orders/**
├── tests-payments.yml   → Triggers on: payments/**
├── tests-contact.yml    → Triggers on: contact/**
├── tests-expiration.yml → Triggers on: expiration/**
└── deploy-*.yaml.old    → Triggers on: push to main + path filter
```

---

## 📁 Project Structure

```
mern-zombilet/
├── .github/
│   └── workflows/              # CI/CD pipelines
│       ├── tests-auth.yml
│       ├── tests-tickets.yml
│       ├── deploy-auth.yaml.old
│       └── ...
├── auth/                       # Authentication microservice
│   ├── src/
│   │   ├── index.ts           # Entry point
│   │   ├── app.ts             # Express app
│   │   ├── nats-wrapper.ts    # NATS client singleton
│   │   ├── routes/            # API endpoints
│   │   ├── models/            # Mongoose models
│   │   ├── services/          # Business logic
│   │   ├── middlewares/       # Custom middleware
│   │   ├── events/            # Publishers/Listeners
│   │   │   ├── publishers/
│   │   │   └── listeners/
│   │   ├── test/              # Test utilities
│   │   └── __mocks__/         # Jest mocks
│   ├── Dockerfile
│   ├── package.json
│   └── tsconfig.json
├── tickets/                    # Tickets microservice
│   └── (same structure as auth)
├── orders/                     # Orders microservice
│   └── (same structure as auth)
├── payments/                   # Payments microservice
│   ├── src/
│   │   ├── stripe.ts          # Stripe client
│   │   └── ...
├── expiration/                 # Expiration microservice
│   ├── src/
│   │   ├── queues/            # Bull.js queues
│   │   └── ...
├── contact/                    # Contact/Email microservice
│   ├── src/
│   │   ├── services/
│   │   │   └── email/         # SendGrid integration
│   │   └── ...
├── client/                     # Next.js frontend
│   ├── pages/                 # Next.js pages
│   ├── components/            # React components
│   ├── hooks/                 # Custom React hooks
│   ├── api/                   # API utilities
│   ├── Dockerfile
│   └── package.json
├── common/                     # Shared library (submodule)
│   └── (Published to NPM as @chato-zombilet/common)
├── infra/
│   ├── k8s/                   # Shared K8s configs
│   │   ├── auth-depl.yaml
│   │   ├── auth-mongo-depl.yaml
│   │   ├── nats-streaming-depl.yaml
│   │   └── ...
│   ├── k8s-dev/               # Development configs
│   │   └── ingress-srv.yaml
│   └── k8s-prod/              # Production configs
│       └── ingress-srv.yaml
├── nats-test/                  # NATS testing utilities
│   └── src/
│       ├── listener.ts
│       └── publisher.ts
├── skaffold.yaml              # Development orchestration
└── README.md
```

### Key Files

**Service Entry Point** (`auth/src/index.ts`):

- Validates environment variables
- Connects to MongoDB
- Connects to NATS
- Starts Express server
- Initializes event listeners

**Express App** (`auth/src/app.ts`):

- Configures middleware
- Registers route handlers
- Error handling

**NATS Wrapper** (`auth/src/nats-wrapper.ts`):

- Singleton pattern for NATS client
- Connection management
- Graceful shutdown

**Dockerfile Pattern**:

```dockerfile
FROM node:16.13-alpine
WORKDIR /app
COPY package.json ./
RUN npm install --only=prod
COPY ./ ./
CMD ["npm", "start"]
```

---

## 🎯 Design Decisions

### Why Event-Driven Architecture?

✅ **Loose Coupling**: Services don't depend on each other directly  
✅ **Scalability**: Services scale independently based on load  
✅ **Resilience**: Service failures don't cascade  
✅ **Flexibility**: Add new services without modifying existing ones  
✅ **Audit Trail**: All state changes tracked via events

### Why Database-Per-Service?

✅ **Service Autonomy**: Each team owns their data model  
✅ **Technology Freedom**: Use optimal database for each service  
✅ **Failure Isolation**: Database issues don't affect other services  
✅ **Independent Scaling**: Scale databases independently

**Trade-off**: No joins across services, data must be replicated via events

### Why NATS Streaming?

✅ **Lightweight**: Lower resource usage than Kafka  
✅ **At-least-once delivery**: With manual acknowledgments  
✅ **Queue groups**: Built-in load balancing  
✅ **Replay capability**: Can redeliver all messages  
✅ **Simple setup**: Easy for development and production

### Why Optimistic Concurrency Control?

✅ **No Distributed Locks**: Avoid complex coordination  
✅ **Performance**: Optimistic approach doesn't block operations  
✅ **Event Ordering**: Prevents out-of-order updates  
✅ **Simplicity**: Mongoose plugin handles complexity

**Trade-off**: Requires retry logic for concurrent updates

### Why Monorepo?

✅ **Unified Development**: All code in one place  
✅ **Atomic Changes**: Update multiple services in one commit  
✅ **Shared Tooling**: Common configs, scripts, dependencies  
✅ **Simplified CI/CD**: Path-based triggers per service  
✅ **Code Sharing**: Common library easily accessible

### Why TypeScript?

✅ **Type Safety**: Catch errors at compile time  
✅ **Better IDE Support**: Autocomplete, refactoring  
✅ **Event Type Safety**: Strongly-typed event interfaces  
✅ **Self-Documenting**: Types serve as documentation

---

## 📝 License

This project was created for educational purposes to demonstrate microservices architecture patterns.

## 🙏 Acknowledgments

Built as a practical exploration of:

- Microservices architecture
- Event-driven systems
- Kubernetes orchestration
- Optimistic concurrency control
- NATS Streaming
- Full-stack TypeScript development

---

**Repository**: [github.com/chatonode/mern-zombilet](https://github.com/chatonode/mern-zombilet)
