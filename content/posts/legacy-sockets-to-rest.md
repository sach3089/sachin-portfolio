---
title: From mainframe sockets to Spring Boot REST APIs
description: A practical guide to sockets, TCP, REST, and endpoints—and a hypothetical migration from mainframe connections to a Spring Boot service on AWS ECS.
date: 2026-10-25
tags: Java, Networking, Mainframe, Spring Boot, AWS
draft: false
---
## A migration scenario

Imagine a business application that exposes operations through a custom TCP socket protocol. A modernization project replaces that interface with a Spring Boot HTTP API deployed on Amazon ECS.

What does “replacing sockets with REST” actually mean? To understand the migration, we need to separate the network connection from the language applications speak over it.

**The central idea: change the application interface, not the existence of network connections.** REST-style HTTP services still use networking underneath.

This guide describes a hypothetical architecture. The examples, payloads, code, and deployment choices are illustrative and do not describe an employer's or client's implementation. Mainframe installations vary; the CICS example below explains a publicly documented pattern.

## End-to-end architecture: from a legacy contract to an HTTP API

The two paths below show the same conceptual business operation before and after migration. They are alternative designs, not two hops of the same request. The actual caller also needs an HTTP implementation or a compatibility adapter; an existing socket client cannot simply change its destination and start speaking HTTP.

```flow
BEFORE · External caller :: Formats a custom record using the legacy contract
TCP connection :: Sends bytes to a listener on an agreed host and port
Mainframe listener / application :: Frames, decodes, and routes the request
Business operation + data access :: Executes the operation within its transaction boundary
Response to the caller :: Encodes a response record and sends it through the connection
```

```flow
AFTER · HTTP-capable caller :: Sends an authenticated HTTPS request with a defined API contract
Application Load Balancer :: Example HTTPS ingress: terminates TLS and routes to healthy targets
ECS service · private application tasks :: Runs Spring Boot containers; target security groups restrict ingress
Spring Boot API boundary :: Authenticates / authorizes the caller, validates input, and maps errors
Application service + adapters :: Executes business logic and translates downstream contracts
Database or downstream service :: Uses scoped credentials and an explicit consistency model
HTTP response to the caller :: Returns a representation and appropriate status through the ingress
```

The ingress and private-task arrangement are example deployment choices. Configure target-side encryption as required; TLS termination at the load balancer does not automatically encrypt the next hop. A valid authentication token also does not replace authorization for a particular resource.

Two supporting paths make this service deployable and operable:

```flow
DELIVERY · Source change :: A GitHub commit triggers a build workflow
Build + test :: Compile and verify the application, then build a container image
Amazon ECR :: Store the versioned container image
ECS deployment :: Update the task definition / service and check rollout health
```

```flow
OBSERVABILITY · Spring Boot service :: Emit telemetry with request / trace context and redact sensitive fields
Logs :: Send configured container stdout / stderr to CloudWatch Logs
Traces + metrics :: Route OpenTelemetry signals through an optional collector to compatible backends
Investigation + feedback :: Correlate requests, diagnose failures, and use findings for the next change
```

ECR, the collector, and telemetry routing are illustrative components, not a record of any organization's infrastructure. Instrumentation and export configuration are required; deploying to ECS does not automatically produce complete distributed traces.

## 1. What is a socket?

A socket is a software endpoint an application uses to communicate over a network. It is not a physical cable, and it is not itself a message format. Your program asks the operating system to create one, then uses it to send and receive data.

An **IP address** identifies a network interface. A **port** helps the operating system deliver traffic to the appropriate service. Think of the IP address as a building address and the port as a reception desk for a particular service. A socket is the program's communication endpoint behind that desk.

For an established TCP connection, the source IP and port, destination IP and port, and transport protocol identify the conversation. Many clients can therefore connect to one listening server port; each accepted connection has a distinct peer.

**This article focuses on TCP sockets.** UDP sockets also exist, but they provide different communication semantics. Socket is the API abstraction; TCP is the transport protocol used in this example.

[Background: Oracle's socket introduction](https://docs.oracle.com/javase/tutorial/networking/sockets/definition.html).

```flow
Client application :: Local IP + temporary port
TCP connection :: Ordered byte stream
Server application :: Server IP + listening port
```

## 2. How does a TCP socket connection work?

A useful lifecycle is: create the endpoint, establish the connection, exchange data, and close it when finished.

On the server, the typical socket API sequence is **socket → bind → listen → accept**. Binding chooses the local address and port; listening makes the endpoint available for incoming connections. Accepting produces a connected socket for a particular client. The listening socket remains available to accept other clients.

The client creates its socket and calls **connect** with the destination address and port. TCP establishes the connection using its handshake: SYN, SYN-ACK, ACK. Application data then travels over the connection in both directions.

```sequence
Client → Server :: SYN — request a connection
Server → Client :: SYN-ACK — acknowledge and respond
Client → Server :: ACK — connection established
Client → Server :: Application request bytes
Server → Client :: Application response bytes
```

TCP supplies an ordered byte stream, with mechanisms for retransmission and flow control. It does **not** supply business-message boundaries. One application write can be read in pieces, and several writes can appear together in a read. TCP segments are not application records.

For a custom protocol, both sides must agree how to find a complete message. Common choices include a fixed-size record, a length prefix, or a delimiter. They must also agree on field encoding and error representation. Closing a connection can signal the end of a message in some protocols, but makes connection reuse harder.

[Further reading: Oracle's server-side socket tutorial](https://docs.oracle.com/javase/tutorial/networking/sockets/clientServer.html) and [the TCP specification, RFC 9293](https://datatracker.ietf.org/doc/html/rfc9293).

## 3. What might a mainframe socket interface look like?

Mainframes can participate in ordinary TCP/IP networks. In a typical custom integration, a caller connects to a known host and port, sends a record that follows an agreed layout, and waits for a response from a business application.

One documented CICS pattern uses an IBM-supplied listener to accept connections and hand work to an application transaction. CICS is a transaction-processing environment; COBOL is a programming language. Neither term means “socket,” and a mainframe application need not use CICS to communicate over TCP/IP.

```flow
External caller :: Writes an agreed request record
TCP listener :: Accepts a network connection
Application transaction :: Parses the record and runs business logic
Response record :: Returns through the connection
```

For example, imagine a fixed-format request containing an operation code and account identifier:

```text
LOOKUP00001234
```

The application needs to know which characters identify the operation, how many identify the account, and when the request is complete. An actual protocol may include headers, lengths, binary fields, or other records; this string is only a readable illustration.

Character conversion also matters. Some mainframe integrations involve EBCDIC, while a web interface commonly uses UTF-8. Some fields can contain packed decimal or binary values. A safe migration discovers the actual layout and encoding rather than treating every payload as plain text or assuming all mainframe traffic uses the same encoding.

[IBM's CICS TCP/IP introduction](https://www.ibm.com/docs/en/zos/3.1.0?topic=guide-introduction-cics-tcpip) and [CICS application transaction/listener documentation](https://www.ibm.com/docs/en/zos/3.1.0?topic=api-cics-application-transaction-listener) describe these platform concepts.

## 4. Why change a working interface?

A socket protocol can be efficient and entirely appropriate. Its age does not automatically make it a bad design. The difficulty arises when consumers must understand transport handling, custom record layouts, and business semantics together.

An HTTP API gives consumers a familiar request/response envelope and tooling. It can make contracts, validation, access control, and diagnostics easier to standardize. The tradeoff is additional protocol overhead and a different failure model; changing the interface does not automatically make the business operation faster or safer.

**The goal is to move the integration boundary to a Spring Boot REST service on AWS while retaining the required business behavior.**

## 5. What are HTTP, REST, an API, and an endpoint?

These terms describe different things:

- **HTTP** is an application protocol with request methods, headers, status codes, and message bodies. HTTPS adds protection using TLS.
- **API** means application programming interface: the contract another program uses to interact with your software. APIs are not limited to the web.
- **REST** means Representational State Transfer. It is an architectural style, not a wire protocol, a Java library, or a synonym for JSON. Resource-oriented URLs, a uniform interface, stateless requests, appropriate caching behavior, and layers are important REST ideas. Full REST also includes hypermedia-driven interactions; many services called “REST APIs” implement a practical subset.
- **Endpoint**, in everyday HTTP API usage, is a callable operation identified by an HTTP method and an address. This use differs from “socket endpoint,” which refers to a networking abstraction.

For example:

```http
GET https://api.example.com/api/v1/accounts/00001234
```

The host is api.example.com. The path identifies an account resource. GET asks to retrieve its representation. A different method on the same path can represent a different operation, so the method is part of the API contract.

A response might be:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "accountId": "00001234",
  "status": "ACTIVE"
}
```

JSON is one representation format, not a REST requirement. Keeping the identifier as a string preserves leading zeroes. A real service would also define authentication, authorization, missing-resource behavior, and which fields a caller is allowed to see.

[MDN's HTTP overview](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview) explains the request/response model; [Fielding's REST chapter](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm) defines the architectural style.

## 6. Does REST eliminate sockets?

**No.** The phrase “socket to REST migration” is shorthand for moving from a custom application protocol over sockets to an HTTP interface designed around REST concepts.

```flow
Resource operation :: GET /api/v1/accounts/00001234
HTTP interface :: Methods, headers, status, representation
Secure transport :: TLS over TCP for a typical HTTP/1.1 or HTTP/2 exchange
Network :: IP routing between systems
```

HTTP/3 uses QUIC over UDP instead, so “HTTP always uses TCP” would be inaccurate. The important distinction is that a web framework handles much of the protocol machinery, while your application focuses on its contract and business logic.

Also, **stateless requests do not mean opening a new network connection for every request**. HTTP connections may be reused. Statelessness means the server does not require conversational client-session context from a previous request to understand the current one; it does not mean the service cannot use a database.

## 7. How to approach the migration

Treat the work as a contract migration, not just a new controller. The following steps describe engineering concerns for the hypothetical system above.

### Understand the old contract before designing the new one

Start with the callers and their business operations. Identify record layouts, encoding, timeouts, connection reuse, errors, and transaction boundaries. Ask what happens if a connection drops after an operation commits but before the caller receives the response.

Without that understanding, a new HTTP endpoint can reproduce the happy path while silently changing failure behavior.

### Separate the HTTP boundary from the business integration

A useful structure is a controller for request mapping, a service layer for the operation, and adapters for downstream interfaces. The controller should not become a container for database conversion, legacy protocol parsing, and business rules.

```flow
HTTP controller :: Accept and validate the request
Application service :: Apply business rules and coordinate work
Integration adapters :: Translate downstream contracts
Downstream systems :: Execute the required operation
```

### Preserve behavior while changing dependencies

A socket interface migration may happen alongside a database or messaging migration. Keep those contracts separate: changing an HTTP boundary does not by itself require changing the database or queue technology.

If dependencies also change, compare delivery guarantees, duplicate handling, data types, and transaction expectations. Replacing a queue name or database connection URL alone does not guarantee equivalent behavior.

### Deploy the service and automate delivery

In this example, a Spring Boot container runs as part of an ECS service. A GitHub Actions workflow builds and tests the code, publishes the image to ECR, and deploys the new task revision. An Application Load Balancer is one possible HTTP ingress, with health checks configured for the application's readiness behavior.

```flow
Caller :: Sends an HTTPS request
HTTP ingress :: Optional load balancer / routing layer
ECS service :: Runs Spring Boot container tasks
Application adapters :: Connect to migrated dependencies
```

ECS manages container tasks; it does not implement REST or automatically make operations stateless. Application design and service configuration still determine behavior.

[Amazon ECS load-balancing documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-load-balancing.html) explains the available ingress options.

## 8. A small Spring Boot example

This is a deliberately minimal controller illustrating the HTTP boundary, not a production-ready account service. The example assumes a lookup service and response type exist.

```java
@RestController
@RequestMapping("/api/v1/accounts")
class AccountController {
    private final AccountLookupService accounts;

    AccountController(AccountLookupService accounts) {
        this.accounts = accounts;
    }

    @GetMapping("/{accountId}")
    AccountResponse getAccount(@PathVariable String accountId) {
        return accounts.lookup(accountId);
    }
}
```

The class maps requests under the accounts path. The method handles GET requests for one identifier. Spring maps the returned object to a response representation through its HTTP message converters. You still need explicit handling for invalid identifiers, authorization failures, absent accounts, and downstream failures.

A method name such as lookup is not evidence that an operation is read-only. Use GET only when the operation is safe; do not hide a state-changing legacy transaction behind it.

[Spring's REST service guide](https://spring.io/guides/gs/rest-service/) shows the framework concepts in a complete introductory project.

## 9. The details that decide whether a migration succeeds

- **Contract equivalence:** compare old and new outputs using representative cases, including malformed input and downstream errors. Preserve identifier formatting and numeric precision.
- **Timeout budgets:** set connection and request timeouts with downstream latency in mind. An HTTP timeout does not prove the downstream work stopped.
- **Retries and duplicates:** retry only when the operation's semantics allow it. A payment-like command may need an idempotency mechanism and reconciliation rather than a blind retry.
- **Transaction boundaries:** an HTTP request, database commit, and message publication do not automatically form one atomic transaction. Design the consistency behavior deliberately.
- **Access control:** TLS protects transport; it does not decide whether a caller is allowed to read a particular record. Authentication and authorization are separate responsibilities.
- **Observability:** request identifiers, traces, logs, and metrics help connect an incoming request to downstream work. Avoid recording confidential payloads or credentials in logs.
- **Rollout:** validate representative traffic, compare behavior, and have a rollback plan. Deployment success is not proof of business correctness.

## The main takeaway

The most useful way to understand this migration is as a change of contract and responsibility. A socket gives programs a channel. A custom protocol defines how they use its bytes. HTTP standardizes the message envelope. A REST-style API organizes the operations and representations consumers see.

In this architecture, Spring Boot exposes the interface, ECS runs the containers, and GitHub Actions automates delivery. The engineering work is connecting those pieces while preserving the behavior callers rely on.

**Modernization is complete when the new interface is understandable, operable, and correct—not merely when it returns HTTP 200.**
