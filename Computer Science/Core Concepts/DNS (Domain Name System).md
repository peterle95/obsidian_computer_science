---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-10-03
scheda: done
visit-count: 3
confidence-level: 2.5
consecutive-correct: 3
last-struggle-date: ""
cssclasses:
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

DNS solves the fundamental problem of translating human-readable domain names into machine-readable IP addresses. Without DNS, users would need to memorize complex numerical IP addresses <mark style="background: #FF5582A6;">(like 192.168.1.1 or 2001:db8::1)</mark> to access websites and services. DNS acts as the internet's phonebook, enabling seamless communication between devices across networks by providing a hierarchical, distributed naming system that scales globally.

This system is crucial because it enables the internet to function as we know it today - allowing billions of devices to locate and communicate with each other using memorable names instead of numerical addresses. It's essential for web browsing, email, file sharing, and virtually all internet-based applications.

# **Core Explanation:**
---

DNS (Domain Name System) is a ==hierarchical, distributed database system that translates domain names into IP addresses.== It operates on a client-server model where DNS resolvers query DNS servers to resolve names.

**Key Characteristics:**
- **Hierarchical Structure**: ==Organized in a tree-like structure with root servers at the top==, followed by top-level domains (TLDs), second-level domains, and subdomains
- **Distributed**: ==No single point of failure==; DNS data is distributed across millions of servers worldwide
- **Cached**: ==Responses are cached at multiple levels to improve performance and reduce load==
- **Recursive Resolution**: DNS resolvers can query multiple servers to resolve a complete domain name

**How DNS Works:**
1. **Query Initiation**: User types a domain name (e.g., example.com)
2. **Local Cache Check**: DNS resolver checks its cache first
3. **Root Server Query**: If not cached, resolver queries root name servers
4. **TLD Server Query**: Root servers direct to appropriate TLD servers (.com, .org, etc.)
5. **Authoritative Server Query**: TLD servers point to authoritative name servers for the domain
6. **Response Return**: Authoritative servers provide the IP address, which is returned to the client

**DNS Record Types:**
- **A Record**: Maps domain to IPv4 address
- **AAAA Record**: Maps domain to IPv6 address
- **CNAME Record**: Creates aliases for domain names
- **MX Record**: Specifies mail servers for email delivery
- **TXT Record**: Stores text information for various purposes
- **NS Record**: Identifies authoritative name servers for a domain

# **Related Concepts:**
---

**IP Addressing**: DNS translates domain names to IP addresses. While IP addresses are the actual network identifiers, DNS provides the human-friendly layer on top.

**DHCP (Dynamic Host Configuration Protocol)**: Often works alongside DNS to automatically assign IP addresses and DNS server configurations to devices joining a network.

**Load Balancing**: DNS can distribute traffic across multiple servers by returning different IP addresses for the same domain name, providing basic load distribution.

**CDN (Content Delivery Networks)**: Leverage DNS to direct users to the nearest server location, using techniques like GeoDNS to optimize content delivery.

**HTTP/HTTPS**: Web protocols that depend on DNS resolution to establish connections to web servers before data transfer can begin.

**Certificate Authorities**: Work with DNS for domain validation in SSL/TLS certificate issuance, ensuring secure connections.

**Reverse DNS**: The opposite of forward DNS - translates IP addresses back to domain names, commonly used for email server verification and network troubleshooting.

# **Examples:**
---

```python
import socket
import dns.resolver

# Example 1: Basic DNS resolution using Python's socket library
def basic_dns_lookup(domain):
    """
    Performs a basic DNS lookup using Python's built-in socket library
    This demonstrates the fundamental DNS resolution process
    """
    try:
        # socket.gethostbyname() performs DNS resolution internally
        # It queries DNS servers to get the IP address for the domain
        ip_address = socket.gethostbyname(domain)
        print(f"IP address for {domain}: {ip_address}")
        return ip_address
    except socket.gaierror as e:
        # This exception occurs when DNS resolution fails
        print(f"DNS resolution failed for {domain}: {e}")
        return None

# Example 2: Advanced DNS queries using dnspython library
def advanced_dns_queries(domain):
    """
    Demonstrates different types of DNS record queries
    Shows how DNS stores various types of information beyond just IP addresses
    """
    resolver = dns.resolver.Resolver()
    
    # Query A record (IPv4 address)
    try:
        a_records = resolver.resolve(domain, 'A')
        print(f"A Records for {domain}:")
        for record in a_records:
            print(f"  {record}")
    except dns.resolver.NXDOMAIN:
        print(f"No A records found for {domain}")
    
    # Query MX record (Mail Exchange - email servers)
    try:
        mx_records = resolver.resolve(domain, 'MX')
        print(f"MX Records for {domain}:")
        for record in mx_records:
            # MX records include priority (lower number = higher priority)
            print(f"  Priority: {record.preference}, Mail Server: {record.exchange}")
    except dns.resolver.NXDOMAIN:
        print(f"No MX records found for {domain}")
    
    # Query TXT records (text information, often used for verification)
    try:
        txt_records = resolver.resolve(domain, 'TXT')
        print(f"TXT Records for {domain}:")
        for record in txt_records:
            print(f"  {record}")
    except dns.resolver.NXDOMAIN:
        print(f"No TXT records found for {domain}")

# Example 3: DNS cache demonstration
def demonstrate_dns_caching():
    """
    Shows how DNS caching works to improve performance
    First query takes longer, subsequent queries are faster due to caching
    """
    import time
    
    domain = "google.com"
    
    # First query - will contact DNS servers
    start_time = time.time()
    ip1 = socket.gethostbyname(domain)
    first_query_time = time.time() - start_time
    
    # Second query - should be faster due to caching
    start_time = time.time()
    ip2 = socket.gethostbyname(domain)
    second_query_time = time.time() - start_time
    
    print(f"First DNS query for {domain}: {first_query_time:.4f} seconds")
    print(f"Second DNS query for {domain}: {second_query_time:.4f} seconds")
    print(f"Caching effect: {((first_query_time - second_query_time) / first_query_time * 100):.1f}% faster")

# Example 4: Custom DNS server configuration
def use_custom_dns_server(domain, dns_server='8.8.8.8'):
    """
    Demonstrates how to use a specific DNS server instead of system default
    Useful for testing or when you want to use a particular DNS provider
    """
    resolver = dns.resolver.Resolver()
    # Configure resolver to use Google's public DNS server
    resolver.nameservers = [dns_server]
    
    try:
        answers = resolver.resolve(domain, 'A')
        print(f"Using DNS server {dns_server} for {domain}:")
        for answer in answers:
            print(f"  {answer}")
    except Exception as e:
        print(f"Error using DNS server {dns_server}: {e}")

# Running the examples
if __name__ == "__main__":
    # Test basic DNS functionality
    basic_dns_lookup("google.com")
    
    # Show different types of DNS records
    advanced_dns_queries("google.com")
    
    # Demonstrate DNS caching
    demonstrate_dns_caching()
    
    # Use custom DNS server
    use_custom_dns_server("cloudflare.com", "1.1.1.1")
````

# **Flashcards:**
---


What is the primary purpose of DNS?;; DNS translates human-readable domain names (like google.com) into machine-readable IP addresses (like 172.217.16.14), acting as the internet's phonebook.

What are the main components of DNS hierarchy from top to bottom?;; Root servers → Top-Level Domain (TLD) servers → Authoritative name servers → Local DNS resolvers/caches.

What is the difference between an A record and a CNAME record?;; An A record maps a domain name directly to an IPv4 address, while a CNAME record creates an alias that points to another domain name.

How does DNS caching improve performance?;; DNS caching stores previously resolved queries locally, reducing the need to contact DNS servers repeatedly and significantly speeding up subsequent requests for the same domain.

What is the purpose of MX records in DNS?;; MX (Mail Exchange) records specify which mail servers are responsible for handling email for a domain, including priority levels for multiple mail servers.

What happens during a recursive DNS query?;; The DNS resolver queries multiple servers in sequence: first root servers, then TLD servers, then authoritative servers, until it gets the final IP address to return to the client.