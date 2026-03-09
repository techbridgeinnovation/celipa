# Celipa Server Architecture Analysis & Microservices Design

## Current Architecture Overview

### Technology Stack
- **Runtime**: Node.js with Express.js
- **Database**: MongoDB (Mongoose ODM)
- **Real-time**: Socket.io
- **File Storage**: AWS S3
- **OCR Services**: AWS Textract, Veryfi SDK
- **Communication**: Twilio (SMS), SendGrid (Email)
- **Authentication**: JWT tokens
- **File Upload**: express-fileupload

### Current Monolithic Structure

```
celipa-server/
├── app.js              # Express app configuration
├── server.js           # HTTP server + Socket.io initialization
├── src/
│   ├── controller/    # Request handlers (6 controllers)
│   ├── services/      # Business logic layer (4 services)
│   ├── db/            # Database models (6 models)
│   ├── routes/        # API route definitions (4 route files)
│   ├── middleware/    # Auth middleware
│   ├── helpers/       # Utility functions & integrations
│   └── utils/         # Calculation utilities
```

### Current Components Breakdown

#### 1. **Controllers** (Request Handlers)
- `auth.controller.js` - Authentication & authorization
- `user.controller.js` - User profile management
- `receipt.controller.js` - Receipt CRUD operations (largest controller)
- `dashboard.controller.js` - Admin dashboard operations
- `incident.controller.js` - Support/incident management
- `global.controller.js` - Global utilities (Veryfi credentials)

#### 2. **Services** (Business Logic)
- `receipt.service.js` - Receipt business logic (697 lines)
- `user.service.js` - User business logic
- `dashboard.service.js` - Analytics & statistics
- `incident.service.js` - Incident management

#### 3. **Database Models**
- `user.model.js` - User accounts
- `receipt.model.js` - Receipts
- `receiptmenu.model.js` - Receipt items/menu
- `receiptparticipants.model.js` - Receipt participants
- `incident.model.js` - Support incidents
- `bills.models.js` - (Unused?)

#### 4. **External Integrations**
- **AWS S3** - File storage (receipt images, profile pictures)
- **AWS Textract** - OCR for receipt scanning
- **Veryfi SDK** - Alternative OCR service
- **Twilio** - SMS notifications
- **SendGrid** - Email notifications
- **Firebase** - Push notifications
- **Socket.io** - Real-time updates

#### 5. **Key Features**
- Receipt creation & scanning (OCR)
- Receipt splitting & item assignment
- Real-time collaboration (Socket.io)
- Payment tracking & confirmation
- User authentication (JWT)
- Admin dashboard
- Notification system (SMS, Email, Push)
- Analytics & reporting

### Current Architecture Issues

1. **Tight Coupling**: All services share the same database connection
2. **Scalability**: Single point of failure, can't scale individual components
3. **Deployment**: Entire app must be redeployed for any change
4. **Technology Lock-in**: All services must use same Node.js version
5. **Resource Allocation**: Can't optimize resources per service
6. **Testing**: Difficult to test services in isolation
7. **Team Collaboration**: Multiple developers working on same codebase

---

## Microservices Architecture Proposal

### Proposed Microservices (8 Services)

#### 1. **API Gateway Service**
**Purpose**: Single entry point, routing, authentication, rate limiting

**Responsibilities**:
- Request routing to appropriate microservices
- JWT token validation
- Rate limiting & throttling
- CORS handling
- Request/response transformation
- Load balancing

**Technology**: 
- Node.js + Express or
- Kong, AWS API Gateway, or NGINX

**Endpoints**: All `/api/*` routes

---

#### 2. **Authentication Service**
**Purpose**: User authentication & authorization

**Responsibilities**:
- User registration
- Login (email/password, Google, Apple)
- JWT token generation & validation
- Password reset
- Phone number verification
- Email verification
- Session management
- User confirmation codes

**Database**: 
- User credentials (email, password hash)
- Tokens
- Verification codes

**Endpoints**:
- `POST /auth/register`
- `POST /auth/login`
- `POST /auth/verify-phone`
- `POST /auth/verify-email`
- `POST /auth/reset-password`
- `POST /auth/logout`
- `POST /auth/refresh-token`

**Dependencies**: 
- Twilio (SMS verification)
- SendGrid (Email verification)

---

#### 3. **User Service**
**Purpose**: User profile management

**Responsibilities**:
- User profile CRUD operations
- Profile picture upload
- User preferences (payment methods)
- Location tracking
- User search & filtering
- User statistics

**Database**:
- User profiles
- User preferences
- User locations

**Endpoints**:
- `GET /users/:id`
- `PATCH /users/:id`
- `GET /users/search`
- `POST /users/:id/profile-picture`
- `PUT /users/:id/payment-preferences`

**Dependencies**:
- File Storage Service (profile pictures)
- Authentication Service (user validation)

---

#### 4. **Receipt Service**
**Purpose**: Core receipt management

**Responsibilities**:
- Receipt creation
- Receipt CRUD operations
- Receipt code generation
- Receipt metadata management
- Receipt search & filtering
- Receipt statistics

**Database**:
- Receipts collection
- Receipt metadata

**Endpoints**:
- `POST /receipts`
- `GET /receipts/:id`
- `PUT /receipts/:id`
- `DELETE /receipts/:id`
- `GET /receipts/user/:userId`
- `GET /receipts/search`
- `POST /receipts/:id/join`

**Dependencies**:
- OCR Service (receipt scanning)
- File Storage Service (receipt images)
- User Service (user validation)
- Notification Service (receipt updates)

---

#### 5. **Receipt Items Service**
**Purpose**: Receipt items & menu management

**Responsibilities**:
- Menu item CRUD operations
- Item assignment to participants
- Item quantity management
- Item price calculations
- Receipt subtotal calculations

**Database**:
- ReceiptMenu collection
- Item assignments

**Endpoints**:
- `POST /receipts/:receiptId/items`
- `GET /receipts/:receiptId/items`
- `PUT /receipts/:receiptId/items/:itemId`
- `DELETE /receipts/:receiptId/items/:itemId`
- `POST /receipts/:receiptId/items/:itemId/assign`
- `GET /receipts/:receiptId/items/participants/:userId`

**Dependencies**:
- Receipt Service (receipt validation)
- User Service (participant validation)
- Real-time Service (live updates)

---

#### 6. **Receipt Participants Service**
**Purpose**: Participant management & payment tracking

**Responsibilities**:
- Add/remove participants
- Payment status tracking
- Payment confirmation
- Participant calculations (tax, tip, total)
- Receipt splitting logic

**Database**:
- ReceiptParticipants collection
- Payment status

**Endpoints**:
- `POST /receipts/:receiptId/participants`
- `GET /receipts/:receiptId/participants`
- `DELETE /receipts/:receiptId/participants/:userId`
- `PUT /receipts/:receiptId/participants/:userId/payment-status`
- `GET /receipts/:receiptId/participants/:userId/calculation`
- `POST /receipts/:receiptId/participants/:userId/confirm-payment`

**Dependencies**:
- Receipt Service
- Receipt Items Service
- Calculation Service
- Notification Service
- Payment Service

---

#### 7. **OCR Service**
**Purpose**: Receipt scanning & data extraction

**Responsibilities**:
- Image upload & processing
- OCR processing (AWS Textract, Veryfi)
- Receipt data extraction
- Receipt validation
- Vendor information extraction

**Endpoints**:
- `POST /ocr/scan`
- `POST /ocr/analyze`
- `GET /ocr/credentials` (Veryfi)

**Dependencies**:
- AWS Textract
- Veryfi SDK
- File Storage Service

**Technology**: 
- Node.js (can be Python for better ML/OCR support)

---

#### 8. **Calculation Service**
**Purpose**: Financial calculations

**Responsibilities**:
- Tip calculations (percentage/dollar)
- Tax calculations
- Participant total calculations
- Receipt total calculations
- Split calculations

**Endpoints**:
- `POST /calculate/tip`
- `POST /calculate/tax`
- `POST /calculate/participant-total`
- `POST /calculate/receipt-total`
- `POST /calculate/split`

**Dependencies**: None (stateless service)

**Technology**: Node.js (can be optimized with caching)

---

#### 9. **Notification Service**
**Purpose**: Multi-channel notifications

**Responsibilities**:
- SMS notifications (Twilio)
- Email notifications (SendGrid)
- Push notifications (Firebase/Expo)
- Notification templates
- Notification queuing
- Delivery tracking

**Endpoints**:
- `POST /notifications/sms`
- `POST /notifications/email`
- `POST /notifications/push`
- `POST /notifications/batch`
- `GET /notifications/:id/status`

**Dependencies**:
- Twilio
- SendGrid
- Firebase
- Message Queue (RabbitMQ/Kafka)

---

#### 10. **File Storage Service**
**Purpose**: File upload & management

**Responsibilities**:
- File upload (S3)
- File retrieval
- Image processing/resizing
- File deletion
- CDN integration

**Endpoints**:
- `POST /files/upload`
- `GET /files/:fileId`
- `DELETE /files/:fileId`
- `POST /files/resize`

**Dependencies**:
- AWS S3
- Image processing library

---

#### 11. **Real-time Service**
**Purpose**: WebSocket connections & real-time updates

**Responsibilities**:
- Socket.io connection management
- Room management (receipt rooms)
- Real-time event broadcasting
- Connection state management

**Endpoints**:
- WebSocket connections
- Socket events: `join_room`, `message`, `receipt_update`

**Dependencies**:
- Receipt Service
- Receipt Items Service
- Receipt Participants Service

**Technology**: Node.js + Socket.io

---

#### 12. **Analytics Service**
**Purpose**: Dashboard & reporting

**Responsibilities**:
- User statistics
- Receipt statistics
- Transaction analytics
- Weekly/monthly reports
- Dashboard data aggregation

**Endpoints**:
- `GET /analytics/users`
- `GET /analytics/receipts`
- `GET /analytics/transactions`
- `GET /analytics/dashboard`
- `GET /analytics/weekly`

**Dependencies**:
- All other services (read-only)
- Time-series database (optional)

---

#### 13. **Payment Service** (Future)
**Purpose**: Payment processing integration

**Responsibilities**:
- Venmo integration
- CashApp integration
- Payment link generation
- Payment status tracking

**Endpoints**:
- `POST /payments/venmo/request`
- `POST /payments/cashapp/request`
- `GET /payments/:id/status`

---

### Supporting Infrastructure

#### 1. **Message Queue** (RabbitMQ/Kafka)
- Async communication between services
- Event-driven architecture
- Notification queuing
- Receipt update events

#### 2. **Service Discovery** (Consul/Eureka)
- Service registration & discovery
- Health checks
- Load balancing

#### 3. **API Gateway** (Kong/AWS API Gateway)
- Single entry point
- Authentication
- Rate limiting
- Request routing

#### 4. **Database per Service**
- Each service has its own database
- Data consistency via events
- Service autonomy

#### 5. **Event Bus** (RabbitMQ/Kafka)
- Event-driven communication
- Receipt updates
- Payment confirmations
- User updates

---

## Microservices Communication Patterns

### 1. **Synchronous Communication** (HTTP/REST)
- Direct service-to-service calls
- Used for: User validation, receipt validation
- API Gateway routes requests

### 2. **Asynchronous Communication** (Message Queue)
- Event-driven architecture
- Used for: Notifications, receipt updates, analytics
- Events: `receipt.created`, `payment.confirmed`, `user.updated`

### 3. **Real-time Communication** (WebSocket)
- Socket.io for live updates
- Real-time Service manages connections
- Broadcasts receipt changes

---

## Data Flow Examples

### Receipt Creation Flow:
1. Client → API Gateway → Receipt Service
2. Receipt Service → OCR Service (if image provided)
3. OCR Service → File Storage Service (upload image)
4. OCR Service → Receipt Service (extracted data)
5. Receipt Service → Receipt Items Service (create items)
6. Receipt Service → Receipt Participants Service (add creator)
7. Receipt Service → Real-time Service (broadcast update)
8. Real-time Service → Notification Service (notify participants)

### Payment Confirmation Flow:
1. Client → API Gateway → Receipt Participants Service
2. Receipt Participants Service → Calculation Service (calculate total)
3. Receipt Participants Service → Notification Service (send email)
4. Receipt Participants Service → Real-time Service (broadcast update)
5. Receipt Participants Service → Analytics Service (update stats)

---

## Migration Strategy

### Phase 1: Extract Services (Low Risk)
1. Calculation Service (stateless, easy to extract)
2. File Storage Service (isolated functionality)
3. Notification Service (independent)

### Phase 2: Core Services
4. Authentication Service
5. User Service
6. OCR Service

### Phase 3: Receipt Services
7. Receipt Service
8. Receipt Items Service
9. Receipt Participants Service

### Phase 4: Supporting Services
10. Real-time Service
11. Analytics Service

### Phase 5: Infrastructure
12. API Gateway
13. Message Queue
14. Service Discovery

---

## Benefits of Microservices Architecture

1. **Scalability**: Scale individual services based on load
2. **Technology Diversity**: Use best tool for each service
3. **Team Autonomy**: Teams can work independently
4. **Fault Isolation**: Failure in one service doesn't crash entire system
5. **Deployment**: Deploy services independently
6. **Testing**: Test services in isolation
7. **Resource Optimization**: Allocate resources per service needs

---

## Challenges & Solutions

### Challenge 1: Data Consistency
**Solution**: Event-driven architecture, eventual consistency

### Challenge 2: Service Communication
**Solution**: API Gateway, Message Queue, Service Discovery

### Challenge 3: Distributed Transactions
**Solution**: Saga pattern, event sourcing

### Challenge 4: Monitoring & Debugging
**Solution**: Centralized logging (ELK), distributed tracing (Jaeger)

### Challenge 5: Deployment Complexity
**Solution**: Containerization (Docker), Orchestration (Kubernetes)

---

## Technology Recommendations

- **Containerization**: Docker
- **Orchestration**: Kubernetes
- **API Gateway**: Kong or AWS API Gateway
- **Message Queue**: RabbitMQ or Apache Kafka
- **Service Discovery**: Consul or Kubernetes Service Discovery
- **Monitoring**: Prometheus + Grafana
- **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana)
- **Tracing**: Jaeger or Zipkin
- **Database**: MongoDB (per service) or PostgreSQL for transactional data

---

## Estimated Service Count: **13 Microservices**

1. API Gateway
2. Authentication Service
3. User Service
4. Receipt Service
5. Receipt Items Service
6. Receipt Participants Service
7. OCR Service
8. Calculation Service
9. Notification Service
10. File Storage Service
11. Real-time Service
12. Analytics Service
13. Payment Service (Future)

---

## Next Steps

1. **Proof of Concept**: Start with Calculation Service (easiest)
2. **Infrastructure Setup**: Docker, Kubernetes, Message Queue
3. **API Gateway**: Set up routing
4. **Gradual Migration**: Move services one by one
5. **Monitoring**: Set up logging and monitoring
6. **Testing**: Integration tests for service communication
7. **Documentation**: API documentation for each service
