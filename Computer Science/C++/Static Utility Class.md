---
memory: to_finish
tags:
  - learned
language:
  - C++
review-date: ""
last-reviewed: 2025-07-10
keywords:
  - static methods
  - private constructors
  - utility class
  - instantiation prevention
  - class design
scheda: done
visit-count: 3
confidence-level: 2.5
consecutive-correct: 3
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
---
The **Static Utility Class** pattern prevents object instantiation by making all constructors, destructors, and assignment operators private, while providing functionality through [[static methods]] only. This design is used when a class serves purely as a collection of related utility functions without needing to maintain any state.

**Key Components:**
- **Private Constructors/Destructors**: Prevent users from creating, copying, or destroying objects
- **Static Methods**: Belong to the class itself, not to any instance - accessible via `ClassName::method()`
- **No Member Variables**: The class maintains no state, making [[Instantiation]] unnecessary
- **Single Responsibility**: Groups related functionality under one namespace-like interface

==**Why Use This Pattern:**==
- ==**Performance**: No object creation/destruction overhead==
- ==**Memory Efficiency**: No memory allocated for unused objects==
- ==**Clear Intent**: Signals that this is purely functional, not object-oriented==
- ==**Simplicity**: Users can't accidentally create or pass around unnecessary objects==
# **Related Concepts:**
---
**Alternative Approaches:**
- **Namespace**: Groups functions without class overhead (`namespace Utils { void func(); }`)
- **Free Functions**: Standalone functions (C-style approach)
- **Singleton Pattern**: Ensures only one instance exists (overkill for stateless utilities)
- **Abstract Base Class**: Uses pure virtual functions (`virtual func() = 0`) - requires inheritance
<!--SR:!2000-01-01,1,250!2025-06-18,4,270!2000-01-01,1,250!2000-01-01,1,250!2000-01-01,1,250-->

**C++ Specifiers:**
- **`static`**: Members belong to class, not instances - accessible without objects
- **`= delete`** (C++11+): Explicitly prevents function usage (modern alternative to private)
- **`= 0`**: Creates pure virtual functions (only valid for virtual functions, not constructors)

**Design Principles:**
- **Utility vs Object-Oriented**: When to use functional vs OOP approaches
- **State Management**: Classes with state need instances, stateless classes don't
- **Access Control**: Using private/public to enforce design intentions

# **Examples:**
---
```cpp
// STATIC UTILITY CLASS IMPLEMENTATION
class ScalarConverter {
private:
   // PREVENT INSTANTIATION: Private default constructor
   // Users cannot write: ScalarConverter obj;
   ScalarConverter() {}
   
   // PREVENT COPYING: Private copy constructor  
   // Users cannot write: ScalarConverter obj2(obj1);
   ScalarConverter(const ScalarConverter& other) { (void)other; }
   
   // PREVENT ASSIGNMENT: Private assignment operator
   // Users cannot write: obj1 = obj2;
   ScalarConverter& operator=(const ScalarConverter& other) { 
       (void)other; return *this; 
   }
   
   // PREVENT DESTRUCTION: Private destructor
   // Even if someone could create an object, they couldn't destroy it properly
   ~ScalarConverter() {}
   
   // HELPER METHODS: Also static, used internally by public methods
   static bool isInt(const std::string& literal) {
       char* end;
       long num = std::strtol(literal.c_str(), &end, 10);
       // Check if entire string was parsed AND result fits in int range
       return (*end == '\0' && num >= INT_MIN && num <= INT_MAX);
   }

public:
   // PUBLIC INTERFACE: Only static method accessible to users
   // No object needed - called via ScalarConverter::convert("42")
   static void convert(const std::string& literal) {
       if (isInt(literal)) {
           int i = std::atoi(literal.c_str());
           std::cout << "int: " << i << std::endl;
           // Convert to other types...
       }
       // Handle other types...
   }
};

// USAGE EXAMPLES
int main() {
   // ✅ CORRECT: Static method call - no object needed
   ScalarConverter::convert("42");
   ScalarConverter::convert("3.14f");
   
   // ❌ ERROR: Cannot instantiate - constructor is private
   // ScalarConverter converter;  // Compilation error
   
   // ❌ ERROR: Cannot create pointer to object
   // ScalarConverter* ptr = new ScalarConverter();  // Compilation error
   
   return 0;
}

// ALTERNATIVE APPROACHES FOR COMPARISON

// 1. NAMESPACE APPROACH (C++ alternative to static class)
namespace ScalarConverter {
   // Functions are not class members - just grouped in namespace
   void convert(const std::string& literal) {
       // Implementation...
   }
   
   bool isInt(const std::string& literal) {
       // Implementation...
   }
}
// Usage: ScalarConverter::convert("42");

// 2. FREE FUNCTIONS (C-style approach)
// Functions exist independently - not grouped
void convertScalar(const std::string& literal) { /* ... */ }
bool isIntLiteral(const std::string& literal) { /* ... */ }
// Usage: convertScalar("42");

// 3. REGULAR CLASS (Object-oriented approach)
class ScalarConverter {
private:
   std::string lastConverted;  // CLASS HAS STATE - needs instances
   
public:
   ScalarConverter() : lastConverted("") {}  // Public constructor
   
   void convert(const std::string& literal) {
       lastConverted = literal;  // Store state
       // Convert and display...
   }
   
   std::string getLastConverted() const { return lastConverted; }
};
// Usage: ScalarConverter converter; converter.convert("42");

// 4. SINGLETON PATTERN (One instance maximum)
class ScalarConverter {
private:
   ScalarConverter() {}  // Private constructor
   
public:
   // Returns reference to single instance
   static ScalarConverter& getInstance() {
       static ScalarConverter instance;  // Created once, persists
       return instance;
   }
   
   void convert(const std::string& literal) { /* ... */ }
};
// Usage: ScalarConverter::getInstance().convert("42");
```
# **Flashcards:**
---
Q: Why are constructors made private in a static utility class?;; A: To prevent instantiation since the class is designed to work only through static methods. The class has no state to maintain, so creating objects would be wasteful and unnecessary. Users should only call ClassName::staticMethod() without creating objects.
<!--SR:!2025-06-15,1,230-->

Q: What does the 'static' keyword do for class methods and when should you use it?;; A: Static methods belong to the class itself, not to any instance. They can be called without creating objects using ClassName::method(). Use static when the method doesn't need access to instance variables and provides utility functionality that logically belongs to the class.

Q: What are three alternatives to the static utility class pattern and their trade-offs?;; A: 1) Namespace - simpler, no class overhead, but less OOP-like. 2) Free functions - C-style, risk of naming conflicts, no grouping. 3) Regular class with instances - needed when maintaining state, but overhead for stateless utilities. Static utility class provides grouping with clear non-instantiable intent.
<!--SR:!2025-06-17,3,250-->