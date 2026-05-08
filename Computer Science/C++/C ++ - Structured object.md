---
memory: to_finish
tags:
  - learned
language:
  - C++
review-date:
last-reviewed: 2025-09-23
scheda: done
cssclasses:
visit-count: 3
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-09-10
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

Structured objects in C++ solve the fundamental problem of ==organizing related data and functionality into cohesive units==. They provide a way to model real-world entities by grouping attributes (data members) and behaviors (member functions) together, enabling data encapsulation, code reusability, and logical organization. This concept is crucial because it forms the foundation of object-oriented programming, allowing developers to create maintainable, scalable software by breaking complex systems into manageable, self-contained components that can interact with each other through well-defined interfaces.

# **Core Explanation:**
---

A structured object in C++ is a user-defined data type that combines data members (variables) and member functions (methods) into a single unit. It can be implemented using either `struct` or `class` keywords, with the primary difference being default access levels (public for struct, private for class).

Key characteristics include:
- **Encapsulation**: Data and functions are bundled together, controlling access through access specifiers (public, private, protected)
- **Data Members**: Variables that store the object's state
- **Member Functions**: Functions that operate on the object's data and define its behavior
- **Constructors**: Special functions that initialize objects when created
- **Destructors**: Special functions that clean up resources when objects are destroyed
- **Access Control**: Mechanisms to hide internal implementation details and provide controlled interfaces

The object works by creating instances that contain their own copy of data members while sharing member functions. Memory is allocated for each object instance, and member functions can access and modify the object's data through the implicit `this` pointer.

# **Related Concepts:**
---

- **Classes vs Structs**: Both create structured objects, but classes default to private access while structs default to public access
- **[[C++ Encapsulation]]**: The principle of bundling data and methods together while controlling access to internal details
- **Abstraction**: Hiding implementation complexity behind simple interfaces, closely related to structured objects
- **Inheritance**: Allows structured objects to derive properties and behaviors from parent objects
- **[[Polymorphism]]**: Enables structured objects to take multiple forms through virtual functions and overriding
- **Composition**: Building complex objects by combining simpler structured objects
- **[[C ++ Constructors]]/Destructors**: Special member functions that manage object lifecycle and resource management
- **[[Access Specifiers]]**: Control visibility and accessibility of object members (public, private, protected)

# **Examples:**
---

```cpp
#include <iostream>
#include <string>

// Basic structured object using struct (default public access)
struct Point {
    // Data members - store the object's state
    double x, y;
    
    // Default constructor - initializes object with default values
    Point() : x(0.0), y(0.0) {}
    
    // Parameterized constructor - initializes with specific values
    Point(double x_val, double y_val) : x(x_val), y(y_val) {}
    
    // Member function - defines behavior/operation on the object
    double distance_from_origin() const {
        return sqrt(x * x + y * y);
    }
    
    // Member function - modifies object state
    void move(double dx, double dy) {
        x += dx;
        y += dy;
    }
    
    // Member function - displays object state
    void display() const {
        std::cout << "Point(" << x << ", " << y << ")" << std::endl;
    }
};

// More complex structured object using class (default private access)
class BankAccount {
private:
    // Private data members - encapsulated, not directly accessible
    std::string account_number;
    std::string owner_name;
    double balance;
    
public:
    // Constructor - initializes account with required information
    BankAccount(const std::string& acc_num, const std::string& name, double initial_balance = 0.0)
        : account_number(acc_num), owner_name(name), balance(initial_balance) {
        // Validation could be added here
        if (initial_balance < 0) {
            balance = 0.0;
        }
    }
    
    // Public interface methods - controlled access to private data
    void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
            std::cout << "Deposited $" << amount << std::endl;
        }
    }
    
    bool withdraw(double amount) {
        if (amount > 0 && amount <= balance) {
            balance -= amount;
            std::cout << "Withdrew $" << amount << std::endl;
            return true;
        }
        std::cout << "Insufficient funds or invalid amount" << std::endl;
        return false;
    }
    
    // Accessor method - provides read-only access to private data
    double get_balance() const {
        return balance;
    }
    
    // Accessor method - provides read-only access to private data
    std::string get_owner() const {
        return owner_name;
    }
    
    // Display method - shows account information
    void display_account_info() const {
        std::cout << "Account: " << account_number 
                  << ", Owner: " << owner_name 
                  << ", Balance: $" << balance << std::endl;
    }
};

// Demonstration of structured objects in action
int main() {
    // Creating and using Point objects
    Point p1;  // Uses default constructor
    Point p2(3.0, 4.0);  // Uses parameterized constructor
    
    std::cout << "=== Point Objects ===" << std::endl;
    p1.display();  // Shows default values
    p2.display();  // Shows initialized values
    
    // Using member functions to interact with objects
    p1.move(1.0, 2.0);  // Modifies object state
    p1.display();  // Shows updated state
    
    std::cout << "Distance from origin: " << p2.distance_from_origin() << std::endl;
    
    // Creating and using BankAccount objects
    std::cout << "\n=== Bank Account Objects ===" << std::endl;
    BankAccount account1("12345", "Alice Johnson", 1000.0);
    BankAccount account2("67890", "Bob Smith");  // Uses default balance
    
    // Demonstrating encapsulation - can only access through public methods
    account1.display_account_info();
    account2.display_account_info();
    
    // Using public interface to interact with private data
    account1.deposit(500.0);
    account1.withdraw(200.0);
    account1.display_account_info();
    
    // Attempting invalid operations
    account1.withdraw(2000.0);  // Should fail due to insufficient funds
    
    return 0;
}
````

# **Flashcards:**

---

What is a structured object in C++?;; A user-defined data type that combines data members (variables) and member functions (methods) into a single unit, enabling encapsulation and organization of related data and functionality.

What is the main difference between struct and class in C++?;; The default access level: struct members are public by default, while class members are private by default. Both can create structured objects with the same capabilities.

What are the three main access specifiers in C++ structured objects?;; Public (accessible from anywhere), Private (accessible only within the same class), and Protected (accessible within the same class and derived classes).

What is encapsulation in the context of structured objects?;; The principle of bundling data and methods together while controlling access to internal details through access specifiers, hiding implementation from external code.

What are constructors and destructors in structured objects?;; Constructors are special member functions that initialize objects when they are created, while destructors are special functions that clean up resources when objects are destroyed.

How do member functions access object data in C++?;; Member functions can access and modify the object's data members through the implicit 'this' pointer, which refers to the current object instance on which the function is called.