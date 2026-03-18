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

---

## Deep Dive: The Hosted Backend Lifecycle
Because Advanced Power-Ups explicitly require a **hosted backend service**, refactoring them to a modern language means fundamentally changing how they are deployed and how they interface with Synapse.

In the legacy Python model, Advanced Power-Ups are often built as a `synapse.lib.cell.Cell` subclass and form a tight cluster with Cortex using `Telepath` and `Nexus`.

In the modernized model, the Advanced Power-Up is a completely standalone container (e.g., a Go binary) that treats Synapse simply as a database and event router with a REST API.

Here is a deep dive into how this looks in practice.

### 1. Deployment Model (The "Sidecar" or Microservice)
You deploy your Advanced Power-Up exactly as you would deploy any modern microservice in Docker or Kubernetes.
*   **The Go/Rust Container:** A scratch or Alpine-based Docker image containing only your compiled binary. It exposes an HTTP port (e.g., `8080`) to receive webhook requests from Synapse.
*   **Network Proximity:** It is deployed alongside Cortex. It only needs network access to the Cortex HTTP API port (usually `4443` or `443`), and Cortex needs network access to the Go/Rust container's port.

### 2. The Bootstrapping Phase
A common challenge is: *How does Synapse know this new backend service exists if they don't share a custom Python RPC framework?*

The Go/Rust service bootstraps itself upon startup:
1.  **Startup:** The Go/Rust service spins up and connects to its own dependencies (Redis, external APIs, etc.).
2.  **Authentication:** It uses a configured Synapse API key or Service Account Token.
3.  **Self-Registration:** The service contains the Storm Package definitions (`.yaml` and `.storm` files) bundled directly inside its binary. On startup, the service makes an HTTP POST request to the Synapse `/api/v1/storm` endpoint, sending a Storm query like: `$lib.pkg.add($my_pkg_def)`.
4.  **Result:** Synapse has now installed the commands, triggers, and crons needed for this Power-Up. The backend service has successfully registered itself without requiring a human administrator to manually load packages.

### 3. Data Flow: Synapse calling the Backend (User Commands)
When an analyst types a command in Optic (e.g., `yara.scan_file`), here is the exact execution flow:

1.  **Storm Execution:** Cortex begins executing the `yara.scan_file` Storm command.
2.  **The HTTP Bridge:** The Storm command (installed during bootstrapping) contains `$lib.inet.http.post()`. It packages the current graph node (e.g., `file:bytes`) and context into a JSON payload.
3.  **Egress:** Cortex makes an HTTP POST request to the Go/Rust backend service (e.g., `http://yara-backend:8080/scan`).
4.  **Processing:** The Go/Rust service receives the JSON, performs the heavy lifting (scanning the file against YARA rules using a high-performance Rust library), and returns a JSON response to Synapse.
5.  **Ingress:** The Storm script parses the JSON response and yields the new nodes (e.g., `it:app:yara:match`) directly to the analyst.

### 4. Data Flow: The Backend calling Synapse (Background Sync)
When the Advanced Power-Up acts as an active collector or scraper, it drives the flow:

1.  **The Backend Loop:** The Go/Rust service runs a highly concurrent background worker pool (goroutines/Rust async). It connects to an external stream (e.g., a Firehose or MISP feed).
2.  **Buffering:** As it receives raw threat data, it translates it into Synapse's node model natively in memory.
3.  **Bulk Ingress:** Periodically (e.g., every 5 seconds or 10,000 events), the Go/Rust service formats a large Storm query containing all the new nodes and relationships.
4.  **Execution:** It POSTs the Storm query to the Synapse `/api/v1/storm` endpoint.
5.  **Result:** Cortex processes the query and updates the graph.

### Why this is a Massive Upgrade
By decoupling the backend service from Synapse's internal Python abstractions (`Cell`, `Telepath`):
*   **Resilience:** If the Go service crashes, it does not disrupt the Cortex cluster.
*   **Scale:** You can horizontally scale the Go/Rust collector pods independently of Cortex.
*   **Polyglot:** You are no longer restricted to Python libraries for parsing complex file formats or handling high-throughput networking.

---

## Operational Considerations & Potential Blind Spots
While moving to a compiled, decoupled backend provides massive architectural benefits, it is crucial to address the operational realities of building and maintaining these Advanced Power-Ups. What are we missing when we remove the tight Python/Telepath coupling?

### 1. Web UI Interface (Optic) Management
**The Concern:** If the backend is written in Go or Rust, how do we build the UI for analysts to interact with it inside Synapse Optic?

**The Reality:** The Optic UI is entirely driven by JSON configuration files located within the `optic` directory of a Storm Package.
*   Because your Go/Rust service *must* install a Storm Package into Cortex to register its commands, you simply include the Optic configuration in that same package.
*   The Optic UI calls Storm commands (which hit your Go/Rust HTTP API), and renders nodes based on the models you defined in the package.
*   **Verdict:** You lose no UI capabilities. Optic does not care what language the backend is written in, because it only ever speaks Storm.

### 2. Configuration and Secrets Management
**The Concern:** Synapse has built-in ways to manage secrets (`$lib.vault`) and Cell configurations (`cell.yaml`). Where do API keys for third-party services (like Shodan or MISP) live now? If the backend is decoupled, does the DevOps team now have to manage secrets in multiple places (a "split-brain" problem)?

**The Reality:** The most modern and secure approach is to treat the Go/Rust backend service as a **Stateless Worker**. Synapse itself remains the single source of truth for all Power-Up configurations and credentials.

You completely avoid the "split-brain" anti-pattern by leveraging Synapse's native `$lib.vault` for *everything*:
*   **Analyst-Provided Secrets:** If the analyst uses a personal API key, they configure it via a setup command (e.g., `my_powerup.setup.apikey <key>`). When the analyst runs a Storm command, the Storm code retrieves the key using `$lib.vault` and securely passes it to the Go/Rust service as an HTTP header (e.g., `X-API-Key`). The Go/Rust service uses the key in-memory for that specific request and discards it.
*   **Global Service Secrets:** If the Power-Up requires a global credential to run a background synchronization job (e.g., an enterprise MISP feed key), the Synapse Administrator still stores it in Synapse's `$lib.vault`.
    *   *How does the background worker get it?* When the Go/Rust service wakes up to run its cron loop, it first makes an authenticated HTTP call to Synapse (`/api/v1/storm`) asking for its configuration: `yield $lib.vault.get('my_powerup:global_key')`.
    *   It retrieves the key Just-In-Time (JIT), uses it to fetch the external data, and then discards the key.

This ensures that backing up Cortex backs up all secrets, migrating Cortex migrates all secrets, and DevOps does not need to inject complex environment variables or Kubernetes Secrets into the Power-Up containers.

### 3. Internal Network Security
**The Concern:** If the Go/Rust service exposes an HTTP API (e.g., `http://my-golang-powerup:8080/scan`) for Synapse to call, what stops someone else on the network from calling it directly and bypassing Synapse's authentication?

**The Reality:** You must secure the HTTP bridge.
*   Because the service is decoupled from Telepath, it no longer inherits Synapse's mutual TLS (mTLS) certificate trust.
*   **Solution:** You must implement a shared secret mechanism. When the Go/Rust service starts, it generates (or is provided via a secure orchestrator) an internal token. When it registers its Storm Package with Cortex, it hardcodes that token into the HTTP headers of its commands. The Go/Rust service must reject any incoming HTTP request that does not contain this token. Alternatively, enforce strict network policies (e.g., Kubernetes NetworkPolicies) so only the Cortex pod can reach the Go/Rust pod.

### 4. Developer Workflows (Creation, Updating, Testing)
**The Concern:** Building a Power-Up now requires knowledge of Go/Rust *and* Storm. How do developers test and package this?

**The Reality:** The developer workflow becomes more complex but highly standardized:
*   **The Monorepo Approach:** The Git repository for the Power-Up must contain both the Go/Rust source code and the `storm` directory (containing the `.yaml` and `.storm` files).
*   **Local Testing:** Developers will use `docker-compose`. They will spin up a local Synapse Cortex, their local Go/Rust service, and a script to auto-load the Storm Package into Cortex. This allows rapid end-to-end testing.
*   **CI/CD Pipeline:** Your build pipeline must do two things:
    1. Compile the Go/Rust binary.
    2. Package the `storm` directory into the binary (using `go:embed` or Rust's `include_bytes!`). This ensures that the single compiled artifact contains exactly the right version of the Storm Package required to interface with it, preventing version drift.
