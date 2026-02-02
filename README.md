# 🚀 Node.js Event Loop & API Flow - Complete Learning Guide

> **Master Node.js Event Loop, Blocking vs Non-Blocking Operations, Worker Threads, and Complete API Request Flow**

This repository contains comprehensive guides, live demonstrations, and practical examples to help you understand how Node.js works under the hood - from event loop internals to complete API request flow including DNS, TCP, TLS/SSL certificates, and HTTP protocol.

---

## 📚 Table of Contents

1. [Event Loop Fundamentals](#1-event-loop-fundamentals)
2. [Blocking vs Non-Blocking](#2-blocking-vs-non-blocking)
3. [Event Loop Phases](#3-event-loop-phases)
4. [Event Loop Priorities Explained](#4-event-loop-priorities-explained)
5. [Worker Threads Solution](#5-worker-threads-solution)
6. [Complete API Request Flow](#6-complete-api-request-flow)
7. [Quick Start](#quick-start)
8. [All Demonstrations](#all-demonstrations)
9. [Key Takeaways](#key-takeaways)

---

## 🎯 What You'll Learn

By studying this repository, you will master:

✅ **Event Loop Architecture** - How Node.js handles millions of requests with a single thread
✅ **Blocking Operations** - Why they're dangerous and how to identify them
✅ **Non-Blocking Patterns** - Best practices for scalable applications
✅ **Event Loop Phases** - Deep dive into all 6 phases, especially Poll phase
✅ **Worker Threads** - How to handle CPU-intensive tasks without blocking
✅ **Complete API Flow** - DNS, TCP, TLS/SSL, HTTP from start to finish
✅ **Performance Optimization** - Making your Node.js apps lightning fast

---

## 1. Event Loop Fundamentals

### What is Event Loop?

**Simple Definition:** Event Loop is Node.js's way of handling multiple operations without creating multiple threads.

**Key Concept:**
- 1 Node.js Process = 1 Event Loop
- 1 Event Loop handles ALL requests (even millions!)
- Single-threaded JavaScript execution
- Multi-threaded I/O operations (background)

### Architecture

```
Million Requests → Single Event Loop → Thread Pool (4-128 threads)
```

**Restaurant Analogy:**
- 1 Waiter (Event Loop) serves all tables
- Kitchen Staff (Thread Pool) does the cooking
- Waiter never waits for food - just takes orders and delivers

### 📖 Read More
- **[EVENT_LOOP_GUIDE.md](./EVENT_LOOP_GUIDE.md)** - Complete architecture guide
- **[event-loop-architecture.ts](./event-loop-architecture.ts)** - Live demo (Run it!)

### 🎮 Try It
```bash
npx ts-node event-loop-architecture.ts
```

---

## 2. Blocking vs Non-Blocking

### The Problem

**Blocking Code:**
```typescript
// ❌ BAD - Blocks entire server!
const data = fs.readFileSync('file.txt');  // ALL users wait here!
```

**Non-Blocking Code:**
```typescript
// ✅ GOOD - Server stays responsive!
const data = await fs.promises.readFile('file.txt');  // Other users continue!
```

### Why It Matters

When you use blocking operations:
- Event loop gets stuck
- All other requests must wait
- Server becomes unresponsive
- Poor user experience

### Real-World Impact

**Without Blocking:**
- 10 users → All served in ~100ms

**With Blocking (1 second block):**
- 10 users → User 1: 1s, User 2: 2s, User 3: 3s... User 10: 10s!

### 📖 Read More
- **[blocking-demo.ts](./blocking-demo.ts)** - See the problem in action!

### 🎮 Try It
```bash
npx ts-node blocking-demo.ts
```

**Watch:** How blocking code delays promises and I/O operations.

---

## 3. Event Loop Phases

### The 6 Phases

Node.js Event Loop has 6 phases that execute in order:

```
1. Timers          → setTimeout(), setInterval()
2. Pending         → I/O callbacks
3. Idle/Prepare    → Internal use
4. Poll ⭐         → I/O operations (MOST IMPORTANT!)
5. Check           → setImmediate()
6. Close           → socket.close()
```

### Poll Phase - The Heart of Event Loop

**What it does:**
- Handles file I/O (`fs.readFile`)
- Handles network I/O (`http.get`)
- Handles database queries
- Spends most time here
- Decides when to move to next phase

**Example:**
```typescript
fs.readFile('file.txt', callback);  // Executes in Poll phase
http.get('http://api.com', callback);  // Executes in Poll phase
```

### Execution Order

```typescript
console.log('1. Synchronous code');

setTimeout(() => console.log('3. Timers phase'), 0);
setImmediate(() => console.log('4. Check phase'));
Promise.resolve().then(() => console.log('2. Microtask'));

// Output:
// 1. Synchronous code
// 2. Microtask (after sync, before phases)
// 3. Timers phase
// 4. Check phase
```

### 📖 Read More
- **[EVENT_LOOP_PHASES.md](./EVENT_LOOP_PHASES.md)** - Complete phase guide
- **[poll-phase-demo.ts](./poll-phase-demo.ts)** - Live demonstration

### 🎮 Try It
```bash
npx ts-node poll-phase-demo.ts
```

**Watch:** Execution order across all phases!

---

## 4. Event Loop Priorities Explained

"Who runs first?" is a common interview question. Let's understand the **VIP Priority System**.

> **Concept:** Not all tasks are created equal. Some are VIPs (run immediately), and some are Regulars (wait in line).

### The Hierarchy (Highest to Lowest)

1.  **👑 `process.nextTick` (Super VIP)**
    *   Runs **immediately** after the current operation finishes.
    *   Before anything else (even Promises!).
    *   *Analogy: "Stop everything, do this NOW."*

2.  **⭐ `Promise` (VIP)**
    *   Runs after `process.nextTick` but before the next Event Loop phase.
    *   Known as "Microtasks".
    *   *Analogy: "Do this before you move to the next customer."*

3.  **🕰️ `setTimeout` / `setInterval` (Timers)**
    *   Runs in the **Timers Phase**.
    *   Standard priority.
    *   *Analogy: "I have an appointment at this time."*

4.  **✅ `setImmediate` (Check)**
    *   Runs in the **Check Phase** (usually after I/O).
    *   *Analogy: "Do this as soon as you're done dealing with I/O."*

### ⚡ The Ultimate Priority Test

Copy-paste this code to see who wins!

```typescript
console.log("1. Script Start");

// 🕰️ Timer
setTimeout(() => {
  console.log("5. setTimeout (Macrotask)");
}, 0);

// ✅ Check
setImmediate(() => {
  console.log("6. setImmediate (Check Phase)");
});

// ⭐ Promise
Promise.resolve().then(() => {
  console.log("4. Promise (Microtask)");
});

// 👑 Next Tick
process.nextTick(() => {
  console.log("3. nextTick (Super VIP)");
});

console.log("2. Script End");
```

### The Output Order
```text
1. Script Start           (Synchronous code runs first)
2. Script End             (Synchronous code finishes)
3. nextTick (Super VIP)   (Runs immediately after sync code)
4. Promise (Microtask)    (Runs after nextTick)
5. setTimeout (Macrotask) (Timers phase)
6. setImmediate (Check)   (Check phase)
```

> **Note:** The order between `setTimeout` and `setImmediate` can vary if not inside an I/O cycle, but `nextTick` and `Promise` will **ALWAYS** be first!

---

## 5. Worker Threads Solution

### The Problem

CPU-intensive tasks block the event loop:

```typescript
// ❌ This blocks everything!
for (let i = 0; i < 1e9; i++) {
  // Heavy computation
}
// Promises and I/O must wait!
```

### The Solution: Worker Threads

**Concept:** Move heavy computation to a separate thread!

**Main Thread (stays free):**
```typescript
import { Worker } from 'worker_threads';

const worker = new Worker('./worker.ts');
worker.on('message', (result) => {
  console.log('Done:', result);
});
// Main thread continues immediately!
```

**Worker Thread (does heavy work):**
```typescript
import { parentPort } from 'worker_threads';

let result = 0;
for (let i = 0; i < 1e9; i++) {
  result += i;  // Heavy computation in separate thread
}
parentPort?.postMessage(result);
```

### Results Comparison

**Without Worker Threads:**
```
Promise resolved after 477ms  ❌ (blocked)
```

**With Worker Threads:**
```
Promise resolved after 4ms    ✅ (immediate!)
```

### 📖 Read More
- **[WORKER_THREADS_COMPARISON.md](./WORKER_THREADS_COMPARISON.md)** - Complete comparison
- **[task-with-worker.ts](./task-with-worker.ts)** - Worker solution
- **[worker.ts](./worker.ts)** - Worker script
- **[comparison-demo.ts](./comparison-demo.ts)** - Side-by-side demo

### 🎮 Try It
```bash
# See blocking behavior
npx ts-node task.ts

# See Worker Threads solution
npx ts-node task-with-worker.ts

# See side-by-side comparison
npx ts-node comparison-demo.ts
```

---

## 6. Complete API Request Flow

### The Journey: Browser to Server

When you make an API request, here's what happens:

```
You type: https://api.example.com/users

Step 1: DNS Resolution (20-100ms)
        api.example.com → 192.168.1.100

Step 2: TCP Connection (50-200ms)
        3-way handshake: SYN → SYN-ACK → ACK

Step 3: TLS/SSL Handshake (100-300ms)
        Certificate verification + Encryption setup

Step 4: HTTP Request (10-50ms)
        GET /users with headers

Step 5: Server Processing (50-500ms)
        Auth → Logic → Database → Response

Step 6: HTTP Response (10-50ms)
        200 OK with JSON data

Step 7: Connection Close/Keep-Alive
        Reuse or close connection
```

### DNS (Domain Name System)

**Role:** Converts domain names to IP addresses

**Process:**
1. Browser cache
2. OS cache
3. Router cache
4. ISP DNS server
5. Root DNS servers
6. TLD servers (.com, .org)
7. Authoritative name servers

**Result:** `api.example.com` → `192.168.1.100`

### TCP (Transmission Control Protocol)

**Role:** Establishes reliable connection

**3-Way Handshake:**
1. Client → Server: SYN (I want to connect)
2. Server → Client: SYN-ACK (OK, confirm)
3. Client → Server: ACK (Confirmed!)

**Port:** 443 (HTTPS), 80 (HTTP)

### TLS/SSL (Transport Layer Security)

**Role:** Encrypts all communication

**SSL Certificate:**
- Proves server identity (like an ID card)
- Contains: domain name, public key, expiry date
- Signed by Certificate Authority (CA)

**Common CAs:**
- Let's Encrypt (Free)
- DigiCert (Paid)
- Comodo (Paid)

**Process:**
1. Client Hello (I support TLS 1.3)
2. Server Hello + Certificate
3. Client verifies certificate
4. Session key exchange
5. Encrypted connection established! 🔒

### HTTP/HTTPS Protocol

**Request Structure:**
```http
GET /users HTTP/1.1
Host: api.example.com
Authorization: Bearer token...
Content-Type: application/json
```

**Response Structure:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{"users": [...]}
```

**Status Codes:**
- `2xx` - Success (200 OK, 201 Created)
- `4xx` - Client Error (404 Not Found)
- `5xx` - Server Error (500 Internal Error)

### HTTP vs HTTPS

| Feature | HTTP | HTTPS |
|---------|------|-------|
| Port | 80 | 443 |
| Security | ❌ Plain text | ✅ Encrypted |
| Certificate | Not needed | Required |
| Browser | "Not Secure" | 🔒 Padlock |

### 📖 Read More
- **[API_REQUEST_FLOW.md](./API_REQUEST_FLOW.md)** - Complete detailed guide
- **[API_FLOW_QUICK_REFERENCE.md](./API_FLOW_QUICK_REFERENCE.md)** - Quick reference
- **[api-flow-demo.ts](./api-flow-demo.ts)** - Live demonstration

### 🎮 Try It
```bash
npx ts-node api-flow-demo.ts
```

**Watch:** Real DNS lookup, certificate verification, and complete request flow!

---

## Quick Start

### Installation

```bash
npm install
```

### Run Any Demo

```bash
# Event Loop Basics
npx ts-node event-loop-architecture.ts

# Blocking vs Non-Blocking
npx ts-node blocking-demo.ts

# Event Loop Phases
npx ts-node poll-phase-demo.ts

# Worker Threads
npx ts-node task.ts                    # See the problem
npx ts-node task-with-worker.ts        # See the solution
npx ts-node comparison-demo.ts         # See side-by-side

# API Request Flow
npx ts-node api-flow-demo.ts
```

---

## All Demonstrations

### 📁 Event Loop

| File | Description | Run Command |
|------|-------------|-------------|
| `event-loop-architecture.ts` | Shows how single event loop handles multiple requests | `npx ts-node event-loop-architecture.ts` |
| `EVENT_LOOP_GUIDE.md` | Complete architecture documentation | Read in editor |

### 📁 Blocking Operations

| File | Description | Run Command |
|------|-------------|-------------|
| `blocking-demo.ts` | Demonstrates blocking vs non-blocking I/O | `npx ts-node blocking-demo.ts` |
| `task.ts` | Shows how CPU tasks block promises | `npx ts-node task.ts` |

### 📁 Event Loop Phases

| File | Description | Run Command |
|------|-------------|-------------|
| `poll-phase-demo.ts` | Live demo of all 6 phases | `npx ts-node poll-phase-demo.ts` |
| `EVENT_LOOP_PHASES.md` | Complete phase documentation | Read in editor |

### 📁 Worker Threads

| File | Description | Run Command |
|------|-------------|-------------|
| `task-with-worker.ts` | Worker Threads solution | `npx ts-node task-with-worker.ts` |
| `worker.ts` | Worker thread script | Used by task-with-worker.ts |
| `comparison-demo.ts` | Side-by-side comparison | `npx ts-node comparison-demo.ts` |
| `WORKER_THREADS_COMPARISON.md` | Complete comparison guide | Read in editor |

### 📁 API Request Flow

| File | Description | Run Command |
|------|-------------|-------------|
| `api-flow-demo.ts` | Live API request demonstration | `npx ts-node api-flow-demo.ts` |
| `API_REQUEST_FLOW.md` | Complete detailed guide | Read in editor |
| `API_FLOW_QUICK_REFERENCE.md` | Quick reference | Read in editor |

---

## Key Takeaways

### ✅ Event Loop

1. **Single event loop handles millions of requests**
2. **Event loop delegates I/O to thread pool**
3. **Never block the event loop!**
4. **Use async/await for all I/O operations**

### ✅ Blocking Operations

1. **Blocking = Event loop stuck = All users wait**
2. **Always use async versions**: `fs.readFile` not `fs.readFileSync`
3. **Avoid long synchronous loops**
4. **Use Worker Threads for CPU-intensive tasks**

### ✅ Event Loop Phases

1. **6 phases execute in order**
2. **Poll phase handles most I/O operations**
3. **Microtasks (Promises, nextTick) execute after each phase**
4. **Understanding phases helps debug timing issues**

### ✅ Worker Threads

1. **Use for CPU-intensive tasks only**
2. **Each worker = separate thread = separate CPU core**
3. **Main thread stays free for requests**
4. **Don't use for I/O (already non-blocking)**

### ✅ API Request Flow

1. **7 steps from browser to server**
2. **DNS converts domain → IP**
3. **TCP establishes connection**
4. **TLS/SSL encrypts communication**
5. **HTTP carries actual data**
6. **Certificates verify server identity**
7. **HTTPS is mandatory for production**

---

## 🎓 Learning Path

**For Beginners:**
1. Start with [EVENT_LOOP_GUIDE.md](./EVENT_LOOP_GUIDE.md)
2. Run `event-loop-architecture.ts` demo
3. Read [blocking-demo.ts](./blocking-demo.ts) code
4. Understand the problem!

**Intermediate:**
1. Study [EVENT_LOOP_PHASES.md](./EVENT_LOOP_PHASES.md)
2. Run `poll-phase-demo.ts`
3. Learn execution order

**Advanced:**
1. Master Worker Threads with [WORKER_THREADS_COMPARISON.md](./WORKER_THREADS_COMPARISON.md)
2. Understand complete API flow with [API_REQUEST_FLOW.md](./API_REQUEST_FLOW.md)
3. Run all demos and understand output

---

## 🚀 Production Best Practices

### Do's ✅

- Use `async/await` for all I/O operations
- Use Worker Threads for heavy computations
- Implement proper error handling
- Use connection pooling for databases
- Enable HTTP/2 for better performance
- Use Keep-Alive connections
- Cache DNS lookups
- Monitor event loop lag

### Don'ts ❌

- Don't use synchronous I/O (`fs.readFileSync`)
- Don't block event loop with long loops
- Don't use Worker Threads for I/O
- Don't ignore SSL certificate validation
- Don't use HTTP in production (use HTTPS)
- Don't create too many Worker Threads

---

## 📝 Additional Resources

### Official Documentation
- [Node.js Event Loop](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- [Worker Threads](https://nodejs.org/api/worker_threads.html)
- [DNS Module](https://nodejs.org/api/dns.html)

### Further Reading
- libuv Design Overview
- Understanding the V8 Engine
- HTTP/2 vs HTTP/1.1
- SSL/TLS Deep Dive

---

## 🙏 Summary

This repository is your **complete guide** to mastering:
- ⚡ Node.js Event Loop
- 🔄 Async/Blocking Operations
- 🧵 Worker Threads
- 🌐 Complete API Request Flow

**Study all files in order, run all demos, and you'll become a master!** 🚀

---

## License

Nest is [MIT licensed](https://github.com/nestjs/nest/blob/master/LICENSE).

---

**Made with ❤️ for learning Node.js internals**
