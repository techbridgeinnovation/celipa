# Celipa Microservices Architecture Design

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT APPLICATIONS                      │
│  (React Native App, Web Dashboard, Website)                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ HTTPS/WSS
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                      API GATEWAY                                │
│  • Request Routing                                             │
│  • Authentication (JWT Validation)                             │
│  • Rate Limiting                                               │
│  • CORS Handling                                               │
│  • Load Balancing                                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  AUTH SERVICE │   │  USER SERVICE  │   │ RECEIPT SERVICE│
│               │   │               │   │               │
│ • Register    │   │ • Profile CRUD│   │ • Create       │
│ • Login       │   │ • Preferences │   │ • Read/Update  │
│ • Verify      │   │ • Search      │   │ • Search       │
│ • JWT Tokens  │   │ • Statistics  │   │ • Join Receipt │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                    │
        │                   │                    │
        ▼                   ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ RECEIPT ITEMS │   │  PARTICIPANTS │   │  OCR SERVICE  │
│    SERVICE    │   │    SERVICE    │   │               │
│               │   │               │   │ • Scan Image  │
│ • CRUD Items  │   │ • Add/Remove  │   │ • Extract Data│
│ • Assign Items│   │ • Payment     │   │ • Validate     │
│ • Quantities  │   │ • Calculations│   │ • Vendor Info │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                    │
        │                   │                    │
        └───────────────────┼────────────────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │  CALCULATION SERVICE  │
                │                       │
                │ • Tip Calculations    │
                │ • Tax Calculations    │
                │ • Split Calculations  │
                │ • Totals              │
                └───────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    SUPPORTING SERVICES                          │
└─────────────────────────────────────────────────────────────────┘

┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ NOTIFICATION │   │ FILE STORAGE   │   │  REAL-TIME     │
│   SERVICE    │   │    SERVICE     │   │    SERVICE     │
│              │   │                │   │                │
│ • SMS        │   │ • Upload       │   │ • WebSocket    │
│ • Email      │   │ • Retrieve     │   │ • Rooms         │
│ • Push       │   │ • Delete       │   │ • Broadcasting │
│ • Templates  │   │ • Resize       │   │ • Events       │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                    │
        │                   │                    │
        ▼                   ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   ANALYTICS   │   │  MESSAGE QUEUE│   │   DATABASES    │
│   SERVICE     │   │  (RabbitMQ/   │   │  (MongoDB per │
│               │   │   Kafka)      │   │   service)     │
│ • Statistics  │   │               │   │                │
│ • Reports     │   │ • Events      │   │ • Auth DB      │
│ • Dashboard   │   │ • Queues      │   │ • User DB      │
│ • Aggregation │   │ • Pub/Sub      │   │ • Receipt DB  │
└───────────────┘   └───────────────┘   └───────────────┘
```

## Service Interaction Patterns

### 1. Receipt Creation Flow

```
Client App
    │
    ▼
API Gateway
    │
    ▼
Receipt Service ──┐
    │             │
    ▼             │
OCR Service       │
    │             │
    ▼             │
File Storage ─────┼──► S3
    │             │
    ▼             │
Receipt Service ◄─┘
    │
    ├──► Receipt Items Service (create items)
    │
    ├──► Receipt Participants Service (add creator)
    │
    ├──► Real-time Service (broadcast)
    │
    └──► Notification Service (notify)
```

### 2. Payment Confirmation Flow

```
Client App
    │
    ▼
API Gateway
    │
    ▼
Receipt Participants Service
    │
    ├──► Calculation Service (calculate total)
    │
    ├──► Notification Service (send email)
    │
    ├──► Real-time Service (broadcast update)
    │
    └──► Analytics Service (update stats)
```

### 3. Real-time Update Flow

```
Receipt Items Service (item assigned)
    │
    ▼
Message Queue (event: item.assigned)
    │
    ├──► Real-time Service ──► WebSocket ──► Clients
    │
    └──► Analytics Service (update stats)
```

## Database Schema per Service

### Authentication Service Database
```javascript
{
  users: {
    _id: ObjectId,
    email: String,
    passwordHash: String,
    tokens: [{
      token: String,
      createdAt: Date
    }],
    verificationCodes: {
      email: String,
      phone: String,
      resetPassword: String
    }
  }
}
```

### User Service Database
```javascript
{
  profiles: {
    _id: ObjectId,
    userId: ObjectId, // Reference to auth service
    firstName: String,
    lastName: String,
    profilePicture: String,
    phoneNumber: String,
    location: {
      lat: Number,
      lng: Number,
      detailed: Object
    },
    preferences: {
      paymentMethods: Array,
      cashappCashtag: String
    }
  }
}
```

### Receipt Service Database
```javascript
{
  receipts: {
    _id: ObjectId,
    receiptCode: Number,
    receiptName: String,
    receiptDate: Date,
    receiptTotal: Number,
    receiptSubtotal: Number,
    receiptTax: Number,
    receiptTip: Number,
    receiptCreator: ObjectId, // User ID
    receiptImage: String, // File URL
    vendorDetails: Object,
    receiptManuallyCreatedParticipants: [ObjectId],
    isVerified: Boolean,
    createdAt: Date
  }
}
```

### Receipt Items Service Database
```javascript
{
  receiptMenu: {
    _id: ObjectId,
    receiptNumber: ObjectId, // Receipt ID
    itemName: String,
    itemPrice: Number,
    itemQuantity: Number,
    itemType: String,
    userIds: [{
      id: ObjectId, // User ID
      quantity: Number
    }],
    createdAt: Date
  }
}
```

### Receipt Participants Service Database
```javascript
{
  receiptParticipants: {
    _id: ObjectId,
    receiptNumber: ObjectId, // Receipt ID
    userId: ObjectId, // User ID
    sendStatus: Boolean,
    confirmPayment: Boolean,
    createdAt: Date
  }
}
```

## Event-Driven Architecture

### Event Types

#### Receipt Events
- `receipt.created`
- `receipt.updated`
- `receipt.deleted`
- `receipt.joined`

#### Item Events
- `item.created`
- `item.updated`
- `item.deleted`
- `item.assigned`

#### Participant Events
- `participant.added`
- `participant.removed`
- `participant.payment.confirmed`
- `participant.payment.sent`

#### User Events
- `user.created`
- `user.updated`
- `user.deleted`

### Event Flow Example

```
Receipt Items Service
    │
    │ (item.assigned event)
    ▼
Message Queue
    │
    ├──► Real-time Service ──► Broadcast to clients
    │
    ├──► Analytics Service ──► Update statistics
    │
    └──► Notification Service ──► Notify participants
```

## API Gateway Routing

### Route Mapping

```
/api/auth/*          → Authentication Service
/api/users/*         → User Service
/api/receipts/*      → Receipt Service
/api/receipts/:id/items/* → Receipt Items Service
/api/receipts/:id/participants/* → Receipt Participants Service
/api/ocr/*           → OCR Service
/api/calculate/*     → Calculation Service
/api/notifications/* → Notification Service
/api/files/*         → File Storage Service
/api/analytics/*     → Analytics Service
/ws/*                → Real-time Service (WebSocket)
```

## Service Dependencies Graph

```
API Gateway
    ├──► Authentication Service
    │       └──► Notification Service (verification)
    │
    ├──► User Service
    │       ├──► Authentication Service (validate user)
    │       └──► File Storage Service (profile pictures)
    │
    ├──► Receipt Service
    │       ├──► Authentication Service (validate user)
    │       ├──► OCR Service (scan receipt)
    │       ├──► File Storage Service (receipt images)
    │       └──► Notification Service (notify participants)
    │
    ├──► Receipt Items Service
    │       ├──► Receipt Service (validate receipt)
    │       ├──► User Service (validate participants)
    │       └──► Real-time Service (broadcast updates)
    │
    ├──► Receipt Participants Service
    │       ├──► Receipt Service (validate receipt)
    │       ├──► Receipt Items Service (get items)
    │       ├──► Calculation Service (calculate totals)
    │       └──► Notification Service (send notifications)
    │
    ├──► OCR Service
    │       └──► File Storage Service (store images)
    │
    ├──► Calculation Service
    │       └──► (Stateless, no dependencies)
    │
    ├──► Notification Service
    │       └──► (External: Twilio, SendGrid, Firebase)
    │
    ├──► File Storage Service
    │       └──► (External: AWS S3)
    │
    ├──► Real-time Service
    │       ├──► Receipt Service (get receipt data)
    │       ├──► Receipt Items Service (get items)
    │       └──► Receipt Participants Service (get participants)
    │
    └──► Analytics Service
            └──► (Read-only access to all services)
```

## Technology Stack per Service

### Core Services (Node.js + Express)
- Authentication Service
- User Service
- Receipt Service
- Receipt Items Service
- Receipt Participants Service
- Calculation Service
- Notification Service
- File Storage Service
- Real-time Service
- Analytics Service

### Specialized Services
- **OCR Service**: Node.js or Python (better ML support)
- **API Gateway**: Kong, AWS API Gateway, or NGINX
- **Message Queue**: RabbitMQ or Apache Kafka

## Deployment Architecture

### Containerization
- Each service in its own Docker container
- Kubernetes for orchestration
- Service mesh (Istio) for service-to-service communication

### Scaling Strategy
- **Stateless Services**: Horizontal scaling (Calculation, OCR)
- **Stateful Services**: Vertical scaling + replication (Database services)
- **Real-time Service**: Horizontal scaling with sticky sessions

### High Availability
- Multiple instances per service
- Load balancing
- Health checks
- Circuit breakers
- Retry mechanisms

## Monitoring & Observability

### Metrics
- Request rate per service
- Error rate
- Response time
- Database query performance
- Queue depth

### Logging
- Centralized logging (ELK Stack)
- Structured logging (JSON)
- Correlation IDs for request tracing

### Tracing
- Distributed tracing (Jaeger/Zipkin)
- Request flow visualization
- Performance bottleneck identification

## Security Considerations

### Authentication
- JWT tokens issued by Authentication Service
- Token validation at API Gateway
- Service-to-service authentication (mTLS)

### Authorization
- Role-based access control (RBAC)
- Service-level permissions
- API Gateway enforces policies

### Data Protection
- Encryption at rest (database)
- Encryption in transit (TLS)
- PII data handling compliance

## Cost Optimization

### Resource Allocation
- Right-size each service based on load
- Auto-scaling based on metrics
- Reserved instances for stable services

### Database Optimization
- Separate read replicas
- Caching layer (Redis)
- Database connection pooling

## Migration Timeline

### Phase 1 (Month 1-2): Foundation
- Set up infrastructure (Docker, Kubernetes)
- Deploy API Gateway
- Extract Calculation Service
- Extract File Storage Service

### Phase 2 (Month 3-4): Core Services
- Extract Authentication Service
- Extract User Service
- Extract OCR Service
- Set up Message Queue

### Phase 3 (Month 5-6): Receipt Services
- Extract Receipt Service
- Extract Receipt Items Service
- Extract Receipt Participants Service
- Extract Real-time Service

### Phase 4 (Month 7-8): Supporting Services
- Extract Notification Service
- Extract Analytics Service
- Set up monitoring and logging
- Performance optimization

### Phase 5 (Month 9+): Optimization
- Fine-tune services
- Optimize database queries
- Implement caching
- Load testing and optimization
