# Build Your First Production MCP Server

A hands-on, end-to-end tutorial. We build one real server — an MCP wrapper around an
internal **Support Tickets** REST API — and take it from `npx` on your laptop to an
authenticated Streamable HTTP service an agent can call in production.

Primary language is **TypeScript** (official `@modelcontextprotocol/sdk`). Python notes
are added where the FastMCP path differs meaningfully.

**Version note (important — the SDKs changed in 2025).** This tutorial targets the
**v1.x line, `@modelcontextprotocol/sdk@^1.29`**, which is the version recommended for
production today. There is a separate **V2** preview (package `@modelcontextprotocol/server`,
`z.object(...)`-wrapped schemas) — do **not** copy V2 snippets into a v1 project; the
import paths and the schema shape differ. Every code block below is written for v1.x.
Sources are listed at the end.

The fictional API we wrap (so the lessons transfer to what you actually build):

```
GET  /tickets/{id}                  -> one ticket
GET  /tickets?status=&assignee=&q=  -> filtered list
POST /tickets/{id}/comments         -> add a comment (a write)
```

---

## 1. Setup

### 1.1 Scaffold

```bash
mkdir tickets-mcp && cd tickets-mcp
npm init -y
npm pkg set type=module
```

### 1.2 Dependencies

Pin Zod to the **v3 line**. `@modelcontextprotocol/sdk@1.29` expects `zod@^3.25`; a bare
`npm i zod` today installs Zod 4 and will break the SDK's schema types. (Source: SDK
issue #925 / #1429.)

```bash
npm i @modelcontextprotocol/sdk@^1.29 zod@^3.25
npm i express jose                      # only needed for the HTTP + auth steps
npm i -D typescript tsx @types/node @types/express
npx tsc --init
```

In `tsconfig.json` set modern module resolution so the `.js`-suffixed ESM imports the SDK
uses resolve correctly:

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "strict": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

> **Node:** use **Node 20+** (Node 22+ if you also want to run MCP Inspector, which
> requires `^22.7.5`).

### 1.3 The two transports you will use

- **stdio** — the server runs as a child process the client spawns. Zero network, zero
  auth. Use it for local dev and for desktop clients (Claude Desktop, Claude Code).
- **Streamable HTTP** — the server is a long-running HTTP service at `/mcp`. Use it for
  production / remote / multi-user. This is where auth and deployment apply.

We will write the tool logic **once** in a shared `getServer()` factory and mount it on
*both* transports. That single factory is the spine of the whole project.

### 1.4 Project layout

```
src/
  api.ts        # thin REST client for the tickets API
  server.ts     # getServer() — registers all tools (transport-agnostic)
  stdio.ts      # dev entry point  (stdio transport)
  http.ts       # prod entry point (Streamable HTTP transport)
  auth.ts       # bearer-token middleware (used by http.ts)
```

---

## 2. The REST client (`src/api.ts`)

Keep the HTTP details in one place so tools stay readable. Nothing MCP-specific here.

```ts
// src/api.ts
const BASE_URL = process.env.TICKETS_API_URL ?? "https://api.internal.example.com";
const API_KEY = process.env.TICKETS_API_KEY ?? "";

export interface Ticket {
  id: string;
  subject: string;
  status: "open" | "pending" | "solved" | "closed";
  priority: "low" | "normal" | "high" | "urgent";
  assignee: string | null;
  requester: string;
  created_at: string;
  updated_at: string;
  description: string;
}

/** Thrown for non-2xx responses so tool handlers can translate them for the agent. */
export class ApiError extends Error {
  constructor(public status: number, message: string) {
    super(message);
  }
}

async function call<T>(path: string, init: RequestInit = {}): Promise<T> {
  const res = await fetch(`${BASE_URL}${path}`, {
    ...init,
    headers: {
      Authorization: `Bearer ${API_KEY}`,
      "Content-Type": "application/json",
      ...init.headers,
    },
  });
  if (!res.ok) {
    const body = await res.text().catch(() => "");
    throw new ApiError(res.status, body || res.statusText);
  }
  return (await res.json()) as T;
}

export const ticketsApi = {
  getTicket: (id: string) => call<Ticket>(`/tickets/${encodeURIComponent(id)}`),

  searchTickets: (params: Record<string, string | undefined>) => {
    const q = new URLSearchParams(
      Object.entries(params).filter(([, v]) => v != null) as [string, string][],
    );
    return call<Ticket[]>(`/tickets?${q.toString()}`);
  },

  addComment: (id: string, body: string, isPublic: boolean) =>
    call<{ id: string; comment_id: string }>(
      `/tickets/${encodeURIComponent(id)}/comments`,
      { method: "POST", body: JSON.stringify({ body, public: isPublic }) },
    ),
};
```

---

## 3. Your first tool — `get_ticket`

Start the server factory with a single, minimal tool. Note three v1.x specifics that
people get wrong:

1. Import paths end in **`.js`** (`/server/mcp.js`), even from TypeScript.
2. `inputSchema` is a **raw Zod shape** — `{ id: z.string() }` — **not** `z.object({...})`.
   (That `z.object(...)` form is V2. In v1.x the SDK wraps the shape for you.)
3. A handler returns `{ content: [...] }`; set `isError: true` to signal failure.

```ts
// src/server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";
import { ticketsApi, ApiError, Ticket } from "./api.js";

export function getServer(): McpServer {
  const server = new McpServer(
    { name: "tickets-mcp", version: "1.0.0" },
    {
      // Surfaced to the model as system-level guidance about the whole server.
      instructions:
        "Tools to read and comment on customer support tickets. " +
        "Prefer search_tickets to find work; never invent ticket IDs.",
    },
  );

  server.registerTool(
    "get_ticket",
    {
      title: "Get ticket",
      description:
        "Fetch a single support ticket by its exact ID (e.g. 'TICK-1042'). " +
        "Returns subject, status, priority, assignee, and full description.",
      inputSchema: {
        id: z.string().describe("Exact ticket ID, e.g. 'TICK-1042'."),
      },
    },
    async ({ id }) => {
      const t = await ticketsApi.getTicket(id);
      return { content: [{ type: "text", text: formatTicket(t) }] };
    },
  );

  return server;
}

/** Compact, model-friendly rendering. Avoid dumping raw JSON the model must re-parse. */
function formatTicket(t: Ticket): string {
  return [
    `Ticket ${t.id} — ${t.subject}`,
    `Status: ${t.status} | Priority: ${t.priority} | Assignee: ${t.assignee ?? "unassigned"}`,
    `Requester: ${t.requester} | Updated: ${t.updated_at}`,
    "",
    t.description,
  ].join("\n");
}
```

### Run it over stdio (`src/stdio.ts`)

```ts
// src/stdio.ts
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { getServer } from "./server.js";

const server = getServer();
const transport = new StdioServerTransport();
await server.connect(transport);
// Never write to stdout here — stdout is the MCP channel. Log to stderr if needed.
```

```bash
npx tsx src/stdio.ts   # waits for a client; jump to step 6 to drive it
```

---

## 4. Designing the tool surface well

**The single most important design rule: an MCP server is not a 1:1 proxy of your REST
API.** A REST API is designed for code that already knows what it wants. An agent is
guessing. So you design tools around *agent intents*, with descriptions, sensible
defaults, and **bounded** output — not around endpoints.

Concretely, do **not** ship a `list_tickets` tool that returns every ticket. That floods
the context window, costs tokens, and buries the answer. Ship a **filterable, bounded
`search_tickets`** instead.

```ts
// add inside getServer(), in src/server.ts
server.registerTool(
  "search_tickets",
  {
    title: "Search tickets",
    description:
      "Find tickets matching filters. Use this to discover work before acting. " +
      "Results are capped at 25; narrow the filters if you need more specific results. " +
      "All filters are optional and combine with AND.",
    inputSchema: {
      status: z
        .enum(["open", "pending", "solved", "closed"])
        .optional()
        .describe("Only tickets in this status."),
      assignee: z
        .string()
        .optional()
        .describe("Username of the assignee, or 'unassigned'."),
      query: z
        .string()
        .optional()
        .describe("Free-text match against subject and description."),
      limit: z
        .number()
        .int()
        .min(1)
        .max(25)
        .default(10)
        .describe("Max results to return (1-25, default 10)."),
    },
  },
  async ({ status, assignee, query, limit }) => {
    const results = await ticketsApi.searchTickets({ status, assignee, q: query });
    const capped = results.slice(0, limit);

    if (capped.length === 0) {
      return {
        content: [
          {
            type: "text",
            text: "No tickets matched. Try removing a filter or broadening the query.",
          },
        ],
      };
    }

    const lines = capped.map(
      (t) =>
        `${t.id}\t[${t.status}/${t.priority}]\t${t.subject}` +
        `\t(${t.assignee ?? "unassigned"})`,
    );
    const more =
      results.length > capped.length
        ? `\n\n…${results.length - capped.length} more not shown. Add filters to narrow.`
        : "";

    return {
      content: [
        { type: "text", text: `${capped.length} ticket(s):\n${lines.join("\n")}${more}` },
      ],
    };
  },
);
```

Design choices baked in above, and why they matter:

- **Bounded output (`max(25)`, `.slice`).** The schema *and* the handler both enforce the
  cap. Schemas can be coaxed; defense-in-depth in the handler is non-negotiable.
- **Enums over free strings** (`status`) — the model can't typo `"opened"`.
- **Descriptions written for a reader who can't see your code.** Each field says what it
  *means* and gives an example. The tool description tells the model *when* to reach for
  it ("discover work before acting").
- **A useful empty result** — tell the agent how to recover, don't just return `[]`.
- **Terse, tab-separated rows** instead of JSON — fewer tokens, still scannable; the agent
  calls `get_ticket` for full detail on the one row it cares about.

---

## 5. Adding more tools (and stopping)

A good first server is **small** — aim for **under ~7 tools**. Each extra tool is more
text in every request, more surface for the model to pick wrong, and more to secure.
Too many tools measurably degrades selection accuracy. Pick the few that cover the real
workflow: **find → inspect → act.**

We have `search_tickets` (find) and `get_ticket` (inspect). Add **one write tool** —
`add_ticket_comment` — and stop. Writes deserve special care: bound them, default to the
*safer* behavior, and make side effects explicit in the description so the agent (and a
human-in-the-loop UI) can reason about confirmation.

```ts
// add inside getServer(), in src/server.ts
server.registerTool(
  "add_ticket_comment",
  {
    title: "Add ticket comment",
    description:
      "Add a comment to a ticket. THIS WRITES DATA and may notify the customer. " +
      "Set public=false for an internal note (the default). " +
      "Confirm the ticket ID and comment text with the user before calling.",
    inputSchema: {
      id: z.string().describe("Exact ticket ID to comment on."),
      body: z
        .string()
        .min(1)
        .max(5000)
        .describe("Comment text. Plain text or markdown."),
      public: z
        .boolean()
        .default(false)
        .describe("true = visible to the customer; false = internal note. Default false."),
    },
    annotations: {
      readOnlyHint: false,
      destructiveHint: false, // adds data, doesn't delete/overwrite
      idempotentHint: false, // calling twice posts two comments
    },
  },
  async ({ id, body, public: isPublic }) => {
    const result = await ticketsApi.addComment(id, body, isPublic);
    const visibility = isPublic ? "PUBLIC (customer-visible)" : "internal";
    return {
      content: [
        {
          type: "text",
          text: `Added ${visibility} comment ${result.comment_id} to ticket ${id}.`,
        },
      ],
    };
  },
);
```

Notes:

- **`annotations`** (`readOnlyHint` / `destructiveHint` / `idempotentHint`) are MCP's
  standard hints. Clients use them to decide what needs a confirmation prompt. Setting
  them honestly is how you get "confirm before write" behavior without inventing your own
  protocol. Mark `get_ticket` / `search_tickets` with `readOnlyHint: true`.
- **Default to the safe side**: `public` defaults to `false`. If the model omits it, you
  post a harmless internal note, never an accidental customer email.
- The description states the side effect in plain words and explicitly asks for
  confirmation — your client's human-in-the-loop step keys off exactly this.

Final surface: **3 tools.** That covers the workflow and leaves room to grow.

---

## 6. Error handling that helps the agent recover

The agent only sees what you return. A bare 500 is a dead end; a specific, actionable
message lets it retry, fix its input, or ask the user. The pattern: **catch known
failures, map them to guidance, and return `isError: true`** (which surfaces the text to
the model as a tool error rather than a normal result).

Wrap every handler once with a helper instead of repeating try/catch:

```ts
// add to src/server.ts (above getServer)
import { ApiError } from "./api.js";

type ToolResult = { content: { type: "text"; text: string }[]; isError?: boolean };

function safe(handler: (a: any, extra: any) => Promise<ToolResult>) {
  return async (args: any, extra: any): Promise<ToolResult> => {
    try {
      return await handler(args, extra);
    } catch (err) {
      if (err instanceof ApiError) {
        const guidance =
          err.status === 404
            ? "No ticket with that ID exists. Use search_tickets to find the correct ID."
            : err.status === 401 || err.status === 403
              ? "The upstream API rejected the request (auth). Do not retry; report this."
              : err.status === 429
                ? "Rate limited upstream. Wait a few seconds before trying again."
                : `Upstream API error (HTTP ${err.status}). Retrying may help.`;
        return {
          content: [{ type: "text", text: `${guidance} (detail: ${err.message})` }],
          isError: true,
        };
      }
      return {
        content: [{ type: "text", text: `Unexpected error: ${(err as Error).message}` }],
        isError: true,
      };
    }
  };
}
```

Then wrap each handler — e.g.:

```ts
server.registerTool(
  "get_ticket",
  { /* …config from step 3… */ },
  safe(async ({ id }) => {
    const t = await ticketsApi.getTicket(id);
    return { content: [{ type: "text", text: formatTicket(t) }] };
  }),
);
```

Rules of thumb:

- **Map status codes to *next actions*** ("use search_tickets to find the ID"), not just
  "404 Not Found".
- **Distinguish retryable from terminal.** 429/5xx → "retry"; 401/403 → "don't retry,
  report". This stops the agent from hammering a broken endpoint.
- **Never leak stack traces, internal hostnames, or tokens** into the message.
- **Validation errors are free** — Zod rejects bad input *before* your handler runs and
  the SDK returns a structured schema error to the client automatically.

---

## 7. Test it: MCP Inspector + Claude Code

### 7.1 MCP Inspector (visual, transport-agnostic)

Inspector is the fastest way to see your tools, fire calls by hand, and read raw
protocol traffic. It spawns the stdio server for you.

```bash
npx @modelcontextprotocol/inspector npx tsx src/stdio.ts
# opens http://localhost:6274
```

In the UI: **Connect → List Tools** (confirm `get_ticket`, `search_tickets`,
`add_ticket_comment` appear with your descriptions) → run `search_tickets` with
`{ "status": "open" }` and inspect the result. For a deployed server, pick the
**Streamable HTTP** transport, enter your `/mcp` URL, and add an `Authorization: Bearer …`
header.

### 7.2 Wire it into Claude Code

Claude Code reads an `.mcp.json` at the project root (commit it to share with your team):

```jsonc
// .mcp.json
{
  "mcpServers": {
    "tickets": {
      "command": "npx",
      "args": ["tsx", "/abs/path/to/tickets-mcp/src/stdio.ts"],
      "env": {
        "TICKETS_API_URL": "https://api.internal.example.com",
        "TICKETS_API_KEY": "dev-key-xxxxx"
      }
    }
  }
}
```

Equivalently from the CLI:

```bash
claude mcp add tickets --env TICKETS_API_KEY=dev-key-xxxxx -- npx tsx /abs/path/to/tickets-mcp/src/stdio.ts
```

For **Claude Desktop**, the same `mcpServers` block goes in its config file
(`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS); restart the
app. **Verify** by asking: *"Search open tickets assigned to maria and summarize the
urgent ones."* You should see the model call `search_tickets`, then `get_ticket`, in the
tool-call trace.

> **Python equivalent:** the same `mcpServers` entry, with
> `"command": "uv", "args": ["run", "server.py"]`.

---

## 8. Auth for production — OAuth 2.1 Resource Server

For a remote Streamable HTTP server, **your MCP server is an OAuth 2.1 Resource Server.**
It does **not** log users in; it **validates** the bearer token the client already holds,
and the two checks that matter are:

1. **Signature + issuer** — the token is real and from your authorization server.
2. **Audience binding (RFC 8707)** — the token's `aud` claim equals *this server's*
   resource URL. This is what stops a token minted for service A from being replayed
   against your MCP server.

You also publish **`/.well-known/oauth-protected-resource`** (RFC 9728) so a client that
gets a `401` can discover *which* authorization server to go get a token from.

Here is a practical, self-contained middleware skeleton using `jose`. (For a
batteries-included alternative, the SDK ships `requireBearerAuth` + `mcpAuthRouter`; this
hand-rolled version shows what they do under the hood.)

```ts
// src/auth.ts
import { createRemoteJWKSet, jwtVerify } from "jose";
import type { NextFunction, Request, Response } from "express";

const OAUTH_ISSUER = process.env.OAUTH_ISSUER!; // e.g. https://auth.example.com/
const PUBLIC_BASE_URL = process.env.PUBLIC_BASE_URL ?? "http://localhost:3333";
const RESOURCE_URL = `${PUBLIC_BASE_URL}/mcp`; // must equal the token's `aud`
const JWKS = createRemoteJWKSet(new URL(`${OAUTH_ISSUER}.well-known/jwks.json`));

// Make the verified identity available to handlers in a type-safe way.
declare global {
  // eslint-disable-next-line @typescript-eslint/no-namespace
  namespace Express {
    interface Request {
      auth?: { token: string; clientId: string; scopes: string[]; sub?: string };
    }
  }
}

function unauthorized(res: Response, error: string) {
  // RFC 9728: point clients at the metadata doc so they can recover.
  res.setHeader(
    "WWW-Authenticate",
    `Bearer error="${error}", resource_metadata="${PUBLIC_BASE_URL}/.well-known/oauth-protected-resource"`,
  );
  res.status(401).json({ error });
}

export async function requireBearerToken(req: Request, res: Response, next: NextFunction) {
  const header = req.header("authorization") ?? "";
  if (!header.toLowerCase().startsWith("bearer ")) {
    return unauthorized(res, "missing_bearer_token");
  }
  const token = header.slice("bearer ".length);
  try {
    const { payload } = await jwtVerify(token, JWKS, {
      issuer: OAUTH_ISSUER,
      audience: RESOURCE_URL, // <-- RFC 8707 audience binding
    });
    const scopes = typeof payload.scope === "string" ? payload.scope.split(" ") : [];
    req.auth = {
      token,
      clientId: (payload.azp ?? payload.client_id) as string,
      scopes,
      sub: typeof payload.sub === "string" ? payload.sub : undefined,
    };
    next();
  } catch {
    return unauthorized(res, "invalid_token");
  }
}

// RFC 9728 protected-resource metadata.
export function protectedResourceMetadata(_req: Request, res: Response) {
  res.json({
    resource: RESOURCE_URL,
    authorization_servers: [OAUTH_ISSUER],
    bearer_methods_supported: ["header"],
    scopes_supported: ["tickets:read", "tickets:write"],
  });
}
```

**Enforce scopes inside the handler**, not just at the gate — a valid token does not
imply permission for *this* action. The SDK threads the verified identity into the
handler as `extra.authInfo`:

```ts
// inside the add_ticket_comment handler (a write -> needs the write scope)
async ({ id, body, public: isPublic }, extra) => {
  const scopes: string[] = extra?.authInfo?.scopes ?? [];
  if (!scopes.includes("tickets:write")) {
    return {
      content: [{ type: "text", text: "Forbidden: token lacks 'tickets:write' scope." }],
      isError: true,
    };
  }
  // …perform the write…
}
```

Golden rules: **bind the audience** (without it any valid token works), **enforce scopes
per-tool**, and **trust the token's `sub`, never a user ID passed as a tool argument** —
the model could be tricked into impersonation.

---

## 9. Deployment — the Streamable HTTP entry point + a container

### 9.1 The production entry point (`src/http.ts`)

This is the canonical v1.x stateful Streamable HTTP pattern: one transport per session,
keyed by the `mcp-session-id` header, created on the `initialize` request and reused
after. `POST /mcp` handles JSON-RPC; `GET /mcp` opens the server→client SSE stream;
`DELETE /mcp` tears a session down.

```ts
// src/http.ts
import express, { Request, Response } from "express";
import { randomUUID } from "node:crypto";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { isInitializeRequest } from "@modelcontextprotocol/sdk/types.js";
import { getServer } from "./server.js";
import { requireBearerToken, protectedResourceMetadata } from "./auth.js";

const app = express();
app.use(express.json());

// Discovery endpoint is public; everything under /mcp requires a valid token.
app.get("/.well-known/oauth-protected-resource", protectedResourceMetadata);

const transports: Record<string, StreamableHTTPServerTransport> = {};

const postHandler = async (req: Request, res: Response) => {
  const sessionId = req.headers["mcp-session-id"] as string | undefined;
  let transport: StreamableHTTPServerTransport;

  if (sessionId && transports[sessionId]) {
    transport = transports[sessionId]; // existing session
  } else if (!sessionId && isInitializeRequest(req.body)) {
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
      onsessioninitialized: (sid) => {
        transports[sid] = transport;
      },
    });
    transport.onclose = () => {
      const sid = transport.sessionId;
      if (sid) delete transports[sid];
    };
    await getServer().connect(transport); // fresh server instance per session
  } else {
    return res.status(400).json({
      jsonrpc: "2.0",
      error: { code: -32000, message: "Bad Request: no valid session ID" },
      id: null,
    });
  }
  await transport.handleRequest(req, res, req.body);
};

// GET (SSE stream) and DELETE (terminate) share the same lookup logic.
const sessionRequestHandler = async (req: Request, res: Response) => {
  const sessionId = req.headers["mcp-session-id"] as string | undefined;
  if (!sessionId || !transports[sessionId]) {
    return res.status(400).send("Invalid or missing session ID");
  }
  await transports[sessionId].handleRequest(req, res);
};

app.post("/mcp", requireBearerToken, postHandler);
app.get("/mcp", requireBearerToken, sessionRequestHandler);
app.delete("/mcp", requireBearerToken, sessionRequestHandler);

const port = Number(process.env.PORT ?? 3333);
app.listen(port, () => console.error(`tickets-mcp listening on :${port}/mcp`));
```

> To run **without** auth during local HTTP testing, drop the `requireBearerToken`
> argument from the three routes. Re-add it before shipping.

### 9.2 Containerize

```dockerfile
# Dockerfile
FROM node:22-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx tsc
EXPOSE 3333
CMD ["node", "dist/http.js"]
```

```bash
docker build -t tickets-mcp .
docker run -p 3333:3333 \
  -e TICKETS_API_URL=https://api.internal.example.com \
  -e TICKETS_API_KEY=$TICKETS_API_KEY \
  -e OAUTH_ISSUER=https://auth.example.com/ \
  -e PUBLIC_BASE_URL=https://mcp.example.com \
  tickets-mcp
```

Put it behind TLS (a load balancer or reverse proxy terminating HTTPS) so `PUBLIC_BASE_URL`
is `https://…` — the audience check compares against that exact URL.

> **Cloudflare Workers alternative:** Cloudflare's `agents` SDK exposes `McpAgent` with a
> built-in Streamable HTTP handler and OAuth provider, so you skip the Express plumbing
> above and `wrangler deploy`. The *tool design* (steps 3–6) is identical — only the
> transport wiring changes.
>
> **Python alternative:** FastMCP collapses this whole step into
> `mcp = FastMCP("tickets"); mcp.run(transport="streamable-http", host="0.0.0.0", port=3333)`,
> with tools declared via `@mcp.tool` and Pydantic/type-hint argument schemas.

---

## 10. Hardening checklist before you ship

Run through this before exposing the server to real agents and real data:

- [ ] **Read-only by default.** Only `add_ticket_comment` writes; reads carry
      `readOnlyHint: true`, the write carries honest `destructiveHint`/`idempotentHint`.
      Consider a `READ_ONLY=true` env flag that skips registering write tools entirely for
      lower-trust deployments.
- [ ] **Scrub PII in responses.** Don't return what the agent doesn't need. Redact emails,
      phone numbers, and full names from ticket bodies before they enter the context window:

      ```ts
      const scrub = (s: string) =>
        s
          .replace(/[\w.+-]+@[\w-]+\.[\w.-]+/g, "[email]")
          .replace(/\+?\d[\d ().-]{7,}\d/g, "[phone]");
      ```

- [ ] **Rate-limit** the HTTP entry point (`express-rate-limit`) per token/IP so a runaway
      agent loop can't melt the upstream API.
- [ ] **Bound everything.** Caps on result counts (step 4) and on input sizes
      (`body.max(5000)`) — already in the schemas; keep them.
- [ ] **Secrets via env, never in code or `.mcp.json` committed to a public repo.** Inject
      `TICKETS_API_KEY` / OAuth config at runtime; keep `.mcp.json` examples keyless.
- [ ] **Audience + scope enforced** (step 8): `audience` set on `jwtVerify`, per-tool scope
      checks in handlers, identity taken from `sub` not from tool args.
- [ ] **No stdout writes on the stdio server** — it corrupts the protocol stream. Log to
      `stderr` only.
- [ ] **Validate egress.** The MCP server should only ever call your tickets API host —
      don't let a tool argument become an arbitrary URL (SSRF).

---

## 11. Connecting it to an agent (production)

Point Claude Code at the deployed HTTPS endpoint with a bearer token:

```jsonc
// .mcp.json  (production, remote)
{
  "mcpServers": {
    "tickets": {
      "type": "http",
      "url": "https://mcp.example.com/mcp",
      "headers": { "Authorization": "Bearer ${TICKETS_MCP_TOKEN}" }
    }
  }
}
```

```bash
# or via CLI
claude mcp add --transport http tickets https://mcp.example.com/mcp \
  --header "Authorization: Bearer $TICKETS_MCP_TOKEN"
```

**Example interaction** (what you should observe in the trace):

> **You:** "Which urgent tickets are still open and unassigned? Add an internal note on the
> oldest one saying I'm picking it up."
>
> 1. Model calls `search_tickets { "status": "open", "assignee": "unassigned", "query": "" }`
>    → gets a bounded list back.
> 2. Filters to urgent, picks the oldest, calls `get_ticket { "id": "TICK-1042" }` for detail.
> 3. Calls `add_ticket_comment { "id": "TICK-1042", "body": "Picking this up.", "public": false }`
>    — your client prompts for confirmation first because of the write hints, and the token
>    must carry `tickets:write` or the handler returns the scope error from step 8.

That find → inspect → act loop, over three well-described, bounded, authenticated tools,
is a production MCP server.

---

## Sources

- TypeScript SDK — server guide & README (v1.x):
  https://github.com/modelcontextprotocol/typescript-sdk/blob/main/docs/server.md ·
  https://ts.sdk.modelcontextprotocol.io/documents/server.html
- Streamable HTTP session pattern (example source):
  https://github.com/modelcontextprotocol/typescript-sdk/blob/v1.x/src/examples/server/simpleStreamableHttp.ts
- npm package & Zod-version compatibility (issues #925, #1429):
  https://www.npmjs.com/package/@modelcontextprotocol/sdk
- MCP Authorization spec (audience binding, RFC 8707 / RFC 9728):
  https://modelcontextprotocol.io/docs/tutorials/security/authorization
- OAuth 2.1 + Streamable HTTP production walkthrough:
  https://nerdleveltech.com/mcp-server-typescript-oauth-streamable-http-production-tutorial
- MCP Inspector: https://modelcontextprotocol.io/docs/tools/inspector ·
  https://github.com/modelcontextprotocol/inspector
- Cloudflare Workers MCP (`McpAgent`): https://developers.cloudflare.com/agents/model-context-protocol/
- Python FastMCP: https://github.com/modelcontextprotocol/python-sdk
