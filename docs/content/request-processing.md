+++
title = "Request Processing Pipeline"
description = "Understanding how Taxy processes requests before they reach the origin server"
weight = 1
+++

# Request Processing Pipeline

This document explains the complete request processing pipeline in Taxy, detailing all the stages a request goes through before it reaches the HTTP origin server behind the reverse proxy.

## Overview

When a client makes a request to Taxy, the request goes through several processing stages before being forwarded to the backend origin server. These stages handle TLS termination, routing, header manipulation, security checks, and connection pooling.

## Request Processing Stages

### 1. Connection Acceptance

**Location**: `taxy/src/proxy/http/mod.rs` - `start_proxy()` and `start()`

When a TCP connection is established:
- The connection is wrapped in a buffered stream (`BufStream<TcpStream>`)
- Local and remote socket addresses are captured for logging and header processing
- The connection is spawned into an asynchronous task for handling

### 2. TLS Termination (if configured)

**Location**: `taxy/src/proxy/http/mod.rs` - `start()` function

If the port is configured for HTTPS/TLS:
- The TLS acceptor performs the TLS handshake
- Server Name Indication (SNI) is extracted from the TLS ClientHello
- The appropriate certificate is selected based on SNI
- The connection is decrypted
- ALPN (Application-Layer Protocol Negotiation) determines HTTP/1.1 or HTTP/2

If HTTP/3 (QUIC) is used:
- QUIC connections are accepted via `start_quic_proxy()`
- H3 protocol negotiation occurs
- Each request stream is processed individually

### 3. Protocol Detection and HTTP Parsing

**Location**: `taxy/src/proxy/http/mod.rs` - `start()` function, lines 424-535

The HTTP protocol is auto-detected:
- Uses hyper's `auto::Builder` to automatically detect HTTP/1.1 or HTTP/2
- For HTTPS, protocol is determined via ALPN during TLS handshake
- For plain HTTP, protocol is detected from the request itself
- Upgrades (like WebSocket) are supported via `serve_connection_with_upgrades()`

### 4. Security Checks

**Location**: `taxy/src/proxy/http/mod.rs` - service function, lines 435-443

Before routing, Taxy performs security checks:

**Domain Fronting Detection**:
- Compares the SNI (from TLS) with the Host header
- If they don't match (domain fronting attack), the request is rejected
- Returns error: `ProxyError::DomainFrontingDetected`

### 5. Routing and Route Selection

**Location**: `taxy/src/proxy/http/route.rs` and `taxy/src/proxy/http/filter.rs`

The router (`Router::get_route()`) determines which backend should handle the request:

**Virtual Host Matching** (`filter.rs`):
- Extracts the host from the Host header or SNI
- Tests against configured virtual hosts (e.g., `example.com`, `*.example.com`)
- Returns `None` if no virtual host matches (and vhosts are configured)

**Path Matching** (`filter.rs`):
- Splits the request path into segments
- Compares with configured route paths
- Matches the longest common prefix
- Remaining path segments are passed to the backend

**Route Selection** (`route.rs`):
- Iterates through all configured routes
- Returns the first route that matches both vhost and path criteria
- If no route matches, returns `ProxyError::NoRouteFound`

### 6. HTTPS Upgrade Handling

**Location**: `taxy/src/proxy/http/mod.rs` - service function, lines 456-481

If configured with `upgrade_insecure`:
- Detects HTTP requests (from `forwarded_proto`)
- Automatically redirects to HTTPS with a 301 response
- Reconstructs the URL with the HTTPS scheme and configured HTTPS port
- Returns redirect response immediately without forwarding to backend

### 7. URL Rewriting

**Location**: `taxy/src/proxy/http/mod.rs` - service function, lines 486-495

The request URI is rewritten for the backend:
- Takes the first server from the matched route's server list
- Constructs the target URL using the server's base URL
- Appends the remaining path segments (after matching prefix is removed)
- Preserves the original query string
- Updates the request URI to point to the backend server

Example:
- Client requests: `http://example.com/api/users?limit=10`
- Route path: `/api`
- Backend server: `http://backend:8080`
- Rewritten URI: `http://backend:8080/users?limit=10`

### 8. Request Header Manipulation

**Location**: `taxy/src/proxy/http/rewriter.rs` - `RequestRewriter`

Headers are manipulated in two phases:

#### 8.1 Pre-Processing (`pre_process()` - lines 71-131)

**Forwarding Headers**:
- Handles `X-Forwarded-For` header:
  - If `trust_upstream_headers` is false, removes existing X-Forwarded-For
  - If true, parses and preserves existing forwarded IPs
  - Appends the client's IP address to the chain
  
- Handles `Forwarded` header (RFC 7239):
  - Constructs standardized forwarded header with:
    - `for=<client-ip>` - Client's IP address (IPv6 wrapped in brackets)
    - `host=<original-host>` - Original Host header value
    - `proto=<http|https>` - Original protocol (http or https)
  - Preserves upstream forwarded directives if trusted

- Sets additional headers:
  - `X-Forwarded-Proto`: The protocol used by the client (http/https)
  - `X-Forwarded-Host`: The original Host header value
  - `X-Real-IP`: Removed if not trusting upstream headers

**Security**:
- By default (`trust_upstream_headers=false`), removes all forwarding headers from the client
- This prevents clients from spoofing their IP address or other metadata
- Only adds headers based on the actual connection information

#### 8.2 Post-Processing (`post_process()` - lines 133-137)

**Via Header**:
- Adds `Via` header if configured
- Indicates that the request passed through Taxy proxy
- Useful for debugging and tracing request paths

**Host Header Update**:
- Updates the Host header to match the backend server's authority
- Ensures the backend receives the correct Host header

### 9. Connection Pooling

**Location**: `taxy/src/proxy/http/pool.rs` - `ConnectionPool`

Before forwarding to the backend:
- The connection pool manages persistent connections to backends
- Reuses existing connections when possible (HTTP/1.1 keep-alive, HTTP/2 multiplexing)
- Creates new connections if needed
- Handles connection lifecycle and timeouts
- Improves performance by avoiding repeated TCP handshakes

### 10. Request Forwarding

**Location**: `taxy/src/proxy/http/mod.rs` - service function, lines 521-535

Finally, the request is forwarded:
- The modified request is sent to the backend via the connection pool
- Request body is streamed to the backend (using `BoxBody`)
- The proxy waits for the response from the backend
- Errors during forwarding are caught and converted to error responses

### 11. Response Processing

**Location**: `taxy/src/proxy/http/rewriter.rs` - `ResponseRewriter`

After receiving the response from the backend:

**Alt-Svc Header** (lines 197-210):
- Removes any existing `Alt-Svc` header from backend
- Adds a new `Alt-Svc` header advertising available protocols:
  - HTTP/2 on the configured HTTPS port: `h2=":443"`
  - HTTP/3 on the configured QUIC port: `h3=":443"`
- This tells clients they can upgrade to better protocols

**Error Handling** (lines 213-224):
- If backend returns an error or is unreachable
- Generates a user-friendly error page using a template
- Maps the error to an appropriate HTTP status code
- Returns the error response to the client

## Summary

In summary, before a request reaches the HTTP origin server, Taxy processes it through these stages:

1. **Connection acceptance** - TCP socket handling
2. **TLS termination** - Decrypt HTTPS traffic, extract SNI
3. **Protocol detection** - HTTP/1.1, HTTP/2, or HTTP/3
4. **Security checks** - Domain fronting detection
5. **Routing** - Virtual host and path matching
6. **HTTPS upgrade** - Redirect HTTP to HTTPS if configured
7. **URL rewriting** - Construct backend URL with remaining path
8. **Header manipulation** - Add forwarding headers, security cleanup
9. **Connection pooling** - Reuse backend connections
10. **Request forwarding** - Send to backend origin server
11. **Response processing** - Add Alt-Svc, handle errors

This comprehensive processing ensures secure, efficient, and properly routed requests to the backend servers while providing features like load balancing, TLS termination, and protocol upgrades.

## Code References

Key source files:
- `taxy/src/proxy/http/mod.rs` - Main HTTP proxy logic
- `taxy/src/proxy/http/filter.rs` - Request filtering (vhost, path matching)
- `taxy/src/proxy/http/route.rs` - Route selection and management
- `taxy/src/proxy/http/rewriter.rs` - Header rewriting (request and response)
- `taxy/src/proxy/http/pool.rs` - Connection pool management
- `taxy/src/proxy/tls.rs` - TLS termination
