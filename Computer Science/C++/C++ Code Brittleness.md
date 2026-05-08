---
memory: to_finish
tags:
  - learned
language:
  - C++
review-date:
last-reviewed: 2025-10-01
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-08-30
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
Code brittleness addresses the critical problem of maintenance and evolution in software systems. Brittle code breaks easily when changes are made elsewhere, creating cascading failures and making systems expensive to modify. In C++, this is particularly important due to the language's complexity, manual memory management, and compilation model where changes can trigger massive rebuilds and subtle runtime errors.

# Core Explanation:
---
Code brittleness refers to code that is ==fragile and prone to breaking when seemingly unrelated changes are made.== Brittle code exhibits high sensitivity to modifications and creates maintenance nightmares.

Key characteristics:
- **High coupling**: Strong dependencies between components
- **Assumption-heavy**: Code relies on implicit assumptions about other parts
- **Change amplification**: Small changes require modifications in many places
- **Hidden dependencies**: Non-obvious relationships between code sections
- **Poor encapsulation**: Internal details exposed and depended upon

Common C++ brittleness sources:
>- Raw pointers and manual memory management
>- Header dependencies and compilation coupling
>- Global state and static variables
>- Tight coupling through inheritance
>- Magic numbers and hardcoded values

# Related Concepts:
---
- **Coupling**: Brittleness often results from high coupling between modules
- **Cohesion**: Low cohesion contributes to brittleness by mixing responsibilities
- **Dependency Inversion**: Pattern that reduces brittleness through abstraction
- **SOLID Principles**: Guidelines that help prevent brittle code design
- **Technical Debt**: Brittleness accumulates as technical debt over time
- **Refactoring**: Process of reducing brittleness while preserving functionality

# Examples:
---
```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <string>

// BRITTLE CODE EXAMPLES

// Example 1: Brittle due to raw pointers and manual memory management
class BrittleDatabase {
private:
    int* data;           // Raw pointer - memory management coupling
    size_t size;
    static int maxSize;  // Global state dependency
    
public:
    BrittleDatabase(size_t s) : size(s) {
        data = new int[s]; // Manual allocation - brittle if exception thrown
        // No initialization - undefined behavior risk
    }
    
    ~BrittleDatabase() {
        delete[] data; // Brittle - what if copied? Double deletion!
    }
    
    // Brittle: No copy constructor/assignment - rule of three violation
    // Default copy will cause double deletion
    
    int getValue(size_t index) {
        return data[index]; // Brittle - no bounds checking
    }
    
    void resize(size_t newSize) {
        delete[] data;           // Brittle - temporary invalid state
        data = new int[newSize]; // What if this throws? data is dangling!
        size = newSize;
    }
};

int BrittleDatabase::maxSize = 1000; // Global state - brittle dependency

// Example 2: Brittle inheritance hierarchy
class BrittleShape {
public:
    int type; // Public data - brittle encapsulation
    double x, y;
    
    // Non-virtual destructor - brittle for inheritance
    ~BrittleShape() {}
    
    void draw() { // Non-virtual - brittle polymorphism
        // Hardcoded logic based on type - brittle to new types
        if (type == 1) {
            std::cout << "Drawing circle at " << x << "," << y << std::endl;
        } else if (type == 2) {
            std::cout << "Drawing rectangle at " << x << "," << y << std::endl;
        }
        // Adding new shape requires modifying this function - brittle!
    }
};

class BrittleCircle : public BrittleShape {
public:
    BrittleCircle(double px, double py) {
        type = 1; // Magic number - brittle assumption
        x = px; y = py;
    }
};

// ROBUST CODE EXAMPLES (Anti-brittleness)

// Example 3: Robust database with proper RAII and encapsulation
class RobustDatabase {
private:
    std::vector<int> data; // RAII container - automatic memory management
    size_t maxSize;        // Instance variable - no global dependency
    
public:
    explicit RobustDatabase(size_t initialSize, size_t max = 10000) 
        : data(initialSize, 0), maxSize(max) {
        // Vector handles allocation safely
        // Default initialization prevents undefined behavior
    }
    
    // Rule of zero - compiler-generated copy/move/destructor are correct
    
    int getValue(size_t index) const {
        if (index >= data.size()) {
            throw std::out_of_range("Index out of bounds"); // Fail fast
        }
        return data[index];
    }
    
    void resize(size_t newSize) {
        if (newSize > maxSize) {
            throw std::invalid_argument("Size exceeds maximum");
        }
        data.resize(newSize, 0); // Exception-safe resize
    }
    
    size_t size() const { return data.size(); }
};

// Example 4: Robust shape hierarchy with proper polymorphism
class RobustShape {
public:
    virtual ~RobustShape() = default; // Virtual destructor for safe inheritance
    virtual void draw() const = 0;    // Pure virtual - forces implementation
    virtual std::unique_ptr<RobustShape> clone() const = 0; // Prototype pattern
    
protected:
    double x, y; // Protected - controlled access
    
    RobustShape(double px, double py) : x(px), y(py) {}
};

class RobustCircle : public RobustShape {
private:
    double radius;
    
public:
    RobustCircle(double px, double py, double r) 
        : RobustShape(px, py), radius(r) {}
    
    void draw() const override {
        std::cout << "Drawing circle at (" << x << "," << y 
                  << ") with radius " << radius << std::endl;
    }
    
    std::unique_ptr<RobustShape> clone() const override {
        return std::make_unique<RobustCircle>(x, y, radius);
    }
};

class RobustRectangle : public RobustShape {
private:
    double width, height;
    
public:
    RobustRectangle(double px, double py, double w, double h) 
        : RobustShape(px, py), width(w), height(h) {}
    
    void draw() const override {
        std::cout << "Drawing rectangle at (" << x << "," << y 
                  << ") size " << width << "x" << height << std::endl;
    }
    
    std::unique_ptr<RobustShape> clone() const override {
        return std::make_unique<RobustRectangle>(x, y, width, height);
    }
};

// Example 5: Configuration management - brittle vs robust
namespace Brittle {
    // Hardcoded values scattered throughout code - very brittle
    void processData() {
        const int BUFFER_SIZE = 1024;    // Magic number
        const double THRESHOLD = 0.95;   // Hardcoded threshold
        const std::string LOG_FILE = "/tmp/app.log"; // Hardcoded path
        
        // Code using these values...
        // If requirements change, must hunt through entire codebase
    }
}

namespace Robust {
    // Centralized configuration - changes isolated to one place
    class Config {
    private:
        static Config instance;
        int bufferSize;
        double threshold;
        std::string logFile;
        
        Config() : bufferSize(1024), threshold(0.95), logFile("/tmp/app.log") {}
        
    public:
        static const Config& getInstance() { return instance; }
        
        int getBufferSize() const { return bufferSize; }
        double getThreshold() const { return threshold; }
        const std::string& getLogFile() const { return logFile; }
        
        // Methods to update configuration from file/environment
        void loadFromFile(const std::string& configFile);
    };
    
    Config Config::instance;
    
    void processData() {
        const auto& config = Config::getInstance();
        int bufferSize = config.getBufferSize();
        double threshold = config.getThreshold();
        const std::string& logFile = config.getLogFile();
        
        // Code using configuration values...
        // Changes centralized - much less brittle
    }
}

int main() {
    // Demonstrate robust patterns
    try {
        RobustDatabase db(10);
        std::cout << "Database size: " << db.size() << std::endl;
        
        std::vector<std::unique_ptr<RobustShape>> shapes;
        shapes.push_back(std::make_unique<RobustCircle>(0, 0, 5));
        shapes.push_back(std::make_unique<RobustRectangle>(10, 10, 3, 4));
        
        for (const auto& shape : shapes) {
            shape->draw(); // Polymorphic call - extensible design
        }
        
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
    }
    
    return 0;
}
```

# Flashcards:
---

What is code brittleness in C++?;; Code that is fragile and prone to breaking when changes are made elsewhere, characterized by high coupling, hidden dependencies, and poor encapsulation that makes maintenance difficult and expensive.

What are three main sources of brittleness in C++ code?;; Raw pointers with manual memory management, tight coupling through inheritance or global state, and hardcoded values scattered throughout the codebase instead of centralized configuration.

How does RAII help reduce code brittleness?;; RAII (Resource Acquisition Is Initialization) automatically manages resources through constructors/destructors, eliminating manual memory management errors and ensuring exception safety, making code more robust to changes.