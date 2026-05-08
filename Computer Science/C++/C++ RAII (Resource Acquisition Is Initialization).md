---
memory: to_finish
tags:
  - learned
language:
  - C++
review-date:
last-reviewed: 2025-10-19
scheda: done
visit-count: 3
confidence-level: 2.5
consecutive-correct: 3
last-struggle-date: ""
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

RAII solves the critical problem of ==resource management in C++==, ==particularly the issue of resource leaks and exception safety==. In C++, resources like memory, file handles, network connections, and mutex locks must be explicitly acquired and released. Manual resource management is error-prone and can lead to memory leaks, dangling pointers, and deadlocks, especially when exceptions occur.

RAII is fundamental to C++ because it provides automatic, deterministic resource management by tying resource lifetime to object lifetime. This eliminates the need for explicit cleanup code, makes programs exception-safe by default, and ensures resources are always properly released regardless of how a scope is exited (normal return, exception, or early return). It's the cornerstone of modern C++ programming and enables writing robust, maintainable code without garbage collection.

# **Core Explanation:**
---

==RAII (Resource Acquisition Is Initialization) is a C++ programming idiom where resource acquisition occurs during object construction and resource release occurs during object destruction.== The core principle is that resources are managed by objects whose lifetimes are controlled by the language's automatic storage duration rules.

**Key Characteristics:**
- **Automatic Resource Management**: Resources are automatically acquired in constructors and released in destructors
- **Exception Safety**: Resources are guaranteed to be released even if exceptions occur
- **Deterministic Cleanup**: Resources are released as soon as objects go out of scope
- **No Manual Cleanup**: Eliminates the need for explicit cleanup code and reduces human error
- **Stack-Based**: Leverages C++'s automatic storage duration and stack unwinding

**How RAII Works:**
1. **Resource Acquisition**: Constructor acquires the resource (memory, file handle, mutex, etc.)
2. **Resource Usage**: The object provides interface to use the resource safely
3. **Automatic Release**: Destructor automatically releases the resource when object goes out of scope
4. **Exception Safety**: Stack unwinding ensures destructors are called even during exceptions

**RAII Principles:**
- ==Every resource should be owned by an object==
- Resources should never be managed manually
- Objects should have clear ownership semantics
- Destructors should never throw exceptions
- Copy/move semantics should be carefully designed for resource-owning classes

# **Related Concepts:**
---

**[[C++ Smart Pointers]]**: Modern C++ smart pointers (unique_ptr, shared_ptr, weak_ptr) are the primary implementation of RAII for memory management, automatically handling allocation and deallocation.

**[[C++ Heap and Stack Allocation]]**: RAII leverages stack-based object lifetime to manage heap-allocated resources, combining the deterministic cleanup of stack allocation with the flexibility of heap allocation.

**Exception Safety**: RAII is essential for exception safety in C++. It ensures that resources are properly cleaned up during stack unwinding when exceptions occur.

**[[Orthodox Canonical Form]]**: RAII classes must carefully manage copy constructor, copy assignment, destructor, and (in C++11+) move constructor and move assignment to prevent resource leaks and double-deletion.

**Scoped Locking**: RAII applied to synchronization primitives like mutexes, where lock acquisition happens in constructor and release happens in destructor.

**Container Classes**: STL containers like vector, string, and map use RAII internally to manage their dynamic memory, providing automatic cleanup.

**Resource Handles**: File handles, network sockets, database connections, and other system resources benefit from RAII wrappers that ensure proper cleanup.

# **Examples:**
---

```cpp
#include <iostream>
#include <memory>
#include <fstream>
#include <mutex>
#include <vector>

// Example 1: Basic RAII class for file handling
class FileManager {
private:
    std::FILE* file;
    std::string filename;

public:
    // Constructor acquires the resource (opens file)
    // This is the "Resource Acquisition Is Initialization" part
    FileManager(const std::string& fname, const char* mode) 
        : filename(fname), file(nullptr) {
        file = std::fopen(fname.c_str(), mode);
        if (!file) {
            throw std::runtime_error("Failed to open file: " + filename);
        }
        std::cout << "File opened: " << filename << std::endl;
    }

    // Destructor automatically releases the resource (closes file)
    // This ensures cleanup happens regardless of how we exit the scope
    ~FileManager() {
        if (file) {
            std::fclose(file);
            std::cout << "File closed: " << filename << std::endl;
        }
    }

    // Delete copy constructor and assignment to prevent double-close
    // This follows the Rule of Five for resource-managing classes
    FileManager(const FileManager&) = delete;
    FileManager& operator=(const FileManager&) = delete;

    // Move constructor transfers ownership
    FileManager(FileManager&& other) noexcept 
        : file(other.file), filename(std::move(other.filename)) {
        other.file = nullptr; // Prevent double-close
    }

    // Move assignment transfers ownership
    FileManager& operator=(FileManager&& other) noexcept {
        if (this != &other) {
            // Close current file if open
            if (file) {
                std::fclose(file);
            }
            // Transfer ownership
            file = other.file;
            filename = std::move(other.filename);
            other.file = nullptr;
        }
        return *this;
    }

    // Safe interface to use the resource
    void write(const std::string& data) {
        if (file) {
            std::fwrite(data.c_str(), 1, data.length(), file);
        }
    }

    bool isOpen() const {
        return file != nullptr;
    }
};

// Example 2: RAII with smart pointers for memory management
class DatabaseConnection {
private:
    std::string connection_string;
    bool connected;

public:
    DatabaseConnection(const std::string& conn_str) 
        : connection_string(conn_str), connected(false) {
        // Simulate database connection
        std::cout << "Connecting to database: " << conn_str << std::endl;
        connected = true;
    }

    ~DatabaseConnection() {
        if (connected) {
            std::cout << "Disconnecting from database" << std::endl;
        }
    }

    // Non-copyable but movable
    DatabaseConnection(const DatabaseConnection&) = delete;
    DatabaseConnection& operator=(const DatabaseConnection&) = delete;
    DatabaseConnection(DatabaseConnection&&) = default;
    DatabaseConnection& operator=(DatabaseConnection&&) = default;

    void query(const std::string& sql) {
        if (connected) {
            std::cout << "Executing query: " << sql << std::endl;
        }
    }
};

// Example 3: RAII for automatic locking (scoped lock)
class ThreadSafeCounter {
private:
    mutable std::mutex mtx;  // mutable allows modification in const methods
    int count;

public:
    ThreadSafeCounter() : count(0) {}

    void increment() {
        // std::lock_guard uses RAII for mutex management
        // Constructor locks the mutex, destructor unlocks it
        // This ensures the mutex is always unlocked, even if exceptions occur
        std::lock_guard<std::mutex> lock(mtx);
        ++count;
        // Mutex is automatically unlocked when lock goes out of scope
    }

    int getValue() const {
        std::lock_guard<std::mutex> lock(mtx);
        return count;
        // Mutex automatically unlocked here
    }
};

// Example 4: Custom RAII wrapper for system resources
class NetworkSocket {
private:
    int socket_fd;
    bool is_open;

public:
    NetworkSocket(int port) : socket_fd(-1), is_open(false) {
        // Simulate socket creation
        socket_fd = port; // Simplified - normally would be actual socket creation
        if (socket_fd > 0) {
            is_open = true;
            std::cout << "Socket opened on port " << port << std::endl;
        } else {
            throw std::runtime_error("Failed to create socket");
        }
    }

    ~NetworkSocket() {
        if (is_open) {
            std::cout << "Socket closed" << std::endl;
            // In real implementation, would call close(socket_fd)
        }
    }

    // Rule of Five implementation
    NetworkSocket(const NetworkSocket&) = delete;
    NetworkSocket& operator=(const NetworkSocket&) = delete;
    
    NetworkSocket(NetworkSocket&& other) noexcept 
        : socket_fd(other.socket_fd), is_open(other.is_open) {
        other.socket_fd = -1;
        other.is_open = false;
    }

    NetworkSocket& operator=(NetworkSocket&& other) noexcept {
        if (this != &other) {
            if (is_open) {
                // Close current socket
                std::cout << "Closing current socket" << std::endl;
            }
            socket_fd = other.socket_fd;
            is_open = other.is_open;
            other.socket_fd = -1;
            other.is_open = false;
        }
        return *this;
    }

    void send(const std::string& data) {
        if (is_open) {
            std::cout << "Sending data: " << data << std::endl;
        }
    }
};

// Example 5: RAII with smart pointers and containers
void demonstrateSmartPointers() {
    std::cout << "\n=== Smart Pointer RAII Demo ===" << std::endl;
    
    // unique_ptr automatically manages memory using RAII
    {
        std::unique_ptr<int> ptr = std::make_unique<int>(42);
        std::cout << "unique_ptr value: " << *ptr << std::endl;
        // Memory is automatically freed when ptr goes out of scope
    }
    
    // shared_ptr with automatic reference counting
    {
        std::shared_ptr<std::vector<int>> vec = 
            std::make_shared<std::vector<int>>(std::initializer_list<int>{1, 2, 3, 4, 5});
        
        std::cout << "shared_ptr use_count: " << vec.use_count() << std::endl;
        
        {
            std::shared_ptr<std::vector<int>> vec2 = vec;
            std::cout << "shared_ptr use_count after copy: " << vec.use_count() << std::endl;
        } // vec2 goes out of scope, use_count decreases
        
        std::cout << "shared_ptr use_count after vec2 destroyed: " << vec.use_count() << std::endl;
        // Memory is automatically freed when last shared_ptr goes out of scope
    }
}

// Example 6: Exception safety with RAII
void demonstrateExceptionSafety() {
    std::cout << "\n=== Exception Safety Demo ===" << std::endl;
    
    try {
        FileManager file("test.txt", "w");
        NetworkSocket socket(8080);
        
        // Simulate some work that might throw
        file.write("Hello, RAII!");
        socket.send("Hello, Network!");
        
        // Simulate an exception
        throw std::runtime_error("Something went wrong!");
        
    } catch (const std::exception& e) {
        std::cout << "Exception caught: " << e.what() << std::endl;
        // Notice that destructors are still called automatically
        // File is closed and socket is closed even though exception occurred
    }
}

int main() {
    std::cout << "=== RAII Demonstration ===" << std::endl;
    
    // Example 1: Basic RAII file management
    {
        std::cout << "\n--- File Management ---" << std::endl;
        FileManager file("example.txt", "w");
        file.write("RAII ensures this file is properly closed!");
        // File automatically closed when 'file' goes out of scope
    }
    
    // Example 2: Database connection
    {
        std::cout << "\n--- Database Connection ---" << std::endl;
        DatabaseConnection db("postgresql://localhost:5432/mydb");
        db.query("SELECT * FROM users");
        // Database automatically disconnected when 'db' goes out of scope
    }
    
    // Example 3: Thread-safe counter
    {
        std::cout << "\n--- Thread-Safe Counter ---" << std::endl;
        ThreadSafeCounter counter;
        counter.increment();
        counter.increment();
        std::cout << "Counter value: " << counter.getValue() << std::endl;
        // Mutex automatically managed by lock_guard
    }
    
    // Example 4: Network socket
    {
        std::cout << "\n--- Network Socket ---" << std::endl;
        NetworkSocket socket(8080);
        socket.send("Hello, RAII World!");
        // Socket automatically closed when 'socket' goes out of scope
    }
    
    // Example 5: Smart pointers
    demonstrateSmartPointers();
    
    // Example 6: Exception safety
    demonstrateExceptionSafety();
    
    std::cout << "\n=== All resources automatically cleaned up! ===" << std::endl;
    
    return 0;
}
````

# **Flashcards:**
---

What does RAII stand for and what is its core principle?;; RAII stands for "Resource Acquisition Is Initialization." Its core principle is that resource acquisition occurs during object construction and resource release occurs during object destruction, tying resource lifetime to object lifetime.

How does RAII provide exception safety in C++?;; RAII provides exception safety through automatic stack unwinding - when an exception occurs, destructors are automatically called for all constructed objects, ensuring resources are properly released even if the normal execution path is interrupted.

What is the Rule of Five and why is it important for RAII classes?;; The Rule of Five states that if a class needs a custom destructor, copy constructor, copy assignment operator, move constructor, or move assignment operator, it likely needs all five. This ensures proper resource management and prevents issues like double-deletion or resource leaks.

How do smart pointers implement RAII for memory management?;; Smart pointers like unique_ptr and shared_ptr automatically manage memory allocation and deallocation. They acquire memory in their constructors and release it in their destructors, eliminating the need for manual new/delete calls and preventing memory leaks.

What is the difference between RAII and garbage collection?;; RAII provides deterministic resource cleanup at the exact moment objects go out of scope, while garbage collection provides non-deterministic cleanup at unpredictable times. RAII works for any resource type, while garbage collection typically only handles memory.

How does std::lock_guard demonstrate RAII principles?;; std::lock_guard acquires a mutex lock in its constructor and releases it in its destructor. This ensures the mutex is always unlocked when the guard goes out of scope, preventing deadlocks and ensuring proper synchronization even if exceptions occur.