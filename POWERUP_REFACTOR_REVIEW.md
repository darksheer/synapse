# Architectural Review: Refactoring Synapse Advanced Power-Ups

## Executive Summary
This report investigates the feasibility and architectural implications of refactoring the supporting systems of Synapse (specifically "Advanced Power-Ups" and external integrations) into modern, compiled languages such as Go or Rust.

Currently, Synapse is a massive, heavily optimized Python codebase that relies on custom frameworks (`Cell` for microservices, `Telepath` for RPC, and `Storm` as a DSL). While the core engine (`Cortex`) is highly specialized in Python, external supporting services often feel like "second-class citizens" due to the steep learning curve of the custom ecosystem and the inherent limitations of Python for packaging and concurrency.

Our findings show that **migrating Advanced Power-Ups to modern languages is entirely feasible and highly recommended** to improve developer experience (DX), performance, and deployment simplicity. This is primarily made possible by Synapse's robust, built-in HTTP/REST APIs, which bypass the need to implement Synapse's custom Python RPC framework in other languages.

---

## Analysis of Current Architecture

To understand how non-Python services can interact with Synapse, we must look at how Python services currently do it.

### 1. `synapse.lib.cell.Cell`
The `Cell` class is the fundamental building block of all Synapse microservices (Cortex, Axon, Aha, etc.).
- **What it does:** It handles startup, SSL/TLS certificate management, clustering (via `nexus`), and exposes both a custom RPC daemon and an HTTP server.
- **Why it matters:** Because every `Cell` (including the main `Cortex`) inherently starts an asynchronous HTTP web application (`tornado.web.Application`), the system is natively ready to handle standard RESTful communication, not just custom RPC.

### 2. `synapse.telepath`
Telepath is Synapse's custom Remote Method Invocation (RMI/RPC) framework.
- **How it works:** It uses an asynchronous protocol over TCP, SSL, or Unix sockets, serializing data via `msgpack`. It relies heavily on Python's dynamic nature (duck typing, dynamic method dispatch via `Proxy` objects, and async generators).
- **The Challenge:** Writing a full Telepath client in a strongly-typed, compiled language (like Rust or Go) would be a massive, brittle undertaking. The protocol expects dynamic object instantiation (`Share`, `Genr`) that maps poorly to static types.
- **The Verdict:** **Do not attempt to implement Telepath in other languages.** It is designed specifically for Python-to-Python communication.

### 3. HTTP/REST Integration Points
Fortunately, Synapse provides alternative integration points that are standard and language-agnostic.
- **Built-in APIs:** The Cortex exposes a robust suite of HTTP APIs (e.g., `/api/v1/storm`, `/api/v1/storm/call`, `/api/v1/model`). External services can authenticate, execute Storm queries, stream results, and modify the hypergraph entirely via JSON over HTTP.
- **Storm ExtApi:** Synapse supports user-defined HTTP endpoints (`/api/ext/*`) that map HTTP requests directly to Storm code. This allows for powerful webhook-style integrations.

---

## Proposal: Building Power-Ups in Modern Languages

To elevate Advanced Power-Ups to "first-class citizens," we recommend shifting away from building them as custom Python `Cell` subclasses connected via Telepath. Instead, they should be built as independent, standalone microservices in Go or Rust that communicate with Cortex via its HTTP API.

### The Target Architecture

1. **The Power-Up Service (Go/Rust):**
   - Built as a standard HTTP server/client.
   - Responsible for interacting with external APIs (e.g., Jira, MISP, Shodan).
   - Handles heavy data processing, concurrent networking, and caching.
   - Communicates with the Synapse Cortex solely via the `/api/v1/storm` HTTP endpoints.
2. **The Synapse Integration (Storm):**
   - The Cortex configuration uses `ExtApi` or custom Storm commands that make outbound HTTP requests (using Storm's `$lib.inet.http`) to the Power-Up Service.
   - Alternatively, the Power-Up Service pushes data into Synapse by POSTing Storm queries to the Cortex.

### Pros of this Approach

*   **Exceptional Developer Experience (DX):** Developers can use standard tooling, IDEs, testing frameworks, and linters for Go/Rust. They are not forced to learn Synapse's custom asynchronous `Cell` architecture.
*   **Strong Typing:** Languages like Rust and Go provide compile-time guarantees, significantly reducing runtime errors when parsing complex external data feeds (a common task for Power-Ups).
*   **Deployment Simplicity:** Both Rust and Go compile to single, statically linked binaries. This eliminates Python dependency hell (no `requirements.txt`, `pip`, or virtual environments needed for deployment) and results in incredibly small, efficient Docker containers.
*   **Concurrency & Performance:** Go's goroutines and Rust's async models are vastly superior to Python's `asyncio` for handling thousands of concurrent network connections (e.g., scraping APIs, ingesting massive threat feeds).
*   **Ecosystem:** You gain access to the massive libraries of Go and Rust for network protocols, parsers, and external SaaS integrations.

### Cons & Technical Challenges

*   **Loss of Telepath Magic:** You lose the ability to easily yield Python async generators directly into the Synapse core. All data transfer must be serialized to JSON/Msgpack over HTTP.
*   **Authentication Overhead:** You must manage Synapse API keys/tokens for authentication instead of relying on Telepath's internal certificate/handshake mechanisms.
*   **Initial Boilerplate:** You will need to write a lightweight, reusable SDK in Go/Rust that wraps the Synapse HTTP API to make executing Storm queries and handling the resulting node data ergonomic.

---

## Conclusion

The feeling that supporting systems are "second-class citizens" is a natural consequence of forcing external integrations into a highly specialized, custom Python ecosystem designed for a core database engine.

By leveraging Synapse's existing HTTP/REST capabilities, you can decouple Advanced Power-Ups from the Python core. Rewriting these connectors in Go or Rust will yield massive improvements in deployment, performance, and developer happiness, all while maintaining full compatibility with the Synapse platform. The first step should be to develop a simple, robust Go/Rust client library that wraps the `/api/v1/storm` endpoints.
### How Commands, Triggers, and Crons are Configured
You might be wondering: if the Advanced Power-Up is written in Go or Rust and sits "next to" Synapse, how does Synapse know about its commands, crons, or triggers?

The answer lies in **Storm Packages** (`synapse.tools.storm.pkg.gen`).

Even if the backend logic runs in a separate Go/Rust service, the "glue" that binds it to Synapse is a standard Storm Package (a YAML/JSON definition containing `.storm` files) loaded into Cortex. The external service does not require *any* modifications to the core Synapse engine.

#### 1. Configuring Commands
A custom command (e.g., `jira.issue.add`) is defined in the Storm Package that comes with your Power-Up. The Storm code for this command acts as a thin wrapper that uses Storm's built-in `$lib.inet.http` module to make an API call to your external Go/Rust service.

```storm
// Inside the Storm Package (e.g., jira.issue.add)
$url = "http://my-golang-powerup:8080/v1/issue/add"
$body = ({ "project": $project, "summary": $summary })
$headers = ({ "Authorization": $lib.auth.getExtApiToken() })

$resp = $lib.inet.http.post($url, headers=$headers, body=$body)
if ($resp.code != 200) {
    $lib.print("Error talking to Go service")
    return()
}
// The Go service handled the API call, created the issue,
// and perhaps even returned nodes for Synapse to yield.
yield $resp.body.nodes
```

#### 2. Configuring Triggers
Triggers in Synapse are native to Cortex (`synapse.lib.trigger`). You define the trigger entirely in Synapse using Storm (e.g., "when an `inet:ipv4` node is created, execute this Storm query").

The Storm query executed by the trigger simply calls your custom command (defined above), which in turn sends an HTTP webhook to your Go/Rust service.
*   **Synapse side:** Defines the trigger rule and calls `$lib.inet.http.post()`.
*   **Go/Rust side:** Listens for incoming webhooks on an HTTP endpoint, receives the node data, processes it, and potentially pushes new intelligence back into Synapse via the `/api/v1/storm` API.

#### 3. Configuring Crons (Scheduled Tasks)
For crons, you have two architectural choices:
*   **Synapse-Driven (Pull):** You create a native Synapse cron job (`synapse.lib.agenda`) that periodically executes a Storm command. That Storm command makes an HTTP GET request to your Go/Rust service, prompting the service to perform work and return results.
*   **Service-Driven (Push):** This is often the better approach for modern languages. You completely bypass Synapse's cron system. Instead, you use standard cron libraries in Go/Rust (which are highly robust). The Go/Rust service wakes up on its own schedule, fetches external data (e.g., scraping a threat feed), and uses the Synapse `/api/v1/storm` HTTP endpoint to bulk-ingest the new nodes into Cortex.

### Summary of the "Next-To" Architecture
*   **No changes to Synapse are required.**
*   The Go/Rust service sits "next to" Synapse as an independent container.
*   The Synapse Administrator installs a lightweight Storm Package (`.yaml`) to register commands and triggers inside Cortex.
*   Communication flows entirely over HTTP (REST/JSON) using `$lib.inet.http` (Cortex -> Service) and `/api/v1/storm` (Service -> Cortex).
