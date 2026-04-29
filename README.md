# Infrastructure (Infra)

Kubernetes-based infrastructure configuration for the TicketHub distributed microservices ticketing system. Contains deployment manifests, service definitions, and ingress routing for all services.

## Overview

The infrastructure directory contains all Kubernetes resources needed to deploy the TicketHub microservices architecture. It includes configurations for development and production environments with proper service mesh, data persistence, message queuing, and API gateway setup.

### Architecture

```
                    ┌──────────────────┐
                    │ NGINX Ingress    │ (ticket.dev)
                    └────────┬─────────┘
           ┌────────────────┼────────────────┐
           │                │                │
      ┌────▼────┐    ┌──────▼──────┐   ┌────▼─────┐
      │ Auth    │    │   Client    │   │API Routes│
      │ Service │    │   Frontend  │   │ (Payments,
      │ :3000   │    │   :3000     │   │  Users,
      └────┬────┘    └─────┬──────┘    │  Tickets,
      │    │        │    │    │       Orders)
      └────┼─────┬──┴────┼────┴────────────┘
           │     │       │
        ┌──▼─┬──▼─┬─────▼──┐
        │    │    │        │
    ┌───▼─┐ │ ┌──▼───┐ ┌──▼──────┐
    │NATS │ │ │Redis │ │Mongo(4) │
    │:4222│ │ │:6379 │ │:27017   │
    └─────┘ │ └──────┘ └─────────┘
            │
         (Event Bus)

Services:
├── auth-srv
├── tickets-srv
├── orders-srv
├── payments-srv
├── expiration-srv
└── client-srv

Databases:
├── auth-mongo
├── tickets-mongo
├── orders-mongo
└── payments-mongo

Message Bus:
└── nats-srv

Cache:
└── expiration-redis
```

## Directory Structure

```
infra/
├── k8s/                              # Production manifests
│   ├── auth-depl.yaml               # Auth service + MongoDB
│   ├── auth-mongo-depl.yaml
│   ├── tickets-depl.yaml            # Tickets service + MongoDB
│   ├── tickets-mongo-depl.yaml
│   ├── orders-depl.yml              # Orders service + MongoDB
│   ├── orders-mongo-depl.yml
│   ├── payments-depl.yml            # Payments service + MongoDB
│   ├── payments-mongo-depl.yml
│   ├── expiration-depl.yml          # Expiration service + Redis
│   ├── expiration-redis-depl.yml
│   ├── client-depl.yaml             # Client frontend
│   └── nats-depl.yaml               # NATS event bus
├── k8s-dev/                          # Development environment
│   └── ingress-srv.yaml             # Ingress with TLS (self-signed)
├── k8s-prod/                         # Production environment
│   └── ingress-srv.yaml             # Ingress production config
└── README.md                         # This file
```

## Services Overview

### Microservices

| Service | Port | Database | Purpose |
|---------|------|----------|---------|
| **auth-srv** | 3000 | auth-mongo | User authentication & JWT management |
| **tickets-srv** | 3000 | tickets-mongo | Event ticket management |
| **orders-srv** | 3000 | orders-mongo | Order creation & management |
| **payments-srv** | 3000 | payments-mongo | Payment processing (Razorpay) |
| **expiration-srv** | (internal) | N/A | Order expiration via job queue |
| **client-srv** | 3000 | N/A | Frontend (Next.js) |

### Data Storage

| Database | Type | Port | Services |
|----------|------|------|----------|
| **auth-mongo** | MongoDB | 27017 | auth-srv |
| **tickets-mongo** | MongoDB | 27017 | tickets-srv |
| **orders-mongo** | MongoDB | 27017 | orders-srv |
| **payments-mongo** | MongoDB | 27017 | payments-srv |
| **expiration-redis** | Redis | 6379 | expiration-srv |

### Infrastructure Services

| Component | Type | Port | Purpose |
|-----------|------|------|---------|
| **nats-srv** | NATS Streaming | 4222 | Event message bus (pub/sub) |
| **ingress** | NGINX | 80/443 | API gateway & routing |

## API Routing

The NGINX Ingress routes all requests through `ticket.dev`:

```
/api/users/*       → auth-srv:3000
/api/tickets/*     → tickets-srv:3000
/api/orders/*      → orders-srv:3000
/api/payments/*    → payments-srv:3000
/*                 → client-srv:3000 (default/frontend)
```

## Prerequisites

- Kubernetes cluster (1.24+)
- kubectl configured to access your cluster
- NGINX Ingress Controller installed
- Secrets configured (see below)

## Setup & Deployment

### 1. Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

### 2. Create Required Secrets

The services require a JWT secret for authentication:

```bash
# Create jwt-secret
kubectl create secret generic jwt-secret --from-literal=Jwt_key=<your-jwt-secret-key>
```

Replace `<your-jwt-secret-key>` with a strong random string.

### 3. Deploy Development Environment

```bash
# Deploy all services and infrastructure
kubectl apply -f infra/k8s/

# Apply development ingress (includes TLS for ticket.dev)
kubectl apply -f infra/k8s-dev/ingress-srv.yaml
```

### 4. Configure Local DNS (Development)

Add to `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows):

```
127.0.0.1 ticket.dev
```

### 5. Verify Deployment

```bash
# Check all deployments
kubectl get deployments

# Check services
kubectl get services

# Check ingress
kubectl get ingress

# Check pods
kubectl get pods

# View logs
kubectl logs -f deployment/auth-depl
```

## Environment Configurations

### Development (`k8s-dev/`)

**Features:**
- TLS enabled with self-signed certificate
- Uses `ticket.dev` domain
- Suitable for local testing

**Notable:** The ingress includes TLS configuration:
```yaml
tls:
  - hosts:
      - ticket.dev
    secretName: ticket-dev-tls
```

### Production (`k8s-prod/`)

**Features:**
- No TLS in this configuration (use proper certificate provider)
- Same routing rules as development
- Ready for cloud deployment

**Recommended for production:**
- Use cert-manager for Let's Encrypt integration
- Update ingress for your production domain
- Implement network policies
- Set resource requests/limits
- Use PersistentVolumes for data storage

## Service Details

### Auth Service (`auth-depl.yaml`)

```yaml
Deployment: auth-depl
Service: auth-srv
Port: 3000
Database: auth-mongo-srv:27017/auth
Environment:
  - MONGO_URI: mongodb://auth-mongo-srv:27017/auth
  - JWT_KEY: (from secret)
```

**Functionality:**
- User registration
- User login/logout
- JWT token generation
- Current user endpoint

### Tickets Service (`tickets-depl.yaml`)

```yaml
Deployment: tickets-depl
Service: tickets-srv
Port: 3000
Database: tickets-mongo-srv:27017/tickets
Environment:
  - MONGO_URI: mongodb://tickets-mongo-srv:27017/tickets
  - JWT_KEY: (from secret)
  - NATS Credentials
```

**Functionality:**
- Create/list/update tickets
- Ticket reservation tracking
- Event publishing (TicketCreated, TicketUpdated)

### Orders Service (`orders-depl.yml`)

```yaml
Deployment: orders-depl
Service: orders-srv
Port: 3000
Database: orders-mongo-srv:27017/orders
Environment:
  - MONGO_URI: mongodb://orders-mongo-srv:27017/orders
  - NATS_URL: http://nats-srv:4222
  - NATS_CLUSTER_ID: ticketing
  - JWT_KEY: (from secret)
```

**Functionality:**
- Create orders for tickets
- Order status management
- Event listening (TicketCreated, TicketUpdated, PaymentCreated)
- Event publishing (OrderCreated, OrderCancelled)

### Payments Service (`payments-depl.yml`)

```yaml
Deployment: payments-depl
Service: payments-srv
Port: 3000
Database: payments-mongo-srv:27017/payments
Environment:
  - MONGO_URI: mongodb://payments-mongo-srv:27017/payments
  - NATS_URL: http://nats-srv:4222
  - NATS_CLUSTER_ID: ticketing
  - JWT_KEY: (from secret)
  - RAZOR_KEY_ID & RAZOR_KEY: (Razorpay credentials)
```

**Functionality:**
- Razorpay integration
- Payment verification
- Event listening (OrderCreated, OrderCancelled, ExpirationComplete)
- Event publishing (PaymentCreated)

### Expiration Service (`expiration-depl.yml`)

```yaml
Deployment: expiration-depl
Service: (internal only)
Cache: expiration-redis-srv:6379
Environment:
  - REDIS_HOST: redis-srv
  - NATS Credentials
```

**Functionality:**
- Job queue for order expiration
- Event listening (OrderCreated)
- Event publishing (ExpirationComplete)
- Uses Redis for persistent job storage

### Client Service (`client-depl.yaml`)

```yaml
Deployment: client-depl
Service: client-srv
Port: 3000
```

**Functionality:**
- Next.js frontend
- Serves static assets
- API gateway connection

### NATS Service (`nats-depl.yaml`)

```yaml
Deployment: nats-depl
Service: nats-srv
Port: 4222 (client)
Port: 8222 (monitoring)
Configuration:
  - Cluster ID: ticketing
  - Health check: 5s interval
```

**Functionality:**
- Event message bus
- Pub/sub system for inter-service communication
- Persistent message storage

## Common Operations

### Deploy All Services

```bash
kubectl apply -f infra/k8s/
```

### Scale a Service

```bash
kubectl scale deployment auth-depl --replicas=3
```

### View Service Logs

```bash
# Follow logs from a deployment
kubectl logs -f deployment/auth-depl

# Follow logs from a specific pod
kubectl logs -f pod/auth-depl-xyz123
```

### Port Forward (Local Access)

```bash
# Access a service locally
kubectl port-forward service/auth-srv 3000:3000

# Access NATS monitoring
kubectl port-forward service/nats-srv 8222:8222
# Then visit http://localhost:8222
```

### Access NATS Monitoring

After port-forwarding:
```bash
kubectl port-forward service/nats-srv 8222:8222
```

Visit: http://localhost:8222

### Restart a Service

```bash
kubectl rollout restart deployment/auth-depl
```

### Update Environment Variable

```bash
# Edit deployment
kubectl edit deployment/auth-depl

# Or use patch
kubectl set env deployment/auth-depl JWT_KEY=newkey
```

### Delete a Deployment

```bash
# Delete specific deployment and service
kubectl delete deployment auth-depl
kubectl delete service auth-srv

# Or use file
kubectl delete -f infra/k8s/auth-depl.yaml
```

## Troubleshooting

### Pods Not Starting

```bash
# Check pod status
kubectl get pods

# Describe pod for events
kubectl describe pod <pod-name>

# View logs
kubectl logs <pod-name>
```

### Service Not Accessible

```bash
# Check if service exists
kubectl get service <service-name>

# Test connectivity from within cluster
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
# nslookup auth-srv
# wget -O- http://auth-srv:3000/api/users/currentuser
```

### Ingress Not Working

```bash
# Check ingress status
kubectl get ingress

# Describe ingress
kubectl describe ingress ingress-service

# Check NGINX controller logs
kubectl logs -f -n ingress-nginx deployment/ingress-nginx-controller
```

### Database Connection Issues

```bash
# Check if MongoDB/Redis is running
kubectl get pods | grep mongo
kubectl get pods | grep redis

# Port-forward to test connection
kubectl port-forward service/tickets-mongo-srv 27017:27017
# mongo mongodb://localhost:27017
```

## Security Considerations

### Current Setup (Development)

- Services communicate within cluster
- TLS enabled on ingress (development certificate)
- JWT secrets stored as Kubernetes secrets
- No network policies configured

### Production Recommendations

- Use cert-manager with Let's Encrypt
- Implement RBAC policies
- Enable network policies
- Use PodSecurityPolicy
- Implement proper secret management (HashiCorp Vault)
- Add resource limits and requests
- Enable container security scanning
- Use private container registries

## Performance Tuning

### Current Configuration

- Single replica per service
- No resource limits/requests
- Default MongoDB/Redis settings

### Production Recommendations

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

- Use horizontal pod autoscaling (HPA)
- Configure vertical pod autoscaling (VPA)
- Use PersistentVolumes for data
- Implement caching layers
- Use CDN for static assets

## Monitoring & Observability

### Current State

- NATS monitoring at :8222
- Basic kubectl monitoring

### Recommended Additions

- **Prometheus**: Metrics collection
- **Grafana**: Visualization
- **ELK Stack**: Logging
- **Jaeger**: Distributed tracing
- **AlertManager**: Alert management

## Updating Manifests

### Change Image Version

```bash
kubectl set image deployment/auth-depl auth=ajaijosephe/auth:v2
```

### Modify Deployment

```bash
# Edit directly
kubectl edit deployment/auth-depl

# Apply new manifest
kubectl apply -f infra/k8s/auth-depl.yaml
```

### Rollback Changes

```bash
kubectl rollout history deployment/auth-depl
kubectl rollout undo deployment/auth-depl --to-revision=1
```

## Backup & Restore

### Backup Manifests

```bash
kubectl get all -n default -o yaml > backup.yaml
```

### Backup Databases

```bash
# MongoDB backup
kubectl exec -it deployment/tickets-mongo-depl -- mongodump --out /backup

# Redis backup
kubectl exec -it service/expiration-redis-srv -- redis-cli BGSAVE
```

## Multi-Environment Setup

### Development Cluster

```bash
kubectl config use-context dev-cluster
kubectl apply -f infra/k8s/
kubectl apply -f infra/k8s-dev/ingress-srv.yaml
```

### Production Cluster

```bash
kubectl config use-context prod-cluster
kubectl apply -f infra/k8s/
kubectl apply -f infra/k8s-prod/ingress-srv.yaml
```

## Related Documentation

- [Tickets Service README](../tickets/README.md)
- [Orders Service README](../orders/README.md)
- [Payments Service README](../payments/README.md)
- [Auth Service README](../auth/README.md)
- [Expiration Service README](../expiration/README.md)
- [Client README](../client/README.md)

## Contributing

When modifying infrastructure:

1. Test changes in development environment first
2. Use kubectl dry-run: `kubectl apply -f manifest.yaml --dry-run=client`
3. Document any new services or changes
4. Update this README if adding new components

## Support

For issues or questions:
- Check Kubernetes logs: `kubectl logs -f deployment/<service>`
- Verify DNS resolution: `kubectl run -it --rm debug --image=busybox -- nslookup <service>`
- Check service connectivity: `kubectl port-forward service/<service> 3000:3000`

## License

ISC
