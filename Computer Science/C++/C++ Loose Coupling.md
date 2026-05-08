---
memory: to_finish
tags:
  - will_learn
language:
  - C++
review-date:
last-reviewed: ""
scheda: done
visit-count: 0
confidence-level: 1
consecutive-correct: 0
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

Loose coupling solves the fundamental problem of excessive dependencies between software components, which leads to rigid, fragile, and hard-to-maintain code. In tightly coupled systems, changes in one component cascade through the entire system, making modifications risky and expensive. Loose coupling promotes modularity by minimizing dependencies between classes and components, making them more independent and interchangeable. This is crucial in C++ because it enables better testability, reusability, maintainability, and parallel development. It supports the Open/Closed Principle (open for extension, closed for modification) and makes large-scale refactoring feasible. Loose coupling is essential for building scalable enterprise applications, game engines, and any system where components need to evolve independently.

# **Core Explanation:**
---
Loose coupling is a design principle where components (classes, modules, functions) have minimal dependencies on each other's internal implementation details. Instead of directly depending on concrete implementations, components depend on abstractions (interfaces, abstract base classes, or protocols).

**Key characteristics:**
- **Abstraction-based dependencies**: Components depend on interfaces, not implementations
- **Minimal knowledge**: Each component knows only what it needs to fulfill its responsibilities
- **Interchangeability**: Components can be replaced without affecting others
- **Independent evolution**: Components can be modified, extended, or replaced independently
- **Testability**: Components can be easily mocked or stubbed for unit testing
- **Reduced compilation dependencies**: Changes don't trigger widespread recompilation

**How it works:**
Loose coupling is achieved through techniques like dependency injection, interface segregation, abstract base classes, event-driven communication, and the use of design patterns (Strategy, Observer, Factory). Instead of creating dependencies directly (`new ConcreteClass()`), components receive their dependencies through constructors, setters, or factory methods. This inversion of control makes the system more flexible and modular.

# **Related Concepts:**
---

- **Tight Coupling**: The opposite of loose coupling, where components are highly dependent on each other's implementation details. Results in brittle, hard-to-maintain code.
- **Dependency Injection**: A technique to achieve loose coupling by providing dependencies from external sources rather than creating them internally.
- **Inversion of Control (IoC)**: Design principle where control flow is inverted - instead of objects controlling their dependencies, dependencies are controlled externally.
- **Interface Segregation Principle**: SOLID principle stating that clients shouldn't depend on interfaces they don't use. Supports loose coupling by creating focused interfaces.
- **Abstract Base Classes**: C++ mechanism to define interfaces and common behavior, enabling polymorphism and loose coupling.
- **Strategy Pattern**: Design pattern that enables loose coupling by encapsulating algorithms and making them interchangeable.
- **Observer Pattern**: Enables loose coupling between subjects and observers through event-driven communication.
- **Cohesion**: Measures how closely related elements within a module are. High cohesion complements loose coupling for good design.

# **Examples:**
---

```cpp
#include <iostream>
#include <memory>
#include <vector>
#include <string>

// Example 1: Tightly Coupled Design (BAD)
namespace TightlyCoupled {
    
    // Concrete email service - hard to test and replace
    class EmailService {
    public:
        void sendEmail(const std::string& recipient, const std::string& message) {
            std::cout << "Sending email to " << recipient << ": " << message << std::endl;
            // Actual email sending logic would go here
        }
    };
    
    // Concrete SMS service - another hard dependency
    class SMSService {
    public:
        void sendSMS(const std::string& phone, const std::string& message) {
            std::cout << "Sending SMS to " << phone << ": " << message << std::endl;
            // Actual SMS sending logic would go here
        }
    };
    
    // Notification manager with tight coupling - BAD DESIGN
    class NotificationManager {
    private:
        EmailService emailService;  // Direct dependency - tightly coupled
        SMSService smsService;      // Direct dependency - tightly coupled
        
    public:
        void sendNotification(const std::string& recipient, const std::string& message, bool useEmail) {
            if (useEmail) {
                // Directly uses concrete implementation
                emailService.sendEmail(recipient, message);
            } else {
                // Directly uses concrete implementation
                smsService.sendSMS(recipient, message);
            }
        }
    };
    
    // Problems with this design:
    // 1. Hard to test - can't mock email/SMS services
    // 2. Hard to extend - adding new notification types requires modifying NotificationManager
    // 3. High coupling - NotificationManager depends on concrete implementations
    // 4. Compilation dependencies - changes to EmailService require recompiling NotificationManager
}

// Example 2: Loosely Coupled Design (GOOD)
namespace LooselyCoupled {
    
    // Abstract interface for notification services
    class INotificationService {
    public:
        virtual ~INotificationService() = default;
        virtual void sendNotification(const std::string& recipient, const std::string& message) = 0;
        virtual std::string getServiceType() const = 0;
    };
    
    // Concrete email service implementing the interface
    class EmailService : public INotificationService {
    public:
        void sendNotification(const std::string& recipient, const std::string& message) override {
            std::cout << "Email to " << recipient << ": " << message << std::endl;
        }
        
        std::string getServiceType() const override {
            return "Email";
        }
    };
    
    // Concrete SMS service implementing the interface
    class SMSService : public INotificationService {
    public:
        void sendNotification(const std::string& recipient, const std::string& message) override {
            std::cout << "SMS to " << recipient << ": " << message << std::endl;
        }
        
        std::string getServiceType() const override {
            return "SMS";
        }
    };
    
    // New service can be added without modifying existing code
    class PushNotificationService : public INotificationService {
    public:
        void sendNotification(const std::string& recipient, const std::string& message) override {
            std::cout << "Push notification to " << recipient << ": " << message << std::endl;
        }
        
        std::string getServiceType() const override {
            return "Push";
        }
    };
    
    // Loosely coupled notification manager
    class NotificationManager {
    private:
        std::vector<std::unique_ptr<INotificationService>> services;
        
    public:
        // Dependency injection through constructor
        NotificationManager(std::vector<std::unique_ptr<INotificationService>> injectedServices) 
            : services(std::move(injectedServices)) {}
        
        // Add services dynamically
        void addService(std::unique_ptr<INotificationService> service) {
            services.push_back(std::move(service));
        }
        
        // Send through all registered services
        void sendToAll(const std::string& recipient, const std::string& message) {
            for (auto& service : services) {
                service->sendNotification(recipient, message);
            }
        }
        
        // Send through specific service type
        void sendViaService(const std::string& serviceType, const std::string& recipient, const std::string& message) {
            for (auto& service : services) {
                if (service->getServiceType() == serviceType) {
                    service->sendNotification(recipient, message);
                    return;
                }
            }
            std::cout << "Service type " << serviceType << " not found" << std::endl;
        }
        
        // List available services
        void listServices() const {
            std::cout << "Available services: ";
            for (const auto& service : services) {
                std::cout << service->getServiceType() << " ";
            }
            std::cout << std::endl;
        }
    };
}

// Example 3: Strategy Pattern for Loose Coupling
namespace StrategyPattern {
    
    // Abstract strategy interface
    class ISortStrategy {
    public:
        virtual ~ISortStrategy() = default;
        virtual void sort(std::vector<int>& data) = 0;
        virtual std::string getName() const = 0;
    };
    
    // Concrete strategies
    class BubbleSort : public ISortStrategy {
    public:
        void sort(std::vector<int>& data) override {
            // Simplified bubble sort implementation
            for (size_t i = 0; i < data.size(); ++i) {
                for (size_t j = 0; j < data.size() - 1 - i; ++j) {
                    if (data[j] > data[j + 1]) {
                        std::swap(data[j], data[j + 1]);
                    }
                }
            }
        }
        
        std::string getName() const override {
            return "Bubble Sort";
        }
    };
    
    class QuickSort : public ISortStrategy {
    public:
        void sort(std::vector<int>& data) override {
            // Simplified - would implement actual quicksort
            std::sort(data.begin(), data.end());
        }
        
        std::string getName() const override {
            return "Quick Sort";
        }
    };
    
    // Context class that uses strategies
    class DataSorter {
    private:
        std::unique_ptr<ISortStrategy> strategy;
        
    public:
        // Constructor injection
        DataSorter(std::unique_ptr<ISortStrategy> sortStrategy) 
            : strategy(std::move(sortStrategy)) {}
        
        // Runtime strategy switching
        void setStrategy(std::unique_ptr<ISortStrategy> newStrategy) {
            strategy = std::move(newStrategy);
        }
        
        void sortData(std::vector<int>& data) {
            if (strategy) {
                std::cout << "Sorting with: " << strategy->getName() << std::endl;
                strategy->sort(data);
            }
        }
    };
}

// Example 4: Observer Pattern for Loose Coupling
namespace ObserverPattern {
    
    // Forward declaration
    class IObserver;
    
    // Subject interface
    class ISubject {
    public:
        virtual ~ISubject() = default;
        virtual void attach(IObserver* observer) = 0;
        virtual void detach(IObserver* observer) = 0;
        virtual void notify() = 0;
    };
    
    // Observer interface
    class IObserver {
    public:
        virtual ~IObserver() = default;
        virtual void update(const std::string& message) = 0;
    };
    
    // Concrete subject
    class NewsAgency : public ISubject {
    private:
        std::vector<IObserver*> observers;
        std::string latestNews;
        
    public:
        void attach(IObserver* observer) override {
            observers.push_back(observer);
        }
        
        void detach(IObserver* observer) override {
            observers.erase(std::remove(observers.begin(), observers.end(), observer), observers.end());
        }
        
        void notify() override {
            for (auto observer : observers) {
                observer->update(latestNews);
            }
        }
        
        void setNews(const std::string& news) {
            latestNews = news;
            notify();
        }
    };
    
    // Concrete observers
    class EmailSubscriber : public IObserver {
    private:
        std::string email;
        
    public:
        EmailSubscriber(const std::string& emailAddress) : email(emailAddress) {}
        
        void update(const std::string& message) override {
            std::cout << "Email notification to " << email << ": " << message << std::endl;
        }
    };
    
    class SMSSubscriber : public IObserver {
    private:
        std::string phone;
        
    public:
        SMSSubscriber(const std::string& phoneNumber) : phone(phoneNumber) {}
        
        void update(const std::string& message) override {
            std::cout << "SMS notification to " << phone << ": " << message << std::endl;
        }
    };
}

// Example 5: Factory Pattern for Loose Coupling
namespace FactoryPattern {
    
    // Abstract product
    class IDatabase {
    public:
        virtual ~IDatabase() = default;
        virtual void connect() = 0;
        virtual void execute(const std::string& query) = 0;
        virtual void disconnect() = 0;
    };
    
    // Concrete products
    class MySQLDatabase : public IDatabase {
    public:
        void connect() override {
            std::cout << "Connected to MySQL database" << std::endl;
        }
        
        void execute(const std::string& query) override {
            std::cout << "Executing MySQL query: " << query << std::endl;
        }
        
        void disconnect() override {
            std::cout << "Disconnected from MySQL database" << std::endl;
        }
    };
    
    class PostgreSQLDatabase : public IDatabase {
    public:
        void connect() override {
            std::cout << "Connected to PostgreSQL database" << std::endl;
        }
        
        void execute(const std::string& query) override {
            std::cout << "Executing PostgreSQL query: " << query << std::endl;
        }
        
        void disconnect() override {
            std::cout << "Disconnected from PostgreSQL database" << std::endl;
        }
    };
    
    // Factory interface
    class IDatabaseFactory {
    public:
        virtual ~IDatabaseFactory() = default;
        virtual std::unique_ptr<IDatabase> createDatabase() = 0;
    };
    
    // Concrete factories
    class MySQLFactory : public IDatabaseFactory {
    public:
        std::unique_ptr<IDatabase> createDatabase() override {
            return std::make_unique<MySQLDatabase>();
        }
    };
    
    class PostgreSQLFactory : public IDatabaseFactory {
    public:
        std::unique_ptr<IDatabase> createDatabase() override {
            return std::make_unique<PostgreSQLDatabase>();
        }
    };
    
    // Client code - loosely coupled to database implementations
    class DatabaseService {
    private:
        std::unique_ptr<IDatabase> database;
        
    public:
        // Constructor injection of factory
        DatabaseService(std::unique_ptr<IDatabaseFactory> factory) {
            database = factory->createDatabase();
        }
        
        void performOperations() {
            database->connect();
            database->execute("SELECT * FROM users");
            database->execute("INSERT INTO logs VALUES (...)");
            database->disconnect();
        }
    };
}

// Demonstration function
void demonstrateLooseCoupling() {
    std::cout << "=== Loose Coupling Demonstration ===" << std::endl;
    
    // Example 1: Notification system with dependency injection
    std::cout << "\n1. Notification System with Dependency Injection:" << std::endl;
    {
        using namespace LooselyCoupled;
        
        // Create services
        std::vector<std::unique_ptr<INotificationService>> services;
        services.push_back(std::make_unique<EmailService>());
        services.push_back(std::make_unique<SMSService>());
        services.push_back(std::make_unique<PushNotificationService>());
        
        // Inject dependencies
        NotificationManager manager(std::move(services));
        
        manager.listServices();
        manager.sendToAll("user@example.com", "Welcome!");
        manager.sendViaService("Email", "user@example.com", "Email-specific message");
    }
    
    // Example 2: Strategy pattern
    std::cout << "\n2. Strategy Pattern:" << std::endl;
    {
        using namespace StrategyPattern;
        
        std::vector<int> data = {64, 34, 25, 12, 22, 11, 90};
        
        // Use different strategies
        DataSorter sorter(std::make_unique<BubbleSort>());
        sorter.sortData(data);
        
        // Switch strategy at runtime
        sorter.setStrategy(std::make_unique<QuickSort>());
        sorter.sortData(data);
    }
    
    // Example 3: Observer pattern
    std::cout << "\n3. Observer Pattern:" << std::endl;
    {
        using namespace ObserverPattern;
        
        NewsAgency agency;
        EmailSubscriber emailSub("user@example.com");
        SMSSubscriber smsSub("123-456-7890");
        
        agency.attach(&emailSub);
        agency.attach(&smsSub);
        
        agency.setNews("Breaking: New C++ standard released!");
    }
    
    // Example 4: Factory pattern
    std::cout << "\n4. Factory Pattern:" << std::endl;
    {
        using namespace FactoryPattern;
        
        // Client code doesn't know which specific database implementation is used
        DatabaseService mysqlService(std::make_unique<MySQLFactory>());
        mysqlService.performOperations();
        
        std::cout << std::endl;
        
        DatabaseService postgresService(std::make_unique<PostgreSQLFactory>());
        postgresService.performOperations();
    }
}

int main() {
    demonstrateLooseCoupling();
    return 0;
}
````

# **Flashcards:**
---

What is the main difference between tight coupling and loose coupling?;; Tight coupling means components depend directly on concrete implementations and internal details of other components. Loose coupling means components depend on abstractions (interfaces) and have minimal knowledge of other components' implementation details.

What are the key benefits of loose coupling in software design?;; Better testability (can mock dependencies), improved maintainability (changes don't cascade), enhanced reusability (components can be used in different contexts), easier parallel development, and better adherence to SOLID principles.

What is dependency injection and how does it promote loose coupling?;; Dependency injection is a technique where dependencies are provided to a component from external sources rather than the component creating them internally. This promotes loose coupling by making components depend on abstractions rather than concrete implementations.

Name three design patterns that help achieve loose coupling in C++;; Strategy Pattern (encapsulates algorithms), Observer Pattern (enables event-driven communication), and Factory Pattern (abstracts object creation). Also Abstract Factory, Dependency Injection, and Command patterns.

What role do abstract base classes play in loose coupling?;; Abstract base classes define interfaces that concrete classes must implement, allowing client code to depend on abstractions rather than concrete implementations. This enables polymorphism and makes components interchangeable without modifying client code.

What is the relationship between loose coupling and the Open/Closed Principle?;; Loose coupling enables the Open/Closed Principle by allowing systems to be open for extension (new implementations can be added) but closed for modification (existing code doesn't need to change). This is achieved through dependency on abstractions rather than concrete implementations.