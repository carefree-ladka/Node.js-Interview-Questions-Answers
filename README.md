# Node.js Interview Questions & Answers

A comprehensive guide to Node.js interview questions asked in top product companies (Amazon, Google, Microsoft, startups).

---

## 1️⃣ Core Node.js Fundamentals

### Q1. What is Node.js?

**Answer:**
Node.js is a JavaScript runtime environment built on Chrome's V8 JavaScript engine. It uses an event-driven, non-blocking I/O model that makes it lightweight and efficient for building scalable network applications.

**Key Points:**
- Executes JavaScript outside the browser
- Uses V8 engine for fast execution
- Single-threaded event loop for handling concurrency
- Best suited for I/O-heavy, real-time applications (chat apps, APIs, streaming services)

**What interviewers look for:** Understanding of runtime vs framework, and when to use Node.js.

---

### Q2. Is Node.js single-threaded?

**Answer:**
Yes and no. JavaScript execution in Node.js is single-threaded, but Node.js uses a thread pool (via libuv) for handling asynchronous operations.

**Detailed Explanation:**
- **JavaScript execution:** Single-threaded
- **Async operations (FS, crypto, DNS):** Uses libuv thread pool (default size = 4 threads)
- **Worker Threads:** Can be used for CPU-intensive tasks
- **Event Loop:** Single-threaded, handles callbacks

**Follow-up concepts:**
- Event loop phases
- Worker threads vs clustering
- Thread pool configuration: `UV_THREADPOOL_SIZE`

---

### Q3. What is the Event Loop?

**Answer:**
The Event Loop is the mechanism that allows Node.js to perform non-blocking I/O operations despite JavaScript being single-threaded. It continuously checks for and executes callbacks from various queues.

**Event Loop Phases (in order):**

1. **Timers:** Executes callbacks scheduled by `setTimeout()` and `setInterval()`
2. **Pending Callbacks:** Executes I/O callbacks deferred to the next iteration
3. **Idle, Prepare:** Internal use only
4. **Poll:** Retrieves new I/O events, executes I/O callbacks
5. **Check:** Executes `setImmediate()` callbacks
6. **Close Callbacks:** Executes close event callbacks (e.g., `socket.on('close')`)

**Special Queues:**
- `process.nextTick()` queue: Runs before all phases
- Microtask queue (Promises): Runs after `nextTick`, before next phase

**Example:**
```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
process.nextTick(() => console.log('4'));
console.log('5');

// Output: 1, 5, 4, 3, 2
```

---

### Q4. Difference between `setImmediate` and `setTimeout`?

**Answer:**

| Feature | `setTimeout` | `setImmediate` |
|---------|--------------|----------------|
| Execution | After minimum delay | After poll phase |
| Timing | Timer-based (at least Xms) | Event loop phase-based |
| Use case | Delay execution | Execute after I/O events |

**Example:**
```javascript
// Outside I/O cycle - order is non-deterministic
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));

// Inside I/O cycle - setImmediate always first
const fs = require('fs');
fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});
// Output: immediate, timeout
```

---

### Q5. What is libuv?

**Answer:**
libuv is a multi-platform C library that provides Node.js with:
- Event loop implementation
- Asynchronous I/O operations
- Thread pool for file system operations
- Platform abstraction layer

**Key Features:**
- Default thread pool size: 4
- Handles: FS operations, DNS lookups, crypto operations
- Can be configured: `process.env.UV_THREADPOOL_SIZE = 8`

---

## 2️⃣ Asynchronous Programming & Internals

### Q6. How does async/await work internally?

**Answer:**
`async/await` is syntactic sugar built on top of Promises. When you use `await`, it pauses the execution of the async function (not the thread) and schedules the remaining code to run after the Promise resolves.

**How it works:**
1. `async` function always returns a Promise
2. `await` pauses the function execution
3. Control returns to the event loop
4. When Promise resolves, execution resumes from where it paused
5. Uses microtask queue for scheduling

**Example:**
```javascript
async function fetchData() {
  console.log('1');
  const data = await fetch('api/data'); // Pauses here
  console.log('2', data);
  return data;
}

// Equivalent to:
function fetchData() {
  console.log('1');
  return fetch('api/data').then(data => {
    console.log('2', data);
    return data;
  });
}
```

---

### Q7. Callback vs Promise vs async/await

| Feature | Callbacks | Promises | async/await |
|---------|-----------|----------|-------------|
| Syntax | Nested functions | Chainable `.then()` | Synchronous-looking |
| Error Handling | Error-first callbacks | `.catch()` | `try/catch` |
| Readability | Callback hell | Better | Best |
| Composition | Difficult | `.all()`, `.race()` | Easy with loops |

**Example:**
```javascript
// Callback Hell
getData(function(a) {
  getMoreData(a, function(b) {
    getMoreData(b, function(c) {
      console.log(c);
    });
  });
});

// Promises
getData()
  .then(a => getMoreData(a))
  .then(b => getMoreData(b))
  .then(c => console.log(c))
  .catch(err => console.error(err));

// async/await
async function fetchAll() {
  try {
    const a = await getData();
    const b = await getMoreData(a);
    const c = await getMoreData(b);
    console.log(c);
  } catch (err) {
    console.error(err);
  }
}
```

---

### Q8. What happens with unhandled promise rejections?

**Answer:**
In modern Node.js versions (15+), unhandled promise rejections will terminate the Node.js process. In older versions, they only logged a warning.

**Best Practices:**
```javascript
// Global handler
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  // Application specific logging, throwing an error, or other logic
});

// Always handle rejections
asyncFunction()
  .then(result => console.log(result))
  .catch(err => console.error(err));

// With async/await
try {
  await asyncFunction();
} catch (err) {
  console.error(err);
}
```

---

### Q9. Difference between `process.nextTick` and `Promise.then`?

**Answer:**

| Feature | `process.nextTick` | `Promise.then` |
|---------|-------------------|----------------|
| Queue | nextTick queue | Microtask queue |
| Priority | Highest (executes first) | After nextTick |
| Risk | Can starve event loop | Safer |
| Use case | Immediate execution | Async operations |

**Example:**
```javascript
Promise.resolve().then(() => console.log('promise1'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise2'));

// Output: nextTick, promise1, promise2
```

**Warning:** Recursive `nextTick` calls can block the event loop:
```javascript
// DON'T DO THIS
function recursiveNextTick() {
  process.nextTick(recursiveNextTick);
}
```

---

## 3️⃣ Memory, Performance & Scaling

### Q10. How does Node.js handle memory?

**Answer:**
Node.js uses V8's memory management with automatic garbage collection.

**V8 Memory Structure:**
- **New Space (Young Generation):** Short-lived objects, ~1-8MB
- **Old Space (Old Generation):** Long-lived objects, ~1.4GB default on 64-bit
- **Large Object Space:** Objects larger than 1MB
- **Code Space:** JIT compiled code
- **Map Space:** Hidden classes and meta information

**Default Heap Limits:**
- 32-bit: ~512MB
- 64-bit: ~1.4GB

**Increase Heap Size:**
```bash
node --max-old-space-size=4096 app.js  # 4GB
node --max-new-space-size=2048 app.js  # 2MB
```

**Monitoring:**
```javascript
console.log(process.memoryUsage());
// {
//   rss: 24576000,        // Resident Set Size
//   heapTotal: 7159808,   // Total heap allocated
//   heapUsed: 4431072,    // Heap used
//   external: 8312,       // C++ objects bound to JS
//   arrayBuffers: 9386    // ArrayBuffers and SharedArrayBuffers
// }
```

---

### Q11. What causes memory leaks in Node.js?

**Answer:**
Common causes:

1. **Global Variables:**
```javascript
// BAD: Never cleaned up
global.cache = [];
setInterval(() => {
  global.cache.push(new Array(1000));
}, 100);
```

2. **Event Listeners Not Removed:**
```javascript
// BAD: Memory leak
function addListener() {
  const emitter = new EventEmitter();
  emitter.on('event', () => { /* handler */ });
  return emitter;
}

// GOOD: Remove listener
emitter.removeListener('event', handler);
// or use once
emitter.once('event', handler);
```

3. **Closures Holding References:**
```javascript
// BAD: Large array never released
function outer() {
  const bigArray = new Array(1000000);
  return function inner() {
    // Closure keeps bigArray in memory
    console.log(bigArray.length);
  };
}
```

4. **Unbounded Cache:**
```javascript
// BAD: No eviction policy
const cache = {};
app.get('/data/:id', (req, res) => {
  cache[req.params.id] = largeData;
});

// GOOD: Use LRU cache
const LRU = require('lru-cache');
const cache = new LRU({ max: 500 });
```

**Detection Tools:**
- Chrome DevTools (heap snapshots)
- `process.memoryUsage()`
- Clinic.js
- heapdump

---

### Q12. How do you scale Node.js applications?

**Answer:**

**1. Clustering (Horizontal Scaling - Same Machine):**
```javascript
const cluster = require('cluster');
const os = require('os');

if (cluster.isMaster) {
  const numCPUs = os.cpus().length;
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  
  cluster.on('exit', (worker) => {
    console.log(`Worker ${worker.id} died`);
    cluster.fork(); // Restart
  });
} else {
  require('./app'); // Start server
}
```

**2. Load Balancer (Nginx/HAProxy):**
```nginx
upstream backend {
  server 127.0.0.1:3000;
  server 127.0.0.1:3001;
  server 127.0.0.1:3002;
}

server {
  listen 80;
  location / {
    proxy_pass http://backend;
  }
}
```

**3. Microservices Architecture:**
- Split application into smaller services
- Each service handles specific functionality
- Independent scaling

**4. Horizontal Scaling (Multiple Machines):**
- Use container orchestration (Kubernetes, Docker Swarm)
- Auto-scaling based on metrics

**5. Worker Threads (CPU-Intensive Tasks):**
```javascript
const { Worker } = require('worker_threads');

function runWorker(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./worker.js', { workerData: data });
    worker.on('message', resolve);
    worker.on('error', reject);
  });
}
```

---

### Q13. What is clustering in Node.js?

**Answer:**
Clustering allows you to create multiple Node.js processes (workers) that share the same server port, enabling better CPU utilization on multi-core systems.

**How it works:**
- Master process spawns worker processes
- Master distributes incoming connections using round-robin (default on most platforms)
- Each worker is an independent process with its own V8 instance
- Workers don't share memory

**Benefits:**
- Utilizes all CPU cores
- Improved fault tolerance (worker dies, others continue)
- Better throughput

**Limitations:**
- No shared memory between workers
- Requires external state management (Redis, DB)
- More complex deployment

---

### Q14. Worker Threads vs Cluster?

**Answer:**

| Feature | Worker Threads | Cluster |
|---------|---------------|---------|
| Purpose | CPU-intensive tasks | Utilize multiple cores |
| Memory | Shared memory (SharedArrayBuffer) | Separate memory spaces |
| Communication | MessageChannel, SharedArrayBuffer | IPC (Inter-Process Communication) |
| Isolation | Threads within same process | Separate processes |
| Overhead | Lower | Higher |
| Use Case | Image processing, crypto | Scaling I/O operations |

**When to use what:**
- **Worker Threads:** CPU-heavy computations without blocking main thread
- **Cluster:** Scaling to handle more requests across multiple cores

---

## 4️⃣ Express.js & APIs

### Q15. What is Middleware in Express?

**Answer:**
Middleware functions have access to the request object (`req`), response object (`res`), and the next middleware function (`next`). They can:
- Execute code
- Modify req/res objects
- End the request-response cycle
- Call the next middleware

**Types of Middleware:**

1. **Application-level:**
```javascript
app.use((req, res, next) => {
  console.log('Time:', Date.now());
  next();
});
```

2. **Router-level:**
```javascript
const router = express.Router();
router.use((req, res, next) => {
  console.log('Router middleware');
  next();
});
```

3. **Error-handling:**
```javascript
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).send('Something broke!');
});
```

4. **Built-in:**
```javascript
app.use(express.json());
app.use(express.static('public'));
```

5. **Third-party:**
```javascript
const morgan = require('morgan');
app.use(morgan('dev'));
```

---

### Q16. Order of middleware execution?

**Answer:**
Middleware executes in the order it's defined (top to bottom).

**Example:**
```javascript
// 1. First middleware
app.use((req, res, next) => {
  console.log('First');
  next();
});

// 2. Second middleware
app.use('/api', (req, res, next) => {
  console.log('API Route');
  next();
});

// 3. Route handler
app.get('/api/users', (req, res) => {
  console.log('Users Route');
  res.send('Users');
});

// 4. Error middleware (must be last)
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).send('Error');
});

// Request to /api/users outputs: First, API Route, Users Route
```

**Important Rules:**
- Error middleware must have 4 parameters and come last
- Must call `next()` to continue to next middleware
- `res.send()` ends the cycle (don't call `next()` after)

---

### Q17. How do you handle errors globally in Express?

**Answer:**

**Error Handling Middleware:**
```javascript
// Must be defined AFTER all routes
app.use((err, req, res, next) => {
  console.error(err.stack);
  
  res.status(err.status || 500).json({
    error: {
      message: err.message,
      ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
    }
  });
});
```

**Creating Custom Error Classes:**
```javascript
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}

// Usage
app.get('/users/:id', (req, res, next) => {
  if (!user) {
    return next(new AppError('User not found', 404));
  }
  res.json(user);
});
```

**Handling Uncaught Exceptions:**
```javascript
process.on('uncaughtException', (err) => {
  console.error('Uncaught Exception:', err);
  process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  process.exit(1);
});
```

---

### Q18. How does Express handle async errors?

**Answer:**
Express (before v5) doesn't automatically catch errors in async route handlers. You must explicitly pass errors to `next()`.

**Problem:**
```javascript
// BAD: Error not caught
app.get('/users', async (req, res) => {
  const users = await User.find(); // If this throws, app crashes
  res.json(users);
});
```

**Solution 1: Try-Catch:**
```javascript
app.get('/users', async (req, res, next) => {
  try {
    const users = await User.find();
    res.json(users);
  } catch (err) {
    next(err);
  }
});
```

**Solution 2: Async Handler Wrapper:**
```javascript
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Usage
app.get('/users', asyncHandler(async (req, res) => {
  const users = await User.find();
  res.json(users);
}));
```

**Solution 3: Express 5 (Native Support):**
```javascript
// Express 5+ automatically catches async errors
app.get('/users', async (req, res) => {
  const users = await User.find();
  res.json(users);
});
```

---

## 5️⃣ Security

### Q19. How do you secure a Node.js application?

**Answer:**

**1. Use Helmet (Security Headers):**
```javascript
const helmet = require('helmet');
app.use(helmet());
// Sets various HTTP headers to secure app
```

**2. CORS Configuration:**
```javascript
const cors = require('cors');
app.use(cors({
  origin: 'https://trusted-domain.com',
  credentials: true,
  methods: ['GET', 'POST']
}));
```

**3. Rate Limiting:**
```javascript
const rateLimit = require('express-rate-limit');
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});
app.use('/api/', limiter);
```

**4. Input Validation:**
```javascript
const { body, validationResult } = require('express-validator');

app.post('/user', [
  body('email').isEmail(),
  body('password').isLength({ min: 8 })
], (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ errors: errors.array() });
  }
  // Process request
});
```

**5. Environment Variables:**
```javascript
require('dotenv').config();
// Never commit .env files
// Use process.env.SECRET_KEY
```

**6. SQL Injection Prevention:**
```javascript
// Use parameterized queries
db.query('SELECT * FROM users WHERE id = ?', [userId]);
```

**7. XSS Protection:**
```javascript
const xss = require('xss-clean');
app.use(xss());
```

**8. HTTPS:**
```javascript
const https = require('https');
const fs = require('fs');

const options = {
  key: fs.readFileSync('private-key.pem'),
  cert: fs.readFileSync('certificate.pem')
};

https.createServer(options, app).listen(443);
```

---

### Q20. How does JWT authentication work?

**Answer:**

**JWT Structure:** `header.payload.signature`

**Flow:**
1. User logs in with credentials
2. Server verifies and creates JWT
3. Server signs token with secret key
4. Client stores token (localStorage/cookie)
5. Client sends token in Authorization header
6. Server verifies token signature
7. If valid, grants access

**Implementation:**
```javascript
const jwt = require('jsonwebtoken');

// 1. Generate Token (Login)
app.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  
  if (!user || !await bcrypt.compare(password, user.password)) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  const token = jwt.sign(
    { userId: user._id, email: user.email },
    process.env.JWT_SECRET,
    { expiresIn: '24h' }
  );
  
  res.json({ token });
});

// 2. Verify Token (Middleware)
const authenticateToken = (req, res, next) => {
  const token = req.headers['authorization']?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
    if (err) {
      return res.status(403).json({ error: 'Invalid token' });
    }
    req.user = decoded;
    next();
  });
};

// 3. Protected Route
app.get('/profile', authenticateToken, (req, res) => {
  res.json({ user: req.user });
});
```

**Best Practices:**
- Store tokens securely (httpOnly cookies for web)
- Use short expiration times
- Implement refresh tokens
- Don't store sensitive data in payload (it's base64, not encrypted)
- Use strong secret keys

---

### Q21. How to prevent SQL/NoSQL injection?

**Answer:**

**SQL Injection Prevention:**

1. **Parameterized Queries:**
```javascript
// BAD: Vulnerable to SQL injection
const query = `SELECT * FROM users WHERE email = '${email}'`;

// GOOD: Parameterized
db.query('SELECT * FROM users WHERE email = ?', [email]);
```

2. **ORM (Sequelize, TypeORM):**
```javascript
const user = await User.findOne({ where: { email } });
```

**NoSQL Injection Prevention (MongoDB):**

1. **Input Validation:**
```javascript
// BAD: Vulnerable
db.collection.find({ username: req.body.username });

// Attack: { "username": { "$ne": null } } returns all users

// GOOD: Validate input type
if (typeof req.body.username !== 'string') {
  return res.status(400).send('Invalid input');
}
```

2. **ODM (Mongoose):**
```javascript
const UserSchema = new mongoose.Schema({
  email: { type: String, required: true }
});
// Mongoose automatically sanitizes
```

3. **Sanitization Libraries:**
```javascript
const mongoSanitize = require('express-mongo-sanitize');
app.use(mongoSanitize());
```

---

## 6️⃣ Streams & Buffers

### Q22. What are Streams in Node.js?

**Answer:**
Streams are objects that let you read/write data in chunks rather than loading everything into memory at once.

**Types of Streams:**

1. **Readable:** Read data from a source
2. **Writable:** Write data to a destination
3. **Duplex:** Both readable and writable
4. **Transform:** Modify data as it's being read/written

**Example:**
```javascript
const fs = require('fs');

// BAD: Loads entire file into memory
fs.readFile('large-file.txt', (err, data) => {
  console.log(data);
});

// GOOD: Streams data in chunks
const readStream = fs.createReadStream('large-file.txt');
readStream.on('data', (chunk) => {
  console.log('Chunk:', chunk.length);
});
```

**Piping Streams:**
```javascript
const fs = require('fs');
const zlib = require('zlib');

// Copy and compress file
fs.createReadStream('input.txt')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('input.txt.gz'));
```

**HTTP Response Stream:**
```javascript
app.get('/download', (req, res) => {
  const readStream = fs.createReadStream('large-video.mp4');
  res.setHeader('Content-Type', 'video/mp4');
  readStream.pipe(res);
});
```

**Use Cases:**
- File uploads/downloads
- Video/audio streaming
- Processing large datasets
- Reading log files

---

### Q23. Buffer vs Stream?

**Answer:**

| Feature | Buffer | Stream |
|---------|--------|--------|
| Memory | Loads entire data | Processes chunks |
| Size Limit | Limited by heap | No practical limit |
| Performance | Fast for small data | Efficient for large data |
| Use Case | Small files, images | Large files, video |

**Buffer Example:**
```javascript
const buffer = Buffer.from('Hello World');
console.log(buffer); // <Buffer 48 65 6c 6c 6f 20 57 6f 72 6c 64>
console.log(buffer.toString()); // Hello World

// Binary operations
buffer[0] = 0x48;
```

**When to use what:**
- **Buffer:** Small data (< 1MB), need random access, binary operations
- **Stream:** Large files, network data, real-time processing

---

## 7️⃣ System Design & Architecture

### Q24. How does Node handle 10k concurrent connections?

**Answer:**

**1. Non-Blocking I/O:**
```javascript
// Each request doesn't block others
app.get('/user/:id', async (req, res) => {
  const user = await db.findUser(req.params.id); // Non-blocking
  res.json(user);
});
```

**2. Event Loop:**
- Single thread handles all connections
- I/O operations delegated to OS/thread pool
- Callbacks executed when operations complete

**3. Connection Pooling:**
```javascript
const mongoose = require('mongoose');
mongoose.connect('mongodb://localhost/db', {
  maxPoolSize: 100, // Reuse connections
  minPoolSize: 10
});
```

**4. Horizontal Scaling:**
- Multiple Node instances behind load balancer
- Each instance handles portion of connections

**5. Caching:**
```javascript
const redis = require('redis');
const client = redis.createClient();

// Reduce DB load
app.get('/user/:id', async (req, res) => {
  const cached = await client.get(`user:${req.params.id}`);
  if (cached) return res.json(JSON.parse(cached));
  
  const user = await db.findUser(req.params.id);
  await client.set(`user:${req.params.id}`, JSON.stringify(user), 'EX', 3600);
  res.json(user);
});
```

**Why Node.js is good for this:**
- Low per-connection overhead
- Event-driven architecture
- Efficient I/O handling
- Small memory footprint per connection

---

### Q25. Where is Node.js NOT a good fit?

**Answer:**

**1. CPU-Intensive Operations:**
```javascript
// BAD: Blocks event loop
app.get('/calculate', (req, res) => {
  let result = 0;
  for (let i = 0; i < 10000000000; i++) {
    result += Math.sqrt(i);
  }
  res.json({ result });
});

// GOOD: Use Worker Threads
const { Worker } = require('worker_threads');
app.get('/calculate', (req, res) => {
  const worker = new Worker('./calculate-worker.js');
  worker.on('message', result => res.json({ result }));
});
```

**Not suitable for:**
- Video encoding
- Image processing (without workers)
- Complex mathematical calculations
- Machine learning inference
- Scientific computing

**Better alternatives:**
- Python (NumPy, TensorFlow)
- Go (concurrent, compiled)
- C++/Rust (system programming)

**2. Applications requiring:**
- Heavy computation
- Multi-threading by default
- Low-level system access
- Real-time performance guarantees

**Node.js excels at:**
- REST APIs
- Real-time applications (chat, notifications)
- Microservices
- Streaming applications
- I/O-bound operations

---

## 8️⃣ Advanced & Tricky Questions

### Q26. Can Node.js do multithreading?

**Answer:**
Yes, through **Worker Threads** and **Child Processes**.

**1. Worker Threads:**
```javascript
// main.js
const { Worker } = require('worker_threads');

function runWorker(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./worker.js', {
      workerData: data
    });
    
    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker stopped with exit code ${code}`));
    });
  });
}

const result = await runWorker({ num: 1000000 });

// worker.js
const { parentPort, workerData } = require('worker_threads');

let sum = 0;
for (let i = 0; i < workerData.num; i++) {
  sum += i;
}

parentPort.postMessage(sum);
```

**2. Child Processes:**
```javascript
const { fork } = require('child_process');

const child = fork('./child.js');
child.send({ task: 'heavy-computation' });
child.on('message', (result) => {
  console.log('Result:', result);
});
```

**Key Differences:**
- **Worker Threads:** Shared memory, lighter weight
- **Child Processes:** Isolated processes, heavier

---

### Q27. What happens if the event loop is blocked?

**Answer:**
If the event loop is blocked, the entire application freezes. No requests can be processed until the blocking operation completes.

**Example of Blocking:**
```javascript
app.get('/block', (req, res) => {
  // This blocks for 10 seconds
  const start = Date.now();
  while (Date.now() - start < 10000) {}
  res.send('Done');
});

// During these 10 seconds, ALL other requests are blocked
```

**Consequences:**
- No incoming requests handled
- No callbacks executed
- Application appears hung
- Timeouts occur
- Poor user experience

**How to Detect:**
```javascript
// Monitor event loop lag
const start = Date.now();
setInterval(() => {
  const lag = Date.now() - start - 1000;
  if (lag > 100) {
    console.warn(`Event loop lag: ${lag}ms`);
  }
  start = Date.now();
}, 1000);
```

**Solutions:**
- Offload to Worker Threads
- Break into smaller chunks with `setImmediate`
- Use async operations
- Profile with `--prof` flag

---

### Q28. Difference between `spawn`, `fork`, and `exec`?

**Answer:**

| Feature | `spawn` | `fork` | `exec` |
|---------|---------|--------|--------|
| Output | Stream (efficient) | Stream + IPC | Buffer (limited) |
| Use Case | Large output, long-running | Node child processes | Small output, shell commands |
| Communication | stdio streams | IPC channel + stdio | No built-in IPC |
| Memory | Low | Medium | High (buffers everything) |

**1. spawn - Stream Output:**
```javascript
const { spawn } = require('child_process');

const ls = spawn('ls', ['-lh', '/usr']);

ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});

ls.stderr.on('data', (data) => {
  console.error(`stderr: ${data}`);
});

ls.on('close', (code) => {
  console.log(`child process exited with code ${code}`);
});

// Good for: Large data, video processing, logs
```

**2. exec - Buffered Output:**
```javascript
const { exec } = require('child_process');

exec('ls -lh /usr', (error, stdout, stderr) => {
  if (error) {
    console.error(`exec error: ${error}`);
    return;
  }
  console.log(`stdout: ${stdout}`);
  console.error(`stderr: ${stderr}`);
});

// Good for: Quick commands, small output
// Danger: Can hit buffer limit (default 1MB)
```

**3. fork - Node.js Processes with IPC:**
```javascript
// parent.js
const { fork } = require('child_process');
const child = fork('./child.js');

child.on('message', (msg) => {
  console.log('Message from child:', msg);
});

child.send({ task: 'compute', data: [1,2,3] });

// child.js
process.on('message', (msg) => {
  if (msg.task === 'compute') {
    const result = msg.data.reduce((a, b) => a + b, 0);
    process.send({ result });
  }
});

// Good for: CPU-intensive Node tasks, microservices
```

**When to use what:**
- **spawn:** Long-running processes, large data, streaming
- **exec:** Quick shell commands, small output
- **fork:** Node.js child processes, need IPC communication

---

## 9️⃣ Real-World Scenarios

### Q29. How would you debug a memory leak?

**Answer:**

**Step 1: Detect the Leak**
```javascript
// Monitor memory over time
setInterval(() => {
  const used = process.memoryUsage();
  console.log({
    rss: `${Math.round(used.rss / 1024 / 1024)}MB`,
    heapUsed: `${Math.round(used.heapUsed / 1024 / 1024)}MB`,
    heapTotal: `${Math.round(used.heapTotal / 1024 / 1024)}MB`
  });
}, 5000);
```

**Step 2: Take Heap Snapshots**
```javascript
// Using heapdump
const heapdump = require('heapdump');

// Trigger on-demand
app.get('/heapdump', (req, res) => {
  heapdump.writeSnapshot((err, filename) => {
    res.send(`Snapshot written to ${filename}`);
  });
});

// Or automatically when memory threshold hit
const v8 = require('v8');
setInterval(() => {
  const heapStats = v8.getHeapStatistics();
  const usedPercent = (heapStats.used_heap_size / heapStats.heap_size_limit) * 100;
  
  if (usedPercent > 80) {
    heapdump.writeSnapshot(`./heap-${Date.now()}.heapsnapshot`);
  }
}, 60000);
```

**Step 3: Analyze with Chrome DevTools**
1. Load heap snapshot in Chrome DevTools
2. Compare snapshots over time
3. Look for objects growing consistently
4. Check "Retainers" to see what's holding references

**Step 4: Common Culprits to Check**
```javascript
// 1. Event listeners
// BAD
class MyClass {
  constructor() {
    setInterval(() => this.update(), 1000); // Never cleared!
  }
}

// GOOD
class MyClass {
  constructor() {
    this.interval = setInterval(() => this.update(), 1000);
  }
  destroy() {
    clearInterval(this.interval);
  }
}

// 2. Closures
// BAD
function createClosure() {
  const largeData = new Array(1000000);
  return function() {
    // Even if largeData isn't used, closure keeps it
    console.log('hello');
  };
}

// 3. Global variables
// BAD
global.cache = {}; // Never cleared

// 4. Unbounded arrays/objects
// BAD
const requests = [];
app.use((req, res, next) => {
  requests.push(req); // Grows forever
  next();
});
```

**Tools:**
- **heapdump / heap-profiler:** Capture snapshots
- **Chrome DevTools:** Analyze snapshots
- **clinic.js:** Complete diagnostics
- **node --inspect:** Live debugging
- **memwatch-next:** Detect leaks in real-time

**Step 5: Fix and Verify**
```javascript
// Before fix: Monitor memory
const beforeMem = process.memoryUsage().heapUsed;

// Simulate load
for (let i = 0; i < 1000; i++) {
  // Your code
}

// After load: Check if memory released
global.gc(); // Run with node --expose-gc
const afterMem = process.memoryUsage().heapUsed;
console.log(`Memory delta: ${(afterMem - beforeMem) / 1024 / 1024}MB`);
```

---

### Q30. How do you handle graceful shutdown?

**Answer:**

Graceful shutdown ensures:
- Ongoing requests complete
- Database connections close
- Resources are cleaned up
- No data corruption

**Implementation:**
```javascript
const express = require('express');
const app = express();

let server;
let isShuttingDown = false;

// Track active connections
const connections = new Set();

server = app.listen(3000, () => {
  console.log('Server running on port 3000');
});

// Track connections
server.on('connection', (conn) => {
  connections.add(conn);
  conn.on('close', () => {
    connections.remove(conn);
  });
});

// Middleware to reject new requests during shutdown
app.use((req, res, next) => {
  if (isShuttingDown) {
    res.set('Connection', 'close');
    return res.status(503).send('Server is shutting down');
  }
  next();
});

// Your routes
app.get('/', (req, res) => {
  res.send('Hello World');
});

// Graceful shutdown function
async function gracefulShutdown(signal) {
  console.log(`${signal} received: closing server gracefully`);
  isShuttingDown = true;

  // Stop accepting new connections
  server.close(() => {
    console.log('HTTP server closed');
  });

  // Close existing connections after timeout
  setTimeout(() => {
    console.log('Forcing connections to close');
    connections.forEach(conn => conn.destroy());
  }, 30000); // 30 second timeout

  try {
    // Close database connections
    await mongoose.connection.close();
    console.log('MongoDB connection closed');

    // Close Redis connection
    await redisClient.quit();
    console.log('Redis connection closed');

    // Close any other resources
    // ...

    console.log('Graceful shutdown complete');
    process.exit(0);
  } catch (err) {
    console.error('Error during shutdown:', err);
    process.exit(1);
  }
}

// Listen for termination signals
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// Handle uncaught errors
process.on('uncaughtException', (err) => {
  console.error('Uncaught Exception:', err);
  gracefulShutdown('uncaughtException');
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  gracefulShutdown('unhandledRejection');
});
```

**Production-Ready Version with Stoppable:**
```javascript
const stoppable = require('stoppable');

const server = stoppable(
  app.listen(3000),
  30000 // Grace period in ms
);

process.on('SIGTERM', () => {
  console.log('SIGTERM received');
  server.stop((err) => {
    if (err) {
      console.error('Error during shutdown:', err);
      process.exit(1);
    }
    // Close other resources
    mongoose.connection.close();
    process.exit(0);
  });
});
```

**Docker/Kubernetes Considerations:**
```yaml
# kubernetes deployment
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
        # Gives time for load balancer to de-register
      terminationGracePeriodSeconds: 30
```

**Best Practices:**
1. Set reasonable timeout (30s typical)
2. Stop accepting new requests immediately
3. Wait for ongoing requests to complete
4. Close database/cache connections
5. Log shutdown stages
6. Exit with appropriate code (0 = success, 1 = error)

---

## 🔟 Performance & Monitoring

### Q31. How do you monitor Node.js applications in production?

**Answer:**

**1. Application Performance Monitoring (APM):**
```javascript
// New Relic
require('newrelic');

// Datadog
const tracer = require('dd-trace').init();

// AppDynamics
require('appdynamics').profile({
  controllerHostName: 'controller.example.com',
  accountName: 'customer1',
  accountAccessKey: 'key'
});
```

**2. Custom Metrics with Prometheus:**
```javascript
const client = require('prom-client');
const express = require('express');
const app = express();

// Create metrics
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code']
});

const activeConnections = new client.Gauge({
  name: 'active_connections',
  help: 'Number of active connections'
});

// Middleware to track metrics
app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestDuration
      .labels(req.method, req.route?.path || req.path, res.statusCode)
      .observe(duration);
  });
  
  next();
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

**3. Health Checks:**
```javascript
app.get('/health', async (req, res) => {
  const health = {
    uptime: process.uptime(),
    timestamp: Date.now(),
    status: 'ok'
  };

  try {
    // Check database
    await mongoose.connection.db.admin().ping();
    health.database = 'connected';

    // Check Redis
    await redisClient.ping();
    health.redis = 'connected';

    // Check memory
    const memUsage = process.memoryUsage();
    health.memory = {
      heapUsed: Math.round(memUsage.heapUsed / 1024 / 1024),
      heapTotal: Math.round(memUsage.heapTotal / 1024 / 1024)
    };

    res.status(200).json(health);
  } catch (err) {
    health.status = 'error';
    health.error = err.message;
    res.status(503).json(health);
  }
});

// Liveness probe (is app running?)
app.get('/healthz', (req, res) => {
  res.status(200).send('OK');
});

// Readiness probe (is app ready to serve?)
app.get('/readyz', async (req, res) => {
  if (isShuttingDown) {
    return res.status(503).send('Shutting down');
  }
  // Check if dependencies are ready
  res.status(200).send('Ready');
});
```

**4. Logging Best Practices:**
```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { 
    service: 'my-app',
    environment: process.env.NODE_ENV 
  },
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

// Add console in development
if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple()
  }));
}

// Usage
logger.info('User logged in', { userId: 123 });
logger.error('Database connection failed', { error: err.message });
```

**5. Error Tracking (Sentry):**
```javascript
const Sentry = require('@sentry/node');

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1 // 10% of transactions
});

// Error handler middleware
app.use(Sentry.Handlers.errorHandler());
```

**Key Metrics to Monitor:**
- **Response Time:** P50, P95, P99
- **Error Rate:** 4xx, 5xx responses
- **Throughput:** Requests per second
- **Memory Usage:** Heap, RSS
- **Event Loop Lag:** Should be < 50ms
- **CPU Usage:** Should be < 80%
- **Active Connections:** Track connection pool
- **Database Query Time:** Slow queries

---

### Q32. How do you optimize Node.js performance?

**Answer:**

**1. Use Clustering:**
```javascript
const cluster = require('cluster');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
} else {
  require('./app');
}
```

**2. Enable Compression:**
```javascript
const compression = require('compression');
app.use(compression());
```

**3. Caching Strategies:**
```javascript
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 600 });

app.get('/user/:id', async (req, res) => {
  const cacheKey = `user:${req.params.id}`;
  
  // Check cache first
  const cached = cache.get(cacheKey);
  if (cached) {
    return res.json(cached);
  }
  
  // Fetch from DB
  const user = await User.findById(req.params.id);
  
  // Store in cache
  cache.set(cacheKey, user);
  res.json(user);
});
```

**4. Database Query Optimization:**
```javascript
// BAD: N+1 query problem
const users = await User.find();
for (let user of users) {
  user.posts = await Post.find({ userId: user.id });
}

// GOOD: Use joins/populate
const users = await User.find().populate('posts');

// Use indexes
userSchema.index({ email: 1 });
userSchema.index({ createdAt: -1, status: 1 });

// Use lean() for read-only data
const users = await User.find().lean(); // Returns plain JS objects
```

**5. Connection Pooling:**
```javascript
mongoose.connect('mongodb://localhost/db', {
  maxPoolSize: 10,
  minPoolSize: 5,
  socketTimeoutMS: 45000
});
```

**6. Async Operations:**
```javascript
// BAD: Sequential
const user = await getUser(id);
const posts = await getPosts(userId);
const comments = await getComments(userId);

// GOOD: Parallel
const [user, posts, comments] = await Promise.all([
  getUser(id),
  getPosts(userId),
  getComments(userId)
]);
```

**7. Stream Large Data:**
```javascript
// BAD: Load entire file
const data = fs.readFileSync('large-file.csv');
res.send(data);

// GOOD: Stream
const stream = fs.createReadStream('large-file.csv');
stream.pipe(res);
```

**8. Use Native Modules:**
```javascript
// Use fast-json-stringify instead of JSON.stringify
const fastJson = require('fast-json-stringify');
const stringify = fastJson({
  type: 'object',
  properties: {
    name: { type: 'string' },
    age: { type: 'integer' }
  }
});
```

**9. Avoid Blocking Operations:**
```javascript
// BAD: Synchronous crypto
const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');

// GOOD: Async crypto
const hash = await new Promise((resolve, reject) => {
  crypto.pbkdf2(password, salt, 100000, 64, 'sha512', (err, derivedKey) => {
    if (err) reject(err);
    resolve(derivedKey.toString('hex'));
  });
});
```

**10. Profile and Optimize:**
```bash
# Generate CPU profile
node --prof app.js

# Process profile
node --prof-process isolate-*.log > processed.txt

# Use clinic.js
npx clinic doctor -- node app.js
npx clinic flame -- node app.js
```

---

## 1️⃣1️⃣ Testing

### Q33. How do you test Node.js applications?

**Answer:**

**1. Unit Testing (Jest/Mocha):**
```javascript
// user.service.js
class UserService {
  async createUser(userData) {
    // Validate
    if (!userData.email) {
      throw new Error('Email required');
    }
    // Create user
    return await User.create(userData);
  }
}

// user.service.test.js
const UserService = require('./user.service');

describe('UserService', () => {
  let userService;
  
  beforeEach(() => {
    userService = new UserService();
  });
  
  describe('createUser', () => {
    it('should create user with valid data', async () => {
      const userData = { email: 'test@example.com', name: 'Test' };
      const user = await userService.createUser(userData);
      
      expect(user.email).toBe('test@example.com');
      expect(user.name).toBe('Test');
    });
    
    it('should throw error if email missing', async () => {
      const userData = { name: 'Test' };
      
      await expect(userService.createUser(userData))
        .rejects.toThrow('Email required');
    });
  });
});
```

**2. Integration Testing (Supertest):**
```javascript
const request = require('supertest');
const app = require('./app');

describe('POST /api/users', () => {
  it('should create a new user', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({
        email: 'test@example.com',
        password: 'password123'
      })
      .expect(201);
    
    expect(response.body).toHaveProperty('id');
    expect(response.body.email).toBe('test@example.com');
  });
  
  it('should return 400 for invalid email', async () => {
    await request(app)
      .post('/api/users')
      .send({ email: 'invalid', password: 'password123' })
      .expect(400);
  });
});
```

**3. Mocking (Jest):**
```javascript
// Mock database
jest.mock('./database');
const db = require('./database');

it('should fetch user from database', async () => {
  // Setup mock
  db.findUser.mockResolvedValue({ id: 1, name: 'Test' });
  
  const user = await userService.getUser(1);
  
  expect(db.findUser).toHaveBeenCalledWith(1);
  expect(user.name).toBe('Test');
});

// Mock external API
jest.mock('axios');
const axios = require('axios');

it('should fetch data from API', async () => {
  axios.get.mockResolvedValue({ data: { result: 'success' } });
  
  const result = await apiService.fetchData();
  
  expect(result).toBe('success');
});
```

**4. E2E Testing (Cypress/Playwright):**
```javascript
// cypress/integration/auth.spec.js
describe('Authentication', () => {
  it('should login successfully', () => {
    cy.visit('/login');
    cy.get('input[name="email"]').type('user@example.com');
    cy.get('input[name="password"]').type('password123');
    cy.get('button[type="submit"]').click();
    
    cy.url().should('include', '/dashboard');
    cy.contains('Welcome back');
  });
});
```

**5. Test Coverage:**
```bash
# Jest
npm test -- --coverage

# NYC (Istanbul)
nyc mocha test/**/*.js
```

**Best Practices:**
- Test business logic, not implementation details
- Use beforeEach/afterEach for setup/teardown
- Mock external dependencies
- Aim for 80%+ code coverage
- Write tests before fixing bugs (TDD)
- Use descriptive test names

---

## 1️⃣2️⃣ Bonus: Common Mistakes & Best Practices

### Q34. Common Node.js mistakes to avoid?

**Answer:**

**1. Not Handling Async Errors:**
```javascript
// BAD
app.get('/users', async (req, res) => {
  const users = await User.find(); // Unhandled rejection
  res.json(users);
});

// GOOD
app.get('/users', async (req, res, next) => {
  try {
    const users = await User.find();
    res.json(users);
  } catch (err) {
    next(err);
  }
});
```

**2. Callback Hell:**
```javascript
// BAD
getData(function(a) {
  getMoreData(a, function(b) {
    getMoreData(b, function(c) {
      // Pyramid of doom
    });
  });
});

// GOOD
const a = await getData();
const b = await getMoreData(a);
const c = await getMoreData(b);
```

**3. Blocking the Event Loop:**
```javascript
// BAD
app.get('/users', (req, res) => {
  const users = JSON.parse(fs.readFileSync('users.json'));
  res.json(users);
});

// GOOD
app.get('/users', async (req, res) => {
  const users = JSON.parse(await fs.promises.readFile('users.json', 'utf8'));
  res.json(users);
});
```

**4. Not Using Environment Variables:**
```javascript
// BAD
const dbUrl = 'mongodb://localhost:27017/myapp';

// GOOD
require('dotenv').config();
const dbUrl = process.env.DATABASE_URL;
```

**5. Ignoring Error-First Callbacks:**
```javascript
// BAD
fs.readFile('file.txt', (data) => {
  console.log(data);
});

// GOOD
fs.readFile('file.txt', (err, data) => {
  if (err) return console.error(err);
  console.log(data);
});
```

**6. Multiple Callbacks:**
```javascript
// BAD
app.get('/user/:id', (req, res) => {
  User.findById(req.params.id, (err, user) => {
    if (err) return res.status(500).send(err);
    if (!user) return res.status(404).send('Not found');
    res.json(user);
    // Oops, forgot to return!
    res.send('More data'); // Error: Can't set headers after sent
  });
});

// GOOD: Always return
app.get('/user/:id', (req, res) => {
  User.findById(req.params.id, (err, user) => {
    if (err) return res.status(500).send(err);
    if (!user) return res.status(404).send('Not found');
    return res.json(user);
  });
});
```

**7. Not Validating Input:**
```javascript
// BAD
app.post('/user', (req, res) => {
  const user = new User(req.body);
  user.save();
});

// GOOD
const { body, validationResult } = require('express-validator');

app.post('/user', [
  body('email').isEmail().normalizeEmail(),
  body('age').isInt({ min: 0, max: 120 })
], (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ errors: errors.array() });
  }
  // Process valid data
});
```

**8. Ignoring Security Headers:**
```javascript
// BAD
app.use(express.json());

// GOOD
const helmet = require('helmet');
app.use(helmet());
app.use(express.json({ limit: '10kb' }));
```

---

## 🎯 Interview Tips

### How to Answer Node.js Interview Questions:

1. **Start with the basics**, then go deeper if asked
2. **Provide code examples** when relevant
3. **Mention trade-offs** and alternatives
4. **Discuss real-world scenarios** from your experience
5. **Ask clarifying questions** before complex answers
6. **Be honest** if you don't know something

### Topics to Master:

✅ Event Loop & Async Programming
✅ Streams & Buffers  
✅ Memory Management
✅ Scaling Strategies
✅ Security Best Practices
✅ Error Handling
✅ Performance Optimization
✅ Testing Strategies

### Quick Reference for Interviews:

**Event Loop Phases:** Timers → I/O → Poll → Check → Close  
**nextTick vs Promise:** nextTick runs first  
**spawn vs exec vs fork:** Stream vs Buffer vs IPC  
**Cluster vs Worker Threads:** Processes vs Threads  
**When NOT to use Node:** CPU-intensive tasks  

---

## 📚 Further Reading

- [Node.js Official Docs](https://nodejs.org/docs)
- [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)
- [You Don't Know Node](https://github.com/azat-co/you-dont-know-node)
- [Node.js Design Patterns](https://www.nodejsdesignpatterns.com/)

---

**Good luck with your interviews! 🚀**
