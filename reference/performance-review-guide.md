# Performance Review Guide

Guidance for performance reviews across frontend, backend, databases, algorithmic complexity, and API behavior.

## Contents

- [Frontend performance (Core Web Vitals)](#frontend-performance-core-web-vitals)
- [JavaScript performance](#javascript-performance)
- [Memory management](#memory-management)
- [Database performance](#database-performance)
- [API performance](#api-performance)
- [Algorithmic complexity](#algorithmic-complexity)
- [Performance review checklist](#performance-review-checklist)

---

## Frontend performance (Core Web Vitals)

### Core metrics (2024)

| Metric | Full name | Target | Meaning |
|--------|-----------|--------|---------|
| **LCP** | Largest Contentful Paint | ≤ 2.5s | Time until largest content is painted |
| **INP** | Interaction to Next Paint | ≤ 200ms | Responsiveness to input (replaces FID in 2024) |
| **CLS** | Cumulative Layout Shift | ≤ 0.1 | Visual stability / layout shift |
| **FCP** | First Contentful Paint | ≤ 1.8s | First paint of any content |
| **TBT** | Total Blocking Time | ≤ 200ms | Main-thread blocking time |

### LCP review

```javascript
// ❌ Lazy-loading the LCP image — delays hero content
<img src="hero.jpg" loading="lazy" />

// ✅ Load the LCP image eagerly
<img src="hero.jpg" fetchpriority="high" />

// ❌ Unoptimized image format
<img src="hero.png" />  // PNG often too large for photos

// ✅ Modern formats + responsive sources
<picture>
  <source srcset="hero.avif" type="image/avif" />
  <source srcset="hero.webp" type="image/webp" />
  <img src="hero.jpg" alt="Hero" />
</picture>
```

**Checklist:**
- [ ] Is `fetchpriority="high"` set on the LCP element?
- [ ] WebP/AVIF in use where appropriate?
- [ ] SSR or static generation for critical HTML?
- [ ] CDN configured correctly?

### FCP review

```html
<!-- ❌ Render-blocking CSS -->
<link rel="stylesheet" href="all-styles.css" />

<!-- ✅ Inline critical CSS + defer the rest -->
<style>/* above-the-fold critical CSS */</style>
<link rel="preload" href="styles.css" as="style" onload="this.onload=null;this.rel='stylesheet'" />

<!-- ❌ Render-blocking font loading -->
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2');
}

<!-- ✅ Font display strategy -->
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2');
  font-display: swap;  /* system font first, swap when loaded */
}
```

### INP review

```javascript
// ❌ Long task blocks the main thread
button.addEventListener('click', () => {
  // 500ms+ of synchronous work
  processLargeData(data);
  updateUI();
});

// ✅ Break up long work
button.addEventListener('click', async () => {
  await scheduler.yield?.() ?? new Promise(r => setTimeout(r, 0));

  for (const chunk of chunks) {
    processChunk(chunk);
    await scheduler.yield?.();
  }
  updateUI();
});

// ✅ Web Worker for heavy CPU work
const worker = new Worker('heavy-computation.js');
worker.postMessage(data);
worker.onmessage = (e) => updateUI(e.data);
```

### CLS review

```css
/* ❌ Media without reserved size */
img { width: 100%; }

/* ✅ Reserve space */
img {
  width: 100%;
  aspect-ratio: 16 / 9;
}

/* ❌ Injected content shifts layout */
.ad-container { }

/* ✅ Fixed min height */
.ad-container {
  min-height: 250px;
}
```

**CLS checklist:**
- [ ] Images/video: explicit `width`/`height` or `aspect-ratio`?
- [ ] Fonts: `font-display: swap` (or appropriate strategy)?
- [ ] Dynamic content: space reserved ahead of time?
- [ ] Avoid inserting content above existing content without reserving space?

---

## JavaScript performance

### Code splitting and lazy loading

```javascript
// ❌ Load everything up front
import { HeavyChart } from './charts';
import { PDFExporter } from './pdf';
import { AdminPanel } from './admin';

// ✅ Lazy load
const HeavyChart = lazy(() => import('./charts'));
const PDFExporter = lazy(() => import('./pdf'));

// ✅ Route-level splitting
const routes = [
  {
    path: '/dashboard',
    component: lazy(() => import('./pages/Dashboard')),
  },
  {
    path: '/admin',
    component: lazy(() => import('./pages/Admin')),
  },
];
```

### Bundle size

```javascript
// ❌ Import entire libraries
import _ from 'lodash';
import moment from 'moment';

// ✅ Import only what you need
import debounce from 'lodash/debounce';
import { format } from 'date-fns';

// ❌ Tree shaking blocked by default export object
export default {
  fn1() {},
  fn2() {},  // bundled even if unused
};

// ✅ Named exports enable tree shaking
export function fn1() {}
export function fn2() {}
```

**Bundle checklist:**
- [ ] Dynamic `import()` for code splitting?
- [ ] Large deps imported narrowly?
- [ ] Bundle analyzed (e.g. webpack-bundle-analyzer)?
- [ ] Unused dependencies removed?

### List rendering

```javascript
// ❌ Huge DOM for huge lists
function List({ items }) {
  return (
    <ul>
      {items.map(item => <li key={item.id}>{item.name}</li>)}
    </ul>
  );  // 10k rows → 10k DOM nodes
}

// ✅ Virtual list — render visible rows only
import { FixedSizeList } from 'react-window';

function VirtualList({ items }) {
  return (
    <FixedSizeList
      height={400}
      itemCount={items.length}
      itemSize={35}
    >
      {({ index, style }) => (
        <div style={style}>{items[index].name}</div>
      )}
    </FixedSizeList>
  );
}
```

**Large data checklist:**
- [ ] Lists over ~100 items: virtual scrolling?
- [ ] Tables: pagination or virtualization?
- [ ] No unnecessary full re-renders of huge trees?

---

## Memory management

### Common leaks

#### 1. Listeners not removed

```javascript
// ❌ Listener survives after unmount
useEffect(() => {
  window.addEventListener('resize', handleResize);
}, []);

// ✅ Cleanup
useEffect(() => {
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```

#### 2. Timers not cleared

```javascript
// ❌ Interval never cleared
useEffect(() => {
  setInterval(fetchData, 5000);
}, []);

// ✅ Clear on unmount
useEffect(() => {
  const timer = setInterval(fetchData, 5000);
  return () => clearInterval(timer);
}, []);
```

#### 3. Closures retaining large objects

```javascript
// ❌ Closure keeps huge array alive
function createHandler() {
  const largeData = new Array(1000000).fill('x');

  return function handler() {
    // largeData retained by closure
    console.log(largeData.length);
  };
}

// ✅ Keep only what the handler needs
function createHandler() {
  const largeData = new Array(1000000).fill('x');
  const length = largeData.length;

  return function handler() {
    console.log(length);
  };
}
```

#### 4. Subscriptions not closed

```javascript
// ❌ WebSocket left open
useEffect(() => {
  const ws = new WebSocket('wss://...');
  ws.onmessage = handleMessage;
}, []);

// ✅ Close on unmount
useEffect(() => {
  const ws = new WebSocket('wss://...');
  ws.onmessage = handleMessage;
  return () => ws.close();
}, []);
```

### Memory review checklist

```markdown
- [ ] Every `useEffect` that needs it has a cleanup?
- [ ] Event listeners removed on unmount?
- [ ] Timers/intervals cleared?
- [ ] WebSocket/EventSource connections closed?
- [ ] Large objects released when no longer needed?
- [ ] No unbounded growth on globals?
```

### Tools

| Tool | Use |
|------|-----|
| Chrome DevTools Memory | Heap snapshots |
| MemLab (Meta) | Automated leak detection |
| Performance Monitor | Live memory trends |

---

## Database performance

### N+1 queries

```python
# ❌ N+1 — 1 + N round trips
users = User.objects.all()  # 1 query
for user in users:
    print(user.profile.bio)  # N more queries

# ✅ Eager load — often 2 queries
users = User.objects.select_related('profile').all()
for user in users:
    print(user.profile.bio)  # no extra queries

# ✅ Many-to-many: prefetch_related
posts = Post.objects.prefetch_related('tags').all()
```

```javascript
// TypeORM example
// ❌ N+1
const users = await userRepository.find();
for (const user of users) {
  const posts = await user.posts;  // query per user
}

// ✅ Eager load
const users = await userRepository.find({
  relations: ['posts'],
});
```

### Indexes

```sql
-- ❌ Full table scan
SELECT * FROM orders WHERE status = 'pending';

-- ✅ Index the filter column
CREATE INDEX idx_orders_status ON orders(status);

-- ❌ Index can’t help: function on column
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- ✅ Sargable range
SELECT * FROM users
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- ❌ Leading wildcard kills index use
SELECT * FROM products WHERE name LIKE '%phone%';

-- ✅ Prefix match can use index
SELECT * FROM products WHERE name LIKE 'phone%';
```

### Query patterns

```sql
-- ❌ SELECT * when you need few columns
SELECT * FROM users WHERE id = 1;

-- ✅ Project only needed columns
SELECT id, name, email FROM users WHERE id = 1;

-- ❌ Huge result set with no cap
SELECT * FROM logs WHERE type = 'error';

-- ✅ Pagination
SELECT * FROM logs WHERE type = 'error' LIMIT 100 OFFSET 0;

-- ❌ Query per id in a loop
for id in user_ids:
    cursor.execute("SELECT * FROM users WHERE id = %s", (id,))

-- ✅ Batch IN query
cursor.execute("SELECT * FROM users WHERE id IN %s", (tuple(user_ids),))
```

### Database review checklist

```markdown
🔴 Must check:
- [ ] Any N+1 patterns?
- [ ] Indexed columns in WHERE/JOIN?
- [ ] Avoid SELECT * on wide/hot tables?
- [ ] LIMIT (or equivalent) on large scans?

🟡 Should check:
- [ ] EXPLAIN / query plans reviewed?
- [ ] Composite index column order correct?
- [ ] Unused indexes identified?
- [ ] Slow query logging / alerting?
```

---

## API performance

### Pagination

```javascript
// ❌ Return entire table
app.get('/users', async (req, res) => {
  const users = await User.findAll();  // could be 100k rows
  res.json(users);
});

// ✅ Paginate + cap page size
app.get('/users', async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);  // max 100
  const offset = (page - 1) * limit;

  const { rows, count } = await User.findAndCountAll({
    limit,
    offset,
    order: [['id', 'ASC']],
  });

  res.json({
    data: rows,
    pagination: {
      page,
      limit,
      total: count,
      totalPages: Math.ceil(count / limit),
    },
  });
});
```

### Caching

```javascript
// ✅ Redis cache-aside
async function getUser(id) {
  const cacheKey = `user:${id}`;

  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  const user = await db.users.findById(id);

  await redis.setex(cacheKey, 3600, JSON.stringify(user));

  return user;
}

// ✅ HTTP cache headers
app.get('/static-data', (req, res) => {
  res.set({
    'Cache-Control': 'public, max-age=86400',  // 24 hours
    'ETag': 'abc123',
  });
  res.json(data);
});
```

### Response compression

```javascript
// ✅ Gzip/Brotli middleware
const compression = require('compression');
app.use(compression());

// ✅ Field selection
// GET /users?fields=id,name,email
app.get('/users', async (req, res) => {
  const fields = req.query.fields?.split(',') || ['id', 'name'];
  const users = await User.findAll({
    attributes: fields,
  });
  res.json(users);
});
```

### Rate limiting

```javascript
// ✅ Rate limit hot endpoints
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 60 * 1000,  // 1 minute
  max: 100,
  message: { error: 'Too many requests, please try again later.' },
});

app.use('/api/', limiter);
```

### API review checklist

```markdown
- [ ] List endpoints paginated?
- [ ] Max page size enforced?
- [ ] Hot reads cached where appropriate?
- [ ] Response compression enabled?
- [ ] Rate limiting in place?
- [ ] Responses trimmed to necessary fields?
```

---

## Algorithmic complexity

### Common growth rates

| Complexity | Name | n=10 | n=1k | n=1M | Example |
|------------|------|------|------|------|---------|
| O(1) | Constant | 1 | 1 | 1 | Hash map lookup |
| O(log n) | Logarithmic | 3 | 10 | 20 | Binary search |
| O(n) | Linear | 10 | 1k | 1M | Single pass |
| O(n log n) | Linearithmic | 33 | ~10k | ~20M | Fast sort |
| O(n²) | Quadratic | 100 | 1M | 1T | Nested loops on n |
| O(2ⁿ) | Exponential | 1024 | ∞ | ∞ | Naive recursive Fibonacci |

### What to spot in review

```javascript
// ❌ O(n²) nested loops
function findDuplicates(arr) {
  const duplicates = [];
  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[i] === arr[j]) {
        duplicates.push(arr[i]);
      }
    }
  }
  return duplicates;
}

// ✅ O(n) with Set
function findDuplicates(arr) {
  const seen = new Set();
  const duplicates = new Set();
  for (const item of arr) {
    if (seen.has(item)) {
      duplicates.add(item);
    }
    seen.add(item);
  }
  return [...duplicates];
}
```

```javascript
// ❌ O(n²) — includes() inside loop
function removeDuplicates(arr) {
  const result = [];
  for (const item of arr) {
    if (!result.includes(item)) {  // O(n) per call
      result.push(item);
    }
  }
  return result;
}

// ✅ O(n) with Set
function removeDuplicates(arr) {
  return [...new Set(arr)];
}
```

```javascript
// ❌ O(n) lookup every time
const users = [{ id: 1, name: 'A' }, { id: 2, name: 'B' }, ...];

function getUser(id) {
  return users.find(u => u.id === id);  // O(n)
}

// ✅ O(1) with Map
const userMap = new Map(users.map(u => [u.id, u]));

function getUser(id) {
  return userMap.get(id);  // O(1)
}
```

### Space complexity

```javascript
// ⚠️ O(n) extra space — new array
const doubled = arr.map(x => x * 2);

// ✅ O(1) extra space — mutate in place if allowed
for (let i = 0; i < arr.length; i++) {
  arr[i] *= 2;
}

// ⚠️ Deep recursion — stack overflow risk
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);  // O(n) stack frames
}

// ✅ Iterative O(1) stack
function factorial(n) {
  let result = 1;
  for (let i = 2; i <= n; i++) {
    result *= i;
  }
  return result;
}
```

### Sample review comments

```markdown
💡 "This nested loop is O(n²); it will hurt at scale."
🔴 "`Array.includes` inside a loop is O(n²) overall — consider a `Set`."
🟡 "This recursion depth risks stack overflow — prefer iteration or bounded depth."
```

---

## Performance review checklist

### 🔴 Blocking / must-fix

**Frontend:**
- [ ] LCP hero image lazy-loaded? (should not be)
- [ ] Any `transition: all`?
- [ ] Animating `width`/`height`/`top`/`left`?
- [ ] Lists >100 rows without virtualization?

**Backend:**
- [ ] N+1 queries?
- [ ] List APIs without pagination?
- [ ] `SELECT *` on large/wide tables?

**General:**
- [ ] O(n²)+ nested loops on large inputs?
- [ ] `useEffect`/listeners missing cleanup?

### 🟡 Important

**Frontend:**
- [ ] Code splitting in place?
- [ ] Large libs tree-shaken / partial import?
- [ ] Images modern formats (WebP/AVIF)?
- [ ] Unused deps removed?

**Backend:**
- [ ] Hot paths cached?
- [ ] WHERE columns indexed?
- [ ] Slow query monitoring?

**API:**
- [ ] Compression enabled?
- [ ] Rate limits?
- [ ] Minimal response payloads?

### 🟢 Nice to have

- [ ] Bundle size tracked?
- [ ] CDN for static assets?
- [ ] Real-user / synthetic monitoring?
- [ ] Benchmarks for critical paths?

---

## Thresholds

### Frontend

| Metric | Good | Needs work | Poor |
|--------|------|------------|------|
| LCP | ≤ 2.5s | 2.5–4s | > 4s |
| INP | ≤ 200ms | 200–500ms | > 500ms |
| CLS | ≤ 0.1 | 0.1–0.25 | > 0.25 |
| FCP | ≤ 1.8s | 1.8–3s | > 3s |
| JS bundle (gzipped) | < 200KB | 200–500KB | > 500KB |

### Backend

| Metric | Good | Needs work | Poor |
|--------|------|------------|------|
| API latency | < 100ms | 100–500ms | > 500ms |
| DB query | < 50ms | 50–200ms | > 200ms |
| Page load (end-to-end) | < 3s | 3–5s | > 5s |

---

## Recommended tools

### Frontend

| Tool | Use |
|------|-----|
| [Lighthouse](https://developer.chrome.com/docs/lighthouse/) | Core Web Vitals |
| [WebPageTest](https://www.webpagetest.org/) | Deep waterfall / filmstrip |
| [webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer) | Bundle composition |
| [Chrome DevTools Performance](https://developer.chrome.com/docs/devtools/performance/) | Runtime profiling |

### Memory

| Tool | Use |
|------|-----|
| [MemLab](https://github.com/facebookincubator/memlab) | Automated leak hunts |
| Chrome Memory panel | Heap snapshots |

### Backend

| Tool | Use |
|------|-----|
| EXPLAIN | Query plans |
| [pganalyze](https://pganalyze.com/) | PostgreSQL insights |
| [New Relic](https://newrelic.com/) / [Datadog](https://www.datadoghq.com/) | APM |

---

## References

- [Core Web Vitals - web.dev](https://web.dev/articles/vitals)
- [Optimizing Core Web Vitals - Vercel](https://vercel.com/guides/optimizing-core-web-vitals-in-2024)
- [MemLab - Meta Engineering](https://engineering.fb.com/2022/09/12/open-source/memlab/)
- [Big O Cheat Sheet](https://www.bigocheatsheet.com/)
- [N+1 Query Problem - Stack Overflow](https://stackoverflow.com/questions/97197/what-is-the-n1-selects-problem-in-orm-object-relational-mapping)
- [API Performance Optimization](https://algorithmsin60days.com/blog/optimizing-api-performance/)
