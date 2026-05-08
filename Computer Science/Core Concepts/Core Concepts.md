---
memory: to_finish
---


---
tags: 
language: 
review-date: 
last-reviewed: 
keywords: 
scheda: to_finish
---
[[REST, REST API & RESTful API]]
[[CPU]]
[[Schema]]
[[GPU]]
 [[IP (Internet Protocol)]]
[[kernel]]
[[Same-Origin Policy (SOP)]]
[[processes]]
[[Virtual Machines]]
[[DNS (Domain Name System)]]
[[RAM]]
[[Generality]]
[[Object-relational mapping (ORM)]]
[[Relational database]]
[[Web framework]]
[[Databases]]
[[Secure Shell Protocol (SSH)]]
[[API Endpoints]]
[[HTTP Methods]]
[[TCP]]
[[API Integration (Frontend-Backend)]]
[[CORS (Cross-Origin Resource Sharing)]]
[[Port Management]]
[[Wrapper]]
[[package.json]]
[[URI - Similarities and Differences with URL]]
[[HTTP Headers]]
[[Cache]]
[[File Descriptors]]
[[HTTP Web Server]]
[[Webhook]]

Based on the sources provided, here is a list of concepts you need to learn and master for the "Webserv" project:

The core of the "Webserv" project is to build a **non-blocking HTTP server in C++98** that can handle multiple client connections. It must be robust, never crashing or terminating unexpectedly, and compatible with standard web browsers.

### I. Core HTTP Protocol Concepts

You will need a deep understanding of the Hypertext Transfer Protocol (HTTP). The **HTTP/1.0 specification (RFC 1945)** is suggested as a primary reference.

- **[[HTTP Message Structure]]**: Understand that HTTP messages consist of a **request/status line**, **header fields**, an **empty line (CRLF)**, and an **optional message body**.
- **[[Request/Response Paradigm]]**: HTTP operates on a request/response model where a client sends a request and a server sends back a response.
- **Request Line**: The first line of a request, containing the **method**, **Request-URI**, and **HTTP-Version** (e.g., `GET /index.html HTTP/1.1`).
- **Status Line**: The first line of a response, including the **HTTP-Version**, **Status Code**, and **Reason-Phrase** (e.g., `HTTP/1.1 200 OK`).
- **HTTP Methods**:
    - **GET**: Used to retrieve information. This includes understanding **conditional GET** with the `If-Modified-Since` header.
    - **HEAD**: Similar to GET, but the server **must not return an Entity-Body** in the response; it's used to obtain metainformation.
    - **POST**: Used to send data to the server, often for annotations, bulletin board messages, or submitting form data. It **requires a `Content-Length` header**.
    - **DELETE**: Used to request the server to delete a specified resource.
    - _Awareness of others_: PUT, LINK, UNLINK are mentioned as additional methods.
- **HTTP Status Codes**: Recognize and accurately implement the different classes of status codes and common codes:
    - `1xx`: Informational (reserved for future use in HTTP/1.0).
    - `2xx`: Success (e.g., **200 OK**, **201 Created**, **202 Accepted**, **204 No Content**).
    - `3xx`: Redirection (e.g., **301 Moved Permanently**, **302 Moved Temporarily**, **304 Not Modified**).
    - `4xx`: Client Error (e.g., **400 Bad Request**, **401 Unauthorized**, **403 Forbidden**, **404 Not Found**).
    - `5xx`: Server Error (e.g., **500 Internal Server Error**, **501 Not Implemented**, **502 Bad Gateway**, **503 Service Unavailable**).
- **Uniform Resource Identifiers (URIs)**: How resources are identified via name, location, or other characteristics, including `absoluteURI` and `abs_path`.
- **Header Fields**: The project requires understanding and handling various header fields:
    - **General-Header Fields**: `Date`, `Pragma`.
    - **Request-Header Fields**: `Authorization`, `From`, `If-Modified-Since`, `Referer`, `User-Agent`.
    - **Response-Header Fields**: `Location`, `Server`, `WWW-Authenticate`.
    - **Entity-Header Fields**: `Allow`, `Content-Encoding`, `Content-Length`, `Content-Type`, `Expires`, `Last-Modified`.
- **Entity Body**: Understanding how the data type (`Content-Type`) and length (`Content-Length`) of the message body are determined.
- **MIME (Multipurpose Internet Mail Extensions)**: HTTP adopts many constructs from MIME for data typing and character sets.
- **Caching Mechanisms**: The use of `Expires`, `If-Modified-Since`, and `Last-Modified` headers to manage cached responses.

### II. Server Architecture and Functionality

You need to implement specific features for the server based on configuration and HTTP interactions.

- **Configuration File Parsing**: Your server must read and apply settings from a configuration file, including:
    - Defining **multiple `interface:port` pairs** to listen on.
    - Setting **default error pages**.
    - Specifying the **maximum allowed size for client request bodies** (`client_max_body_size`).
    - Defining rules per **URL/route** (e.g., `location` blocks) for:
        - **Allowed HTTP methods** (GET, POST, DELETE).
        - **HTTP redirection** (`return` directive).
        - **Root directory** for requested files.
        - **Enabling/disabling directory listing** (`autoindex`).
        - **Default file** to serve for directories (`index`).
        - **File upload storage location**.
- **Serving Static Content**: The ability to deliver static HTML documents, images, stylesheets, etc..
- **File Uploads**: Clients must be able to upload files to the server.
- **[[Common Gateway Interface (CGI)]]**:
    - The ability to **execute external programs** (like PHP or Python scripts) based on file extension.
    - Understanding **environment variables** involved in web server-CGI communication.
    - Handling **chunked requests** for CGI (un-chunking them).
    - Managing CGI output, especially when no `Content-Length` is returned.
    - Ensuring CGI scripts run in the **correct directory** for relative path access.
- **Error Handling and Robustness**: Beyond status codes, ensure the server provides default error pages when custom ones are not specified, and remains operational under stress.
- **Security Considerations**: Be aware of limitations of basic authentication, sensitive information transfer (e.g., `Referer`, `From`, `Server` headers), and preventing attacks based on file/path names.

### III. C++98 and Networking Implementation Details

The project enforces specific technical constraints and requires strong C++98 programming and network programming skills.

- **C++98 Compliance**: All functionality must be implemented in **C++98**, without external or Boost libraries. Prefer C++ versions of functions over C when possible.
- **Non-Blocking I/O**: This is a critical requirement.
    - **Single Polling Mechanism**: Use **only one `poll()`** (or equivalent like `select()`, `epoll()`, `kqueue()`) for **all I/O operations** between clients and the server, including listening sockets.
    - **Simultaneous Monitoring**: The polling mechanism must monitor for **both reading (`POLLIN`) and writing (`POLLOUT`)** events simultaneously.
    - **No Blocking Operations**: You **must never perform a `read()` or `write()` operation without going through `poll()`** (or equivalent) first.
    - **No `errno` checking**: Do not check `errno` to adjust server behavior _after_ a read or write operation.
    - **`pollfd` Struct**: Understand its components: `fd`, `events`, and `revents`.
- **Socket Programming (Using C System Calls)**:
    - **Socket Creation**: Using `socket()` (e.g., `AF_INET`, `SOCK_STREAM`).
    - **Binding**: Attaching a socket to a specific address and port using `bind()`, configuring `sockaddr_in` struct, `INADDR_ANY`.
    - **Listening**: Setting up a socket to accept incoming connections with `listen()`.
    - **Accepting Connections**: Handling new client connections with `accept()`.
    - **Data Transfer**: Sending and receiving data using `send()`, `recv()`, `read()`, `write()`.
    - **Socket Options**: Using `setsockopt()` (e.g., `SOL_SOCKET`, `SO_REUSEADDR`) to reuse addresses.
    - **Byte Order Conversion**: Understanding and using `htons()` and `htonl()` for converting between host byte order and network byte order.
- **File Descriptors**:
    - `fcntl()`: For macOS, specifically for setting file descriptors to **non-blocking mode** using flags like `F_SETFL`, `O_NONBLOCK`, and `FD_CLOEXEC`.
    - `dup`, `dup2`: For duplicating file descriptors, potentially for I/O redirection with CGI.
- **Process Management (for CGI)**: The use of `fork()`, `execve()`, `waitpid()`, `kill()`, and `signal()` specifically for handling CGI processes.
- **File System Operations**: Functions like `chdir()`, `access()`, `stat()`, `open()`, `opendir()`, `readdir()`, and `closedir()` will be necessary for serving files and managing directories.
- **Resource Management**: Properly handling client disconnections and ensuring requests do not hang indefinitely.
- **Chunking Data**: Implementing a `BUFFER_SIZE` mechanism to process large amounts of data in chunks, preventing the server from freezing.

By focusing on these concepts, you will build a solid foundation for your "Webserv" project. Remember to constantly test your server with web browsers and tools like `telnet` or `NGINX` for comparison.