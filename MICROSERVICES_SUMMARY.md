# Celipa Server: Current Architecture & Microservices Proposal - Summary

## Current Architecture (Monolithic)

### Structure
- **Single Node.js/Express Application**
- **MongoDB Database** (shared across all features)
- **Socket.io** for real-time updates
- **6 Controllers**: Auth, User, Receipt, Dashboard, Incident, Global
- **4 Services**: Receipt, User, Dashboard, Incident
- **6 Models**: User, Receipt, ReceiptMenu, ReceiptParticipants, Incident, Bills

### Key Integrations
- AWS S3 (file storage)
- AWS Textract (OCR)
- Veryfi SDK (OCR alternative)
- Twilio (SMS)
- SendGrid (Email)
- Firebase (Push notifications)

### Current Issues
1. ❌ Tight coupling between components
2. ❌ Single point of failure
3. ❌ Can't scale individual features
4. ❌ Entire app redeploy for any change
5. ❌ Difficult to test in isolation
6. ❌ Technology lock-in

---

## Proposed Microservices Architecture

### **13 Microservices Total**

#### Core Business Services (6)
1. **Authentication Service** - Login, registration, JWT tokens
2. **User Service** - Profile management, preferences
3. **Receipt Service** - Receipt CRUD, metadata
4. **Receipt Items Service** - Menu items, assignments
5. **Receipt Participants Service** - Participants, payment tracking
6. **Calculation Service** - Financial calculations (stateless)

#### Supporting Services (4)
7. **OCR Service** - Receipt scanning, data extraction
8. **Notification Service** - SMS, Email, Push notifications
9. **File Storage Service** - File upload/download (S3)
10. **Real-time Service** - WebSocket connections, live updates

#### Infrastructure Services (3)
11. **API Gateway** - Routing, authentication, rate limiting
12. **Analytics Service** - Dashboard, statistics, reporting
13. **Payment Service** - Payment integrations (future)

---

## Service Breakdown by Responsibility

### Authentication Service
- User registration & login
- JWT token management
- Phone/email verification
- Password reset
- **Dependencies**: Twilio, SendGrid

### User Service
- User profiles
- Profile pictures
- Payment preferences
- Location tracking
- User search
- **Dependencies**: File Storage Service

### Receipt Service
- Create/read/update receipts
- Receipt codes
- Receipt search
- Receipt metadata
- **Dependencies**: OCR Service, File Storage, User Service

### Receipt Items Service
- Menu item CRUD
- Item assignments
- Quantity management
- **Dependencies**: Receipt Service, User Service, Real-time Service

### Receipt Participants Service
- Add/remove participants
- Payment status tracking
- Payment confirmations
- Split calculations
- **Dependencies**: Receipt Service, Receipt Items, Calculation, Notification

### Calculation Service
- Tip calculations
- Tax calculations
- Participant totals
- Split calculations
- **Dependencies**: None (stateless)

### OCR Service
- Image processing
- OCR (AWS Textract, Veryfi)
- Data extraction
- Vendor information
- **Dependencies**: File Storage Service

### Notification Service
- SMS (Twilio)
- Email (SendGrid)
- Push notifications (Firebase)
- Notification templates
- **Dependencies**: Twilio, SendGrid, Firebase

### File Storage Service
- File upload (S3)
- File retrieval
- Image processing
- **Dependencies**: AWS S3

### Real-time Service
- WebSocket connections
- Room management
- Event broadcasting
- **Dependencies**: Receipt, Items, Participants services

### Analytics Service
- User statistics
- Receipt statistics
- Dashboard data
- Reports
- **Dependencies**: Read-only access to all services

### API Gateway
- Request routing
- JWT validation
- Rate limiting
- CORS handling
- **Dependencies**: All services

---

## Communication Patterns

### Synchronous (HTTP/REST)
- Direct service-to-service calls
- Used for: Validation, immediate data retrieval
- Via API Gateway

### Asynchronous (Message Queue)
- Event-driven architecture
- Used for: Notifications, analytics, real-time updates
- Events: `receipt.created`, `payment.confirmed`, `item.assigned`

### Real-time (WebSocket)
- Live updates to clients
- Managed by Real-time Service
- Broadcasts receipt changes

---

## Infrastructure Components

### Required Infrastructure
1. **Containerization**: Docker
2. **Orchestration**: Kubernetes
3. **API Gateway**: Kong or AWS API Gateway
4. **Message Queue**: RabbitMQ or Apache Kafka
5. **Service Discovery**: Consul or Kubernetes Service Discovery
6. **Databases**: MongoDB per service (or PostgreSQL for transactional)
7. **Caching**: Redis (optional)
8. **Monitoring**: Prometheus + Grafana
9. **Logging**: ELK Stack
10. **Tracing**: Jaeger or Zipkin

---

## Benefits of Microservices

✅ **Scalability** - Scale services independently  
✅ **Technology Diversity** - Best tool for each service  
✅ **Team Autonomy** - Independent development  
✅ **Fault Isolation** - One service failure doesn't crash system  
✅ **Independent Deployment** - Deploy services separately  
✅ **Easier Testing** - Test services in isolation  
✅ **Resource Optimization** - Allocate resources per service needs  

---

## Migration Strategy

### Phase 1: Foundation (Month 1-2)
- Infrastructure setup (Docker, Kubernetes)
- API Gateway deployment
- Extract: Calculation Service, File Storage Service

### Phase 2: Core Services (Month 3-4)
- Extract: Authentication, User, OCR Services
- Set up Message Queue

### Phase 3: Receipt Services (Month 5-6)
- Extract: Receipt, Receipt Items, Receipt Participants, Real-time Services

### Phase 4: Supporting Services (Month 7-8)
- Extract: Notification, Analytics Services
- Set up monitoring and logging

### Phase 5: Optimization (Month 9+)
- Performance tuning
- Database optimization
- Caching implementation

---

## Key Metrics to Track

### Per Service
- Request rate
- Error rate
- Response time
- Database query performance
- Resource utilization

### System-wide
- End-to-end request latency
- Service availability
- Message queue depth
- Database connection pools

---

## Security Considerations

- **Authentication**: JWT tokens at API Gateway
- **Authorization**: RBAC per service
- **Service-to-Service**: mTLS
- **Data**: Encryption at rest and in transit
- **PII**: Compliance with data protection regulations

---

## Cost Considerations

### Optimization Strategies
- Right-size services based on actual load
- Auto-scaling based on metrics
- Reserved instances for stable services
- Separate read replicas for databases
- Caching layer to reduce database load

### Estimated Costs
- Infrastructure: Higher initially (containers, orchestration)
- Development: Higher (more services to maintain)
- Operations: Higher (more moving parts)
- **But**: Better scalability and fault tolerance

---

## Success Criteria

✅ Each service can be deployed independently  
✅ Services can scale independently  
✅ Failure in one service doesn't affect others  
✅ Services can be developed by different teams  
✅ Clear API contracts between services  
✅ Comprehensive monitoring and logging  
✅ High availability (99.9%+ uptime)  

---

## Next Steps

1. **Review & Approve** architecture proposal
2. **Proof of Concept** - Start with Calculation Service
3. **Infrastructure Setup** - Docker, Kubernetes, Message Queue
4. **API Gateway** - Set up routing and authentication
5. **Gradual Migration** - Move services one by one
6. **Monitoring** - Set up logging and metrics
7. **Documentation** - API docs for each service
8. **Testing** - Integration tests for service communication

---

## Documentation Files

1. **CELIPA_SERVER_ANALYSIS.md** - Detailed current architecture analysis
2. **MICROSERVICES_ARCHITECTURE.md** - Complete microservices design with diagrams
3. **MICROSERVICES_SUMMARY.md** - This summary document

---

## Questions to Consider

1. **Team Size**: Do we have enough developers to maintain 13 services?
2. **Complexity**: Is the added complexity worth the benefits?
3. **Timeline**: Can we afford the migration time?
4. **Infrastructure**: Do we have DevOps expertise for Kubernetes?
5. **Budget**: Can we afford the infrastructure costs?
6. **Current Scale**: Is the current system actually struggling with scale?

---

## Recommendation

**Start Small**: Begin with extracting 2-3 low-risk services (Calculation, File Storage, Notification) to validate the approach before committing to full microservices architecture.

**Alternative**: Consider a **modular monolith** approach first - separate code into modules but deploy as one application. This provides some benefits without the full complexity of microservices.
