---
title: From mainframe sockets to Spring Boot REST APIs
description: A practical guide to sockets, TCP, REST, and endpoints—and the integration migration I worked on, from mainframe connections to a Spring Boot service on AWS ECS.
date: 2026-10-25
tags: Java, Networking, Mainframe, Spring Boot, AWS
draft: false
---
## The change I worked on

At Deloitte, I worked on modernizing legacy COBOL applications for Java-based execution on AWS using the Innovate runtime. My part included building a Spring Boot integration microservice from requirements through deployment on Amazon ECS, replacing legacy socket-based connections with REST endpoints, and creating its GitHub Actions CI/CD pipeline.

But what does “replacing sockets with REST” actually mean? To understand the migration, we need to separate the network connection from the language applications speak over it.

**The central idea: we changed the application interface, not the existence of network connections.** REST-style HTTP services still use networking underneath.

The account of my work above describes the project. The account-lookup examples, payloads, port numbers, code, and topology below are simplified teaching examples—not production contracts or a disclosure of client systems. Mainframe installations vary; the CICS example explains one common pattern rather than identifying the listener used in our environment.

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

**The goal of my migration was to move the integration boundary to a Spring Boot REST service on AWS while retaining the required business behavior.**

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

## 7. How I approached the migration

My scope was broader than adding a controller. I built the integration microservice, repointed interfaces, deployed it on ECS, and created its delivery pipeline. The following breakdown explains the engineering concerns in that kind of migration; it is not a claim that every illustrative mechanism below was used unchanged in our implementation.

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

### Preserve behavior while repointing dependencies

My modernization work included repointing mainframe Db2 interfaces to Db2 LUW pureScale, migrating MQ interfaces to Amazon SQS, and moving NDS/CICS interface handling toward the integration microservice. These are related changes, but a database migration, queue migration, and HTTP boundary migration are not interchangeable tasks.

For example, moving from MQ to SQS requires checking the actual queue type, delivery guarantees, duplicate handling, and transaction expectations. Replacing a queue name alone does not preserve those semantics. Similarly, database repointing requires checking compatibility and transactional behavior beyond a connection URL.

### Deploy the service and automate delivery

The Spring Boot service ran on AWS ECS, and I created its GitHub Actions build/deployment pipeline. A typical HTTP deployment can place an Application Load Balancer in front of ECS tasks for routing and health checks. That load balancer is an illustrative option here, not a verified detail of my project's topology.

```flow
Caller :: Sends an HTTPS request
HTTP ingress :: Optional load balancer / routing layer
ECS service :: Runs Spring Boot container tasks
Application adapters :: Connect to migrated dependencies
```

ECS manages container tasks; it does not implement REST or automatically make operations stateless. Application design and service configuration still determine behavior.

[Amazon ECS load-balancing documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-load-balancing.html) explains the available ingress options.

## 8. A small Spring Boot example

This is a deliberately minimal controller illustrating the HTTP boundary, not production code or the actual account service from my work. The example assumes a lookup service and response type exist.

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

## What I took away

The most useful way to understand this migration is as a change of contract and responsibility. A socket gives programs a channel. A custom protocol defines how they use its bytes. HTTP standardizes the message envelope. A REST-style API organizes the operations and representations consumers see.

Spring Boot helped expose that interface, ECS provided the container runtime, and GitHub Actions automated delivery. The engineering work was in connecting those pieces while preserving the behavior callers relied on.

**Modernization is complete when the new interface is understandable, operable, and correct—not merely when it returns HTTP 200.**
