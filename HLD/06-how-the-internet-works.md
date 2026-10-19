# How the Internet Works — DNS, TCP, TLS, and HTTP

## 1. The problem

When the frontend calls a backend such as `https://api.foodapp.com/restaurants/123`, the client must find the server, establish communication, protect the connection, describe the requested operation, and receive a response.

For a new HTTPS connection over TCP, the high-level flow is:

~~~text
URL / API call
     |
     v
DNS resolution -> Find an IP address
     |
     v
TCP handshake -> Establish reliable transport
     |
     v
TLS handshake -> Establish secure communication
     |
     v
HTTP request -> Backend processes the operation
     |
     v
HTTP response <- Status, headers, and body
~~~

DNS may be cached and existing connections may be reused, so every setup step does not necessarily repeat for each request.

## 2. DNS resolution — find the server's address

**DNS (Domain Name System)** lets clients resolve human-readable hostnames to records such as IP addresses.

An **IP address** identifies a network destination. In the URL `https://api.foodapp.com/restaurants/123`:
- Scheme: `https`
- Hostname: `api.foodapp.com`
- Path: `/restaurants/123`

DNS resolves the hostname, not the path or API operation.

A simplified lookup:
1. The client checks available DNS caches.
2. If there is no usable cached answer, it asks a configured DNS resolver.
3. The resolver may query the DNS hierarchy: root servers, top-level-domain (TLD) servers, and the domain's authoritative DNS servers.
4. The resolver returns the relevant DNS answer to the client.

A **DNS resolver** obtains DNS answers on behalf of a client. A **DNS record** describes information associated with a name.

| Record | Purpose |
|---|---|
| A | Maps a hostname to an IPv4 address |
| AAAA | Maps a hostname to an IPv6 address |
| CNAME | Aliases one hostname to another hostname |

The **TTL (Time to Live)** controls how long an answer may be cached. Caching avoids repeated lookups, but changes may not be observed everywhere immediately while cached answers remain valid.

## 3. TCP handshake — establish reliable transport

**TCP (Transmission Control Protocol)** is a transport-layer protocol that provides a reliable, ordered byte stream between endpoints. It detects certain transmission problems and retransmits lost data as needed. It does not inherently preserve HTTP request boundaries as individual messages.

A typical TCP connection starts with a three-way handshake:

~~~text
Client                           Server
  |                                |
  |------------- SYN ------------->|
  |<---------- SYN-ACK -------------|
  |------------- ACK -------------->|
  |                                |
  |        TCP connection ready     |
~~~

- **SYN:** The client initiates a connection and provides initial sequence-number information.
- **SYN-ACK:** The server acknowledges the client's SYN and sends its own SYN information.
- **ACK:** The client acknowledges the server's SYN.

Sequence numbers help TCP track bytes sent and acknowledged.

An IP address identifies a network destination; a **port** helps the operating system direct traffic to the correct service. Port 443 is the standard HTTPS port, and port 80 is the standard HTTP port.

TCP provides reliable, ordered transport. It does not understand the meaning of a restaurant endpoint or HTTP method.

## 4. TLS handshake — secure the connection

**TLS (Transport Layer Security)** provides cryptographic protection for communication, including encryption and integrity protection, and can authenticate the server.

For conventional HTTPS over TCP, TLS negotiation follows TCP connection establishment.

At a high level:
1. Client and server negotiate compatible security parameters.
2. The server presents a certificate to help authenticate its identity.
3. The client validates the certificate, including trust and hostname checks.
4. The endpoints establish cryptographic session keys.
5. HTTP data can be exchanged in protected form.

A **certificate** is a digitally signed document that binds an identity, such as a domain name, to a public key. A public key is part of a cryptographic key pair. Exact handshake details depend on the TLS version and configuration.

**HTTPS is HTTP protected by TLS.** TLS protects communication in transit; it does not replace HTTP's request/response semantics.

## 5. HTTP request — describe the operation

**HTTP (Hypertext Transfer Protocol)** is an application-layer protocol that defines request and response semantics.

An illustrative HTTP/1.1 request:

~~~http
GET /restaurants/123 HTTP/1.1
Host: api.foodapp.com
Accept: application/json
~~~

- **Method:** The intended operation. `GET` commonly retrieves a resource; `POST` submits data or initiates an operation; `PUT` commonly creates or replaces a resource; `PATCH` requests a partial update; `DELETE` requests deletion.
- **Path:** Identifies the resource, here restaurant 123.
- **Headers:** Metadata such as the target hostname or accepted response format.
- **Body:** Optional content sent with the request, such as JSON for a `POST /orders` request.

Methods' exact behavior depends on HTTP semantics and the API contract. A GET request commonly has no body; APIs often represent GET inputs through the path and query parameters.

## 6. The request reaches the backend

The public IP address may belong to a load balancer, reverse proxy, or gateway rather than directly to the Spring Boot machine.

An illustrative flow:

~~~text
Browser / Client
       |
       v
Public endpoint / Load Balancer
       |
       v
Spring Boot instance
       |
       v
DispatcherServlet
       |
       v
Controller -> Service -> Repository
                         |
                         v
                     PostgreSQL
~~~

A **load balancer** routes requests to an appropriate backend instance. In a typical Spring MVC application, `DispatcherServlet` helps route HTTP requests to matching controllers. The application runs business logic and may query PostgreSQL.

The layers have distinct responsibilities:
- DNS resolves the hostname.
- TCP establishes transport for TCP-based HTTP versions.
- TLS protects communication.
- HTTP defines the application request and response.
- Spring Boot implements the endpoint's behavior.

## 7. HTTP response — return the result

An illustrative HTTP response:

~~~http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 123,
  "name": "Spice Garden",
  "rating": 4.5
}
~~~

- **Status code:** Communicates the outcome.
- **Headers:** Describe response metadata, such as content type.
- **Body:** Contains the result, such as JSON restaurant details.

| Status | Meaning |
|---|---|
| 200 OK | Request succeeded |
| 201 Created | A resource was created |
| 400 Bad Request | Request is invalid or cannot be processed as submitted |
| 401 Unauthorized | Authentication credentials are missing or not accepted |
| 403 Forbidden | Server refuses the operation, commonly due to authorization |
| 404 Not Found | Resource was not found or is not being disclosed |
| 500 Internal Server Error | Unexpected server-side error |

The frontend processes the response and may parse JSON and render the result. Over HTTPS, the response is protected in transit too.

## 8. Connection reuse and caching

The setup flow does not necessarily repeat for every request:
- **DNS caching** can avoid a new hostname lookup.
- **Persistent connections** allow multiple HTTP requests to use an established connection.
- Reusing a connection can avoid another TCP handshake and, where applicable, another TLS handshake.

These techniques reduce repeated setup work and latency. Cache expiration and connection lifetimes still affect behavior.

## 9. HTTP/1.1 vs. HTTP/2 vs. HTTP/3

| Version | Key idea |
|---|---|
| HTTP/1.1 | Supports persistent connections; requests and responses follow HTTP/1.1 connection rules |
| HTTP/2 | Multiplexes multiple logical streams over one TCP connection, reducing some HTTP-level blocking limitations |
| HTTP/3 | Uses QUIC over UDP instead of TCP and integrates TLS 1.3 into QUIC connection establishment |

**Multiplexing** means multiple logical streams can share a connection.

HTTP/3 does not use the TCP three-way handshake. It uses QUIC, a transport protocol over UDP that incorporates TLS 1.3 into connection establishment. The DNS -> TCP -> TLS -> HTTP sequence above describes the familiar TCP-based path, not every possible modern HTTP connection.

## 10. Summary card

- **Idea:** DNS locates the hostname; TCP provides reliable ordered transport; TLS secures communication; HTTP defines requests and responses.
- **Flow:** URL/API call -> DNS -> TCP -> TLS -> HTTP request -> backend -> HTTP response for a new HTTPS-over-TCP connection.
- **Backend:** A load balancer may route the request to Spring Boot, which processes it and may query PostgreSQL.
- **Efficiency:** DNS caching and connection reuse reduce repeated setup work.
- **Tradeoff:** TCP adds connection setup but provides reliable, ordered delivery; TLS adds security and handshake work. HTTP/3 uses QUIC over UDP instead of TCP.
- **Interview one-liner:** “For a new HTTPS request over TCP, the client resolves the hostname through DNS, establishes TCP, negotiates TLS, sends an HTTP request, and receives an HTTP response. Caching and connection reuse can skip some setup on subsequent requests.”
