# Comprehensive Example: Caching Heavy Computations with Redis

## Problem Scenario
You're building an analytics dashboard that:
- Processes millions of data points
- Generates complex reports (user behavior, sales trends)
- Takes 15-30 seconds to compute each report
- Experiences traffic spikes during business hours

## Solution Architecture
We'll implement Redis caching to:
1. Store computed results with unique keys
2. Serve cached results for identical requests
3. Refresh cache periodically or on-demand
4. Handle cache invalidation when source data changes

## Implementation Code

### 1. Redis Setup (same as previous example)
```javascript
// redis-client.js
const redis = require('redis');
const { promisify } = require('util');

const client = redis.createClient({
  host: process.env.REDIS_HOST,
  port: process.env.REDIS_PORT,
  password: process.env.REDIS_PASSWORD
});

// Promisify Redis methods
module.exports = {
  get: promisify(client.get).bind(client),
  set: promisify(client.set).bind(client),
  setex: promisify(client.setex).bind(client),
  del: promisify(client.del).bind(client),
  keys: promisify(client.keys).bind(client),
};
```

### 2. Analytics Service with Caching
```javascript
// analytics-service.js
const redis = require('./redis-client');
const db = require('./database'); // Your database client

class AnalyticsService {
  constructor() {
    this.cacheTTL = 3600; // 1 hour cache duration
  }

  // Helper method to generate cache key
  _getCacheKey(reportType, params) {
    return `analytics:${reportType}:${JSON.stringify(params)}`;
  }

  async getSalesReport({ startDate, endDate, region }) {
    const cacheKey = this._getCacheKey('sales', { startDate, endDate, region });
    
    try {
      // 1. Check cache first
      const cachedReport = await redis.get(cacheKey);
      if (cachedReport) {
        console.log('Returning cached sales report');
        return JSON.parse(cachedReport);
      }

      // 2. Compute heavy report if not cached
      console.log('Computing fresh sales report...');
      const report = await this._computeSalesReport(startDate, endDate, region);
      
      // 3. Cache the result
      await redis.setex(cacheKey, this.cacheTTL, JSON.stringify(report));
      
      return report;
    } catch (error) {
      console.error('Sales report error:', error);
      throw error;
    }
  }

  async _computeSalesReport(startDate, endDate, region) {
    // Simulating heavy computation (15-30 seconds)
    return new Promise(resolve => {
      setTimeout(async () => {
        // Actual computation would happen here
        const result = {
          summary: {
            totalSales: await db.query(/* complex query */),
            averageOrder: await db.query(/* another query */),
            // ... more metrics
          },
          trends: await this._computeTrends(startDate, endDate, region),
          // ... more data
        };
        resolve(result);
      }, 15000); // 15 second delay to simulate computation
    });
  }

  async _computeTrends(startDate, endDate, region) {
    // Another heavy computation
    return db.query(/* complex trend analysis query */);
  }

  // Invalidate cache when source data changes
  async invalidateCache(reportType, params = {}) {
    let cacheKey;
    if (params) {
      cacheKey = this._getCacheKey(reportType, params);
      await redis.del(cacheKey);
    } else {
      // Invalidate all reports of this type if no params
      const keys = await redis.keys(`analytics:${reportType}:*`);
      await Promise.all(keys.map(key => redis.del(key)));
    }
  }
}

module.exports = new AnalyticsService();
```

### 3. Express API Endpoints
```javascript
// server.js
const express = require('express');
const analyticsService = require('./analytics-service');

const app = express();
app.use(express.json());

app.get('/analytics/sales', async (req, res) => {
  try {
    const { startDate, endDate, region } = req.query;
    const report = await analyticsService.getSalesReport({
      startDate,
      endDate,
      region
    });
    res.json(report);
  } catch (error) {
    res.status(500).json({ error: 'Failed to generate sales report' });
  }
});

// Webhook to invalidate cache when data changes
app.post('/invalidate-cache', async (req, res) => {
  try {
    await analyticsService.invalidateCache(req.body.reportType, req.body.params);
    res.json({ status: 'Cache invalidated' });
  } catch (error) {
    res.status(500).json({ error: 'Cache invalidation failed' });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Real-World Example Flow

1. **First Request (Cache Miss)**
   - User requests sales report for Jan 2023, North America
   - System computes report (takes 15 seconds)
   - Stores result in Redis with key: `analytics:sales:{"startDate":"2023-01-01","endDate":"2023-01-31","region":"north_america"}`
   - TTL: 1 hour

2. **Subsequent Request (Cache Hit)**
   - Same user or different user requests same report
   - System returns cached result in <5ms
   - No recomputation needed

3. **Data Update Scenario**
   - Backoffice system updates sales records
   - Calls `/invalidate-cache` webhook with `{ reportType: "sales" }`
   - Next request will compute fresh data

## Performance Comparison

| Metric          | Uncached | Cached |
|-----------------|---------|--------|
| Response Time   | 15-30s  | <5ms   |
| Database Load   | High    | None   |
| CPU Usage       | High    | None   |
| Concurrent Users Supported | 10-20 | 1000+ |

## Advanced Considerations

1. **Cache Warming**
```javascript
// Pre-compute reports during off-peak hours
async function warmCache() {
  await analyticsService.getSalesReport({
    startDate: getPreviousMonthStart(),
    endDate: getPreviousMonthEnd(),
    region: 'global'
  });
  // Repeat for other common reports
}
```

2. **Stale-While-Revalidate**
```javascript
// Serve stale data while refreshing in background
async function getReportWithStaleFallback(params) {
  const cached = await redis.get(cacheKey);
  if (cached) {
    // Refresh in background if data is older than 30 minutes
    if (isDataStale(cached)) {
      this._computeSalesReport(params).then(freshData => {
        redis.setex(cacheKey, this.cacheTTL, JSON.stringify(freshData));
      });
    }
    return JSON.parse(cached);
  }
  return this.getSalesReport(params);
}
```

3. **Cache Key Versioning**
```javascript
// Add schema version to cache keys
_getCacheKey(reportType, params) {
  return `v2:analytics:${reportType}:${JSON.stringify(params)}`;
  // Change version when report structure changes
}
```

## Monitoring Setup

1. **Cache Metrics**
```javascript
// Add to analytics service
this.metrics = {
  cacheHits: 0,
  cacheMisses: 0,
  computeTimes: [],
};

// Update in getSalesReport()
if (cachedReport) {
  this.metrics.cacheHits++;
} else {
  this.metrics.cacheMisses++;
  const start = Date.now();
  // ... after computation
  this.metrics.computeTimes.push(Date.now() - start);
}
```

2. **Redis Monitoring Commands**
```bash
# Check memory usage
redis-cli info memory

# Check keys
redis-cli keys "analytics:*"

# Check TTL for specific key
redis-cli ttl "analytics:sales:{...}"
```

This implementation provides a complete solution for caching heavy computations, significantly improving performance while maintaining data accuracy through proper cache invalidation strategies.
