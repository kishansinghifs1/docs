# Knight.dev – Technical Interview Prep & Coding Assignment Playbook

This guide is your master resource for technical interviews and full-stack coding assignments. It details the professional engineering terminology to use, provides deep-dive code patterns, lists advanced interview Q&As, and outlines a systematic playbook for completing take-home developer challenges.

---

## 1. Professional Software Engineering Jargon Sheet

Use these exact terms when explaining your architecture to interviewers:

* **Headless UI Primitives:** The use of unstyled, accessible component bases (via **Radix UI**) that handle complex keyboard interactions, screen reader announcements (WAI-ARIA compliance), and focus management, allowing the styling layer (Tailwind CSS) to remain completely separate.
* **Hydration Mismatch:** A React error occurring when the server-rendered HTML doesn't match the initial client-side rendered DOM (common with dynamic data like dates or random word slugs). Solved using React lifecycle hooks (`useEffect` state flags or `suppressHydrationWarning`).
* **Durable Execution / Serverless Workflows:** Background jobs (**Inngest**) that checkpoint execution progress at each `step.run`. This ensures long-running AI agent tasks can pause, resume, and retry without restarting the entire function, escaping standard serverless timeout constraints.
* **Micro-VM Sandboxing:** Spinning up isolated, lightweight virtual machines (**E2B Firecracker VMs**) rather than simple OS processes or shared Docker containers. This prevents shell execution escapes, guarantees security isolation, and provides hot-reload preview endpoints.
* **Network Request Batching:** The process where **tRPC Http Batch Link** aggregates multiple queries fired during the same event-loop tick into a single HTTP payload, avoiding network bottlenecking.
* **Stateless Database Polling:** Fetching UI message updates using React Query's `refetchInterval` (polling) instead of WebSockets. This maintains compatibility with serverless/edge functions, avoids maintaining active server socket connections, and simplifies scalability.
* **Database Cascading Referential Integrity:** Database constraint directives (`onDelete: Cascade`) configured at the schema level. This ensures when a parent entity (e.g. `Project`) is deleted, all dependent children (e.g. `Message`, `Fragment`) are cleaned up automatically by the database engine, avoiding orphan records and database bloat.
* **JIT (Just-In-Time) Styling Engine:** Tailwind's compilation process that dynamically scans your React code for utility class names and generates only the exact CSS required, resulting in minimal CSS bundle sizes (often under 50KB for large applications).

---

## 2. Key Code Snippets & Interview Explanations

### Topic A: Recursive UI Component (The File Tree Explorer)
**Interview Question:** *"How did you build a file explorer that renders nested folder structures of infinite depth without crashing?"*

#### Code Implementation (`src/components/tree-view.tsx` conceptual representation)
```typescript
import { useState } from "react";
import { TreeItem } from "@/types";

interface TreeViewProps {
  item: TreeItem;
}

export function TreeView({ item }: TreeViewProps) {
  const [isOpen, setIsOpen] = useState(false);

  // Case 1: The item is a file (leaf node)
  if (typeof item === "string") {
    return <div className="pl-4 py-1 text-sm text-foreground/80">{item}</div>;
  }

  // Case 2: The item is a folder containing sub-items (recursive node)
  const [folderName, ...children] = item;

  return (
    <div className="pl-2">
      <div 
        onClick={() => setIsOpen(!isOpen)} 
        className="cursor-pointer py-1 font-semibold text-sm hover:text-primary transition"
      >
        📁 {folderName}
      </div>
      {isOpen && (
        <div className="border-l border-border ml-2 pl-2">
          {children.map((child, index) => (
            <TreeView key={index} item={child} /> // Recursive Self-Invocation
          ))}
        </div>
      )}
    </div>
  );
}
```

#### Talk Track (How to explain it):
> "To render an arbitrary folder structure, I implemented a **recursive React component**. First, I defined a recursive union type `TreeItem` which represents either a file `string` or a folder `[string, ...TreeItem[]]`. The component evaluates the item type: if it's a file, it renders a leaf node; if it's a folder, it renders a header and maps over its children, recursively mounting instances of itself. This separates data processing from rendering, is highly performant, and scales to infinite nesting depths."

---

### Topic B: Prefetching & SSR Dehydration (Eliminating Layout Shift)
**Interview Question:** *"How did you ensure that project pages render instantly on initial load without a loading screen, while still allowing the client to poll for database updates?"*

#### Code Implementation (`src/app/projects/[projectId]/page.tsx`)
```typescript
import { getQueryClient, trpc } from "@/trpc/server";
import { dehydrate, HydrationBoundary } from "@tanstack/react-query";
import { ProjectView } from "@/modules/projects/ui/views/project-view";

export default async function Page({ params }: { params: Promise<{ projectId: string }> }) {
  const { projectId } = await params;
  const queryClient = getQueryClient();

  // Prefetch database entity on the server securely
  await queryClient.prefetchQuery(
    trpc.projects.getOne.queryOptions({ id: projectId })
  );

  return (
    // Pass dehydrated server cache to client
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ProjectView projectId={projectId} />
    </HydrationBoundary>
  );
}
```

#### Talk Track (How to explain it):
> "I implemented an **SSR Hybrid Hydration pattern** using TanStack React Query and Next.js Server Components. Before the client receives any HTML, the server queries the database and populates the cache using `queryClient.prefetchQuery`. By wrapping the layout with `HydrationBoundary` and serializing the cache via `dehydrate()`, the client components wake up with the database data already present in cache memory. This eliminates layout shifts (CLS), hides initial spinners, and enables immediate background synchronization/polling."

---

### Topic C: Durable Step Executions (Preventing Request Timeouts)
**Interview Question:** *"AI generation takes several minutes. How did you run this background logic without hitting standard serverless function timeouts?"*

#### Code Implementation (`src/inngest/functions.ts` conceptual representation)
```typescript
export const knightFunction = inngest.createFunction(
  { id: "knight-function" },
  { event: "knight/run" },
  async ({ event, step }) => {
    // 1. Create isolation sandbox (resilient step)
    const sandboxId = await step.run("create-sandbox", async () => {
      const sandbox = await Sandbox.create("knights-nextjs-template");
      return sandbox.sandboxId;
    });

    // 2. Invoke Gemini LLM coding model (durable step)
    const filesJson = await step.run("run-ai-agent", async () => {
      const files = await callGeminiAgent(event.data.prompt);
      return files;
    });

    // 3. Write updates directly to database
    await step.run("write-to-db", async () => {
      await prisma.project.update({
        where: { id: event.data.projectId },
        data: { files: filesJson },
      });
    });
  }
);
```

#### Talk Track (How to explain it):
> "To execute slow processes like code sandboxing and LLM reasoning, I designed an **event-driven workflow with Inngest**. When the user makes a request, we send a lightweight event and return an immediate response to the client. The background executor catches the event. By splitting the logic into isolated `step.run` callbacks, Inngest serializes progress at each checkpoint. If a later step fails or exceeds execution limits, the engine retries only the failed step using the cached outputs of the completed steps."

---

## 3. High-Value Interview Q&A

### Q1: Why did you use tRPC instead of a standard REST API?
* **Answer:** "tRPC gives us **compile-time end-to-end type safety**. There's no need to build OpenAPI configurations or write duplicate TypeScript interfaces on the client. If I change a database column in Prisma, tRPC automatically highlights compile errors on the frontend page. This speeds up feature iteration and prevents API shape mismatches."

### Q2: Why choose polling over WebSockets for displaying live AI progress?
* **Answer:** "WebSockets are stateful, requiring persistent open connections which are expensive to scale, difficult to manage in serverless runtimes (like Vercel or Cloudflare Workers), and prone to dropping on mobile networks. Polling the tRPC endpoint (`refetchInterval: 2000`) is **stateless**, making it infinitely scalable, highly reliable, simple to configure, and perfectly compatible with serverless architecture."

### Q3: How do you handle security in E2B sandboxes? What stops a user from writing a script to hack your server?
* **Answer:** "E2B utilizes **Firecracker Micro-VMs**, which provide hardware-level virtualization. The code runs inside an isolated guest operating system, not a shared Docker kernel. There is no route back to our hosting infrastructure. Furthermore, all operations are secured behind tRPC middleware checking authenticated Clerk user tokens."

### Q4: Explain Next.js 15 App Router rendering options: SSG vs SSR vs ISR. Which is used here?
* **Answer:** "Next.js 15 supports Static Site Generation (SSG), Server-Side Rendering (SSR), and Incremental Static Regeneration (ISR). 
  - **SSG:** Compiles static HTML files at build time.
  - **SSR:** Generates HTML dynamically on every request.
  - **ISR:** Statically builds pages but updates them in the background after a specified timeout.
  
  In this project, the dashboard `/` and pricing `/pricing` pages leverage **SSG** for instant delivery. However, the workspace workspace page `/projects/[projectId]` uses **SSR (Dynamic Rendering)** because it relies on Clerk authentication context and dynamic project parameters that must be loaded fresh from PostgreSQL on every request."

### Q5: What is the benefit of React Server Components (RSC) on bundle size?
* **Answer:** "Since React Server Components execute exclusively on the server, all of their dependencies (like markdown parsers, date formatting libraries, or database clients) remain on the server and are never shipped to the client's browser. This keeps the bundle size small, speeds up the Page Load time, and improves Core Web Vitals like LCP (Largest Contentful Paint) and INP (Interaction to Next Paint)."

### Q6: How do you handle race conditions or duplicate project runs?
* **Answer:** "We solve this at two levels:
  1. **UI Optimistic State Control:** As soon as the project mutation triggers, we disable the form submit buttons, preventing the user from hitting Enter multiple times.
  2. **Inngest Concurrency Keys:** Inngest allows defining a `concurrency` key based on the `projectId`. If a user attempts to trigger multiple runs for the same project, Inngest serializes the queue, ensuring only one run executes at any given time."

---

## 4. Take-Home Assignment & Coding Challenge Playbook

When given a coding assignment (typically to build a full-stack feature, a mini-SaaS page, or connect an API), follow this structured playbook to stand out as a senior engineer.

### Phase 1: Planning and Architecture Setup
1. **Define Core Entities First:** Do not write UI components immediately. Start by sketching out your data models on paper.
2. **Setup the Database Schema:** Modify your Prisma schema file (`schema.prisma`). Ensure you configure appropriate relations, indexes (for faster querying on search fields), and cascade deletes.
   ```prisma
   // Example: Add index on foreign keys
   @@index([userId])
   ```
3. **Database Syncing:** Run `npx prisma migrate dev` to compile the client.
4. **Environment Check:** Configure `.env.local` with fallback development credentials, making sure no production secrets are checked into git.

### Phase 2: Building the API Contracts (tRPC Routers)
1. **Define Validation Schemas:** Create Zod schemas for all client inputs.
2. **Develop Router Procedures:** Write your backend procedures in `/server/procedures.ts`.
3. **Implement Middleware Early:** If the route requires authentication, wrap it in a protected procedure helper.
4. **Test the Contracts:** Call the backend mock endpoint using a client runner to verify inputs reject invalid payloads and return the correct structures.

### Phase 3: Dynamic Layouts & State Sync
1. **Use Route Segments Correctly:** Group presentation-focused pages under route groups `(home)` and dynamic project-space views under `[projectId]`.
2. **Optimize Rendering with SSR Hydration:** Use `prefetchQuery` and `HydrationBoundary` on dynamic pages to pre-fill TanStack Query cache.
3. **Implement Accessible Layouts:** Build layouts using Resizable Panels to prevent Layout Shift. Ensure elements (like tabs or dropdowns) utilize Radix UI primitives.
4. **Handle Loading & Error States:** Use Next.js loading skeletons (`loading.tsx`) and error boundaries (`error.tsx`) to catch component crashes gracefully.

### Phase 4: Long-Running & Event Tasks
1. **Delegate Work:** If a task takes longer than 2 seconds, do not execute it synchronously. Instead, trigger an Inngest background event (`inngest.send`).
2. **Design Durable Step Chains:** Split complex workflows into distinct `step.run` blocks, caching API responses or database writes.
3. **Configure Live Polling:** On the frontend, configure React Query with `refetchInterval` to poll for changes. This keeps your client synced with the database worker automatically.

### Phase 5: Submission Checklist
Before submitting the code challenge, double-check these details:
* **Remove Placeholders:** Ensure there are no left-over mock states or hardcoded IDs.
* **Fix Hydration Warnings:** Ensure elements that depend on client-side values (like dates or random slugs) are wrapped in custom lifecycle checks.
* **Document Everything:** Write a clear, comprehensive `README.md` containing:
  - An architecture block diagram explaining the data flow.
  - Setup and execution instructions (`npm run dev`).
  - Highlights of design choices (like choosing headless Radix elements for accessibility and tRPC for safety).
