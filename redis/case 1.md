# Comprehensive Example: Caching Slow Third-Party APIs with Redis

## Problem Scenario
You're building an e-commerce application that integrates with multiple external payment gateways (Stripe, PayPal, etc.). During peak traffic periods:
- Payment gateway APIs become slow (500-1000ms response times)
- Your checkout process becomes unresponsive
- Conversion rates drop due to poor user experience

## Solution Architecture
We'll implement Redis caching to:
1. Store successful API responses
2. Serve cached responses for identical requests
3. Automatically refresh stale data

## Implementation Code

### 1. Setup Redis Connection

```javascript
// redis-client.js
const redis = require('redis');
const { promisify } = require('util');

const client = redis.createClient({
  host: process.env.REDIS_HOST,
  port: process.env.REDIS_PORT,
  password: process.env.REDIS_PASSWORD
});

client.on('error', (err) => {
  console.error('Redis error:', err);
});

module.exports = {
  get: promisify(client.get).bind(client),
  set: promisify(client.set).bind(client),
  setex: promisify(client.setex).bind(client),
  del: promisify(client.del).bind(client),
  quit: promisify(client.quit).bind(client),
};
```

### 3. Payment Service with Caching

```javascript
// payment-service.js
const axios = require('axios');
const { get, setex } = require('./redis-client');

class PaymentService {
  constructor() {
    this.cacheTTL = 300; // 5 minutes
  }

  async processPayment(paymentData) {
    const cacheKey = `payment:${paymentData.transactionId}`;
    
    try {
      // Check for cached result
      const cachedResult = await get(cacheKey);
      if (cachedResult) {
        console.log('Returning cached payment result');
        return JSON.parse(cachedResult);
      }

      // Process with Stripe
      const stripeResponse = await axios.post(
        'https://api.stripe.com/v1/charges',
        paymentData,
        {
          headers: { Authorization: `Bearer ${process.env.STRIPE_KEY}` },
          timeout: 5000 // 5s timeout
        }
      );

      // Cache successful payment
      await setex(
        cacheKey,
        this.cacheTTL,
        JSON.stringify(stripeResponse.data)
      );

      return stripeResponse.data;
    } catch (error) {
      console.error('Payment processing error:', error);
      throw error;
    }
  }
  
  async getPaymentStatus(transactionId) {
    const cacheKey = `payment_status:${transactionId}`;
    
    try {
      // Check cache first
      const cachedStatus = await get(cacheKey);
      if (cachedStatus) {
        return JSON.parse(cachedStatus);
      }

      // Fallback to API
      const response = await axios.get(
        `https://api.stripe.com/v1/charges/${transactionId}`,
        {
          headers: { Authorization: `Bearer ${process.env.STRIPE_KEY}` },
          timeout: 3000
        }
      );

      // Cache status for 1 minute (shorter TTL for status checks)
      await setex(cacheKey, 60, JSON.stringify(response.data));

      return response.data;
    } catch (error) {
      console.error('Status check error:', error);
      throw error;
    }
  }
}

module.exports = new PaymentService();
```
### 4. Express API Endpoint

```javascript
// server.js
const express = require('express');
const cacheMiddleware = require('./cache-middleware');
const paymentService = require('./payment-service');

const app = express();
app.use(express.json());

// Apply caching middleware to all GET requests
app.get('*', cacheMiddleware);

app.post('/process-payment', async (req, res) => {
  try {
    const result = await paymentService.processPayment(req.body);
    res.json(result);
  } catch (error) {
    res.status(500).json({ error: 'Payment processing failed' });
  }
});

app.get('/payment-status/:id', async (req, res) => {
  try {
    const status = await paymentService.getPaymentStatus(req.params.id);
    res.json(status);
  } catch (error) {
    res.status(500).json({ error: 'Status check failed' });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```



