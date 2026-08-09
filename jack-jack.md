# AlgoCode — Complete Architecture Deep-Dive

> Microservices-based code submission & evaluation platform  
> **4 Services** | **BullMQ** | **Docker** | **MongoDB Atlas** | **Redis** | **JWT Auth**

---

## Table of Contents

1. [High-Level Platform Architecture](#1-high-level-platform-architecture)
2. [Service Interaction Map (End-to-End Flow)](#2-end-to-end-flow)
3. [Docker Deep-Dive](#3-docker-deep-dive-evaluator-service)
4. [BullMQ Deep-Dive](#4-bullmq-deep-dive-both-services)
5. [User-Service Deep-Dive](#5-user-service-port-6000)
6. [Problem-Service Deep-Dive](#6-problem-service-port-3000)
7. [Submission-Service Deep-Dive](#7-submission-service-port-5000)
8. [Evaluator-Service Deep-Dive](#8-evaluator-service-port-4000)
9. [Known Issues & Gaps](#9-known-issues--gaps)
10. [Selected Interview Q&A](#10-key-interview-questions--answers)

---

## 1. High-Level Platform Architecture

```
┌──────────────┐     ┌──────────────┐     ┌───────────────┐     ┌──────────────────┐
│  User-Service │     │ Problem-Svc  │     │Submission-Svc │     │ Evaluator-Svc    │
│  (Express)    │     │ (Express)    │     │ (Fastify)     │     │ (Express + TS)   │
│  Port: 6000   │     │ Port: 3000   │     │ Port: 5000    │     │ Port: 4000       │
│               │     │              │     │               │     │                  │
│ Auth + Users  │     │ CRUD Probs.  │     │ Submissions   │     │ Dockerized       │
│ JWT + Refresh │     │ Markdown     │     │ Queue Prod    │     │ Code Execution   │
│ AWS S3 Avatars│     │ Sanitization │     │ Webhook       │     │ Bull Board UI    │
└──────┬────────┘     └──────┬───────┘     └───────┬───────┘     └────────┬─────────┘
       │                     │                     │                      │
       │    MongoDB          │    MongoDB          │    MongoDB           │  Docker Socket
       │    (Atlas)          │    (Atlas)          │    (Atlas)           │  + Docker Images
       │                     │                     │                      │
       │                     │      ┌──────────────┘                      │
       │                     │      │  axios call to fetch                │
       │                     │      │  problem details                    │
       │                     │      ▼                                     │
       │                     │   Submission-Service                       │
       │                     │   fetches testCases,                       │
       │                     │   codeStubs from                           │
       │                     │   Problem-Service                          │
       │                     │                                            │
       └─────────────────────┴────────────────────────────────────────────┘
                                        │
                                        │  Redis (BullMQ)
                                        │  Queue: "SubmissionQueue"
                                        │  Job: "SubmissionJob"
                                        │
                        ┌───────────────┴───────────────┐
                        │                               │
                   Producer Side                  Consumer Side
                   (Submission-Svc)              (Evaluator-Svc)
                        │                               │
                        │  adds job with:               │  Worker picks up,
                        │  - attempts: 3                │  executes in Docker
                        │  - exponential backoff(2s)    │  containers, sends
                        │  - removeOnComplete: true     │  webhook back to
                        │  - removeOnFail: false        │  Submission-Service
                        ▼                               ▼
```

### Service Matrix

| Service | Language | Framework | DB | Port | Key Libs |
|---------|----------|-----------|----|------|----------|
| **User-Service** | JavaScript | Express 5 | MongoDB Atlas | 6000 | JWT, bcryptjs, AWS SDK S3, Winston, Multer |
| **Problem-Service** | JavaScript | Express 5 | MongoDB Atlas | 3000 | Zod, marked, sanitize-html, turndown, Winston |
| **Submission-Service** | JavaScript | Fastify 4 | MongoDB Atlas | 5000 | BullMQ, ioredis, Joi, Axios, Mongoose 8 |
| **Evaluator-Service** | TypeScript | Express 4 | — | 4000 | BullMQ, Dockerode, Bull Board, ioredis |

---

## 2. End-to-End Flow

```
 Step 1: User submits code
         POST /api/v1/submissions  →  { userId, problemId, language, code }
              │
 Step 2: Submission-Service fetches problem details from Problem-Service
         GET {PROBLEM_ADMIN_SERVICE_URL}/api/v1/problems/{problemId}
              │
 Step 3: Code boilerplate injection
         codeCreator(startSnippet, userCode, endSnippet)
         Wraps user code in problem's start/end snippets for that language
              │
 Step 4: Save submission to MongoDB with status "PENDING"
         submissionRepository.createSubmission(submissionPayload)
              │
 Step 5: Enqueue "SubmissionJob" on BullMQ "SubmissionQueue"
         SubmissionProducer(payload)
         Config: attempts=3, backoff=exponential(2s), removeOnComplete=true
              │
 Step 6: Evaluator-Service Worker picks up job from Redis
         SubmissionWorker("SubmissionQueue")
              │
 Step 7: Run code in isolated Docker containers
         For each testCase:
           - Pull image (once) → Python:3.11-slim / gcc:13 / eclipse-temurin:21
           - Create container with security limits (256MB, 1CPU, no network, 64 PIDs)
           - Execute code with input, collect output
           - Compare with expected output → PASS / FAIL
              │
 Step 8: Send webhook callback to Submission-Service
         POST /api/v1/submissions/{submissionId}/evaluate-result
         Body: { submissionId, testResults, overallStatus, executionTime... }
              │
 Step 9: Submission-Service updates submission status to "COMPLETED"
         updateSubmission(submissionId, { status: 'COMPLETED', ...results })
              │
Step 10: If webhook POST fails → WebhookRetryService retries
         Every 60s, find failed webhooks, retry up to 5 times with exponential backoff
```

---

## 3. Docker Deep-Dive (Evaluator-Service)

### 3.1 Multi-Stage Dockerfile

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --omit=dev

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
EXPOSE 4000
CMD ["node", "dist/index.js"]
```

**Why multi-stage?**
- Stage 1 (builder): Has TypeScript compiler, devDependencies — compiles to `dist/`
- Stage 2 (production): Only `dist/` + production `node_modules` — NO TypeScript, NO devDeps
- Result: ~80MB vs ~300MB single-stage image
- Layer caching: `package*.json` copied first → `npm ci` cached unless deps change

**`npm ci` vs `npm install`:** `ci` is deterministic — uses exact versions from `package-lock.json`. Fails if lockfile is out of sync. Ideal for CI/Docker.

### 3.2 Single-Stage Dockerfiles (Other 3 Services)

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY src ./src
EXPOSE {PORT}
CMD ["npm", "start"]
```

- `--omit=dev`: Skip devDependencies in production
- `npm cache clean --force`: Reduces image size by removing npm cache
- All identical except port numbers (3000/5000/6000)
- Simpler because JS doesn't need compilation

### 3.3 Container Factory (`containerFactory.ts`)

```typescript
import Docker from 'dockerode';

async function createContainer(imageName: string, cmdExecutable: string[]) {
    const docker = new Docker();

    const container = await docker.createContainer({
        Image: imageName,
        Cmd: cmdExecutable,
        AttachStdin: true,
        AttachStdout: true,
        AttachStderr: true,
        Tty: false,
        HostConfig: {
            Memory: 256 * 1024 * 1024,   // 256MB hard limit
            NanoCpus: 1000000000,         // 1 CPU core
            PidsLimit: 64,                // Prevents fork bombs
            NetworkMode: 'none',          // No external networking
        },
        OpenStdin: true
    });

    return container;
}
```

**Security Hardening Summary:**

| Constraint | Value | Purpose |
|-----------|-------|---------|
| `NetworkMode` | `'none'` | Blocks all network access — prevents data exfiltration, downloading payloads |
| `Memory` | 256 MB | Hard OOM limit — container killed if exceeded |
| `NanoCpus` | 1,000,000,000 | 1 CPU core — prevents CPU exhaustion |
| `PidsLimit` | 64 | Prevents fork bombs (`:(){ :\|:& };:`) — max 64 processes |
| `Tty` | `false` | No pseudo-terminal — prevents background daemon processes |
| `OpenStdin` | `true` | Keeps stdin open for piping input |

### 3.4 Docker Images Used

```typescript
// from utils/constants.ts
export const PYTHON_IMAGE = "python:3.11-slim";        // ~150MB
export const JAVA_IMAGE = "eclipse-temurin:21-jdk-jammy"; // ~400MB
export const CPP_IMAGE = "gcc:13";                      // ~1.4GB
```

Images pulled once per executor instance via `imageAlreadyPulled` flag — not per request.

### 3.5 Docker Image Pull (`pullImage.ts`)

```typescript
export default async function pullImage(imageName: string) {
    const docker = new Docker();
    return new Promise<void>((res, rej) => {
        docker.pull(imageName, (err, stream) => {
            if (err) return rej(err);
            docker.modem.followProgress(
                stream,
                (err) => err ? rej(err) : res(),
                (event) => console.log(`[${imageName}] ${event.status}`)
            );
        });
    });
}
```

**How it works:**
- `docker.pull()` returns a progress stream
- `docker.modem.followProgress()` accumulates stream chunks, calls progress callback per layer, resolves when done
- Error callback rejects if pull fails (image not found, network issue)

### 3.6 Docker Stream Protocol (`dockerHelper.ts`)

Docker Engine API returns container output as a **multiplexed stream**:

```
┌──────────────────────────────────────────────────┐
│  HEADER (8 bytes)     │  PAYLOAD (length bytes)  │
├──────┬────────┬───────┼──────────────────────────┤
│ type │  pad   │length │  actual data              │
│ 1 B  │  3 B   │ 4 B   │  (length bytes)           │
│      │        │uint32 │                           │
│      │        │  BE   │                           │
└──────┴────────┴───────┴──────────────────────────┘

Type: 0 = stdin, 1 = stdout, 2 = stderr
```

```typescript
export function decodeDockerStream(buffer: Buffer): DockerStreamOutput {
    let offset = 0;
    const output: DockerStreamOutput = { stdout: '', stderr: '' };

    while (offset < buffer.length) {
        const typeOfStream = buffer[offset];              // byte 0: stream type
        const length = buffer.readUint32BE(offset + 4);   // bytes 4-7: payload length
        offset += DOCKER_STREAM_HEADER_SIZE;              // skip 8-byte header (constant=8)

        if (typeOfStream === 1) {
            output.stdout += buffer.toString('utf-8', offset, offset + length);
        } else if (typeOfStream === 2) {
            output.stderr += buffer.toString('utf-8', offset, offset + length);
        }

        offset += length;
    }
    return output;
}
```

### 3.7 Stream Decoding with Timeout (`fetchDecodedStream`)

```typescript
export function fetchDecodedStream(
    loggerStream: NodeJS.ReadableStream,
    rawLogBuffer: Buffer[],
    timeoutMs: number = 5000
): Promise<string> {
    return new Promise((res, rej) => {
        let isResolved = false;

        const timeout = setTimeout(() => {
            if (!isResolved) {
                isResolved = true;
                (loggerStream as Readable).destroy();
                rej("TLE");  // Time Limit Exceeded
            }
        }, timeoutMs);

        const cleanup = () => {
            clearTimeout(timeout);
            loggerStream.removeAllListeners();
        };

        loggerStream.on('end', () => {
            if (!isResolved) {
                isResolved = true;
                cleanup();
                const completeBuffer = Buffer.concat(rawLogBuffer);
                const decodedStream = decodeDockerStream(completeBuffer);
                if (decodedStream.stderr) {
                    rej(decodedStream.stderr);
                } else {
                    res(decodedStream.stdout);
                }
            }
        });

        loggerStream.on('error', (error) => {
            if (!isResolved) {
                isResolved = true;
                cleanup();
                rej(error);
            }
        });
    });
}
```

**`isResolved` flag** prevents race conditions between timeout, `end`, and `error` events — ensures the promise settles exactly once.

### 3.8 Template Method Pattern (`AbstractExecutor.ts`)

```typescript
export abstract class AbstractExecutor implements CodeExecutorStrategy {
    protected abstract imageName: string;
    private imageAlreadyPulled = false;

    async execute(code: string, inputTestCase: string, outputTestCase: string): Promise<ExecutionResponse> {
        const rawLogBuffer: Buffer[] = [];
        let container: Docker.Container | null = null;

        try {
            // 1. Pull image (only once per instance)
            if (!this.imageAlreadyPulled) {
                await pullImage(this.imageName);
                this.imageAlreadyPulled = true;
            }
        } catch (error) {
            return { output: "Failed to pull image", status: "ERROR" };
        }

        // 2. Get language-specific command
        const runCommand = this.fetchCommand(code, inputTestCase);

        try {
            // 3. Create & start container
            container = await createContainer(this.imageName, ['/bin/sh', '-c', runCommand]);
            await container.start();

            // 4. Follow logs
            const loggerStream = await container.logs({
                stdout: true, stderr: true, timestamps: false, follow: true
            });

            // 5. OLE check: 5MB max output
            let accumulatedSize = 0;
            const MAX_MEMORY_LIMIT = 5 * 1024 * 1024;
            loggerStream.on('data', (chunk) => {
                accumulatedSize += chunk.length;
                if (accumulatedSize > MAX_MEMORY_LIMIT) {
                    (loggerStream as Readable).destroy(new Error("Output Limit Exceeded"));
                } else {
                    rawLogBuffer.push(chunk);
                }
            });

            // 6. Decode stream with 10s timeout
            const codeResponse = await fetchDecodedStream(loggerStream, rawLogBuffer, 10000);

            // 7. Compare output
            if (codeResponse.trim() === outputTestCase.trim()) {
                return { output: codeResponse, status: "SUCCESS" };
            } else {
                return { output: codeResponse, status: "WA" };
            }
        } catch (error) {
            if (error === "TLE") {
                if (container) await container.kill();  // Force kill on timeout
                return { output: "Time Limit Exceeded", status: "ERROR" };
            }
            return { output: error.message || String(error), status: "ERROR" };
        } finally {
            // 8. Cleanup: stop + remove container
            if (container) {
                try {
                    const state = await container.inspect().catch(() => null);
                    if (state?.State.Running) await container.stop({ t: 2 });
                    await container.remove().catch(() => null);
                } catch (error) { /* best-effort cleanup */ }
            }
        }
    }

    protected abstract fetchCommand(code: string, inputTestCase: string): string;
}
```

**Error Handling Matrix:**

| Scenario | Detection | Status Returned | Container Action |
|----------|-----------|----------------|------------------|
| Output matches expected | String comparison | `SUCCESS` | Stop + remove |
| Output mismatches | String comparison | `WA` (Wrong Answer) | Stop + remove |
| Execution > 10s | setTimeout → reject("TLE") | `ERROR` | Kill forcefully |
| Output > 5MB | `accumulatedSize > 5MB` | `ERROR` | Destroy stream, kill |
| stderr present | `fetchDecodedStream` reject | `ERROR` | Stop + remove |
| Image pull fails | `pullImage` throws | `ERROR` | N/A |
| Container create/start fails | Exception caught | `ERROR` | Best-effort cleanup |

### 3.9 Concrete Executors

```typescript
// CppExecutor
class CppExecutor extends AbstractExecutor {
    protected imageName = CPP_IMAGE;  // "gcc:13"
    protected fetchCommand(code, inputTestCase): string {
        return `echo '${code.replace(/'/g, "'\\''")}' > main.cpp && g++ main.cpp -o main && echo '${inputTestCase.replace(/'/g, "'\\''")}' | ./main`;
    }
}

// JavaExecutor
class JavaExecutor extends AbstractExecutor {
    protected imageName = JAVA_IMAGE;  // "eclipse-temurin:21-jdk-jammy"
    protected fetchCommand(code, inputTestCase): string {
        return `echo '${code.replace(/'/g, "'\\''")}' > Main.java && javac Main.java && echo '${inputTestCase.replace(/'/g, "'\\''")}' | java Main`;
    }
}

// PythonExecutor
class PythonExecutor extends AbstractExecutor {
    protected imageName = PYTHON_IMAGE;  // "python:3.11-slim"
    protected fetchCommand(code, inputTestCase): string {
        return `echo '${code.replace(/'/g, "'\\''")}' > test.py && echo '${inputTestCase.replace(/'/g, "'\\''")}' | python3 test.py`;
    }
}
```

**Shell injection mitigation:** The `replace(/'/g, "'\\''")` pattern escapes single quotes:
- Closes current single-quoted string: `'`
- Appends escaped quote: `\'`
- Opens new single-quoted string: `'`
- This prevents code containing `'` from breaking out of the echo command

### 3.10 Strategy Pattern (`ExecutorFactory.ts`)

```typescript
export default function createExecutor(codeLanguage: string): CodeExecutorStrategy | null {
    if (codeLanguage.toLowerCase() === "python") return new PythonExecutor();
    if (codeLanguage.toLowerCase() === "java")   return new JavaExecutor();
    if (codeLanguage.toLowerCase() === "cpp")    return new CppExecutor();
    return null;
}
```

Adding a new language requires:
1. Create new executor class extending `AbstractExecutor`
2. Add case to `createExecutor()` factory
3. Pull the Docker image

---

## 4. BullMQ Deep-Dive (Both Services)

### 4.1 Redis Key Structure

```
Redis Keyspace:
  bull:SubmissionQueue:wait          → List of waiting jobs
  bull:SubmissionQueue:active        → List of jobs being processed
  bull:SubmissionQueue:completed     → Set of completed job IDs
  bull:SubmissionQueue:failed        → Set of failed job IDs
  bull:SubmissionQueue:delayed       → Sorted set (score = timestamp) for retry-delayed jobs
  bull:SubmissionQueue:priority      → Sorted set for priority-ordered jobs
  bull:SubmissionQueue:id            → Auto-incrementing job ID counter
  bull:SubmissionQueue:stalled-check → Timestamps for stalled job detection
  bull:SubmissionQueue:meta          → Queue metadata (paused/active, timestamp)
  bull:SubmissionQueue:events        → Stream of job events (completed, failed, progress...)

  bull:EvaluationQueue:*             → Same structure for legacy secondary queue
```

### 4.2 Redis Connection (Both Services — Identical Pattern)

```typescript
import Redis from "ioredis";

const redisConfig = {
    port: ServerConfig.REDIS_PORT,       // env REDIS_PORT || "6379"
    host: ServerConfig.REDIS_HOST,       // env REDIS_HOST || '127.0.0.1'
    maxRetriesPerRequest: null           // CRITICAL: disables ioredis retries
};

const redisConnection = new Redis(redisConfig);
```

**Why `maxRetriesPerRequest: null`?**  
BullMQ implements its own retry logic at the job level (attempts, backoff). If ioredis retries a command, BullMQ might think it failed and retry the job when the original command actually succeeded. Setting to `null` gives BullMQ full control over retries.

### 4.3 Producer (Submission-Service) — The Main Enqueuer

```javascript
// submissionQueueProducer.js
module.exports = async function (payload) {
    const submissionId = Object.keys(payload)[0];
    
    const job = await submissionQueue.add("SubmissionJob", payload, {
        attempts: 3,                    // Retry 3 times on failure
        backoff: {
            type: 'exponential',        // Delay: 2s → 4s → 8s
            delay: 2000                 // Base delay: 2000ms
        },
        removeOnComplete: true,         // Auto-delete successful jobs (saves Redis memory)
        removeOnFail: false             // Keep failed jobs for debugging/inspection
    });
    
    return job.id;
};
```

**Retry sequence:**
```
Attempt 1 → fails → wait 2s → Attempt 2 → fails → wait 4s → Attempt 3 → fails → moved to "failed" queue
```

**Why `removeOnFail: false`?** Failed jobs contain valuable debug information (error messages, stack traces, input data). Keeping them in the failed queue allows inspection via Bull Board or programmatic access.

### 4.4 Producer (Evaluator-Service) — Simple Re-enqueuer

```typescript
// producers/submissionQueueProducer.ts
export default async function (payload: Record<string, unknown>) {
    await submissionQueue.add(submission_job, payload);  // "SubmissionJob"
    console.log("Successfully added a new submission job");
}
```

No retry config — simpler producer, likely for re-enqueuing or testing purposes.

### 4.5 Main Worker (Evaluator-Service)

```typescript
// workers/SubmissionWorker.ts
export default function SubmissionWorker(queueName: string) {
    new Worker(
        queueName,  // "SubmissionQueue"
        async (job: Job) => {
            if (job.name === submission_job) {  // "SubmissionJob"
                const submissionJobInstance = new SubmissionJob(job.data);
                submissionJobInstance.handle(job);
                return true;  // Marks job as completed in BullMQ
            }
        },
        { connection: redisConnection }
    );
}
```

**Key details:**
- `return true` is the completion signal — without it, BullMQ marks job as failed
- Worker processes jobs one at a time (concurrency default: 1)
- If handler throws → BullMQ applies retry config (attempts, backoff)
- If handler never returns → BullMQ detects stall and retries

### 4.6 Job Handler (`SubmissionJob.ts`)

```typescript
class SubmissionJob implements IJob {
    name: string;
    payload: Record<string, SubmissionPayload>;

    handle = async (job?: Job) => {
        const { code, language, testCases, submissionId, userId } = extractFromPayload();

        // Strategy pattern: get language-specific executor
        const strategy = createExecutor(language);

        const testResults: TestResult[] = [];
        let passedCount = 0;
        const startTime = Date.now();

        // Execute each test case sequentially
        for (let i = 0; i < testCases.length; i++) {
            const testCase = testCases[i];
            try {
                const response = await strategy.execute(
                    code,
                    testCase.input,
                    testCase.output
                );
                const testPassed = response.status === "SUCCESS";
                if (testPassed) passedCount++;

                testResults.push({
                    testCaseIndex: i,
                    input: testCase.input,
                    expectedOutput: testCase.output,
                    actualOutput: response.output || "",
                    status: testPassed ? "PASS" : "FAIL",
                });
            } catch (error) {
                testResults.push({
                    testCaseIndex: i,
                    input: testCase.input,
                    expectedOutput: testCase.output,
                    actualOutput: "",
                    status: "FAIL",
                    error: error instanceof Error ? error.message : String(error),
                });
            }
        }

        const executionTime = Date.now() - startTime;

        const evaluationResult: EvaluationResult = {
            submissionId, userId,
            totalTestCases: testCases.length,
            passedTestCases: passedCount,
            failedTestCases: testCases.length - passedCount,
            overallStatus:
                passedCount === testCases.length ? "SUCCESS"
                : passedCount > 0 ? "PARTIAL"
                : "FAILED",
            testResults,
            executionTime,
        };

        await this.sendWebhookCallback(evaluationResult);
    };

    failed = (job?: Job): void => {
        console.error("Job failed", job?.id, job?.failedReason);
    };
}
```

**Overall Status Logic:**

| Passed | Total | Result |
|--------|-------|--------|
| All | N | `SUCCESS` |
| Some | N | `PARTIAL` |
| 0 | N | `FAILED` |

### 4.7 Webhook Callback Routing

```typescript
private sendWebhookCallback = async (evaluationResult: EvaluationResult) => {
    const webhookBaseUrl =
        process.env.SUBMISSION_SERVICE_WEBHOOK_BASE_URL ||
        process.env.SUBMISSION_SERVICE_WEBHOOK_URL ||
        "http://submission_service:5000/api/v1/submissions";

    const webhookUrl = webhookBaseUrl.includes(":submissionId")
        ? webhookBaseUrl.replace(":submissionId", evaluationResult.submissionId) + "/evaluate-result"
        : `${webhookBaseUrl}/${evaluationResult.submissionId}/evaluate-result`;

    await axios.post(webhookUrl, evaluationResult, { timeout: 5000 });
};
```

The URL supports two patterns:
- `http://host/submissions/:submissionId/evaluate-result` (placeholder-based)
- `http://host/submissions/{id}/evaluate-result` (direct append)

Default target: `http://submission_service:5000/api/v1/submissions/{id}/evaluate-result` (Docker service name)

### 4.8 Legacy Worker (Submission-Service) — `evaluationWorker.js`

```javascript
function evaluationWorker(queue) {
    new Worker('EvaluationQueue', async job => {
        if (job.name === 'EvaluationJob') {
            await axios.post('http://localhost:3001/sendPayload', {
                userId: job.data.userId,
                payload: job.data
            });
        }
    }, { connection: redisConnection });
}
```

**Note:** This worker listens on a **different queue** (`EvaluationQueue`) and forwards to `localhost:3001/sendPayload`. This is a legacy/alternate evaluation pathway — not the primary flow. The primary flow uses the Evaluator-Service's `SubmissionWorker`.

### 4.9 Bull Board UI

```typescript
// config/bullBoardConfig.ts
import { createBullBoard } from "@bull-board/api";
import { BullMQAdapter } from "@bull-board/api/bullMQAdapter";
import { ExpressAdapter } from "@bull-board/express";
import submissionQueue from "../queues/submissionQueue";

const serverAdapter = new ExpressAdapter();
serverAdapter.setBasePath('/ui');

createBullBoard({
    queues: [new BullMQAdapter(submissionQueue)],
    serverAdapter,
});

// Mounted in index.ts:
app.use("/ui", bullBoardAdapter.getRouter());
```

**Available at `http://localhost:4000/ui`** — provides:
- Real-time job counts (waiting, active, completed, failed, delayed)
- Per-job data inspection
- Retry / remove / move job controls
- Pause / resume queue
- Job search and filtering

---

## 5. User-Service (Port 6000)

### 5.1 Entry Point (`index.js`)

```javascript
const express = require('express');
const app = express();

app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));
app.use(cors({ origin: true, methods: ['GET','PUT','POST','DELETE'], allowedHeaders: ['Content-Type','Authorization'] }));

app.use('/api', apiRouter);  // → /api/v1/users/...

app.get('/ping', (req, res) => res.json({ message: 'User Service is alive' }));
app.use(errorHandler);

app.listen(PORT, async () => {
    await connectToDB.connect();
});

// Graceful shutdown
process.on('SIGINT', () => shutdown('SIGINT'));
process.on('SIGTERM', () => shutdown('SIGTERM'));
```

### 5.2 User Model (`user.model.js`)

```javascript
const userSchema = new mongoose.Schema({
    username: {
        type: String,
        required: true,
        unique: true,
        trim: true,
        minlength: 3,
        maxlength: 30,
        match: [/^[a-zA-Z0-9_]+$/, 'Username can only contain letters, numbers, and underscores']
    },
    email: {
        type: String,
        required: true,
        unique: true,
        trim: true,
        lowercase: true,
        match: [/^\S+@\S+\.\S+$/, 'Please provide a valid email address']
    },
    password: {
        type: String,
        required: true,
        minlength: [8, 'Password must be at least 8 characters long']
    },
    firstName:  { type: String, trim: true, maxlength: 50 },
    lastName:   { type: String, trim: true, maxlength: 50 },
    avatarUrl:  { type: String, default: null },
    role:       { type: String, enum: ['user', 'admin'], default: 'user' },
    isActive:   { type: Boolean, default: true }
}, { timestamps: true });

// Index on email for fast lookups
userSchema.index({ email: 1 });

// Pre-save hook: auto-hash password
userSchema.pre('save', async function () {
    if (!this.isModified('password')) return;
    this.password = await hashPassword(this.password);
});

// Strip password from JSON responses
userSchema.methods.toJSON = function () {
    const userObject = this.toObject();
    delete userObject.password;
    return userObject;
};
```

**Key patterns:**
- **`unique: true`** at Mongoose level (NOT a replacement for DB-level unique index — Mongoose does optimistic checks, DB enforces at commit time)
- **`pre('save')`** runs only when `password` is modified → safe for partial updates
- **`toJSON()`** override → password never leaked in API responses, even through `res.json(user)`

### 5.3 Auth System — Dual Token Strategy

| Token | Signing Secret | Default Expiry | Usage |
|-------|---------------|----------------|-------|
| Access Token | `JWT_SECRET` | 24h (`JWT_EXPIRES_IN`) | Authenticate API requests |
| Refresh Token | `JWT_REFRESH_SECRET` | 7d (`JWT_REFRESH_EXPIRES_IN`) | Get new access token |

```javascript
// jwt.utils.js
function generateAccessToken(payload) {
    return jwt.sign(payload, JWT_SECRET, { expiresIn: JWT_EXPIRES_IN });
}

function generateRefreshToken(payload) {
    return jwt.sign(payload, JWT_REFRESH_SECRET, { expiresIn: JWT_REFRESH_EXPIRES_IN });
}

function verifyAccessToken(token) {
    return jwt.verify(token, JWT_SECRET);  // Throws JsonWebTokenError or TokenExpiredError
}
```

**Token refresh flow:**
```
1. Login → get { accessToken (24h), refreshToken (7d) }
2. Access token expires → POST /api/v1/users/refresh → verify refresh token → get new access token
3. Refresh token expires → must re-login
```

### 5.4 Authentication Middleware

```javascript
function authenticate(req, res, next) {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
        throw new Unauthorized('No token provided');
    }
    
    const token = authHeader.substring(7);  // Remove "Bearer " prefix
    const decoded = verifyAccessToken(token);
    req.user = decoded;  // { id, email, username, role }
    next();
}
```

**Error handling for JWT:**
- `JsonWebTokenError` → "Invalid token" (tampered payload, wrong secret, malformed)
- `TokenExpiredError` → "Token expired" (client should refresh)

### 5.5 Password Security (`password.utils.js`)

```javascript
async function hashPassword(password) {
    const salt = await bcrypt.genSalt(10);       // 2^10 = 1024 iterations
    return await bcrypt.hash(password, salt);
}

async function comparePassword(password, hashedPassword) {
    return await bcrypt.compare(password, hashedPassword);
}

function validatePasswordStrength(password) {
    const errors = [];
    if (!password || password.length < 8) errors.push('at least 8 characters');
    if (!/[A-Z]/.test(password))         errors.push('at least one uppercase');
    if (!/[a-z]/.test(password))         errors.push('at least one lowercase');
    if (!/[0-9]/.test(password))         errors.push('at least one number');
    if (!/[!@#$%^&*(),.?":{}|<>]/.test(password)) errors.push('at least one special char');
    return { isValid: errors.length === 0, errors };
}
```

- **bcryptjs** (pure JS) chosen over **bcrypt** (native C++) → Docker-friendly, no build tools needed
- Salt rounds: 10 (standard, ~100ms on modern hardware)
- Password validation: Upper + Lower + Digit + Special + 8+ chars

### 5.6 AWS S3 Avatar Upload

```javascript
async uploadAvatar(userId, fileBuffer, mimetype) {
    const extension = mimetype.split('/')[1] || 'png';
    const fileName = `avatars/${userId}-${Date.now()}.${extension}`;
    
    const command = new PutObjectCommand({
        Bucket: process.env.S3_BUCKET_NAME,
        Key: fileName,
        Body: fileBuffer,
        ContentType: mimetype
    });
    
    await s3Client.send(command);
    
    user.avatarUrl = `https://${bucketName}.s3.${process.env.AWS_REGION}.amazonaws.com/${fileName}`;
    await user.save();
    
    return user.toJSON();
}
```

**File handling:**
- Multer with `memoryStorage()` (in-memory buffer, no temp files)
- File size limit: 2MB
- MIME type validation: `image/jpeg`, `image/png`, `image/webp` only
- URL pattern: `https://{bucket}.s3.{region}.amazonaws.com/{key}`

### 5.7 DB Connection — Singleton Pattern

```javascript
class DBConnection {
    constructor() {
        if (DBConnection.instance) return DBConnection.instance;  // Singleton
        this.isConnected = false;
        DBConnection.instance = this;
    }

    async connect() {
        if (this.isConnected) {
            console.log('Using existing connection');
            return;  // Reuse connection — no double-connect
        }
        await mongoose.connect(ATLAS_DB_URL);
        this.isConnected = true;
        console.log('New connection established');
    }

    async disconnect() {
        if (this.isConnected) {
            await mongoose.disconnect();
            this.isConnected = false;
        }
    }
}
```

**Singleton pattern** ensures only one mongoose connection per Node process.

### 5.8 Routes

```
POST   /api/v1/users/register   → Register (public)
POST   /api/v1/users/login      → Login (public)
POST   /api/v1/users/refresh    → Refresh token (public)
GET    /api/v1/users/profile    → Get profile (authenticated)
PUT    /api/v1/users/profile    → Update profile (authenticated)
PUT    /api/v1/users/password   → Change password (authenticated)
POST   /api/v1/users/avatar     → Upload avatar (authenticated + multer)
```

### 5.9 Error Hierarchy

```
BaseError { name, statusCode, description, details }
  ├── BadRequest (400)
  │     constructor(propertyName, details)
  │     message: "Invalid request parameter: {propertyName}"
  ├── NotFound (404)
  │     constructor(resourceName, resourceValue)
  │     message: "{resourceName} not found with value: {resourceValue}"
  ├── Unauthorized (401)
  │     constructor(message, details)
  │     message: custom message
  └── InternalServerError (500)
        constructor(details)
        message: "Something went wrong"
```

**Error Handler Middleware:**
```javascript
function errorHandler(err, req, res, next) {
    if (err instanceof BaseError) {
        return res.status(err.statusCode).json({ success: false, message: err.message, error: err.details });
    }
    if (err.name === 'ValidationError')       → 400 (Mongoose validation)
    if (err.code === 11000)                    → 409 (Mongoose duplicate key)
    if (err.name === 'JsonWebTokenError')      → 401 (Invalid JWT)
    if (err.name === 'TokenExpiredError')      → 401 (Expired JWT)
    // Default → 500
}
```

---

## 6. Problem-Service (Port 3000)

### 6.1 Entry Point (`index.js`)

```javascript
const app = express();

app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));
app.use(bodyParser.text());  // Also accept raw text body
app.use(cors(/* ... */));

app.use('/api', apiRouter);  // → /api/v1/problems/...

app.get('/ping', (req, res) => res.json({ message: 'Problem Service is alive' }));
app.use(errorHandler);

app.listen(PORT, async () => {
    await connectToDB.connect();
});
```

### 6.2 Problem Model (`problem.model.js`)

```javascript
const problemSchema = new mongoose.Schema({
    title: {
        type: String,
        required: [true, 'Title cannot be empty'],
        trim: true,
        minlength: [3, 'Title must be at least 3 characters long']
    },
    description: {
        type: String,
        required: [true, 'Description cannot be empty'],
        trim: true
    },
    difficulty: {
        type: String,
        enum: { values: ['easy', 'medium', 'hard'], message: 'Difficulty must be easy, medium, or hard' },
        required: true,
        default: 'easy',
        lowercase: true
    },
    testCases: {
        type: [{
            input:  { type: String, required: true, trim: true },
            output: { type: String, required: true, trim: true }
        }],
        validate: {
            validator: arr => arr && arr.length > 0,
            message: 'At least one test case is required'
        }
    },
    codeStubs: {
        type: [{
            language:      { type: String, required: true, trim: true, lowercase: true },
            startSnippet:  { type: String, trim: true },
            userSnippet:   { type: String, default: '', trim: true },
            endSnippet:    { type: String, trim: true }
        }],
        validate: {
            validator: arr => arr && arr.length > 0,
            message: 'At least one code stub is required'
        }
    },
    editorial: { type: String, default: '', trim: true }
}, { timestamps: true, strict: true });
```

**`codeStubs` Design:**
```
startSnippet    →  imports, class declaration, function signature
userSnippet     →  placeholder for user code (empty string = user writes everything)
endSnippet      →  closing braces, main method, test harness
```

The Submission-Service assembles: `startSnippet + userCode + endSnippet` → complete executable code.

### 6.3 Markdown Sanitization Pipeline

```javascript
const marked = require('marked');
const sanitizeHtmlLibrary = require('sanitize-html');
const TurndownService = require('turndown');

function sanitizeMarkdownContent(markdownContent) {
    const turndownService = new TurndownService();

    // Step 1: Markdown → HTML
    const convertedHtml = marked.parse(markdownContent);

    // Step 2: HTML → Sanitized HTML (XSS prevention)
    const sanitizedHtml = sanitizeHtmlLibrary(convertedHtml, {
        allowedTags: sanitizeHtmlLibrary.defaults.allowedTags.concat(['img'])
    });

    // Step 3: Sanitized HTML → Sanitized Markdown
    const sanitizedMarkdown = turndownService.turndown(sanitizedHtml);

    return sanitizedMarkdown;
}
```

**Why this 3-step pipeline?**
1. `marked.parse()` — Converts Markdown to HTML (including `<script>` tags in raw HTML within markdown)
2. `sanitize-html` — Strips all dangerous HTML tags/attributes, keeping only safe ones (+ `img`)
3. `turndown` — Converts cleaned HTML back to Markdown for storage

**This prevents stored XSS** — even if a user submits Markdown containing `<script>alert('xss')</script>`, it gets stripped.

### 6.4 Validation — Zod 4

```javascript
const createProblemSchema = z.object({
    title:       z.string().min(3).trim(),
    description: z.string().min(1).trim(),
    difficulty:  z.enum(['easy', 'medium', 'hard']),
    testCases:   z.array(z.object({
                    input: z.string().trim(),
                    output: z.string().trim()
                 })).min(1),
    codeStubs:   z.array(z.object({
                    language:      z.string().trim().toLowerCase(),
                    startSnippet:  z.string().trim(),
                    userSnippet:   z.string().trim(),
                    endSnippet:    z.string().trim().optional()
                 })).min(1),
    editorial:   z.string().trim().optional()
});

const validateRequest = (schema) => (req, res, next) => {
    try {
        req.body = schema.parse(req.body);  // Zod coerces + validates
        next();
    } catch (error) {
        return res.status(400).json({
            success: false, message: 'Invalid request data', error: error.errors
        });
    }
};
```

### 6.5 Logger — Winston (3 Transports)

1. **Console** — colored, human-readable (`YYYY-MM-DD HH:mm:ss [LEVEL]: message`)
2. **MongoDB** — error-level only → `logs` collection in `LOG_DB_URL` database
3. **File** — `app.log` (all levels)

### 6.6 Routes

```
GET    /api/v1/problems/ping    → Health check
POST   /api/v1/problems         → Create problem (validated)
GET    /api/v1/problems         → List all problems
GET    /api/v1/problems/:id     → Get problem by ID
PUT    /api/v1/problems/:id     → Update problem (validated)
DELETE /api/v1/problems/:id     → Delete problem
```

### 6.7 Controller Layer — Dependency Injection via Constructor

```javascript
// controller
const problemService = new ProblemService(new ProblemRepository());

async function addProblem(req, res, next) {
    const newproblem = await problemService.createProblem(req.body);
    return res.status(StatusCodes.CREATED).json({
        success: true, message: 'Successfully created a new problem', data: newproblem
    });
}
```

**Note:** The controller imports `ProblemRepository` from `../repositories` but the actual file exists in `../services` — no separate repository layer exists. This is a bug.

---

## 7. Submission-Service (Port 5000)

### 7.1 Tech Stack

- **Fastify 4** (not Express) — ~2x faster, built-in schema serialization, plugin system
- **Joi** for validation (not Zod)
- **BullMQ 5.7** for job queues
- **Mongoose 8** for MongoDB
- **fastify-plugin** for dependency injection via decorators

### 7.2 Plugin Architecture

```javascript
// app.js
async function app(fastify, options) {
    await fastify.register(require('@fastify/cors'), { /* ... */ });
    await fastify.register(repositoryPlugin);   // Decorates: fastify.submissionRepository
    await fastify.register(servicePlugin);       // Decorates: fastify.submissionService
    await fastify.register(apiRoutes, { prefix: '/api' });
}
module.exports = fastifyPlugin(app);
```

**Plugin chain:**
```
repositoryPlugin → fastify.decorate('submissionRepository', new SubmissionRepository())
  ↓
servicePlugin → fastify.decorate('submissionService', new SubmissionService(fastify.submissionRepository))
  ↓
apiRoutes → defines POST /, GET /, GET /:submissionId, POST /:submissionId/evaluate-result
```

**Why Fastify plugins?** Fastify's plugin system creates encapsulated scopes. `fastify-plugin` makes plugins globally available across all scopes (otherwise, decorations are scoped).

### 7.3 Entry Point (`index.js`)

```javascript
const fastify = require('fastify')({ logger: true });
const app = require('./app');
const connectToDB = require('./config/dbConfig');

fastify.register(app);
fastify.setErrorHandler(errorHandler);

fastify.listen({ port: serverConfig.PORT, host: '0.0.0.0' }, async (err) => {
    if (err) { fastify.log.error(err); process.exit(1); }
    await connectToDB();
    console.log(`Server up at port ${serverConfig.PORT}`);
});
```

**`host: '0.0.0.0'`** — binds to all interfaces. Required in Docker containers to accept connections from outside the container.

### 7.4 Submission Model (`submissionModel.js`)

```javascript
const submissionSchema = new mongoose.Schema({
    userId:    { type: String, required: true, index: true },
    problemId: { type: String, required: true, index: true },
    code:      { type: String, required: true },
    language:  { type: String, required: true },
    status:    {
        type: String,
        enum: ["PENDING", "PROCESSING", "COMPLETED", "ERROR"],
        default: "PENDING"
    },
    testResults: [{
        testCaseIndex: Number,
        input: String,
        expectedOutput: String,
        actualOutput: String,
        status: { type: String, enum: ["PASS", "FAIL"], default: "FAIL" },
        error: String
    }],
    totalTestCases:   { type: Number, default: 0 },
    passedTestCases:  { type: Number, default: 0 },
    failedTestCases:  { type: Number, default: 0 },
    overallStatus:    { type: String, enum: ["SUCCESS", "PARTIAL", "FAILED", null], default: null },
    executionError:   String,
    submittedAt:      { type: Date, default: Date.now, index: true },
    completedAt:      Date,
    executionTime:    Number,
    
    // Idempotency & Webhook Retry
    idempotencyKey:    { type: String, unique: true, sparse: true },
    webhookAttempts:   { type: Number, default: 0 },
    lastWebhookAttempt: Date,
    nextRetryAt:       Date,
    webhookFailed:     { type: Boolean, default: false },
    
    __v: { type: Number, select: false }  // Exclude __v from queries
}, { 
    timestamps: true,
    indexes: [
        { userId: 1, submittedAt: -1 },        // User's submissions, newest first
        { problemId: 1, userId: 1 },           // Find user's submissions for problem
        { status: 1, submittedAt: -1 },         // Filter by status
        { webhookFailed: 1, nextRetryAt: 1 }    // Efficient webhook retry queries
    ]
});
```

**`idempotencyKey` design:** `unique: true, sparse: true` — sparse index excludes documents with `null`/missing key. This prevents duplicate submissions by checking a unique key, but allows documents without the key.

### 7.5 Submission Flow (`submissionService.js`)

```javascript
async addSubmission(submissionPayload) {
    // 1. Validate language (only cpp, java, python)
    if (!ALLOWED_LANGUAGES.includes(submissionPayload.language.toLowerCase())) {
        throw new SubmissionCreationError(`Language not supported`);
    }

    // 2. Fetch problem details from Problem-Service
    problemAdminApiResponse = await fetchProblemDetails(problemId);
    // GET {PROBLEM_ADMIN_SERVICE_URL}/api/v1/problems/{problemId}

    // 3. Find code stub for user's language
    const languageCodeStub = problemAdminApiResponse.data.codeStubs.find(
        cs => cs.language.toLowerCase() === submissionPayload.language.toLowerCase()
    );

    // 4. Assemble complete code: startSnippet + userCode + endSnippet
    const completeCode = codeCreator(
        languageCodeStub.startSnippet,
        submissionPayload.code,
        languageCodeStub.endSnippet
    );
    submissionPayload.code = completeCode;

    // 5. Save to MongoDB with status "PENDING"
    const submission = await this.submissionRepository.createSubmission(submissionPayload);

    // 6. Enqueue to BullMQ
    const payload = {
        [submission._id]: {
            code: submission.code,
            language: submission.language,
            testCases: problemAdminApiResponse.data.testCases,
            userId, submissionId: submission._id, problemId
        }
    };
    await SubmissionProducer(payload);
    // Config: attempts=3, exponential backoff(2s), removeOnComplete=true, removeOnFail=false
}
```

### 7.6 Code Creator — Boilerplate Injection

```javascript
function codeCreator(startSnippet, userCode, endSnippet) {
    const start = (startSnippet || '').trim();
    const user  = (userCode || '').trim();
    const end   = (endSnippet || '').trim();

    // Detect if user already included boilerplate
    const includesStart = !!(start && user.includes(start));
    const includesEnd   = !!(end && user.includes(end));

    if (includesStart && includesEnd) return user;           // Already complete
    if (includesStart && !includesEnd)  return [user, end].join('\n');   // Add end
    if (!includesStart && includesEnd)  return [start, user].join('\n'); // Add start
    return [start, user, end].filter(Boolean).join('\n');    // Full assembly
}
```

**Prevents double-wrapping** on resubmission — if user code already contains start/end snippets, returns as-is.

### 7.7 Webhook Retry Service (`webhookRetryService.js`)

```javascript
const WEBHOOK_CONFIG = {
    MAX_RETRIES: 5,
    INITIAL_DELAY: 5000,        // 5 seconds
    MAX_DELAY: 3600000,          // 1 hour
    BACKOFF_MULTIPLIER: 2
};

// Exponential backoff with cap:
// Attempt 0 → 5s
// Attempt 1 → 10s
// Attempt 2 → 20s
// Attempt 3 → 40s
// Attempt 4 → 80s
// Attempt 5 → 160s (then capped at 3600s/1h for subsequent)
```

**Retry lifecycle:**
```javascript
async retryFailedWebhooks() {
    const failed = await submissionRepository.findFailedWebhookSubmissions();
    // query: { webhookFailed: true, nextRetryAt: { $lt: new Date() } }

    for (const submission of failed) {
        if (submission.webhookAttempts >= MAX_RETRIES) {
            // Give up — could trigger alert/notification here
            continue;
        }

        const result = await this.sendWebhookCallback(submission);

        if (result.success) {
            // Reset webhook state
            await updateSubmission(id, { webhookFailed: false, webhookAttempts: 0 });
        } else {
            // Schedule next retry
            const nextRetry = calculateNextRetry(submission.webhookAttempts);
            await incrementWebhookAttempt(id, nextRetry);
        }
    }
}

// Runs every 60 seconds
startRetryJob(60000);
```

**Retry delay formula:**
```javascript
calculateNextRetry(attemptCount) {
    let delay = 5000 * Math.pow(2, attemptCount);
    delay = Math.min(delay, 3600000);
    return new Date(Date.now() + delay);
}
```

### 7.8 Routes

```
POST   /api/v1/submissions                          → Create submission
GET    /api/v1/submissions?userId=X&limit=20&offset=0 → Get user's submissions (paginated)
GET    /api/v1/submissions/:submissionId             → Get submission by ID
POST   /api/v1/submissions/:submissionId/evaluate-result → Webhook endpoint (called by Evaluator-Service)
```

### 7.9 API Response Helper (`apiResponse.js`)

```javascript
class ApiResponse {
    static success(data, message = 'Success') {
        return { success: true, message, data, error: null, timestamp: new Date().toISOString() };
    }
    
    static error(message, error = null, data = null) {
        return { success: false, message, data, error: error || message, timestamp: new Date().toISOString() };
    }
    
    static paginated(data, total, limit, offset, message = 'Success') {
        return {
            success: true, message, data,
            pagination: { total, limit, offset, page: Math.floor(offset/limit)+1, pages: Math.ceil(total/limit) },
            timestamp: new Date().toISOString()
        };
    }
}
```

### 7.10 Validation — Joi

```javascript
const submissionValidationSchema = joi.object({
    userId:    joi.string().required().trim(),
    problemId: joi.string().required().trim(),
    language:  joi.string().required().valid('cpp', 'java', 'python').lowercase(),
    code:      joi.string().required().max(102400)  // 100KB max
});

const evaluationResultSchema = joi.object({
    submissionId:    joi.string().required(),
    userId:          joi.string().required(),
    totalTestCases:  joi.number().required().min(0),
    passedTestCases: joi.number().required().min(0),
    failedTestCases: joi.number().required().min(0),
    overallStatus:   joi.string().required().valid('SUCCESS', 'PARTIAL', 'FAILED'),
    testResults:     joi.array().required().min(0),
    executionTime:     joi.number().required().min(0)
}).unknown(true);  // Allow extra fields from Evaluator Service
```

### 7.11 Inter-Service Communication — Axios Client

```javascript
// apis/problemAdminApi.js
const PROBLEM_ADMIN_API_URL = `${PROBLEM_ADMIN_SERVICE_URL}/api/v1`;

async function fetchProblemDetails(problemId) {
    const uri = PROBLEM_ADMIN_API_URL + `/problems/${problemId}`;
    const response = await axiosInstance.get(uri);
    return response.data;
}
```

**Note:** No auth headers between services — all inter-service calls use plain HTTP.

---

## 8. Evaluator-Service (Port 4000)

### 8.1 Tech Stack

- **TypeScript** (strict mode — only service using TS)
- **Express 4**
- **BullMQ** (consumer side)
- **dockerode** (Docker Engine API client)
- **Bull Board** (queue monitoring UI)
- **Axios** (webhook callback)

### 8.2 TypeScript Configuration (Strict Mode)

```json
{
  "compilerOptions": {
    "target": "es2016",
    "module": "commonjs",
    "outDir": "./dist",
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "skipLibCheck": true
  }
}
```

### 8.3 Entry Point

```typescript
const app = express();

app.use("/ui", bullBoardAdapter.getRouter());  // Bull Board at /ui
app.get("/ping", (res) => res.status(200).send("Pong"));

SubmissionWorker(submission_queue);  // Start worker immediately on boot

app.listen(serverConfig.PORT, () => {
    console.log(`Evaluator Service is up`);
    console.log(`BullMQ UI available at http://localhost:${serverConfig.PORT}/ui`);
});
```

**No CRUD API routes** — this service is purely a background worker + monitoring UI.

### 8.4 Type Definitions (`types.ts`)

```typescript
export interface CodeExecutorStrategy {
    execute(code: string, inputTestCase: string, outputTestCase: string): Promise<ExecutionResponse>;
}

export type ExecutionResponse = { output: string; status: string };

export interface IJob {
    name: string;
    payload?: Record<string, unknown>;
    handle: (job?: Job) => void;
    failed: (job?: Job) => void;
}

export type SubmissionPayload = {
    code: string;
    language: string;
    testCases: TestCase[];
    userId: string;
    submissionId: string;
    problemId?: string;
};

export type TestResult = {
    testCaseIndex: number;
    input: string;
    expectedOutput: string;
    actualOutput: string;
    status: 'PASS' | 'FAIL';
    error?: string;
};

export type EvaluationResult = {
    submissionId: string;
    userId: string;
    totalTestCases: number;
    passedTestCases: number;
    failedTestCases: number;
    overallStatus: 'SUCCESS' | 'PARTIAL' | 'FAILED';
    testResults: TestResult[];
    executionTime?: number;
};

export type TestCase = { input: string; output: string };
```

### 8.5 Server Configuration

```typescript
export default {
    PORT: process.env.PORT || 3000,
    REDIS_PORT: parseInt(process.env.REDIS_PORT || "6379", 10),
    REDIS_HOST: process.env.REDIS_HOST || '127.0.0.1'
};
```

---

## 9. Known Issues & Gaps

| Issue | Location | Detail |
|-------|----------|--------|
| **No `docker-compose.yml`** | Root | Services must be run manually — no orchestration defined |
| **No API Gateway** | — | No load balancer, rate limiter, auth gateway between services |
| **Problem-Service missing repository** | `problem.controller.js:3` | Imports `ProblemRepository` from `../repositories` but directory doesn't exist |
| **Submission-Service `this` binding** | `submissionController.js` | Uses `this.submissionService` / `this.submissionRepository` but functions aren't bound to Fastify context |
| **Legacy dual-queue evaluation** | `evaluationWorker.js` vs `SubmissionWorker.ts` | Two evaluation paths: `EvaluationQueue` (legacy, `localhost:3001`) and `SubmissionQueue` (primary, Evaluator-Service) |
| **No inter-service auth** | All axios calls | Cross-service HTTP calls have no auth headers/tokens |
| **No health/readiness checks** | All services | No `/health` or `/ready` endpoints for container orchestration |
| **Inconsistent DB connection patterns** | User vs Problem services | User-Service uses singleton, Problem-Service uses plain function — different implementations of same concept |
| **No input sanitization on code** | Submission-Service | User code is passed raw to Docker containers (relies on container security, not input filtering) |
| **Hardcoded `localhost:3001`** | `evaluationWorker.js` | Legacy worker uses hardcoded URL instead of env variable |
| **`axiosInstance` bare** | `axiosInstance.js` | Creates axios instance with no interceptors, timeouts, or retry config |
| **Container image caching** | `AbstractExecutor.ts` | `imageAlreadyPulled` flag caches per executor instance, not globally — each new instance re-pulls |

---

## 10. Key Interview Questions & Answers

### Q1: Why microservices for a code submission platform?

**A:** The Evaluator-Service is fundamentally different from CRUD services — it's resource-intensive (Docker containers, CPU, memory) and latency-tolerant (async queue). Separating it enables:
- **Independent scaling**: 10 evaluator instances behind 1 submission API during peak load
- **Independent deployment**: Update evaluations without touching user/problem APIs
- **Tech stack freedom**: TypeScript for evaluator (type safety with Docker API), Fastify for high-throughput submission API, Express for simple CRUD
- **Fault isolation**: Evaluator crash doesn't affect user profile retrieval
- **Resource isolation**: Evaluator runs on beefy instances, other services on lightweight ones

### Q2: How do you prevent malicious code from harming the system?

**A:** Defense-in-depth with multiple layers:
1. **`NetworkMode: 'none'`** (most critical) — blocks ALL network access. Code cannot exfiltrate data, download payloads, or attack internal services
2. **`Memory: 256MB`** — Hard OOM limit. Container is killed immediately, not allowed to swap
3. **`NanoCpus: 1000000000`** — 1 CPU core maximum. Prevents CPU exhaustion attacks
4. **`PidsLimit: 64`** — Prevents classic fork bombs (`:(){ :|:& };:`). Container cannot spawn >64 PIDs
5. **10-second timeout** — Infinite loops are killed by `setTimeout` + `container.kill()`
6. **5MB output limit** — Prevents output spamming (e.g., infinite printing)
7. **Isolated Docker daemon** — Evaluator should run on its own Docker host, isolated from production services

### Q3: Why BullMQ instead of RabbitMQ or Kafka?

**A:**
| Factor | BullMQ | RabbitMQ | Kafka |
|--------|--------|----------|-------|
| Infrastructure | Redis (already needed) | Separate broker | Separate broker + ZK |
| Complexity | Simple API | AMQP protocol | Complex partitioning |
| Job retry | Built-in (attempts, backoff) | Manual via DLQ | Manual via consumers |
| Monitoring UI | Bull Board (built-in) | Management Plugin | Multiple tools |
| Best for | Task queues, job processing | Message routing | Event streaming, high throughput |

For this use case (async code evaluation), BullMQ is ideal — simple API, built-in retry, visual monitoring. If the platform grew to millions of submissions/day, Kafka would provide better throughput and event replay.

### Q4: How does retry work at different levels?

**A:** Three retry mechanisms:

1. **BullMQ Job Retry** (`attempts: 3`, exponential backoff with 2s base)
   - When `SubmissionJob.handle()` throws
   - Delay: 2s → 4s → 8s → failed queue
   - Managed entirely by BullMQ

2. **Webhook Callback Retry** (`WebhookRetryService`)
   - When Evaluator-Service's POST to Submission-Service fails
   - Polls every 60s, retries up to 5 times
   - Delay formula: `5000 * 2^attempt`, capped at 1 hour

3. **Docker Execution Timeout** (`AbstractExecutor`)
   - 10s timeout per test case
   - Kills container if exceeded (TLE)

### Q5: Explain the Template Method pattern used in code execution.

**A:** `AbstractExecutor.execute()` defines the invariant algorithm skeleton:
```
1. Pull Docker image (once)
2. Build execution command (abstract — subclass implements)
3. Create Docker container with security limits
4. Start container
5. Follow logs with output size monitoring
6. Decode multiplexed stream with timeout
7. Compare output with expected
8. Cleanup container (stop + remove)
```

Subclasses (`CppExecutor`, `JavaExecutor`, `PythonExecutor`) only override `fetchCommand()` — the language-specific command construction. The `imageName` is an abstract property. This is the classic Template Method pattern from the GoF book.

### Q6: How does the Strategy Pattern enable language extensibility?

**A:** `ExecutorFactory.createExecutor()` returns a `CodeExecutorStrategy` instance based on language:
```typescript
createExecutor("python") → new PythonExecutor()
createExecutor("java")   → new JavaExecutor()
createExecutor("cpp")    → new CppExecutor()
```

Adding a new language (e.g., JavaScript) requires:
1. `class JavaScriptExecutor extends AbstractExecutor { imageName = "node:20-alpine"; fetchCommand(...) { ... } }`
2. Add `if (language === "javascript") return new JavaScriptExecutor();` to factory
3. Pull `node:20-alpine` image

No changes to the job handler, queue, or worker. Open/Closed Principle.

### Q7: Why Fastify over Express for Submission-Service?

**A:**
- **~2x faster throughput** — Fastify's router is optimized, uses schema-based serialization
- **Plugin system** — Dependencies injected via `fastify.decorate()`, scoped by plugin
- **Built-in logging** — Pino logger with request ID tracking, slower-route detection
- **Schema validation** — Joi/JSON Schema integration with automatic error responses
- **Better for microservices** — Lower overhead, designed for high-throughput internal APIs

Express is fine for User-Service and Problem-Service (low request rate, simple CRUD, developer familiarity).

### Q8: How does the Docker multiplexed stream protocol work?

**A:** Docker Engine API v1.12+ uses multiplexing to interleave stdout and stderr in a single TCP stream:

```
[STREAM_HEADER (8 bytes)] [PAYLOAD (n bytes)] [STREAM_HEADER] [PAYLOAD] ...
```

Each header:
- **Byte 0**: Stream type (`0x01` = stdout, `0x02` = stderr)
- **Bytes 1-3**: Padding (unused)
- **Bytes 4-7**: Payload length as uint32 big-endian

The `decodeDockerStream()` function reads headers sequentially, extracts payloads, and separates stdout/stderr. This cannot be done with simple `toString()` — raw output contains binary header frames.

### Q9: How does the dual-token JWT refresh strategy work?

**A:**

| Step | Token | Expiry | 
|------|-------|--------|
| Login | Access (24h) + Refresh (7d) | Both returned |
| API calls | Access token in `Authorization: Bearer <token>` | Expires in 24h |
| Refresh | Refresh token sent to `/api/v1/users/refresh` | Valid for 7d |
| New access | Server issues fresh 24h access token | Refresh token unchanged |

**Benefits:**
- Access tokens are short-lived (if stolen, limited window)
- Refresh tokens are long-lived but used rarely (fewer network exposures)
- Different signing secrets prevent token-type confusion attacks
- Server can revoke refresh tokens without invalidating all access tokens

### Q10: How would you deploy this platform to production?

**A:**

1. **Orchestration**: Create `docker-compose.yml` or Kubernetes manifests with:
   - 4 services (each auto-restart, health checks)
   - Redis (persistent volume, append-only mode)
   - MongoDB Atlas (or self-hosted replica set)

2. **API Gateway**: Add Nginx/Traefik/Kong for:
   - Rate limiting per user/IP
   - Request routing to correct service
   - TLS termination
   - Authentication at edge (validate JWT before forwarding)

3. **Scaling**:
   - User-Service: 2 replicas (stateless, CRUD)
   - Problem-Service: 2 replicas (stateless, CRUD)
   - Submission-Service: 3 replicas (handles all user submissions)
   - Evaluator-Service: 5+ replicas (resource-bound, needs beefy instances)

4. **Monitoring**:
   - Bull Board for queue visibility
   - Prometheus + Grafana for service metrics
   - Centralized logging (ELK/Loki)

5. **Security improvements**:
   - mTLS between services
   - API key for Evaluator-Service → Submission-Service webhook
   - Dedicated Docker host for Evaluator (isolated from other services)
   - seccomp/AppArmor profiles on evaluation containers
