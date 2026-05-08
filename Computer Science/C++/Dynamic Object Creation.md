---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date: ""
last-reviewed: 2025-07-20
keywords:
 - dynamic
 - new keyword
 - delete
 - runtime
 - object
 - heap
 - memory allocation
scheda: done
visit-count: 4
confidence-level: 3
consecutive-correct: 4

---
```dataviewjs
const currentPage = dv.current;
let visitCount = currentPage.file.frontmatter["visit-count"] || 0;
let confidence = currentPage.file.frontmatter["confidence-level"] || 1;
let streak = currentPage.file.frontmatter["consecutive-correct"] || 0;

const container = this.container.createEl('div');
container.style.cssText = `
 background:

# 2a2a2a; border: 1px solid

# 404040; border-radius: 6px;
 padding: 12px; margin: 10px 0; display: inline-block;
`;

// Status display
const status = container.createEl('div');
status.innerHTML = `
 <strong>Learning Progress</strong><br>
 Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
`;
status.style.cssText = 'margin-bottom: 10px; font-size: 13px; color:

# cccccc;';

// Quick feedback buttons
const buttonContainer = container.createEl('div');
['Got it! ✅', 'Struggled ⚠️', 'Failed ❌'].forEach((label, index) => {
 const btn = buttonContainer.createEl('button');
 btn.textContent = label;
 btn.style.cssText = `
 margin-right: 8px; padding: 4px 8px; border: none; border-radius: 3px;
 cursor: pointer; font-size: 11px;
 background: ${['

# 28a745', '

# ffc107', '

# dc3545'][index]};
 color: ${index === 1 ? '

# 000' : '

# fff'};
 `;

 btn.addEventListener('click', async => {
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
 fm["last-reviewed"] = new Date.toISOString.split('T');
 if (index > 0) fm["last-struggle-date"] = new Date.toISOString.split('T');
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
const currentPage = dv.current;
const content = await app.vault.read(app.vault.getAbstractFileByPath(currentPage.file.path));

// Split content into lines
const lines = content.split('\n');
let flashcardLines = ;
let inCodeBlock = false;

// Collect all potential flashcard lines - simplified approach
for (let i = 0; i < lines.length; i++) {
 const line = lines[i];

 // Track code blocks
 if (line.trim.startsWith('```')) {
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
 line.trim.startsWith('const ') ||
 line.trim.startsWith('let ') ||
 line.trim.startsWith('function ') ||
 line.trim.startsWith('return ') ||
 line.trim.startsWith('if (') ||
 line.trim.startsWith('for (') ||
 line.trim.startsWith('while (') ||
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
const flashcards = ;
for (let i = 0; i < filteredLines.length; i++) {
 const line = filteredLines[i];
 try {
 const separatorIndex = line.indexOf(';;');
 if (separatorIndex === -1) continue;

 const front = line.substring(0, separatorIndex).trim;
 const back = line.substring(separatorIndex + 2).trim;

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
 errorMsg.style.cssText = 'background:

# 2a2a2a; padding: 15px; border-radius: 6px; color:

# cccccc;';
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
 background:

# 2a2a2a;
 border: 1px solid

# 404040;
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
title.style.cssText = 'margin: 0; color:

# ffffff;';

const progress = header.createEl('div');
progress.style.cssText = 'color:

# cccccc; font-size: 14px; text-align: right;';

// Card container
const cardContainer = container.createEl('div');
cardContainer.style.cssText = `
 background:

# 1a1a1a;
 border: 2px solid

# 404040;
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
 color:

# ffffff;
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
 background:

# 4a9eff; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; font-weight: 500;
`;

const easyButton = buttonContainer.createEl('button');
easyButton.textContent = 'Easy ✅';
easyButton.style.cssText = `
 background:

# 28a745; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const hardButton = buttonContainer.createEl('button');
hardButton.textContent = 'Hard ❌';
hardButton.style.cssText = `
 background:

# dc3545; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const nextButton = buttonContainer.createEl('button');
nextButton.textContent = 'Next →';
nextButton.style.cssText = `
 background:

# 6c757d; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const prevButton = buttonContainer.createEl('button');
prevButton.textContent = '← Prev';
prevButton.style.cssText = `
 background:

# 6c757d; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const shuffleButton = buttonContainer.createEl('button');
shuffleButton.textContent = '🔀 Shuffle';
shuffleButton.style.cssText = `
 background:

# 17a2b8; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

// Functions
function updateDisplay {
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
 cardContainer.style.borderColor = '

# ffc107';
 cardContainer.style.backgroundColor = '

# 252525';
 } else {
 easyButton.style.display = 'none';
 hardButton.style.display = 'none';
 flipButton.textContent = 'Flip Card';
 cardContainer.style.borderColor = '

# 404040';
 cardContainer.style.backgroundColor = '

# 1a1a1a';
 }

 // Update navigation buttons
 prevButton.style.display = currentCardIndex > 0 ? 'inline-block' : 'none';
 nextButton.textContent = currentCardIndex < flashcards.length - 1 ? 'Next →' : 'Restart';
}

function flipCard {
 showingBack = !showingBack;
 updateDisplay;
}

function nextCard {
 if (currentCardIndex < flashcards.length - 1) {
 currentCardIndex++;
 } else {
 currentCardIndex = 0;
 }
 showingBack = false;
 updateDisplay;
}

function prevCard {
 if (currentCardIndex > 0) {
 currentCardIndex--;
 showingBack = false;
 updateDisplay;
 }
}

function markCorrect {
 if (showingBack) {
 correctCount++;
 totalReviewed++;
 nextCard;
 }
}

function markIncorrect {
 if (showingBack) {
 totalReviewed++;
 nextCard;
 }
}

function shuffleCards {
 for (let i = flashcards.length - 1; i > 0; i--) {
 const j = Math.floor(Math.random * (i + 1));
 [flashcards[i], flashcards[j]] = [flashcards[j], flashcards[i]];
 }
 currentCardIndex = 0;
 showingBack = false;
 correctCount = 0;
 totalReviewed = 0;
 updateDisplay;
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
instructions.style.cssText = 'font-size: 12px; color:

# 888; text-align: center; line-height: 1.4;';
instructions.innerHTML = `
 <strong>Controls:</strong> Click card to flip | Navigation buttons | Easy/Hard to mark
`;

// Initialize
updateDisplay;
```

# **Purpose/Why**:

---
Dynamic Object Creation ==solves the problem of creating objects when you don't know at compile time what type of object you need, how many objects you need, or when you need them==. This is essential for:

- **Factory Patterns**: Creating different object types based on runtime parameters (like string identifiers)
- **Polymorphic Collections**: Storing different derived objects in the same container
- **Memory Flexibility**: Allocating objects that outlive their creation scope
- **Runtime Decision Making**: Creating objects based on user input, configuration files, or network data

Without dynamic allocation, you'd be limited to stack-allocated objects with fixed types and lifetimes, severely restricting program flexibility and design patterns.

# **Core Explanation:**

---
**Dynamic Object Creation** is the process of allocating and constructing ==objects on the heap at runtime using the `new` operator, returning a pointer to the created object.==

**Key Characteristics:**

- **Heap Allocation**: Objects are created in heap memory, not on the stack
- **Manual Memory Management**: You must explicitly call `delete` to free memory
- **Pointer Return**: `new` returns a pointer to the allocated object
- **Runtime Flexibility**: Object type and creation timing determined at runtime
- **Outlives Scope**: Objects persist beyond the scope where they were created

**How it Works:**

1. `new` operator allocates memory on the heap
2. Constructor is called to initialize the object
3. Pointer to the object is returned
4. Object exists until `delete` is explicitly called
5. `delete` calls destructor and frees memory

**Memory Responsibility**: The caller becomes responsible for memory management - every `new` must have a corresponding `delete`.

# **Related Concepts:**

---
**Stack vs Heap Allocation**: Stack objects are automatically destroyed when leaving scope; heap objects require manual deletion.

**[[C++ Smart Pointers]]**: Modern C++ alternatives (`unique_ptr`, `shared_ptr`) that automatically manage memory to prevent leaks.

**Factory Pattern**: Often uses dynamic creation to instantiate different derived classes based on parameters.

**Polymorphism**: Dynamic creation enables storing different derived objects through base class pointers.

**Memory Leaks**: Failure to `delete` dynamically created objects leads to memory leaks.

**RAII (Resource Acquisition Is Initialization)**: Principle that resources should be tied to object lifetime to ensure proper cleanup.

# **Examples:**

---
```cpp

# include <iostream>

# include <string>

class AForm
{
public:
 virtual ~AForm {} // Virtual destructor for proper cleanup
 virtual void execute = 0;
};

class ShrubberyForm : public AForm
{
private:
 std::string target;
public:
 ShrubberyForm(const std::string& t) : target(t) {}
 void execute override
 {
 std::cout << "Creating shrubbery for " << target << std::endl;
 }
};

class RobotomyForm : public AForm {
private:
 std::string target;
public:
 RobotomyForm(const std::string& t) : target(t) {}
 void execute override {
 std::cout << "Robotomizing " << target << std::endl;
 }
};

class FormFactory {
public:
 // Factory method using dynamic object creation
 static AForm* createForm(const std::string& type, const std::string& target) {
 // Decision made at runtime based on string parameter
 if (type == "shrubbery") {
 // Dynamic creation - object allocated on heap
 return new ShrubberyForm(target);
 }
 else if (type == "robotomy") {
 // Another dynamic creation - different type based on input
 return new RobotomyForm(target);
 }
 // Return NULL if invalid type - no object created
 return NULL;
 }
};

int main {
 // Dynamic object creation based on runtime string
 AForm* form1 = FormFactory::createForm("shrubbery", "garden");
 AForm* form2 = FormFactory::createForm("robotomy", "target");
 AForm* form3 = FormFactory::createForm("invalid", "test");

 // Check if objects were successfully created
 if (form1) {
 form1->execute; // Polymorphic call
 delete form1; // CRITICAL: Manual memory cleanup
 form1 = NULL; // Good practice: avoid dangling pointer
 }

 if (form2) {
 form2->execute; // Different object type, same interface
 delete form2; // Each 'new' needs corresponding 'delete'
 form2 = NULL;
 }

 if (form3) {
 form3->execute;
 delete form3;
 } else {
 std::cout << "Form3 creation failed - no cleanup needed" << std::endl;
 }

 // Array of dynamically created objects
 AForm* forms;
 forms = new ShrubberyForm("home");
 forms = new RobotomyForm("robot");
 forms = new ShrubberyForm("office");

 // Process all forms polymorphically
 for (int i = 0; i < 3; i++)
 {
 if (forms[i])
 {
 forms[i]->execute;
 delete forms[i]; // Clean up each dynamically created object
 forms[i] = NULL;
 }
 }

 return 0;
}

// Common pitfall - memory leak example:
void memoryLeakExample
{
 AForm* form = new ShrubberyForm("leak");
 form->execute;
 // BUG: Missing delete - memory leak!
 // Correct: delete form;
}

// Correct pattern with exception safety:
void safePatternExample {
 AForm* form = NULL;
 try {
 form = new ShrubberyForm("safe");
 form->execute;
 delete form; // Cleanup in normal path
 form = NULL;
 }
 catch (...) {
 delete form; // Cleanup in exception path
 throw; // Re-throw exception
 }
}
```

# **Flashcards:**

---
What operator creates objects dynamically on the heap and what must you do afterward?;; The `new` operator creates objects on the heap, and you must call `delete` to free the memory and prevent memory leaks.

What is the main difference between stack and dynamic object creation in terms of memory management?;; Stack objects are automatically destroyed when leaving scope, while dynamically created objects require manual `delete` calls and persist until explicitly freed.

In a factory pattern, why is dynamic object creation essential?;; Dynamic creation allows the factory to decide at runtime which specific derived class to instantiate based on parameters, returning different object types through a common base pointer.