# Quick Start Guide - Production Payment Gateway

## Prerequisites
- Docker & Docker Compose installed
- Node.js 18+ (for local development)
- PostgreSQL 15 (included in Docker)
- Redis 7 (included in Docker)

## Setup & Launch

### 1. Start All Services
```bash
cd payment-gateway
docker-compose up --build
```

Services will start in this order (health checks ensure dependencies):
1. PostgreSQL (port 5432)
2. Redis (port 6379)
3. API Server (port 8000)
4. Worker (background jobs)
5. Frontend Dashboard (port 3000)
6. Checkout Page (port 3001)

### 2. Verify Services
```bash
# Check API health
curl http://localhost:8000/health

# Check job queue status
curl http://localhost:8000/api/v1/test/jobs/status

# Check Redis
redis-cli -p 6379 ping
```

### 3. Get Test Credentials
```bash
curl http://localhost:8000/api/v1/test/merchant
```

Response:
```json
{
  "id": "merchant-uuid",
  "api_key": "key_test_abc123",
  "api_secret": "secret_test_xyz789",
  "webhook_url": null,
  "webhook_secret": "whsec_test_abc123"
}
```

## Testing the Payment Flow

### Step 1: Login to Dashboard
1. Open http://localhost:3000
2. Login with test credentials from above
3. View dashboard

### Step 2: Create Test Order
1. Click "Create test order" button
2. Automatically redirects to checkout page
3. Complete payment (uses simulated payment processor)

### Step 3: Monitor Processing
1. Open http://localhost:8000/api/v1/test/jobs/status
2. Watch "pending" → "processing" → "completed" progression
3. Status updates every 2 seconds as jobs process

### Step 4: Check Webhook Delivery
1. (Terminal) Start test merchant webhook receiver:
   ```bash
   cd test-merchant
   npm install
   node webhook.js
   ```

2. In dashboard, configure webhook:
   - URL: `http://host.docker.internal:4000/webhook` (Mac/Windows)
   - URL: `http://172.17.0.1:4000/webhook` (Linux)

3. Create new payment and observe webhook logs in test merchant terminal

### Step 5: Test Refunds
Create a refund via API:
```bash
curl -X POST http://localhost:8000/api/v1/payments/{payment_id}/refunds \
  -H "X-Api-Key: key_test_abc123" \
  -H "X-Api-Secret: secret_test_xyz789" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50000,
    "reason": "Customer requested"
  }'
```

## API Examples

### Create Order
```bash
curl -X POST http://localhost:8000/api/v1/orders \
  -H "X-Api-Key: key_test_abc123" \
  -H "X-Api-Secret: secret_test_xyz789" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50000,
    "currency": "INR",
    "receipt": "receipt_123"
  }'
```

### Create Payment (with Idempotency)
```bash
curl -X POST http://localhost:8000/api/v1/payments \
  -H "X-Api-Key: key_test_abc123" \
  -H "X-Api-Secret: secret_test_xyz789" \
  -H "Idempotency-Key: unique-request-123" \
  -H "Content-Type: application/json" \
  -d '{
    "order_id": "order_abc123",
    "method": "upi",
    "vpa": "user@paytm"
  }'
```

### Get Payment Status
```bash
curl http://localhost:8000/api/v1/payments/{payment_id}/public
```

### List Webhook Logs
```bash
curl http://localhost:8000/api/v1/webhooks \
  -H "X-Api-Key: key_test_abc123" \
  -H "X-Api-Secret: secret_test_xyz789"
```

### Retry Failed Webhook
```bash
curl -X POST http://localhost:8000/api/v1/webhooks/{webhook_id}/retry \
  -H "X-Api-Key: key_test_abc123" \
  -H "X-Api-Secret: secret_test_xyz789"
```

## SDK Integration

### Embed on Your Website
```html
<script src="http://localhost:3001/checkout.js"></script>
<button id="pay-button">Pay Now</button>

<script>
document.getElementById('pay-button').addEventListener('click', () => {
  const checkout = new PaymentGateway({
    key: 'key_test_abc123',
    orderId: 'order_xyz',
    onSuccess: (response) => {
      console.log('Payment successful:', response.paymentId);
      // Handle success (e.g., show confirmation, update order)
    },
    onFailure: (error) => {
      console.log('Payment failed:', error);
      // Handle failure (e.g., show error message)
    },
    onClose: () => {
      console.log('Modal closed');
      // Handle close (e.g., cleanup)
    }
  });
  checkout.open();
});
</script>
```

## Testing Modes

### Deterministic Payment Processing
Set environment variables in backend `.env`:

```env
# Force all payments to succeed
TEST_MODE=true
TEST_PAYMENT_SUCCESS=true
TEST_PROCESSING_DELAY=1000

# Force all payments to fail
TEST_MODE=true
TEST_PAYMENT_SUCCESS=false
TEST_PROCESSING_DELAY=500
```

### Fast Webhook Retry Testing
```env
# Test complete retry cycle in < 1 minute
WEBHOOK_RETRY_INTERVALS_TEST=true
```

Retry schedule (test mode):
- Attempt 1: Immediate
- Attempt 2: After 5 seconds
- Attempt 3: After 10 seconds
- Attempt 4: After 15 seconds
- Attempt 5: After 20 seconds

## Database Debugging

### Connect to PostgreSQL
```bash
psql postgresql://gateway_user:gateway_pass@localhost:5432/payment_gateway
```

### Useful Queries
```sql
-- View recent payments
SELECT id, status, created_at FROM payments ORDER BY created_at DESC LIMIT 10;

-- View webhook logs
SELECT id, event, status, attempts, created_at FROM webhook_logs ORDER BY created_at DESC;

-- View refunds
SELECT id, payment_id, status, created_at FROM refunds ORDER BY created_at DESC;

-- View idempotency keys
SELECT key, merchant_id, expires_at FROM idempotency_keys;
```

## Redis Debugging

### Connect to Redis
```bash
redis-cli -p 6379
```

### Useful Commands
```bash
# Check queue length
LLEN bull:payments:wait
LLEN bull:webhooks:wait
LLEN bull:refunds:wait

# Get queue stats
HGETALL bull:payments:stats

# Monitor in real-time
MONITOR
```

## Logs & Troubleshooting

### View Logs
```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f api
docker-compose logs -f worker
```

### Common Issues

**Webhooks not delivering:**
- Check merchant webhook_url is set in dashboard
- Verify merchant can receive POST requests
- Check test-merchant webhook receiver is running
- Look at webhook logs in dashboard

**Jobs not processing:**
- Verify Redis is running: `redis-cli ping`
- Check worker container logs: `docker-compose logs worker`
- Verify job queue status: `curl http://localhost:8000/api/v1/test/jobs/status`

**Payment timeout:**
- Payments have 30-second timeout
- In test mode, reduce TEST_PROCESSING_DELAY
- Check worker concurrency settings

**Signature mismatch:**
- Verify webhook secret matches
- Ensure JSON payload has no whitespace changes
- Check encoding is UTF-8

## Performance Tuning

### Worker Concurrency
Edit `src/workers/*.js`:
```javascript
new Worker('queue-name', processor, {
  connection: redisConnection,
  concurrency: 10  // Increase for more parallel processing
});
```

### Redis Connection Pool
Edit `src/queues/index.js`:
```javascript
const redisConnection = new IORedis({
  host: 'redis',
  port: 6379,
  maxRetriesPerRequest: null,
  enableReadyCheck: false,
  enableOfflineQueue: true
});
```

### Database Connection Pool
Edit `src/config/database.js`:
```javascript
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,  // Max connections
  min: 5,   // Min connections
  idle: 10000
});
```

## Stopping Services
```bash
# Stop all containers
docker-compose down

# Stop and remove volumes
docker-compose down -v

# Restart services
docker-compose restart
```

## Production Deployment

### Before Going Live

1. **Environment Variables**
   - Set strong API key/secret pairs
   - Configure real webhook URLs
   - Enable HTTPS everywhere
   - Set `TEST_MODE=false`
   - Configure production Redis with persistence

2. **Database**
   - Use managed PostgreSQL service
   - Enable automated backups
   - Set up monitoring and alerts

3. **Redis**
   - Use managed Redis service (e.g., AWS ElastiCache)
   - Enable persistence
   - Configure replication

4. **Security**
   - Rotate webhook secrets regularly
   - Implement rate limiting
   - Use VPCs and security groups
   - Enable WAF protection

5. **Monitoring**
   - Set up error tracking (e.g., Sentry)
   - Monitor job queue depths
   - Alert on failed jobs
   - Track webhook delivery rates

6. **Scaling**
   - Run multiple worker instances
   - Use load balancer for API
   - Scale Redis horizontally

## Support & Documentation

- API Docs: http://localhost:3000/dashboard/docs
- Webhook Config: http://localhost:3000/dashboard/webhooks
- Dashboard: http://localhost:3000/dashboard
- Implementation Guide: `IMPLEMENTATION_GUIDE.md`

---

Happy testing! 🎉
