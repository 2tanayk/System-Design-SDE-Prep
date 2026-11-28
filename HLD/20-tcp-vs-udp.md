# TCP vs. UDP — Transport Protocols, Ports, and Sockets

## 1. Where do TCP and UDP fit?

When one machine sends data to another over the internet, IP helps route packets toward the destination machine. But IP is best-effort: by itself, it does not guarantee that packets arrive, arrive in order, or arrive only once.

The transport layer adds communication behavior on top of IP. Two major transport protocols are:

- **TCP — Transmission Control Protocol**
- **UDP — User Datagram Protocol**

A simplified network stack:

    Application layer    HTTP, DNS, PostgreSQL protocol, SSH, etc.
            |
    Transport layer      TCP or UDP
            |
    Internet layer       IP
            |
    Network access       Ethernet, Wi-Fi, etc.
            |
            v
          Network

The important distinction:

- IP helps get packets to the destination network endpoint.
- TCP or UDP provides transport behavior between endpoints.
- Application protocols define what the application-level messages mean.

Not every application communicates over HTTP. HTTP is one application-layer protocol among many.

## 2. What is an IP address, a port, and a socket?

Imagine one machine is running several services:

- A Spring Boot application.
- A PostgreSQL database.
- An SSH server.

The machine has a network address, but the operating system also needs to know which service should receive incoming communication.

That is where **port numbers** come in.

### IP address

An IP address identifies a network interface/host destination for routing.

Example:

    192.168.1.10

### Port

A port number identifies a transport endpoint on that machine.

For example:

    192.168.1.10:8080  -> Spring Boot HTTP server
    192.168.1.10:5432  -> PostgreSQL server
    192.168.1.10:22    -> SSH server

A server process creates a socket and listens on a local IP address and port. When data arrives for that endpoint, the operating system can deliver it to the appropriate socket.

**A port is not the application protocol itself.** It is a number used by the transport layer and operating system to identify an endpoint. The service listening there understands the relevant application protocol.

### Socket

A socket is an operating-system communication endpoint used by an application to send or receive network data.

For a listening server, it is associated with a local address and port, along with a transport protocol. A connected TCP socket also has peer information.

A useful mental model:

- IP address: which machine/network interface?
- Port: which transport endpoint on that machine?
- TCP or UDP: which transport protocol?
- Application protocol: what do the messages mean?

### Common port conventions

| Service | Common default port | Typical transport / application protocol |
|---|---:|---|
| HTTP | 80 | HTTP over TCP (HTTP/1.1, HTTP/2) |
| HTTPS | 443 | HTTPS; commonly TCP for HTTP/1.1 and HTTP/2, UDP for HTTP/3 over QUIC |
| SSH | 22 | SSH over TCP |
| PostgreSQL | 5432 | PostgreSQL wire protocol over TCP |
| DNS | 53 | Usually UDP for ordinary queries; TCP is also used |

These are conventions/defaults, not hard rules. Services can be configured to listen on different ports.

Also, TCP and UDP have separate port-number spaces. For example, TCP port 53 and UDP port 53 are distinct transport endpoints even though they use the same numeric port.

## 3. A practical backend example: why PostgreSQL uses port 5432

A database server normally does not communicate with your backend using HTTP.

Consider a Spring Boot application with PostgreSQL:

    Browser
       |
       | HTTP request to port 8080
       v
    Spring Boot application
       |
       | PostgreSQL wire protocol over TCP to port 5432
       v
    PostgreSQL server

The browser might send an HTTP request:

    GET /api/posts/123 HTTP/1.1
    Host: localhost:8080

Spring Boot receives that HTTP request. When it needs data, its PostgreSQL JDBC driver establishes a TCP connection to PostgreSQL and speaks the PostgreSQL wire protocol.

Example JDBC URL:

    jdbc:postgresql://localhost:5432/blogdb

Breakdown:

- jdbc: PostgreSQL is being accessed through JDBC.
- postgresql: the JDBC driver for PostgreSQL is used.
- localhost: the database host.
- 5432: the port on which PostgreSQL is listening.
- blogdb: the target database.

The database protocol is not HTTP. JDBC is the Java database connectivity API; the PostgreSQL driver uses the PostgreSQL protocol over the network.

### When can a database be accessed over HTTP?

An application can expose database operations through a REST API or another HTTP interface:

    Browser
       |
       | HTTP
       v
    REST API
       |
       | PostgreSQL protocol over TCP
       v
    PostgreSQL

Here, the REST API speaks HTTP to the browser. The database still speaks its native protocol to the backend.

Some managed database products offer additional HTTP APIs, but that is an extra interface, not the normal PostgreSQL connection mechanism.

## 4. TCP — Transmission Control Protocol

TCP is a **connection-oriented** transport protocol. It provides a reliable, ordered byte stream between two endpoints.

The key properties are:
- Connection establishment.
- Reliable delivery using acknowledgments and retransmission.
- Ordered delivery.
- Flow control.
- Congestion control.

### A. Connection establishment: three-way handshake

Before normal application data is exchanged, TCP establishes a logical connection using a three-way handshake.

    Client                              Server

      |                                  |
      |------------ SYN ---------------->|
      |                                  |
      |<--------- SYN-ACK ---------------|
      |                                  |
      |------------ ACK ---------------->|
      |                                  |
      |       Connection established     |

1. **SYN:** The client requests connection establishment and sends its initial sequence number.
2. **SYN-ACK:** The server acknowledges the client's SYN and sends its own SYN.
3. **ACK:** The client acknowledges the server's SYN.

The connection is a logical relationship maintained by the two endpoints. It does not reserve a dedicated physical network path.

### B. Reliable delivery

Suppose the sender transmits three portions of data:

    Segment 1
    Segment 2
    Segment 3

Segment 2 is lost in the network.

TCP uses sequence numbers and acknowledgments to detect missing data. Depending on the situation, the sender detects loss through duplicate acknowledgments or a retransmission timeout and retransmits the missing data.

Conceptually:

    Sender                              Receiver

    Segment 1 ------------------------> Received
    Segment 2 --------X                 Lost
    Segment 3 ------------------------> Received

                                        Detects a gap
    Retransmit Segment 2 ------------> Missing bytes recovered

TCP's sequence numbers identify positions in the byte stream, and acknowledgments indicate what data has been received. The receiver can reconstruct the correct ordered byte stream.

Important: a TCP acknowledgment confirms receipt by the peer's TCP stack. It does not prove that the receiving application has processed the data or committed a database transaction.

### C. Ordered delivery

Suppose data is transmitted in this order:

    A -> B -> C -> D

But the network delivers the segments out of order:

    A -> C -> D -> B

TCP uses sequence numbers to reconstruct the correct byte order. The receiving application reads the stream in order:

    A -> B -> C -> D

This is useful for file transfers, database communication and ordinary HTTP communication, where the application expects a coherent byte stream.

### D. TCP is a byte stream, not a message protocol

This is an important interview detail.

Suppose the sender performs two writes:

    write("HELLO")
    write("WORLD")

The sender might think it sent two messages. TCP does not preserve application write boundaries.

The receiving application might read:

    HEL
    LOWORLD

Or:

    HELLOWORLD

Or several other chunk combinations.

TCP guarantees an ordered stream of bytes, not the boundaries of application-level messages.

If an application needs message boundaries, it must define a framing mechanism. Common approaches include:

- Fixed-length messages.
- Length-prefixed messages.
- Delimiters, such as newline characters.

This matters when implementing a custom protocol over a persistent TCP connection.

### E. Flow control

Flow control prevents a fast sender from overwhelming a slow receiver.

For example:
- A sender can produce data at 100 MB/s.
- The receiver can process data at only 10 MB/s.

TCP uses a receiver-advertised window to regulate how much unacknowledged data the sender can have in flight.

**Flow control protects the receiver.**

### F. Congestion control

Congestion control helps prevent senders from overwhelming the network.

Suppose many machines transmit large amounts of data through the same network bottleneck. TCP adjusts its sending behavior in response to congestion signals, such as packet loss and acknowledgments.

**Congestion control protects the network.**

Flow control and congestion control solve different problems:
- Flow control: can the receiver keep up?
- Congestion control: can the network path handle the traffic?

## 5. UDP — User Datagram Protocol

UDP is a **connectionless** transport protocol. It sends independent units of data called datagrams without establishing a TCP-style connection first.

Conceptually:

    Client                              Server

    Datagram 1 ------------------------>
    Datagram 2 ------------------------>
    Datagram 3 ------------------------>

There is no TCP three-way handshake.

### What does UDP provide?

UDP provides:
- Port-based delivery to the appropriate socket.
- Preservation of datagram boundaries.
- A checksum for detecting certain data corruption (checksum requirements differ between IPv4 and IPv6).

UDP does not provide built-in:
- Guaranteed delivery.
- Retransmission.
- Ordered delivery.
- Duplicate suppression.
- Flow control.
- Congestion control.

Suppose the sender transmits:

    A -> B -> C -> D

The receiver might get:

    A -> C -> D

B was lost.

Or the datagrams might arrive in a different order:

    B -> A -> D -> C

UDP itself does not repair the loss or reorder the datagrams. If the application needs those properties, it must implement them or use another protocol that provides them.

### Why use UDP if it does not guarantee delivery?

Sometimes waiting for reliable, ordered delivery is worse than losing an occasional piece of data.

Imagine a live voice call. One audio datagram is lost.

Should the application pause playback until that old audio is retransmitted?

Usually not. By the time it arrives, it may already be too late to be useful. Continuing with newer audio can provide a better real-time experience.

This is why UDP is useful for time-sensitive communication.

## 6. TCP vs. UDP comparison

| Property | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented | Connectionless |
| Handshake | Three-way handshake | No TCP-style handshake |
| Reliability | Acknowledgments and retransmission | No built-in delivery guarantee |
| Ordering | Ordered byte stream | No ordering guarantee |
| Data model | Byte stream | Individual datagrams |
| Message boundaries | Not preserved | Preserved |
| Flow control | Built in | No built-in mechanism |
| Congestion control | Built in | No built-in mechanism |
| Typical overhead | More transport machinery | Simpler transport machinery |
| Common uses | HTTP/1.1, HTTP/2, PostgreSQL, SSH, file transfer | DNS queries, real-time media, many game protocols |

UDP is not automatically faster in every situation. It simply avoids TCP's built-in connection and reliability machinery. Actual performance depends on the application, network, protocol behavior and workload.

## 7. Real-world use cases

### TCP

- **HTTP/1.1 and HTTP/2:** Traditionally use TCP for reliable, ordered delivery.
- **PostgreSQL:** Commonly uses TCP and the PostgreSQL wire protocol, usually on port 5432.
- **File transfer:** The receiver needs the complete data in the correct order.
- **SSH:** Remote terminal communication needs reliable, ordered data.

### UDP

- **Real-time voice and video:** Late media data may no longer be useful, so timely delivery can matter more than retransmission.
- **Online multiplayer games:** Frequent state updates are time-sensitive; a newer position may make an older update obsolete.
- **DNS:** Traditional DNS queries commonly use UDP, while TCP is also used in cases such as zone transfers and responses that require it.

Not every game or media protocol uses UDP. Applications can also build reliability on top of UDP when they need it.

## 8. Can an application make UDP reliable?

Yes. Saying "UDP is unreliable" does not mean applications using UDP can never provide reliability.

An application can build mechanisms such as:
- Sequence numbers to detect missing or reordered datagrams.
- Acknowledgments to confirm receipt.
- Retransmission of important messages.
- Timeouts to detect unresponsive peers.
- Congestion-control mechanisms.

The distinction:

    TCP:
    Reliability and ordering are provided by the transport protocol.

    UDP:
    Reliability and ordering, if needed, are implemented by
    the application or another protocol built on top.

This flexibility is useful when an application wants custom transport behavior rather than TCP's exact semantics.

## 9. QUIC and HTTP/3 — a modern connection

QUIC is a modern transport protocol built on top of UDP.

It implements reliability, congestion control, encryption integration and stream management on top of UDP. HTTP/3 uses QUIC.

This is a useful reminder: choosing UDP does not mean giving up all reliability. QUIC builds transport capabilities on top of UDP while making different design choices from TCP.

We'll explore HTTP/1.1, HTTP/2 and HTTP/3 in the next topic.

## 10. Head-of-line blocking in TCP

Suppose a TCP connection carries:

    A -> B -> C -> D

B gets lost, but C and D reach the receiver.

Can the receiving application consume C and D immediately as part of the ordered TCP byte stream?

No. TCP must recover the missing earlier bytes before delivering later bytes in order.

This is called **head-of-line blocking**.

It can increase latency when later data is available but an earlier portion of the same TCP stream is missing.

UDP does not impose this same transport-level ordered-delivery requirement across datagrams. An application can process C and D even when B is missing, provided its own protocol allows that.

QUIC uses independent streams so loss affecting one stream does not necessarily block delivery on unrelated streams. Packet loss and congestion can still affect multiple streams, but the transport-level ordering behavior is different from TCP's single byte stream.

## 11. How to choose in a system design interview

Start with the application's requirements instead of saying only "TCP is reliable, UDP is fast."

| Requirement | Likely choice |
|---|---|
| Every byte must arrive correctly and in order | TCP |
| File transfer | TCP |
| Typical database connection | TCP |
| Time-sensitive updates where old data can be discarded | UDP |
| Real-time media transport | Often UDP-based |
| Custom transport behavior | UDP can be a foundation |

The central trade-off:

**TCP provides reliable, ordered byte delivery by default. UDP provides lightweight datagram delivery and leaves reliability and ordering decisions to the application.**

## 12. Interview summary

- **IP address:** Identifies the network destination.
- **Port:** Identifies a transport endpoint on that destination machine.
- **Socket:** An operating-system communication endpoint used by an application.
- **Application protocol:** Defines the meaning and structure of application messages, such as HTTP or PostgreSQL protocol.
- **TCP:** Connection-oriented, reliable, ordered byte stream.
- **Three-way handshake:** SYN -> SYN-ACK -> ACK.
- **Sequence numbers:** Help reconstruct byte order and detect missing data.
- **Acknowledgments and retransmission:** Help TCP recover missing data.
- **Flow control:** Prevents overwhelming the receiver.
- **Congestion control:** Helps prevent overwhelming the network.
- **UDP:** Connectionless datagram delivery, without built-in reliability or ordering.
- **Head-of-line blocking:** Later TCP bytes cannot be delivered to the application before missing earlier bytes in the same stream are recovered.
- **QUIC:** Provides modern transport capabilities over UDP; HTTP/3 uses it.

**Remember:**
- TCP asks: "How do I deliver this byte stream reliably and in order?"
- UDP asks: "How do I send this datagram with minimal transport machinery?"
- A port is not the same thing as HTTP. A service listens on a port and speaks an application protocol over TCP or UDP.

## References

- [RFC 9293 — Transmission Control Protocol (TCP)](https://www.rfc-editor.org/rfc/rfc9293)
- [RFC 768 — User Datagram Protocol (UDP)](https://www.rfc-editor.org/rfc/rfc768)
- [IANA — Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)
- [PostgreSQL Documentation — Connection URI](https://www.postgresql.org/docs/current/libpq-connect.html)
- [PostgreSQL Documentation — Protocol Overview](https://www.postgresql.org/docs/current/protocol.html)
- [RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport](https://www.rfc-editor.org/rfc/rfc9000)
- [RFC 9114 — HTTP/3](https://www.rfc-editor.org/rfc/rfc9114)
