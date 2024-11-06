# gRPC from the Browser (without using `grpc-web`)

This repository documents an architecture and implementation that allows **browsers to talk directly to gRPC services over HTTP/2**, without using `grpc-web`, `protoc` plugins, or protocol translation layers.

The goal is to keep:

*   **gRPC semantics intact**
    
*   **Browser-native APIs only**
    
*   **Infrastructure responsibilities cleanly separated**
    

* * *

## Motivation

gRPC provides:

*   Strongly typed service contracts
    
*   First-class streaming (client, server, bidirectional)
    
*   A single protocol usable across heterogeneous systems
    

However, browser support has traditionally required `grpc-web`, which introduces:

*   A custom protocol
    
*   Extra tooling (`protoc` plugins)
    
*   Envoy translation (custom grpc) filters
    
*   A split ecosystem
    

This project explores and documents a different path:

> **Use native browser capabilities to speak gRPC over real HTTP/2.**

* * *

## Browser-side gRPC (no `grpc-web`)

The browser client is implemented using:

*   **`fetch()`** — native HTTP/2 support
    
*   **`Uint8Array`** — raw gRPC binary framing
    
*   **`ReadableStream`** — streaming in both directions: a stream *as* the request body, and the
    response read as it arrives
    
*   **[`protobufjs`](https://www.npmjs.com/package/protobufjs)** — protobuf in pure JavaScript, so
    no `protoc` and no native toolchain
    

This allows the browser to:

*   Encode and decode protobuf messages
    
*   Stream data to the server
    
*   Receive streaming responses
    
*   Preserve gRPC framing and semantics — `Content-Type: application/grpc`, unchanged end to end
    

### Why `protobufjs` instead of `protoc`?

*   **No Native Binary Toolchains:** Eliminates the need to install or maintain OS-dependent `protoc` compiler binaries or custom plug-ins in your local workspace or CI/CD pipelines. The whole pipeline stays inside npm
    
*   **No per-language plugins** — nothing like `protoc-gen-*` to install, match versions with, or
    reproduce on a build machine

*   **Code generation is one npm script** — `gen-web-client-protos` emits the client from your
    `.proto` files, like any other build step

The result is a **pure-JS gRPC client**, suitable for browsers — and, unchanged, for Node 20+.

Reference implementation:  
**[`@emmveqz/grpc-web-native`](https://www.npmjs.com/package/@emmveqz/grpc-web-native)**

* * *

## Authentication: JWT

Authentication is handled via **JWT tokens** passed with each gRPC request.

Key points:

*   The browser only knows about the token
    
*   Token issuance and signing are **out of scope** here
    
*   Validation happens at the infrastructure boundary
    

This keeps:

*   Application services stateless
    
*   Authentication centralized
    
*   gRPC services free from auth concerns
    

* * *

## Public Entry Point: Envoy

Envoy acts as the **single public-facing component**.

Responsibilities:

*   TLS termination
    
*   HTTP/2 negotiation (ALPN)
    
*   CORS handling
    
*   JWT validation
    
*   Routing and load balancing
    

Envoy runs as:

*   A single ECS service
    
*   One or more tasks (often one is enough)
    
*   Stateless and horizontally scalable
    

> Envoy is the **protocol boundary** between the public internet and private gRPC services.

The gRPC applications behind Envoy do not:

*   Handle TLS
    
*   Handle CORS
    
*   Know anything about browsers
    
*   Face the internet directly
    

> If you can't use a Proxy layer, an alternative to remove Envoy can be used with [`@emmveqz/grpc-node-web`](https://www.npmjs.com/package/@emmveqz/grpc-node-web) which can handle CORS natively. In that case, the gRPC service will also need to handle Auth and TLS, and be exposed to the internet.

* * *

## AWS Network Load Balancer (NLB)

The Network Load Balancer sits in front of Envoy.

Why NLB:

*   Layer 4 (TCP)
    
*   Preserves HTTP/2 streams
    
*   No request buffering
    
*   No protocol interference
    

The NLB:

*   Does **not** terminate TLS
    
*   Does **not** inspect HTTP
    
*   Simply forwards TCP connections to Envoy
    

This is critical for real gRPC streaming.  
(Do not use AWS Application Load Balancer, nor its built-in gRPC/HTTP protocols. Use TCP)

* * *

## Private Application Layer: gRPC Services

The application services are implemented using:

*   **[`@grpc/grpc-js`](https://www.npmjs.com/package/@grpc/grpc-js)** (proven in practice, but it should work in other languages)
    
*   HTTP/2 cleartext (`h2c`)
    
*   ECS tasks in private subnets
    

Characteristics:

*   Services only accept traffic from Envoy
    
*   No public exposure
    
*   No environment-specific logic
    

From the application’s perspective:

*   Local development
    
*   Staging
    
*   Production
    

…are essentially identical.

Only Envoy configuration changes between environments.

* * *

## Why gRPC for the Application Layer

gRPC is not just about typed APIs.

This architecture treats gRPC as a **universal internal protocol**.

### Benefits

*   Strong service contracts
    
*   Bidirectional streaming
    
*   High-performance binary transport
    
*   One protocol for all internal systems
    

### Example gRPC Domains

*   **Web gRPC** — exposed through Envoy to browsers
    
*   **Mail gRPC** — internal mail server coordination
    
*   **Telephony gRPC** — PBX / VoIP orchestration
    
*   **Worker gRPC** — background processing
    
*   **Admin gRPC** — internal tooling
    

All these services:

*   Speak the same protocol
    
*   Share tooling
    
*   Can communicate directly with each other
    

The browser-facing gRPC is just **one consumer** of the same ecosystem.

* * *

## Architectural Overview

```
Browser (fetch, streams)
        |
        | HTTPS + HTTP/2
        v
AWS NLB (TCP)
        |
        v
Envoy (TLS, CORS, JWT)
        |
        | h2c
        v
gRPC Services (running in AWS ECS)
```

(See [/docs/diagrams/](/docs/diagrams/grpc-browser-native-architecture.png) for visual versions.)

* * *

## What This Is — and Isn’t

**This is:**

*   Native gRPC over HTTP/2
    
*   Browser-compatible
    
*   Streaming-capable
    
*   Infrastructure-clean
    

**This is not:**

*   REST
    
*   `grpc-web`
    
*   WebSockets
    
*   API Gateway–based
    

* * *

See [/docs/architecture.md](/docs/architecture.md) for more details.
