---
memory: to_finish
tags:
  - to_learn
language:
  - C++
review-date: 2025-11-20
last-reviewed: 2025-10-22
scheda: done
visit-count: 3
confidence-level: 1
consecutive-correct: 0
last-struggle-date: 2025-10-22
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
# Purpose/Why:
---

  The SOLID principles are a set of five design guidelines for object-oriented programming intended to <mark style="background: #ABF7F7A6;">make software designs more understandable, flexible, and maintainable</mark>. As <mark style="background: #ABF7F7A6;">applications grow in complexity, without a good design foundation, they become
  rigid, fragile, and difficult to change</mark>. SOLID principles address this by reducing dependencies and coupling, so that developers can modify one part of the software without impacting others. In C++, a language that offers powerful object-oriented features, applying SOLID is crucial for building robust, scalable, and resilient systems that can evolve over time.

# Core Explanation:
---

 S.O.L.I.D. is an acronym for five principles:

   1. ==S - Single Responsibility Principle (SRP)==: A <mark style="background: #FF5582A6;">class should have only one reason to change</mark>. This means a <mark style="background: #FF5582A6;">class should only have one job or responsibility</mark>. When a class has multiple responsibilities, changes to one might inadvertently affect the others.


   2. ==O - [[Open⧸Closed Principle]]:== Software entities (classes, modules, functions, etc.) should be <mark style="background: #FF5582A6;">open for extension but closed for modification</mark>. This means you should be able to add new functionality without changing existing code. <mark style="background: #FF5582A6;">This is often achieved through interfaces, abstract classes, and polymorphism.</mark>


   3. ==L - Liskov Substitution Principle (LSP)==: <mark style="background: #ABF7F7A6;">Subtypes must be substitutable for their base types</mark> without altering the correctness of the program. If you have a class Derived that inherits from Base, you <mark style="background: #ABF7F7A6;">should be able to use an object of Derived wherever an object of Base is expected, without causing issues.</mark>


   4. ==I - Interface Segregation Principle (ISP)==: No client should be forced to depend on methods it does not use. This principle suggests that large, monolithic interfaces should be split into smaller, more specific ones, so that clients only need to know about the methods that are relevant to them.


   5. ==D - Dependency Inversion Principle (DIP)==: High-level modules should not depend on low-level modules. Both should depend on abstractions (e.g., interfaces). Furthermore, abstractions should not depend on details. Details should depend on abstractions. This decouples modules, allowing for more flexibility and easier testing.

# Related Concepts:
---

   * Design Patterns: SOLID principles are not design patterns themselves, but high-level guidelines. Many design patterns (like Strategy, Factory, Observer) are concrete implementations of one or more SOLID principles. For example, the Strategy pattern is a classic implementation of the Open/Closed Principle.
   * GRASP (General Responsibility Assignment Software Patterns): These are another set of principles for assigning responsibilities to objects. Concepts like "High Cohesion" and "Low Coupling" from GRASP are the goals that SOLID principles help achieve.
   * DRY (Don't Repeat Yourself): This principle is about reducing repetition of information. While SOLID focuses on class design and dependencies, DRY focuses on avoiding duplicate code. They are complementary.
   * KISS (Keep It Simple, Stupid): This principle suggests that systems work best if they are kept simple rather than made complicated. SOLID principles can help achieve simplicity by breaking down complex systems into smaller, more manageable parts.

  
# Examples:
---

```cpp
// SOLID Principles in C++

// --- Single Responsibility Principle (SRP) ---
// A class should have only one reason to change.

class Journal {
    std::string title;
    std::vector<std::string> entries;
public:
    explicit Journal(const std::string& title) : title{title} {}
    
    void add_entry(const std::string& entry) {
        entries.push_back(entry);
    }
    // If we added a save() method here, the Journal class would have two
    // responsibilities: managing entries and persistence. This violates SRP.
};

struct PersistenceManager {
    // This class has a single responsibility: saving data.
    // It is decoupled from the Journal's entry management logic.
    static void save(const Journal& j, const std::string& filename) {
        // In a real app, this would write to a file.
        std::cout << "Saving journal '" << j.title << "' to " << filename << std::endl;
    }
};


// --- Open/Closed Principle (OCP) ---
// Open for extension, closed for modification.
// We can add new filter specifications without modifying the Filter class.

// The Specification interface defines a contract for checking if an item meets a criteria.
// This is the "abstraction" that allows for extension.
template <typename T> struct Specification {
    virtual bool is_satisfied(T* item) = 0;
};

// The Filter interface is also an abstraction. It is "closed" for modification.
// Its behavior can be extended by giving it new Specification objects.
template <typename T> struct Filter {
    virtual std::vector<T*> filter(std::vector<T*> items, Specification<T>& spec) = 0;
};

struct Product {
    std::string name;
    int color;
};

// We can create new specifications to extend filtering capabilities
// without ever touching the BetterFilter or Specification code.
struct ColorSpecification : Specification<Product> {
    int color;
    explicit ColorSpecification(const int color) : color{color} {}
    bool is_satisfied(Product* item) override {
        return item->color == color;
    }
};

// This filter implementation is "open" to new specifications.
struct BetterFilter : Filter<Product> {
    std::vector<Product*> filter(std::vector<Product*> items, Specification<Product>& spec) override {
        std::vector<Product*> result;
        for (auto& item : items) {
            if (spec.is_satisfied(item)) {
                result.push_back(item);
            }
        }
        return result;
    }
};


// --- Liskov Substitution Principle (LSP) ---
// Subtypes must be substitutable for their base types.
class Rectangle {
protected:
    int width, height;
public:
    Rectangle(int width, int height) : width{width}, height{height} {}
    virtual void set_width(int w) { width = w; }
    virtual void set_height(int h) { height = h; }
    int get_area() const { return width * height; }
};

class Square : public Rectangle {
public:
    Square(int size) : Rectangle(size, size) {}
    // This violates LSP. A Square must maintain its squareness (width == height).
    // Overriding the setters changes the behavior in a way a user of Rectangle would not expect.
    // A function that takes a Rectangle and sets width and height independently will fail with a Square.
    void set_width(int w) override { this->width = this->height = w; }
    void set_height(int h) override { this->width = this->height = h; }
};

// This function demonstrates the LSP violation.
// It works for Rectangle, but produces an unexpected result for Square.
void process(Rectangle& r) {
    int w = 5;
    r.set_width(w);
    r.set_height(10);
    // For a Rectangle, we expect the area to be w * 10 = 50.
    // For a Square, because set_height also sets the width, the area becomes 10 * 10 = 100.
    // This breaks the program's logic, violating LSP.
    std::cout << "Expected area: " << w * 10 << ", Got: " << r.get_area() << std::endl;
}


// --- Interface Segregation Principle (ISP) ---
// Clients should not be forced to depend on interfaces they do not use.

// Monolithic interface - BAD
struct IMachine {
    virtual void print(const std::string& doc) = 0;
    virtual void scan(const std::string& doc) = 0;
    virtual void fax(const std::string& doc) = 0;
};

// A simple printer is forced to implement scan and fax, which it cannot do.
struct SimplePrinter : IMachine {
    void print(const std::string& doc) override { std::cout << "Printing " << doc << std::endl; }
    void scan(const std::string& doc) override { /* Do nothing or throw exception */ }
    void fax(const std::string& doc) override { /* Do nothing or throw exception */ }
};

// Segregated interfaces - GOOD
struct IPrinter {
    virtual void print(const std::string& doc) = 0;
};
struct IScanner {
    virtual void scan(const std::string& doc) = 0;
};

// Now, a simple printer only needs to implement the interface it actually uses.
struct JustAPrinter : IPrinter {
    void print(const std::string& doc) override { std::cout << "Printing " << doc << std::endl; }
};

// A multifunction device can implement multiple interfaces.
struct Photocopier : IPrinter, IScanner {
    void print(const std::string& doc) override { /* ... */ }
    void scan(const std::string& doc) override { /* ... */ }
};


// --- Dependency Inversion Principle (DIP) ---
// High-level modules should not depend on low-level modules. Both should depend on abstractions.

// This is the abstraction both high-level and low-level modules will depend on.
struct IMessageSender {
    virtual void send(const std::string& message) = 0;
};

// This is a low-level module (a concrete implementation detail).
struct EmailSender : IMessageSender {
    void send(const std::string& message) override {
        std::cout << "Sending email: " << message << std::endl;
    }
};

// This is the high-level module.
// It does not depend on the concrete EmailSender, but on the IMessageSender abstraction.
// This allows us to change the sending mechanism (e.g., to SMS) without changing NotificationService.
class NotificationService {
    std::unique_ptr<IMessageSender> sender;
public:
    // The dependency (the sender) is injected via the constructor.
    NotificationService(std::unique_ptr<IMessageSender> s) : sender{std::move(s)} {}

    void send_notification(const std::string& message) {
        sender->send(message);
    }
};

int main() {
    // Example usage of DIP
    auto email_sender = std::make_unique<EmailSender>();
    NotificationService notifier(std::move(email_sender));
    notifier.send_notification("Hello, SOLID!");
    
    // Example usage of SRP
    Journal journal("My Journal");
    journal.add_entry("I learned about SOLID principles today");
    PersistenceManager::save(journal, "journal.txt");
    
    // Example usage of OCP
    std::vector<Product*> products = {
        new Product{"Apple", 1}, 
        new Product{"Banana", 2},
        new Product{"Orange", 1}
    };
    ColorSpecification red_spec(1);
    BetterFilter filter;
    auto red_products = filter.filter(products, red_spec);
    for (auto& p : red_products) {
        std::cout << "Red product: " << p->name << std::endl;
    }
    
    // Example usage of LSP
    Rectangle rect(5, 5);
    process(rect); // Works as expected
    Square square(5);
    process(square); // Demonstrates LSP violation
    
    // Cleanup
    for (auto p : products) delete p;
    
    return 0;
}
```


# Flashcards:
---

What does the 'S' in SOLID stand for?;; Single Responsibility Principle: A class should have only one reason to change, meaning it should have only one job.

What does the 'O' in SOLID stand for?;; Open/Closed Principle: Software entities should be open for extension, but closed for modification.

What does the 'L' in SOLID stand for?;; Liskov Substitution Principle: Subtypes must be substitutable for their base types without altering the correctness of the program.

What does the 'I' in SOLID stand for?;; Interface Segregation Principle: No client should be forced to depend on methods it does not use. Prefer smaller, specific interfaces over large, monolithic ones.

What does the 'D' in SOLID stand for?;; Dependency Inversion Principle: High-level modules should not depend on low-level modules. Both should depend on abstractions.

Why are SOLID principles important in C++?;; They help create maintainable, scalable, and flexible object-oriented code by reducing coupling and increasing cohesion, making software easier to change and understand.
