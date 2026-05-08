---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date: ""
last-reviewed: 2025-08-12
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
**Pointers vs Objects** solve several fundamental problems in C++:

1. **Memory Management Control**: Objects can be created on stack (automatic cleanup) or heap (manual control over lifetime)
2. **Polymorphism Implementation**: Base class pointers can point to derived class objects, enabling runtime behavior selection
3. **Dynamic Object Creation**: Creating objects at runtime based on user input or configuration (Factory Pattern)
4. **Resource Efficiency**: Pointers allow sharing objects without copying, and enable optional objects (nullable)
5. **Data Structure Implementation**: Essential for linked lists, trees, graphs, and other dynamic data structures

This distinction is crucial because it determines **when and how objects are created/destroyed**, **memory location**, and **access patterns** - fundamental aspects that affect program performance, memory usage, and design flexibility.

# **Core Explanation:**

---
#

# **Objects (Direct Instances)**

- **Definition**: Actual instances of a class stored directly in memory
- **Stack Allocation**: `ClassName objectName;` - automatic storage duration
- **Automatic Cleanup**: Destructors called automatically when scope ends
- **Direct Access**: Use dot operator (`.`) to access members
- **Known Type**: Exact type determined at compile time

#

# **Pointers (Indirect Access)**

- **Definition**: Variables that store memory addresses of objects
- **Declaration**: `ClassName* pointerName;` - just holds an address
- **Heap Allocation**: `new ClassName` - manual memory management required
- **Indirect Access**: Use arrow operator (`->`) or dereference (`*ptr`)
- **Polymorphic**: Base class pointers can point to derived class objects

#

# **Key Memory Locations**

- **Stack**: Automatic storage, fast allocation/deallocation, limited size
- **Heap**: Dynamic storage, manual management, larger space available

#

# **Dereferencing Process**

- `*pointer` converts pointer to the actual object it points to
- Essential when functions expect object references but you have pointers
- `pointer->member` is equivalent to `(*pointer).member`

# **Related Concepts:**

---
#

# **Factory Design Pattern**

- Uses pointers to return different derived class objects through base class interface
- Enables runtime object type selection based on string parameters or configuration

#

# **Polymorphism**

- Base class pointers pointing to derived objects enable virtual function calls
- Runtime behavior determined by actual object type, not pointer type

#

# **RAII (Resource Acquisition Is Initialization)**

- Stack objects automatically manage resources through constructors/destructors
- Heap objects require manual `delete` calls to prevent memory leaks

#

# **References vs Pointers**

- References: Aliases to existing objects, cannot be null, cannot be reassigned
- Pointers: Can be null, can be reassigned, require explicit dereferencing
- [[C++ Pointer-to-Reference]]

#

# **[[C++ Smart Pointers]]**

- `std::unique_ptr`, `std::shared_ptr` provide automatic memory management
- Combine benefits of pointers with automatic cleanup

# **Examples:**

---
```cpp

# include <iostream>

class AForm {
public:
 virtual void execute = 0; // Pure virtual function
 virtual ~AForm {} // Virtual destructor for proper cleanup
};

class RobotomyRequestForm : public AForm {
private:
 std::string _target;
public:
 RobotomyRequestForm(const std::string& target) : _target(target) {}
 void execute override {
 std::cout << "Robotomizing " << _target << std::endl;
 }
};

class Intern {
public:
 // Factory method - returns base class pointer to derived object
 AForm* makeForm(const std::string& formType, const std::string& target) {
 if (formType == "robotomy request") {
 // Dynamic allocation on heap - caller must delete
 return new RobotomyRequestForm(target);
 }
 return nullptr; // Invalid form type
 }
};

class Bureaucrat {
private:
 std::string _name;
public:
 Bureaucrat(const std::string& name) : _name(name) {}

 // Method expects AForm reference (not pointer)
 void signForm(AForm& form) {
 std::cout << _name << " is signing the form" << std::endl;
 form.execute; // Polymorphic call - actual type determines behavior
 }
};

int main {
 // === OBJECT CREATION (Stack Allocation) ===
 Bureaucrat bob("Bob"); // Stack object - automatic cleanup
 Intern someIntern; // Stack object - automatic cleanup

 // === POINTER DECLARATION ===
 AForm* formPtr; // Pointer variable - no object created yet
 // Points nowhere initially (uninitialized)

 // === DYNAMIC OBJECT CREATION (Heap Allocation) ===
 formPtr = someIntern.makeForm("robotomy request", "Bender");
 // 1. makeForm creates RobotomyRequestForm object on heap
 // 2. Returns AForm* pointing to that object
 // 3. formPtr stores the address of the heap object
 // 4. Polymorphism: base pointer points to derived object

 // Memory layout at this point:
 // Stack: bob [Bureaucrat object]
 // someIntern [Intern object]
 // formPtr [pointer variable] ──┐
 // │
 // Heap: [RobotomyRequestForm object] ←─┘

 if (formPtr != nullptr) { // Check if form creation succeeded
 // === DEREFERENCING POINTER TO PASS OBJECT ===
 bob.signForm(*formPtr); // *formPtr dereferences pointer to get object
 // ↑ // signForm expects AForm& (reference)
 // └─── Converts: AForm* → AForm& (pointer to reference)

 // Alternative access methods (equivalent):
 // formPtr->execute; // Arrow operator for direct method call
 // (*formPtr).execute; // Dereference + dot operator
 }

 // === MANUAL MEMORY CLEANUP ===
 delete formPtr; // Must manually delete heap objects
 formPtr = nullptr; // Good practice - avoid dangling pointers

 // === AUTOMATIC CLEANUP ===
 // bob and someIntern automatically destroyed when main ends
 // No delete needed for stack objects

 return 0;
}

// === COMPARISON: DIFFERENT APPROACHES ===
void demonstrateAlternatives {
 // Approach 1: Direct object creation (stack)
 RobotomyRequestForm directForm("Target1"); // Stack object
 Bureaucrat clerk("Clerk"); // Stack object
 clerk.signForm(directForm); // Pass object directly (no dereferencing)
 // Both objects automatically destroyed at end of scope

 // Approach 2: Pointer to stack object
 RobotomyRequestForm stackForm("Target2"); // Stack object
 AForm* ptrToStack = &stackForm; // Pointer to stack object
 clerk.signForm(*ptrToStack); // Dereference to pass object
 // stackForm automatically destroyed, ptrToStack becomes invalid

 // Approach 3: Heap allocation (manual management)
 AForm* heapForm = new RobotomyRequestForm("Target3"); // Heap object
 clerk.signForm(*heapForm); // Dereference to pass
 delete heapForm; // Manual cleanup required

 // Approach 4: Factory pattern (runtime type selection)
 Intern factory;
 AForm* factoryForm = factory.makeForm("robotomy request", "Target4");
 if (factoryForm) {
 clerk.signForm(*factoryForm); // Polymorphic behavior
 delete factoryForm; // Factory caller responsible for cleanup
 }
}
```

# **Flashcards:**

---
What is the key difference between `ClassName object;` and `ClassName* pointer;` in terms of memory and object creation?;; `ClassName object;` creates an actual object on the stack with automatic cleanup, while `ClassName* pointer;` only declares a pointer variable that can hold an address - no object is created until you assign it (e.g., with `new` or `&existingObject`).

Why do we write `high.signForm(*rrf)` instead of `high.signForm(rrf)` when rrf is a pointer?;; Because `signForm` expects an AForm reference (`AForm& form`), but `rrf` is a pointer (`AForm* rrf`). The `*rrf` dereferences the pointer to get the actual object, converting from pointer to reference that the method requires.

In the Factory Pattern with `AForm* form = intern.makeForm("robotomy request", "Bender")`, what type of object is actually created and why can we store it in an AForm pointer?;; A `RobotomyRequestForm` object is actually created on the heap. We can store it in an `AForm*` pointer because `RobotomyRequestForm` inherits from `AForm` (IS-A relationship), enabling polymorphism where a base class pointer can point to any derived class object.