---
memory: to_finish
tags:
  - mastered
language:
  - C++
review-date: ""
last-reviewed: 2025-09-06
scheda: done
visit-count: 2
confidence-level: 2
consecutive-correct: 2
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
# **Core Explanation:**

Encapsulation is one of the fundamental principles of object-oriented programming (OOP). It refers to the bundling of data (attributes or member variables) and the methods (functions) that operate on that data into a single unit, called a class. Encapsulation also involves hiding the internal implementation details of an object and exposing only a controlled interface to interact with it.

**Extensive Explanation:**

Think of encapsulation as a protective capsule around an object's internal workings. This capsule prevents direct, uncontrolled access to the data and forces interaction through well-defined methods. This control is crucial for maintaining the integrity of the object's state and for managing complexity in larger software systems.

**Key Aspects of Encapsulation:**

* **Bundling Data and Methods:**  The primary aspect of encapsulation is combining the data that represents the state of an object with the methods that define its behavior. These data and methods are grouped together within a class. This association makes the code more organized and easier to understand, as the data and the operations that manipulate it are logically linked.

* **Data Hiding (Information Hiding):**  Encapsulation often involves hiding the internal representation of the data from the outside world. This is achieved through access modifiers like `private` and `protected` in C++.
    * **`private` members:**  These members (data and methods) are only accessible from within the class itself. External code cannot directly access or modify private members. This ensures that the internal state of the object can only be changed by the methods defined within the class, providing a controlled mechanism for modification.
    * **[[protected members]]:** These members are accessible from within the class itself and from its derived classes (in the context of [[C++ - Inheritance]]). They are not accessible from external code. This level of access is useful when designing class hierarchies, allowing subclasses to interact with certain internal aspects of their parent class without exposing them to the general public.
    * **`public` members:** These members are accessible from anywhere. They form the public interface of the class, defining how external code can interact with objects of that class.

* **Controlled Access (Interface):**  While data hiding restricts direct access, encapsulation provides controlled access through `public` methods. These methods act as intermediaries, allowing external code to interact with the object's data in a safe and predictable manner.
    * **[[Getter Methods]] (Accessors):** Public methods that provide read-only access to the object's private data. They allow external code to retrieve the values of internal attributes without allowing direct modification. The naming convention often follows `getVariableName()` or simply `variableName()` for boolean attributes (e.g., `isActive()`).
    * **[[Setter Methods]] (Mutators):** Public methods that provide controlled write access to the object's private data. They allow external code to modify the internal state of the object, but typically include logic to validate the input and ensure that the object remains in a consistent and valid state. The naming convention often follows `setVariableName(newValue)`.
    Getter and setter methods are fundamental patterns within the concept of encapsulation in object-oriented programming. They provide controlled access to an object's private data members, enabling interaction with the object's state while maintaining data integrity and flexibility.

**Benefits of Encapsulation:**

* **Data Protection:**  Encapsulation protects the internal state of an object from being accidentally or intentionally corrupted by external code. By restricting direct access to data and forcing interaction through methods, you can enforce rules and constraints on how the data can be modified, ensuring data integrity.

* **Modularity and Maintainability:** Encapsulation promotes modularity by breaking down a system into self-contained units (objects). Changes to the internal implementation of a class do not necessarily affect other parts of the system, as long as the public interface remains the same. This makes code easier to maintain, debug, and modify.

* **Abstraction:** Encapsulation helps achieve abstraction by hiding the complex internal workings of an object and presenting a simplified interface to the user. The user of an object only needs to know how to use its public methods, without needing to understand the intricate details of its implementation. This reduces cognitive load and simplifies the use of complex objects.

* **Flexibility and Evolution:**  Because the internal implementation is hidden, you can change the way a class works internally without breaking code that uses the class, as long as the public interface remains consistent. This allows for greater flexibility and makes it easier to evolve the software over time.

* **Code Reusability:** Well-encapsulated classes are more likely to be reusable in different parts of the application or in other projects, as their dependencies are minimized and their behavior is well-defined through their interface.    

**In Summary:**

Encapsulation is a cornerstone of OOP in C++. By bundling data and methods and controlling access through well-defined interfaces, encapsulation leads to more robust, maintainable, and understandable code. It promotes data integrity, modularity, and abstraction, making it a crucial concept for building complex software systems.

# **Examples:**

**Achieving Encapsulation in C++:**

Encapsulation is primarily achieved in C++ through the use of **classes** and **access modifiers**.

```cpp
#include <iostream>
#include <string>

class BankAccount {
private:
    std::string accountNumber; // Private data member
    double balance;           // Private data member

public:
    // Constructor
    BankAccount(std::string accNum, double initialBalance) : accountNumber(accNum), balance(initialBalance) {}

    // Getter methods (accessors)
    std::string getAccountNumber() const {
        return accountNumber;
    }

    double getBalance() const {
        return balance;
    }

    // Setter methods (mutators) - with validation
    void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
            std::cout << "Deposit of " << amount << " successful. New balance: " << balance << std::endl;
        } else {
            std::cout << "Invalid deposit amount." << std::endl;
        }
    }

    void withdraw(double amount) {
        if (amount > 0 && balance >= amount) {
            balance -= amount;
            std::cout << "Withdrawal of " << amount << " successful. New balance: " << balance << std::endl;
        } else {
            std::cout << "Insufficient funds or invalid withdrawal amount." << std::endl;
        }
    }

    // Public method to display account information
    void displayAccountInfo() const {
        std::cout << "Account Number: " << accountNumber << std::endl;
        std::cout << "Balance: $" << balance << std::endl;
    }
};

int main() {
    BankAccount myAccount("1234567890", 1000.0);

    // Accessing data through public getter methods
    std::cout << "Account number: " << myAccount.getAccountNumber() << std::endl;
    std::cout << "Current balance: $" << myAccount.getBalance() << std::endl;

    // Modifying data through public setter methods (with validation)
    myAccount.deposit(500.0);
    myAccount.withdraw(200.0);
    myAccount.withdraw(2000.0); // Attempt to withdraw more than the balance

    // Displaying account information
    myAccount.displayAccountInfo();

    // Attempting to access private members directly (will result in a compilation error)
    // std::cout << myAccount.balance; // Error! 'balance' is a private member

    return 0;
}```

**Explanation of the Example:**

- The BankAccount class encapsulates the accountNumber and balance (data) along with the deposit, withdraw, and displayAccountInfo methods (behavior).
    
- accountNumber and balance are declared as private, meaning they can only be accessed and modified from within the BankAccount class itself.
    
- Public methods like getBalance(), deposit(), and withdraw() provide controlled access to the account's data.
    
- The deposit() and withdraw() methods include validation logic to ensure the balance remains consistent and valid.
    
- The main() function interacts with the BankAccount object only through its public interface, demonstrating how encapsulation protects the internal state.