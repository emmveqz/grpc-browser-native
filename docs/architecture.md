# Architecture

This document explains the architecture behind exposing **gRPC services directly to browsers**, without using `grpc-web`, protocol translation, or code generation plugins.

The focus is on **separation of concerns**, **protocol correctness**, and **long-term ecosystem consistency**.

* * *

## Architectural Goals

*  Use **real gRPC** end-to-end (HTTP/2)
    
*  Avoid protocol translation ([`grpc-web`](https://www.npmjs.com/package/grpc-web))
    
*  Keep browser code **browser-native**
    
*  Centralize cross-cutting concerns (TLS, auth, CORS)
    
*  Keep application services environment-agnostic
    
*  Allow the same protocol to be used across _all_ internal systems
    

* * *

## High-Level Components

The system is composed of five primary layers:

1.  [Browser gRPC client](#1-browser-grpc-client)
    
2.  [Network Load Balancer (NLB)](#2-network-load-balancer-aws-nlb)
    
3.  [Envoy proxy](#3-envoy-public-edge-gateway)
    
4.  [Service discovery](#4-service-discovery)
    
5.  [gRPC application services](#5-grpc-application-services)
    

Each layer has a single responsibility.

* * *

## 1\. Browser gRPC Client

The browser communicates using:

*   `fetch()` for HTTP/2 transport
    
*   `ReadableStream` streaming in both directions: a stream *as* the request body, and the
    response read as it arrives
    
*   `Uint8Array` raw gRPC binary framing
    
*   [`protobufjs`](https://www.npmjs.com/package/protobufjs) protobuf in pure JavaScript, so
    no `protoc` and no native toolchain
    

The client:

*   Encode and decode protobuf messages
    
*   Streams data to the server
    
*   Receives streaming responses
    
*   Preserve gRPC framing and semantics — `Content-Type: application/grpc`, unchanged end to end
    

### Why no `grpc-web`

`grpc-web` introduces:

*   A custom wire protocol
    
*   A translation proxy
    
*   OS-dependent `protoc` compiler binaries or custom plug-ins
    

This architecture instead treats gRPC as:

> **An HTTP/2 application protocol**, which browsers already support.

* * *

## 2\. Network Load Balancer (AWS NLB)

The Network Load Balancer is the first AWS-managed component in the request path.

### Responsibilities

*   Accept public TCP connections
    
*   Forward raw TCP streams to Envoy
    

### Non-responsibilities

*   No TLS termination
    
*   No HTTP inspection
    
*   No buffering
    
*   No routing
    

NLB is used because it is:

*   Layer 4 (TCP)
    
*   HTTP/2-safe
    
*   Streaming-safe
    

This ensures HTTP/2 frames are preserved intact.

* * *

## 3\. Envoy: Public Edge Gateway

Envoy acts as the **public door** into the system.

It is the only component exposed to the internet.

### Responsibilities

Envoy handles:

*   TLS termination
    
*   ALPN negotiation (HTTP/2)
    
*   CORS validation
    
*   JWT authentication
    
*   Routing to backend services
    
*   Load balancing across service instances
    

Envoy does **not**:

*   Implement business logic
    
*   Transform gRPC payloads (no custom grpc filters needed)
    
*   Translate protocols
    

### TLS & ALPN

Envoy terminates TLS and advertises ALPN's:

*   `h2`
    
*   `http/1.1` (Optional. Do not use unless sharing with other services. gRPC must use `h2`)
    

Without ALPN configuration, browsers cannot negotiate HTTP/2 and connections would fail silently.

> If you can't use a Proxy layer, an alternative to remove Envoy can be used with [`@emmveqz/grpc-node-web`](https://www.npmjs.com/package/@emmveqz/grpc-node-web) which can handle CORS natively. In that case, the gRPC service will also need to handle Auth and TLS, and be exposed to the internet.

* * *

## 4\. Service Discovery

Backend gRPC services are deployed as ECS tasks and registered via AWS Cloud Map.

Envoy resolves services using DNS:

`grpc-service.grpc-namespace`

Service discovery enables:

*   Dynamic scaling
    
*   Zero configuration changes when tasks are added/removed
    
*   Automatic load balancing
    

Envoy periodically re-resolves DNS and updates its backend pool.

* * *

## 5\. gRPC Application Services

The application layer consists of one or more **gRPC services**, implemented using:

*   [`@grpc/grpc-js`](https://www.npmjs.com/package/@grpc/grpc-js) (proven in practice, but it should work in other languages)
    
*   HTTP/2 cleartext (`h2c`)
    
*   ECS tasks in private subnets
    

### Key Properties

*   No TLS
    
*   No CORS
    
*   No authentication logic
    
*   No environment-specific configuration
    

The services trust Envoy completely.

This allows:

*   Local development without special config
    
*   Production deployment without code changes
    
*   Consistent behavior across environments
    

* * *

## Internal vs External gRPC

This architecture treats gRPC as a **universal internal protocol**.

### Example Domains

*   **Web gRPC**  
    Exposed via Envoy to browsers
    
*   **Mail gRPC**  
    Internal mail server coordination
    
*   **Telephony gRPC**  
    PBX / VoIP orchestration
    
*   **Worker gRPC**  
    Background jobs and schedulers
    
*   **Admin gRPC**  
    Internal tooling and control planes
    

All services:

*   Share the same protocol
    
*   Use the same tooling
    
*   Can communicate directly with each other
    

The browser is just another client.

* * *

## Request Flow (Step-by-Step)

1.  Browser initiates `fetch()` to gRPC endpoint
    
2.  TLS handshake occurs at Envoy
    
3.  HTTP/2 is negotiated via ALPN
    
4.  Browser sends OPTIONS (CORS preflight)
    
5.  Envoy validates CORS and JWT
    
6.  Envoy routes request to gRPC service
    
7.  gRPC service processes request (unary or streaming)
    
8.  Response stream flows back through Envoy
    
9.  Browser receives streamed data
    

* * *

## Separation of Concerns

| Layer | Responsibility |
| --- | --- |
| Browser | Encoding, decoding, UI logic |
| NLB | Connection distribution |
| Envoy | Security, routing, transport |
| Cloud Map | Service discovery |
| gRPC Services | Business logic |

This separation keeps each layer simple and replaceable.

* * *

## Why This Architecture Scales

*   Stateless Envoy
    
*   Stateless gRPC services
    
*   DNS-based discovery
    
*   No shared session state
    
*   Horizontal scaling everywhere
    

Scaling Envoy and gRPC services is independent.

* * *

## Trade-offs

### Pros

*   Real gRPC in browsers
    
*   Streaming support
    
*   Typed contracts
    
*   Unified protocol across systems
    

### Cons

*   More initial setup than REST
    
*   Requires HTTP/2 literacy
    
*   Proxy literacy (Envoy configuration)
    

These trade-offs are intentional.

* * *

## Summary

This architecture demonstrates that:

*   Browsers can speak real gRPC
    
*   gRPC does not require [`grpc-web`](https://www.npmjs.com/package/grpc-web)
    
*   Infrastructure concerns belong at the edge
    
*   Application services should remain simple
    

The result is a **cohesive, scalable, protocol-driven system** that treats gRPC as a first-class citizen everywhere.

* * *
