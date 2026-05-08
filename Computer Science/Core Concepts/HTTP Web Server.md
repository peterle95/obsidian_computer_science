---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-09-17
scheda: done
visit-count: 5
confidence-level: 3
consecutive-correct: 4
last-struggle-date: 2025-08-06
cssclasses:
  - important
---

```dataviewjs
const currentPage = dv.current();
let visitCount = currentPage.file.frontmatter["visit-count"] || 0;
let confidence = currentPage.file.frontmatter["confidence-level"] || 1;
let streak = currentPage.file.frontmatter["consecutive-correct"] || 0;

const container = this.container.createEl('div');
container.style.cssText = `
    background: #2a2a2a; border: 1px solid #404040; border-radius: 6px;
    padding: 12px; margin: 10px 0; display: inline-block;
`;

// Status display
const status = container.createEl('div');
status.innerHTML = `
    <strong>Learning Progress</strong><br>
    Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
`;
status.style.cssText = 'margin-bottom: 10px; font-size: 13px; color: #cccccc;';

// Quick feedback buttons
const buttonContainer = container.createEl('div');
['Got it! ✅', 'Struggled ⚠️', 'Failed ❌'].forEach((label, index) => {
    const btn = buttonContainer.createEl('button');
    btn.textContent = label;
    btn.style.cssText = `
        margin-right: 8px; padding: 4px 8px; border: none; border-radius: 3px;
        cursor: pointer; font-size: 11px;
        background: ${['#28a745', '#ffc107', '#dc3545'][index]};
        color: ${index === 1 ? '#000' : '#fff'};
    `;
    
    btn.addEventListener('click', async () => {
        visitCount++;
        if (index === 0) { // Got it
            confidence = Math.min(5, confidence + 0.5);
            streak++;
        } else { // Struggled or failed
            confidence = Math.max(1, confidence - 0.5);
            streak = 0;
        }
        
        const file = app.vault.getAbstractFileByPath(currentPage.file.path);
        await app.fileManager.processFrontMatter(file, (fm) => {
            fm["visit-count"] = visitCount;
            fm["confidence-level"] = confidence;
            fm["consecutive-correct"] = streak;
            fm["last-reviewed"] = new Date().toISOString().split('T')[0];
            if (index > 0) fm["last-struggle-date"] = new Date().toISOString().split('T')[0];
        });
        
        status.innerHTML = `
            <strong>Learning Progress</strong><br>
            Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
        `;
    });
});
```

```dataviewjs
// Get all flashcards from the current note
const currentPage = dv.current();
const content = await app.vault.read(app.vault.getAbstractFileByPath(currentPage.file.path));

// Split content into lines
const lines = content.split('\n');
let flashcardLines = [];
let inCodeBlock = false;

// Collect all potential flashcard lines - simplified approach
for (let i = 0; i < lines.length; i++) {
    const line = lines[i];
    
    // Track code blocks
    if (line.trim().startsWith('```')) {
        inCodeBlock = !inCodeBlock;
        continue;
    }
    
    // Skip everything inside code blocks
    if (inCodeBlock) continue;
    
    // Check if line contains flashcard separator
    if (line.includes(';;')) {
        flashcardLines.push(line);
    }
}

const filteredLines = flashcardLines.filter(line => {
    // Only filter out lines that are clearly part of the JavaScript code
    // Be more specific with patterns to avoid false positives
    return !(
        line.trim().startsWith('const ') ||
        line.trim().startsWith('let ') ||
        line.trim().startsWith('function ') ||
        line.trim().startsWith('return ') ||
        line.trim().startsWith('if (') ||
        line.trim().startsWith('for (') ||
        line.trim().startsWith('while (') ||
        line.includes('dataviewjs') ||
        line.includes('content.split') ||
        line.includes('flashcardLines') ||
        line.includes('this.container') ||
        line.includes('addEventListener') ||
        line.includes('console.log') ||
        /\.\w+\(/.test(line) && (line.includes('.map(') || line.includes('.filter(') || line.includes('.forEach(') || line.includes('.find('))
    );
});

// Process the flashcards
const flashcards = [];
for (let i = 0; i < filteredLines.length; i++) {
    const line = filteredLines[i];
    try {
        const separatorIndex = line.indexOf(';;');
        if (separatorIndex === -1) continue;
        
        const front = line.substring(0, separatorIndex).trim();
        const back = line.substring(separatorIndex + 2).trim();
        
        // Very minimal validation - just check they exist
        if (front && back) {
            flashcards.push({ 
                front: front, 
                back: back,
                index: i // Keep track of original order
            });
        }
    } catch (error) {
        console.log('Error processing flashcard line:', line, error);
    }
}

// Debug information
console.log('All lines with ;;:', flashcardLines.length);
console.log('After filtering:', filteredLines.length);
console.log('Filtered lines:', filteredLines);
console.log('Final flashcards:', flashcards.length);

if (flashcards.length === 0) {
    const errorMsg = this.container.createEl('div');
    errorMsg.style.cssText = 'background: #2a2a2a; padding: 15px; border-radius: 6px; color: #cccccc;';
    errorMsg.innerHTML = `
        <strong>No flashcards found!</strong><br><br>
        Lines with ';;' found: ${flashcardLines.length}<br>
        After filtering: ${filteredLines.length}<br>
        Valid flashcards: ${flashcards.length}<br><br>
        <strong>All lines with ;;:</strong><br>
        ${flashcardLines.map((line, i) => `${i+1}. ${line.substring(0, 50)}...`).join('<br>')}
    `;
    return;
}

// Flashcard state
let currentCardIndex = 0;
let showingBack = false;
let correctCount = 0;
let totalReviewed = 0;

// Create main container
const container = this.container.createEl('div');
container.style.cssText = `
    background: #2a2a2a;
    border: 1px solid #404040;
    border-radius: 8px;
    padding: 20px;
    margin: 15px 0;
    max-width: 700px;
`;

// Header with progress
const header = container.createEl('div');
header.style.cssText = 'display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px;';

const title = header.createEl('h3');
title.textContent = `Flashcards (${flashcards.length} total)`;
title.style.cssText = 'margin: 0; color: #ffffff;';

const progress = header.createEl('div');
progress.style.cssText = 'color: #cccccc; font-size: 14px; text-align: right;';

// Card container
const cardContainer = container.createEl('div');
cardContainer.style.cssText = `
    background: #1a1a1a;
    border: 2px solid #404040;
    border-radius: 6px;
    padding: 30px;
    margin: 20px 0;
    min-height: 120px;
    display: flex;
    align-items: center;
    justify-content: center;
    text-align: center;
    cursor: pointer;
    transition: all 0.2s;
`;

const cardText = cardContainer.createEl('div');
cardText.style.cssText = `
    font-size: 16px;
    line-height: 1.5;
    color: #ffffff;
    word-wrap: break-word;
    max-width: 100%;
`;

// Button container
const buttonContainer = container.createEl('div');
buttonContainer.style.cssText = 'display: flex; gap: 10px; justify-content: center; flex-wrap: wrap; margin-bottom: 15px;';

// Control buttons
const flipButton = buttonContainer.createEl('button');
flipButton.textContent = 'Flip Card';
flipButton.style.cssText = `
    background: #4a9eff; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px; font-weight: 500;
`;

const easyButton = buttonContainer.createEl('button');
easyButton.textContent = 'Easy ✅';
easyButton.style.cssText = `
    background: #28a745; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const hardButton = buttonContainer.createEl('button');
hardButton.textContent = 'Hard ❌';
hardButton.style.cssText = `
    background: #dc3545; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const nextButton = buttonContainer.createEl('button');
nextButton.textContent = 'Next →';
nextButton.style.cssText = `
    background: #6c757d; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const prevButton = buttonContainer.createEl('button');
prevButton.textContent = '← Prev';
prevButton.style.cssText = `
    background: #6c757d; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const shuffleButton = buttonContainer.createEl('button');
shuffleButton.textContent = '🔀 Shuffle';
shuffleButton.style.cssText = `
    background: #17a2b8; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

// Functions
function updateDisplay() {
    const card = flashcards[currentCardIndex];
    cardText.textContent = showingBack ? card.back : card.front;
    
    progress.innerHTML = `Card ${currentCardIndex + 1} of ${flashcards.length}`;
    if (totalReviewed > 0) {
        progress.innerHTML += `<br>Correct: ${correctCount}/${totalReviewed} (${Math.round(correctCount/totalReviewed*100)}%)`;
    }
    
    // Show/hide difficulty buttons
    if (showingBack) {
        easyButton.style.display = 'inline-block';
        hardButton.style.display = 'inline-block';
        flipButton.textContent = 'Show Front';
        cardContainer.style.borderColor = '#ffc107';
        cardContainer.style.backgroundColor = '#252525';
    } else {
        easyButton.style.display = 'none';
        hardButton.style.display = 'none';
        flipButton.textContent = 'Flip Card';
        cardContainer.style.borderColor = '#404040';
        cardContainer.style.backgroundColor = '#1a1a1a';
    }
    
    // Update navigation buttons
    prevButton.style.display = currentCardIndex > 0 ? 'inline-block' : 'none';
    nextButton.textContent = currentCardIndex < flashcards.length - 1 ? 'Next →' : 'Restart';
}

function flipCard() {
    showingBack = !showingBack;
    updateDisplay();
}

function nextCard() {
    if (currentCardIndex < flashcards.length - 1) {
        currentCardIndex++;
    } else {
        currentCardIndex = 0;
    }
    showingBack = false;
    updateDisplay();
}

function prevCard() {
    if (currentCardIndex > 0) {
        currentCardIndex--;
        showingBack = false;
        updateDisplay();
    }
}

function markCorrect() {
    if (showingBack) {
        correctCount++;
        totalReviewed++;
        nextCard();
    }
}

function markIncorrect() {
    if (showingBack) {
        totalReviewed++;
        nextCard();
    }
}

function shuffleCards() {
    for (let i = flashcards.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [flashcards[i], flashcards[j]] = [flashcards[j], flashcards[i]];
    }
    currentCardIndex = 0;
    showingBack = false;
    correctCount = 0;
    totalReviewed = 0;
    updateDisplay();
}

// Event listeners
cardContainer.addEventListener('click', flipCard);
flipButton.addEventListener('click', flipCard);
easyButton.addEventListener('click', markCorrect);
hardButton.addEventListener('click', markIncorrect);
nextButton.addEventListener('click', nextCard);
prevButton.addEventListener('click', prevCard);
shuffleButton.addEventListener('click', shuffleCards);

// Instructions
const instructions = container.createEl('div');
instructions.style.cssText = 'font-size: 12px; color: #888; text-align: center; line-height: 1.4;';
instructions.innerHTML = `
    <strong>Controls:</strong> Click card to flip | Navigation buttons | Easy/Hard to mark
`;

// Initialize
updateDisplay();
```
# **Purpose/Why**:
---

An HTTP Web Server solves the fundamental problem of ==**delivering web content** (like HTML pages, images, videos, and API data) from a server to a client (typically a web browser) over the internet==. It acts as the backbone of the World Wide Web, enabling users to access information and interact with online applications by standardizing how requests are received, processed, and responses are sent back.

Its primary application is to host websites and web services, making them accessible to users worldwide. In computer science, understanding HTTP web servers is crucial because it embodies core networking concepts ([[TCP]]/IP, [[sockets]], [[I⧸O Multiplexing]]), protocol implementation (HTTP), data parsing, file serving, and concurrency management, which are foundational for distributed systems and internet infrastructure. Without compliant web servers, the vast ecosystem of the internet as we know it would not exist.

# **Core Explanation:**
---

A basic HTTP web server consists of several components that work together to receive and process HTTP requests from clients and send back responses. Below are the main parts of our webserver.

### Server Core
The networking part of a web server that ==handles [[TCP]] connections and performs tasks such as listening for incoming requests and sending back responses==. It is responsible for the ==low-level networking== tasks of the web server, such as ==creating and managing sockets, handling input and output streams, and managing the flow of data between the server and clients==. Before writing your webserver, I would recommend reading this awesome guide on building simple TCP client/server in C as it will help you get a good understanding of how TCP works in C/C++. Also, you would need to understand [[I⧸O Multiplexing]], this video will help you grasp the main idea of `select()`.

### Request Parser
The parsing part of a web server refers to the process that is responsible for i<mark style="background: #FF5582A6;">nterpreting and extracting information from HTTP requests</mark>. In this web server, the parsing of requests is performed by the `HttpRequest` class. An `HttpRequest` <mark style="background: #FF5582A6;">object receives an incoming request, parses it, and extracts the relevant information such as the method, path, headers, and message body (if present)</mark>. If any syntax error was found in the request during parsing, error flags are set and parsing stops. Requests can be fed to the object through the method `feed()` either fully or partially; this is possible because the parser scans the request byte at a time and updates the parsing state whenever needed. The same way of parsing is used by Nginx and Node.js request parsers.
### Response Builder
The response builder is responsible for <mark style="background: #BBFABBA6;">constructing and formatting the HTTP responses that are sent back to clients in response to their requests</mark>. In this web server, the `Response` class is responsible for building and storing the HTTP response, including the status line, headers, and message body. The response builder may also perform tasks such as <mark style="background: #BBFABBA6;">setting the appropriate status code and reason phrase based on the result of the request, adding headers to the response to provide additional information about the content or the server, and formatting the message body according to the content type and encoding of the response</mark>. For example, if the server receives a request for a webpage from a client, the server will parse the request and pass it to a `Response` object which will fetch the contents of the webpage and construct the HTTP response with the HTML content in the message body and the appropriate headers, such as the `Content-Type` and `Content-Length` headers.
### Configuration File
A configuration file is a <mark style="background: #ADCCFFA6;">text file that contains various settings and directives that dictate how the web server should operate</mark>. These settings can include things like the port number that the web server should listen on, the location of the web server's root directory, and many other settings.

```nginx
server {
  listen 8001;                        # listening port, mandatory parameter
  host 127.0.0.1;                     # host or 127.0.0.1 by default
  server_name test;                   # specify server_name, need to be added into /etc/hosts to work
  error_page 404 /error/404.html;     # default error page
  client_max_body_size 1024;          # max request body size in bytes
  root docs/fusion_web/;              # root folder of site directory, full or relative path, mandatory parameter
  index index.html;                   # default page when requesting a directory, index.html by default

  location /tours {                   
      root docs/fusion_web;           # root folder of the location, if not specified, taken from the server. 
                                      # EX: - URI /tours           --> docs/fusion_web/tours
                                      #     - URI /tours/page.html --> docs/fusion_web/tours/page.html 
      autoindex on;                   # turn on/off directory listing
      allow_methods POST GET;         # allowed methods in location, GET only by default
      index index.html;               # default page when requesting a directory, copies root index by default
      return abc/index1.html;         # redirection
      alias  docs/fusion_web;         # replaces location part of URI. 
                                      # EX: - URI /tours           --> docs/fusion_web
                                      #     - URI /tours/page.html --> docs/fusion_web/page.html 
  }

  location cgi-bin {
      root ./;                                                 # cgi-bin location, mandatory parameter
      cgi_path /usr/bin/python3 /bin/bash;                     # location of interpreters installed on the current system, mandatory parameter
      cgi_ext .py .sh;                                         # extensions for executable files, mandatory parameter
  }
}
```
### CGI (Common Gateway Interface)
CGI is a standard for running external programs from a web server. When a user requests a web page that should be handled by a CGI program, the web server executes the program and returns the output to the user's web browser. CGI programs are simply scripts that can be written in any programming language, such as Perl, Python, or bash, and are typically used to process data submitted by a user through a web browser, or to generate dynamic content on a web page.


<img src="assets/images/CGI.jpg" alt="CGI" style="width: 750px; height: auto;" />

# **Related Concepts:**
---

*   **TCP_IP**: The fundamental set of networking protocols on which HTTP operates. TCP provides reliable, connection-oriented communication, which the Web Server Core uses to establish and maintain client connections.
*   **[[Sockets]]**: The endpoint of a two-way communication link between two programs running on the network. The Server Core uses sockets to listen for connections and send/receive data.
*   **[[I⧸O Multiplexing]] (e.g., `select()`, `poll()`, `epoll()`)**: Techniques used by the Server Core to efficiently monitor multiple file descriptors (sockets) for I/O readiness, allowing a single thread to handle many client connections concurrently without blocking. This is crucial for performance in a web server.
*   **[[HTTP Message Structure]]**: The defined format for requests and responses that the Request Parser interprets and the Response Builder constructs. This includes the start-line, headers, and optional body.
*   **[[HTTP Methods]]**: The actions clients request to perform on a resource (e.g., GET, POST, DELETE), which the Request Parser extracts and the server logic then processes.
*   **[[HTTP Headers]]**: Metadata key-value pairs within HTTP messages crucial for communication (e.g., `Content-Type`, `Content-Length`, `Host`). The Request Parser extracts them, and the Response Builder adds them.
*   **[[HTTP Versions]]**: Specify the protocol rules (e.g., HTTP/1.0, HTTP/1.1, HTTP/2). The server must adhere to a specific version for parsing and response generation.
*   **Client-Server Model**: The foundational architectural pattern where the web server acts as the server, providing services to clients (web browsers).

# **Examples:**
---
### Configuration File Example
Here is an example file that shows config file format and supported directives. This type of file dictates the server's behavior, similar to Nginx's configuration.

```nginx
server {
  listen 8001;                        # listening port, mandatory parameter
  host 127.0.0.1;                     # host or 127.0.0.1 by default
  server_name test;                   # specify server_name, need to be added into /etc/hosts to work
  error_page 404 /error/404.html;     # default error page
  client_max_body_size 1024;          # max request body size in bytes
  root docs/fusion_web/;              # root folder of site directory, full or relative path, mandatory parameter
  index index.html;                   # default page when requesting a directory, index.html by default

  location /tours {
    root docs/fusion_web;           # root folder of the location, if not specified, taken from the server.
                                    # EX: - URI /tours           --> docs/fusion_web/tours
                                    #     - URI /tours/page.html --> docs/fusion_web/tours/page.html
    autoindex on;                   # turn on/off directory listing
    allow_methods POST GET;         # allowed methods in location, GET only by default
    index index.html;               # default page when requesting a directory, copies root index by default
    return abc/index1.html;         # redirection
    alias  docs/fusion_web;         # replaces location part of URI.
                                    # EX: - URI /tours           --> docs/fusion_web
                                    #     - URI /tours/page.html --> docs/fusion_web/page.html
  }

  location cgi-bin {
    root ./;                                                 # cgi-bin location, mandatory parameter
    cgi_path /usr/bin/python3 /bin/bash;                     # location of interpreters installed on the current system, mandatory parameter
    cgi_ext .py .sh;                                         # extensions for executable files, mandatory parameter
  }
}
```

### Conceptual C++ Web Server Flow (Simplified)
This example illustrates the high-level interaction between the Server Core, Request Parser, and Response Builder. Real-world implementations are much more complex, especially for robust error handling, concurrency, and performance.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <map>

// --- Server Core (Conceptual) ---
// Represents the low-level networking part.
// In a real server, this would involve sockets, bind, listen, accept, etc.
class ServerCore {
public:
    // Simulates listening for a connection and accepting a request.
    std::string receiveRequest() {
        std::cout << "ServerCore: Listening for incoming requests...\n";
        // In a real scenario, this would read from a socket.
        // For demonstration, returning a hardcoded raw HTTP request.
        return "GET /index.html HTTP/1.1\r\nHost: example.com\r\nUser-Agent: my-browser\r\n\r\n";
    }

    // Simulates sending a response back to the client.
    void sendResponse(const std::string& response) {
        std::cout << "ServerCore: Sending response to client.\n";
        // In a real scenario, this would write to a socket.
        std::cout << "--- RESPONSE SENT ---\n" << response << "\n---------------------\n";
    }
};

// --- Request Parser (Conceptual) ---
// Interprets the raw HTTP request string.
class HttpRequest {
public:
    std::string method;
    std::string path;
    std::string httpVersion;
    std::map<std::string, std::string> headers;
    std::string body;

    // Simulates parsing the raw request. In a real system, 'feed' would read byte-by-byte.
    void feed(const std::string& rawRequest) {
        std::cout << "RequestParser: Parsing incoming request...\n";
        // Simple parsing logic (not robust for real HTTP, just for concept)
        std::istringstream iss(rawRequest);
        std::string line;

        // Parse Request Line (e.g., GET /index.html HTTP/1.1)
        if (std::getline(iss, line) && !line.empty()) {
            std::istringstream lineStream(line);
            lineStream >> method >> path >> httpVersion;
        }

        // Parse Headers (key: value)
        while (std::getline(iss, line) && line != "\r") { // Headers end with a blank line
            size_t colonPos = line.find(':');
            if (colonPos != std::string::npos) {
                std::string headerName = line.substr(0, colonPos);
                std::string headerValue = line.substr(colonPos + 2); // +2 to skip ': '
                headers[headerName] = headerValue;
            }
        }

        // Parse Body (if any, typically for POST/PUT)
        std::string bodyLine;
        while (std::getline(iss, bodyLine)) {
            body += bodyLine + "\n";
        }
        if (!body.empty()) body.pop_back(); // Remove last newline
        
        std::cout << "RequestParser: Method: " << method << ", Path: " << path << "\n";
        std::cout << "RequestParser: Headers parsed: " << headers.size() << "\n";
    }

    // Checks for parsing errors (simplified)
    bool hasError() const {
        return method.empty() || path.empty(); // Basic check
    }
};

// --- Response Builder (Conceptual) ---
// Constructs the HTTP response.
class Response {
public:
    std::string statusLine;
    std::map<std::string, std::string> headers;
    std::string body;

    // Simulates building the response based on request processing outcome.
    void buildResponse(const HttpRequest& request, int statusCode, const std::string& statusPhrase, const std::string& content) {
        std::cout << "ResponseBuilder: Building response...\n";
        statusLine = request.httpVersion + " " + std::to_string(statusCode) + " " + statusPhrase;
        headers["Content-Type"] = "text/html";
        headers["Content-Length"] = std::to_string(content.length());
        headers["Server"] = "MyWebserv/1.0";
        body = content;
    }

    // Converts the built response objects into a single raw HTTP response string.
    std::string toRawResponse() const {
        std::string raw = statusLine + "\r\n";
        for (const auto& header : headers) {
            raw += header.first + ": " + header.second + "\r\n";
        }
        raw += "\r\n"; // Blank line separating headers and body
        raw += body;
        return raw;
    }
};

// --- Main Server Loop (Conceptual) ---
// Orchestrates the interaction between components.
int main() {
    ServerCore server;
    
    // Simulate one request-response cycle
    std::string rawRequest = server.receiveRequest();

    HttpRequest request;
    request.feed(rawRequest); // The request parser processes the raw request data

    Response response;
    if (request.hasError()) {
        response.buildResponse(request, 400, "Bad Request", "<html><body><h1>400 Bad Request</h1></body></html>");
    } else if (request.method == "GET" && request.path == "/index.html") {
        // Simulate fetching content (e.g., from a file system based on root from config)
        std::string fileContent = "<html><body><h1>Hello from My Web Server!</h1><p>This is a simulated index page.</p></body></html>";
        response.buildResponse(request, 200, "OK", fileContent);
    } else {
        // Handle other paths/methods or return 404 (Not Found)
        response.buildResponse(request, 404, "Not Found", "<html><body><h1>404 Not Found</h1></body></html>");
    }

    std::string rawResponse = response.toRawResponse(); // Response builder formats the final response string
    server.sendResponse(rawResponse); // Server core sends the formatted response

    return 0;
}
```

# **Flashcards:**
---

What is the primary function of the Server Core component in an HTTP web server?;;The Server Core handles low-level networking tasks like managing TCP connections, listening for requests, creating/managing sockets, and controlling data flow between the server and clients.

What is the responsibility of the Request Parser in an HTTP web server?;;The Request Parser interprets incoming HTTP requests, extracting key information such as the method, path, headers, and message body, and sets error flags if syntax issues are found.

How does the Request Parser's `feed()` method allow for partial request processing?;;The `feed()` method allows requests to be processed byte by byte, updating the parsing state incrementally, enabling handling of incomplete or streaming requests similar to Nginx and Node.js.

What is the role of the Response Builder in an HTTP web server?;;The Response Builder constructs and formats HTTP responses, including setting the status line (version, code, phrase), adding appropriate headers (e.g., Content-Type, Content-Length), and formatting the message body.

What is a web server's Configuration File used for?;;A Configuration File is a text file that dictates how the web server operates, containing settings like the listening port, host, server name, error page locations, maximum request body size, root directories, and access rules for specific URI locations.

Explain the purpose of CGI in the context of a web server. ;;CGI (Common Gateway Interface) is a standard that allows a web server to execute external programs or scripts (e.g., in Python, Perl) to generate dynamic content or process user input, returning the output as part of the HTTP response.