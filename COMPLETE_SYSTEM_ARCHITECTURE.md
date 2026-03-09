# Complete Celipa System Architecture
## Mobile App + Microservices Integration

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    CELIPA ECOSYSTEM                              │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  Mobile App      │  │  Web Dashboard   │  │  Marketing Site  │
│  (React Native)  │  │  (React)         │  │  (React)         │
└────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                               │ HTTPS/WSS
                               │
                    ┌──────────▼──────────┐
                    │   API Gateway       │
                    │   (Kong/AWS)        │
                    │                     │
                    │ - Routing           │
                    │ - Auth (JWT)        │
                    │ - Rate Limiting     │
                    │ - CORS              │
                    └──────────┬──────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
    ┌─────────┐          ┌─────────┐          ┌─────────┐
    │ Auth    │          │ User    │          │ Receipt │
    │ Service │          │ Service │          │ Service │
    └─────────┘          └─────────┘          └─────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
    ┌─────────┐          ┌─────────┐          ┌─────────┐
    │ Receipt │          │ Receipt │          │   OCR   │
    │ Items   │          │ Partici-│          │ Service │
    │ Service │          │ pants   │          │         │
    └─────────┘          └─────────┘          └─────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Real-time Service   │
                    │  (Socket.io)        │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Message Queue      │
                    │  (RabbitMQ/Kafka)   │
                    └─────────────────────┘
```

## Complete Receipt Flow - End to End

### 1. Receipt Scanning & Creation

```
┌─────────────────────────────────────────────────────────────┐
│                    MOBILE APP                                │
└─────────────────────────────────────────────────────────────┘

User Action: "Scan Receipt"
    │
    ▼
┌──────────────────┐
│ ReceiptPicture   │
│   Screen         │
│                  │
│ 1. Camera        │
│ 2. Take Photo    │
│ 3. Compress      │
└────────┬─────────┘
         │
         │ Upload Image
         ▼
┌──────────────────┐
│  OCR Service      │
│  (Microservice)   │
│                  │
│ 1. Receive image  │
│ 2. Process (Veryfi│
│    or Textract)   │
│ 3. Extract data   │
│ 4. Return JSON    │
└────────┬─────────┘
         │
         │ Extracted Data
         ▼
┌──────────────────┐
│ UserReceipt      │
│   Screen         │
│                  │
│ 1. Display items │
│ 2. Edit items    │
│ 3. Edit tax      │
│ 4. Confirm       │
└────────┬─────────┘
         │
         │ POST /receipts
         ▼
┌──────────────────┐
│ Receipt Service  │
│  (Microservice)   │
│                  │
│ 1. Create receipt│
│ 2. Generate code │
│ 3. Store metadata│
└────────┬─────────┘
         │
         │ Create Items
         ▼
┌──────────────────┐
│ Receipt Items    │
│   Service        │
│                  │
│ 1. Create items  │
│ 2. Store in DB   │
└────────┬─────────┘
         │
         │ Add Creator
         ▼
┌──────────────────┐
│ Receipt Partici- │
│   pants Service  │
│                  │
│ 1. Add creator   │
│ 2. Set as leader │
└────────┬─────────┘
         │
         │ Upload Image
         ▼
┌──────────────────┐
│ File Storage     │
│   Service        │
│                  │
│ 1. Upload to S3  │
│ 2. Return URL    │
└────────┬─────────┘
         │
         │ Emit Event
         ▼
┌──────────────────┐
│ Message Queue    │
│                  │
│ Event:           │
│ receipt.created  │
└────────┬─────────┘
         │
         │ Broadcast
         ▼
┌──────────────────┐
│ Real-time Service│
│                  │
│ Broadcast to:    │
│ - Creator        │
└────────┬─────────┘
         │
         │ Navigate
         ▼
┌──────────────────┐
│ VirtualReceipt   │
│   Screen         │
│                  │
│ 1. Display items │
│ 2. Socket.io     │
│    connected     │
│ 3. Ready for     │
│    collaboration │
└──────────────────┘
```

### 2. Item Assignment Flow

```
┌──────────────────┐
│ VirtualReceipt   │
│   Screen         │
└────────┬─────────┘
         │
         │ User taps item
         ▼
┌──────────────────┐
│ assignItem()     │
│                  │
│ Check:           │
│ - Quantity       │
│ - User role      │
└────────┬─────────┘
         │
         │ PUT /receipts/:id/items/:itemId/assign
         ▼
┌──────────────────┐
│ Receipt Items    │
│   Service        │
│                  │
│ 1. Validate      │
│ 2. Update DB     │
│ 3. Emit event    │
└────────┬─────────┘
         │
         │ Event: item.assigned
         ▼
┌──────────────────┐
│ Message Queue    │
│                  │
│ Queue event      │
└────────┬─────────┘
         │
         │ Consume event
         ▼
┌──────────────────┐
│ Real-time Service│
│                  │
│ 1. Get updated   │
│    receipt data  │
│ 2. Broadcast to  │
│    all clients   │
└────────┬─────────┘
         │
         │ Socket.io event
         ▼
┌──────────────────┐
│ All Clients      │
│ (VirtualReceipt) │
│                  │
│ 1. Receive update│
│ 2. Update state  │
│ 3. Re-render UI  │
└──────────────────┘
```

### 3. Payment Confirmation Flow

```
┌──────────────────┐
│ VirtualReceipt   │
│   Screen         │
└────────┬─────────┘
         │
         │ User confirms payment
         ▼
┌──────────────────┐
│ AcceptPayment    │
│   Modal          │
│                  │
│ 1. Show total    │
│ 2. Show items    │
│ 3. Confirm       │
└────────┬─────────┘
         │
         │ User selects payment method
         ▼
┌──────────────────┐
│ PaymentMethods   │
│   Modal          │
│                  │
│ Select: Venmo,   │
│ CashApp, etc.    │
└────────┬─────────┘
         │
         │ PUT /receipts/:id/participants/:userId/confirm
         ▼
┌──────────────────┐
│ Receipt Partici- │
│   pants Service  │
│                  │
│ 1. Update status │
│ 2. Calculate     │
│    total         │
│ 3. Emit event    │
└────────┬─────────┘
         │
         ├──► Calculation Service (calculate total)
         │
         ├──► Notification Service (send email)
         │
         └──► Message Queue (payment.confirmed event)
                 │
                 ▼
         ┌──────────────────┐
         │ Real-time Service│
         │                  │
         │ Broadcast update │
         └────────┬─────────┘
                  │
                  ▼
         ┌──────────────────┐
         │ All Clients      │
         │                  │
         │ Update UI        │
         └──────────────────┘
```

## Data Flow - VirtualReceipt Screen

```
┌─────────────────────────────────────────────────────────────┐
│              VirtualReceipt Screen Lifecycle                │
└─────────────────────────────────────────────────────────────┘

1. Screen Mount
    │
    ├──► Initialize Socket.io connection
    │    └──► Connect to Real-time Service
    │
    ├──► Fetch Receipt Data
    │    ├──► GET /receipts/:id (Receipt Service)
    │    ├──► GET /receipts/:id/items (Receipt Items Service)
    │    ├──► GET /receipts/:id/participants (Receipt Participants Service)
    │    └──► GET /receipts/:id/participants/items (All items)
    │
    └──► Set up Socket.io Listeners
         ├──► socket.on(receiptId) - Receipt updates
         └──► socket.on(`${receiptId}pu`) - Payment updates

2. User Interactions
    │
    ├──► Assign Item
    │    ├──► PUT /receipts/:id/items/:itemId/assign
    │    ├──► Receipt Items Service updates DB
    │    ├──► Emits event to Message Queue
    │    ├──► Real-time Service broadcasts
    │    └──► All clients receive update
    │
    ├──► Add Participant
    │    ├──► POST /receipts/:id/participants
    │    ├──► Receipt Participants Service adds user
    │    ├──► Notification Service sends SMS/Email
    │    ├──► Emits event to Message Queue
    │    └──► Real-time Service broadcasts
    │
    ├──► Edit Tip/Tax (Creator only)
    │    ├──► POST /receipts/:id/update
    │    ├──► Receipt Service updates
    │    ├──► Calculation Service recalculates
    │    ├──► Emits event to Message Queue
    │    └──► Real-time Service broadcasts
    │
    └──► Confirm Payment
         ├──► PUT /receipts/:id/participants/:userId/confirm
         ├──► Receipt Participants Service updates
         ├──► Notification Service sends email
         ├──► Emits event to Message Queue
         └──► Real-time Service broadcasts

3. Real-time Updates
    │
    ├──► Receipt Update Received
    │    ├──► Update menuItems state
    │    ├──► Update participants state
    │    ├──► Update receipt state
    │    ├──► Recalculate totals
    │    └──► Re-render UI
    │
    └──► Payment Update Received
         ├──► Update payment statuses
         ├──► Update UI indicators
         └──► Show notifications

4. Screen Unmount
    │
    └──► Disconnect Socket.io
```

## API Call Mapping - Current to Microservices

### Authentication

**Current:**
```javascript
POST /user/login
POST /user/signup
POST /user/confirm
```

**Microservices:**
```javascript
POST /auth/login          → Authentication Service
POST /auth/register       → Authentication Service
POST /auth/verify         → Authentication Service
```

### User Management

**Current:**
```javascript
GET /user/:id
PATCH /user/profile
PUT /user/payment/preferred
```

**Microservices:**
```javascript
GET /users/:id            → User Service
PATCH /users/:id          → User Service
PUT /users/:id/payment    → User Service
```

### Receipt Operations

**Current:**
```javascript
POST /receipt/create
GET /current/user/:id
GET /receipt/join/:userId/:code
GET /user/receipt/:id
```

**Microservices:**
```javascript
POST /receipts            → Receipt Service
GET /receipts/:id        → Receipt Service
GET /receipts/join/:code → Receipt Service
GET /receipts/user/:id   → Receipt Service
```

### Receipt Items

**Current:**
```javascript
GET /user/menuitems/:receiptId
PUT /menu/addparticipant/:userId/:menuId/:receiptId/:quantity
PUT /receipt/update/item/:itemId/:receiptId
DELETE /receipt/item/:itemId/:receiptId
```

**Microservices:**
```javascript
GET /receipts/:id/items                    → Receipt Items Service
POST /receipts/:id/items/:itemId/assign    → Receipt Items Service
PUT /receipts/:id/items/:itemId            → Receipt Items Service
DELETE /receipts/:id/items/:itemId        → Receipt Items Service
```

### Participants

**Current:**
```javascript
GET /user/participants/:receiptId
PUT /receipt/add
PUT /receipt/status/:userId/:receiptId
PUT /receipt/confirmStatus/:userId/:receiptId
POST /receipt/participants/localparticipants/:receiptId
```

**Microservices:**
```javascript
GET /receipts/:id/participants                    → Receipt Participants Service
POST /receipts/:id/participants                   → Receipt Participants Service
PUT /receipts/:id/participants/:userId/status     → Receipt Participants Service
PUT /receipts/:id/participants/:userId/confirm     → Receipt Participants Service
POST /receipts/:id/participants/contacts          → Receipt Participants Service
```

### OCR

**Current:**
```javascript
GET /global/credentials
(Direct Veryfi API call)
```

**Microservices:**
```javascript
GET /ocr/credentials     → OCR Service
POST /ocr/scan           → OCR Service
```

## Socket.io Event Mapping

### Current Events

**Client Receives:**
- `receiptId` - Full receipt update
- `${receiptId}pu` - Payment updates

**Server Emits (via API triggers):**
- Receipt changes trigger broadcasts
- Payment changes trigger broadcasts

### Microservices Events

**Event Types:**
- `receipt.created`
- `receipt.updated`
- `item.assigned`
- `item.updated`
- `participant.added`
- `participant.removed`
- `payment.confirmed`
- `payment.sent`
- `tip.updated`
- `tax.updated`

**Event Flow:**
```
Service Action
    │
    ▼
Update Database
    │
    ▼
Emit Event to Message Queue
    │
    ▼
Real-time Service Consumes Event
    │
    ▼
Broadcast to Connected Clients
    │
    ▼
Mobile App Updates State
    │
    ▼
UI Re-renders
```

## State Management Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Redux Store                              │
└─────────────────────────────────────────────────────────────┘

User Action
    │
    ▼
Component Dispatch Action
    │
    ▼
Redux Action Creator
    │
    ▼
API Call (via api.js)
    │
    ▼
Microservice (via API Gateway)
    │
    ▼
Response
    │
    ▼
Redux Reducer Updates State
    │
    ▼
Component Re-renders
    │
    ▼
UI Updates
```

## Error Handling Flow

```
API Call
    │
    ├──► Success
    │    └──► Update State
    │
    └──► Error
         │
         ├──► Network Error
         │    └──► Show offline message
         │
         ├──► 401 Unauthorized
         │    └──► Clear token, redirect to login
         │
         ├──► 404 Not Found
         │    └──► Show error message
         │
         └──► 500 Server Error
              └──► Show error, retry option
```

## Caching Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    Caching Layers                           │
└─────────────────────────────────────────────────────────────┘

1. Redux Store (In-Memory)
   - Current user
   - Current receipt
   - Recent receipts

2. Redux Persist (AsyncStorage)
   - User token
   - User profile
   - Preferences

3. Component State
   - UI state
   - Form data
   - Temporary data

4. API Response Cache (Future)
   - Receipt data
   - User data
   - Menu items
```

## Performance Optimizations

### Current
- Image compression
- Lazy loading
- Redux memoization
- Socket.io connection reuse

### Microservices Enhancements
- API response caching
- Request batching
- GraphQL for complex queries
- CDN for images
- Service worker for offline

## Security Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Security Layers                           │
└─────────────────────────────────────────────────────────────┘

1. App Level
   - JWT token in AsyncStorage
   - Secure storage (Keychain/Keystore)
   - Certificate pinning

2. API Gateway
   - JWT validation
   - Rate limiting
   - CORS policies

3. Service Level
   - Service-to-service auth (mTLS)
   - Role-based access control
   - Data encryption

4. Database Level
   - Encryption at rest
   - Access controls
   - Audit logging
```

## Monitoring & Observability

```
┌─────────────────────────────────────────────────────────────┐
│                    Monitoring Stack                          │
└─────────────────────────────────────────────────────────────┘

Mobile App
    │
    ├──► Analytics (Mixpanel)
    ├──► Crash Reporting (Sentry)
    └──► Performance Monitoring

API Gateway
    │
    ├──► Request Logging
    ├──► Error Tracking
    └──► Performance Metrics

Microservices
    │
    ├──► Health Checks
    ├──► Metrics (Prometheus)
    ├──► Logging (ELK Stack)
    └──► Tracing (Jaeger)

Message Queue
    │
    ├──► Queue Depth
    ├──► Message Rates
    └──► Error Tracking
```

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Deployment Stack                          │
└─────────────────────────────────────────────────────────────┘

Mobile App
    │
    ├──► Expo Build Service
    ├──► App Store / Play Store
    └──► OTA Updates (Expo Updates)

API Gateway
    │
    ├──► Kubernetes
    ├──► Load Balancer
    └──► Auto-scaling

Microservices
    │
    ├──► Docker Containers
    ├──► Kubernetes Pods
    ├──► Service Mesh (Istio)
    └──► Auto-scaling

Databases
    │
    ├──► MongoDB (per service)
    ├──► Read Replicas
    └──► Backup Strategy

Message Queue
    │
    ├──► RabbitMQ Cluster
    └──► High Availability
```

## Summary

### Current Architecture
- **Monolithic server** handling all operations
- **Single database** for all data
- **Direct Socket.io** connection to server
- **Simple API structure**

### Microservices Architecture
- **13 specialized services** handling specific domains
- **Database per service** for data isolation
- **Dedicated Real-time Service** for WebSocket connections
- **Message Queue** for event-driven communication
- **API Gateway** for unified entry point

### App Changes Required
1. **Minimal changes** - API Gateway handles routing
2. **Socket.io connection** - Point to Real-time Service
3. **Error handling** - Service-specific errors
4. **Caching** - Enhanced caching strategy
5. **Offline support** - Better offline capabilities

### Benefits
- **Scalability** - Scale services independently
- **Reliability** - Fault isolation
- **Maintainability** - Clear service boundaries
- **Performance** - Optimize each service
- **Team autonomy** - Independent development

The mobile app is well-architected and can seamlessly integrate with a microservices backend with minimal changes, primarily in the API service layer and Socket.io connection management.
