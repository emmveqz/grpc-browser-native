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
    
*   **`Uint8Array`** — raw binary framing
    
*   **`ReadableStream`** — upstream streaming
    
*   **`protobufjs`** — runtime protobuf parsing (no codegen, no `protoc`)
    

This allows the browser to:

*   Encode/decode protobuf messages dynamically
    
*   Stream data to the server
    
*   Receive streaming responses
    
*   Preserve gRPC framing and semantics
    

### Why `protobufjs` instead of `protoc`?

*   No build-time code generation
    
*   No language-specific bindings
    
*   Protobuf becomes a **data schema**, not a compiled artifact
    

The result is a **pure-JS gRPC client**, suitable for browsers.

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

## Private Application Layer: Node.js gRPC

The application services are implemented using:

*   **`@grpc/grpc-js`**
    
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
Node.js gRPC Services
```

(See [`/docs/diagrams/`](/docs/diagrams/grpc-browser-native-architecture.png) for visual versions.)

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
