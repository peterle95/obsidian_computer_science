---
memory: to_finish
tags:
 - will_learn
language:
 - Core Concepts
review-date:
last-reviewed: ""
scheda: to_finish
visit-count: 0
confidence-level: 1
consecutive-correct: 0
last-struggle-date: ""
cssclasses:

---
# **Purpose/Why**:
---

QUIC (Quick UDP Internet Connections) is a modern transport layer network protocol designed to improve performance and security for internet connections. It was initially developed by Google and later standardized by the IETF. The fundamental problem QUIC solves is the inherent latency and performance bottlenecks of TCP (Transmission Control Protocol), especially for encrypted connections using TLS (Transport Layer Security).

Its primary applications are in services that require a quick response, such as web browsing, streaming, online gaming, and VoIP. QUIC is crucial because it addresses several long-standing issues with TCP, such as head-of-line blocking and slow connection establishment, leading to a faster and more reliable user experience on the web.

# **Core Explanation:**
---

QUIC is a connection-oriented transport protocol built on top of the connectionless User Datagram Protocol (UDP). This design choice allows QUIC to bypass the often slow and rigid mechanisms of TCP that are implemented in operating system kernels.

**Key Characteristics:**

- **Reduced Connection Latency:** QUIC significantly speeds up connection establishment. While a typical TCP connection requires a 3-way handshake, and a secure TLS connection adds even more round trips, QUIC combines the transport and cryptographic handshakes. For a new connection, it takes only one round trip (1-RTT) to establish a secure channel. For subsequent connections to a known server, it can achieve zero round-trip time (0-RTT) by caching previous connection details.

- **Multiplexing without Head-of-Line Blocking:** HTTP/2 introduced multiplexing, allowing multiple requests and responses to be sent over a single TCP connection. However, with TCP, a single lost packet can hold up all subsequent data streams until it is retransmitted, a problem known as head-of-line blocking. QUIC resolves this by allowing independent data streams. If a packet for one stream is lost, only that stream is affected while others can continue to be processed.

- **Built-in Security:** Security is a core feature of QUIC, with TLS 1 encryption integrated directly into the protocol. This ensures that connections are always encrypted and authenticated.

- **Connection Migration:** QUIC uses a unique connection identifier to maintain a connection even if the client's IP address changes, for instance, when switching from a Wi-Fi network to a cellular network. This provides a more seamless and uninterrupted experience for mobile users.

- **Improved Congestion Control:** QUIC employs more modern and flexible congestion control algorithms to better handle varying network conditions and avoid congestion.


**How it Works:**

QUIC operates on top of UDP, which by itself is unreliable. QUIC adds a layer of reliability with features like packet sequencing, acknowledgments, and retransmissions, similar to TCP. However, because it is implemented at the application layer, it can evolve and be updated more easily than TCP, which is deeply integrated into operating system kernels. When a client connects to a server using QUIC, it sends an initial packet that includes the necessary information for the server to set up an encrypted connection. This streamlined handshake process is a key contributor to its reduced latency.

# **Related Concepts:**
---

- **TCP (Transmission Control Protocol):** The traditional protocol for reliable data transfer over the internet. QUIC is designed as a successor to TCP for many applications. While both provide reliability, QUIC offers lower latency, better handling of multiplexed streams, and built-in encryption.

- **UDP (User Datagram Protocol):** A connectionless and unreliable transport protocol. QUIC is built on top of UDP, leveraging its speed and flexibility while adding its own mechanisms for reliability and security.

- **HTTP/2:** The second major version of the Hypertext Transfer Protocol. While HTTP/2 introduced multiplexing, it is limited by the head-of-line blocking issue inherent in its use of TCP.

- **HTTP/3:** The third major version of HTTP, which is designed to use QUIC as its underlying transport protocol. The adoption of HTTP/3 is a primary driver for the widespread implementation of QUIC.

# **Examples:**
---

Since QUIC is a transport layer protocol, providing a simple code example like a "hello world" script is not practical. Instead, here are some conceptual examples:

**1. QUIC Handshake (Simplified):**

A key feature of QUIC is its faster handshake process compared to TCP with TLS.

- **TCP + TLS Handshake (Simplified):**

 1. **Client -> Server:** SYN (Initiate connection)
 2. **Server -> Client:** SYN-ACK (Acknowledge and initiate)
 3. **Client -> Server:** ACK (Connection established)
 4. **Client -> Server:** ClientHello (Start TLS handshake)
 5. **Server -> Client:** ServerHello, Certificate, etc.
 6. **Client -> Server:** Finished (TLS handshake complete)
 7. **Server -> Client:** Finished (TLS handshake complete)
 This process involves multiple round trips, leading to higher latency.

- **QUIC Handshake (1-RTT, Simplified):**

 7. **Client -> Server:** Initial Packet (contains TLS ClientHello)
 8. **Server -> Client:** Initial Packet (contains TLS ServerHello, Certificate, etc.) and Handshake Packet
 9. **Client -> Server:** Handshake Packet
 The connection is established and encrypted in a single round trip.


**2. Checking for QUIC in Your Browser:**

You can observe QUIC in action using your web browser's developer tools.

1. Open a web browser that supports QUIC (like Chrome or Firefox).
2. Navigate to a website that uses QUIC (many Google services do).
3. Open the Developer Tools (usually by pressing F12).
4. Go to the "Network" tab.
5. Right-click on the column headers and ensure "Protocol" is checked.
6. Reload the page. In the "Protocol" column, you may see "h3" or "http/3", which indicates that QUIC is being used.


**3. QUIC Packet Structure:**

A QUIC packet has a distinct structure that includes a header and a payload containing one or more frames. The header can be a "long header" used during connection setup or a "short header" for established connections. The frames carry the actual data, such as stream data or control information.

# **Flashcards:**
---

What is the primary goal of the QUIC protocol?;; To reduce latency and improve the performance of web applications compared to TCP.

How does QUIC reduce connection establishment time?;; By combining the transport and cryptographic (TLS 1.3) handshakes into a single round trip (1-RTT) or even zero round trips (0-RTT) for subsequent connections.

What is "head-of-line blocking" and how does QUIC solve it?;; Head-of-line blocking is when a lost packet in a TCP stream blocks all subsequent data. QUIC solves this with independent streams, so a lost packet only affects its own stream.

What is the relationship between QUIC and UDP?;; QUIC is built on top of the UDP protocol, which allows it to be more flexible and avoid the rigidness of TCP. QUIC adds its own reliability and security features on top of UDP.

What is connection migration in QUIC?;; The ability to maintain a connection even if the user's IP address changes (e.g., switching from Wi-Fi to cellular data), thanks to a unique connection identifier.

How is security handled in QUIC?;; Security is built-in with integrated TLS 1 encryption, meaning all QUIC connections are encrypted by default.