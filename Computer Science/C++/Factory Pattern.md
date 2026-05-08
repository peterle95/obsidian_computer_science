---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date: ""
last-reviewed: 2025-07-22
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-06-30

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
The Factory Pattern solves the problem of **object creation complexity** and **tight coupling** between client code and concrete classes. Instead of having client code directly instantiate specific objects (which creates dependencies), ==the factory encapsulates the creation logic and returns objects through a common interface.==

**Key Problems it Solves:**

- **Runtime Object Creation**: Creating objects based on runtime information (user input, configuration files, string identifiers)
- **Decoupling**: Separating object creation from object usage
- **Extensibility**: Adding new object types without modifying existing client code
- **Centralized Creation Logic**: Avoiding scattered `new` statements throughout the codebase

This is crucial in C++ because it promotes **[[C++ Loose Coupling]]**, **maintainability**, and follows the **[[Open⧸Closed Principle]]** (open for extension, closed for modification).

# **Core Explanation:**

---
The **Factory Pattern** is a creational design pattern that provides an interface for creating objects without specifying their exact class. Instead of calling constructors directly, clients request objects from a factory method.

**Key Characteristics:**

1. **Abstract Product**: Common interface/base class for all created objects
2. **Concrete Products**: Specific implementations of the abstract product
3. **Factory**: Class or method responsible for object creation
4. **Client**: Uses the factory to get objects, doesn't know concrete types

**How it Works:**

1. ==Client calls factory method with parameters (often strings or enums)==
2. ==Factory analyzes parameters and decides which concrete class to instantiate==
3. ==Factory creates the object using `new` and returns it as a pointer to the abstract base==
4. Client uses the object through the abstract interface
5. Client is responsible for memory cleanup (`delete`)

**Benefits:**

- Reduces code duplication
- Centralizes object creation logic
- Makes code more maintainable and testable
- Supports polymorphism naturally

# **Related Concepts:**

---
**Abstract Base Classes**: The "product" in factory pattern is typically an abstract base class with pure virtual functions.

**Function Pointers**: Advanced factory implementations use function pointers or function objects to avoid long if/else chains.

**Strategy Pattern**: Similar in that both use polymorphism, but Strategy focuses on algorithms while Factory focuses on object creation.

**Builder Pattern**: Another creational pattern, but Builder focuses on constructing complex objects step-by-step, while Factory creates objects in one call.

**Singleton Pattern**: Often combined with Factory - the factory itself might be a singleton to ensure centralized creation.

**RAII (Resource Acquisition Is Initialization)**: Factory-created objects often need proper cleanup, making RAII principles important.

# **Examples:**

---
```cpp
// ABSTRACT PRODUCT - Common interface for all forms
class AForm {
protected:
 std::string _name;
 bool _signed;
 int _gradeToSign;
 int _gradeToExecute;

public:
 AForm(std::string name, int signGrade, int execGrade)
 : _name(name), _signed(false), _gradeToSign(signGrade), _gradeToExecute(execGrade) {}

 virtual ~AForm {} // Virtual destructor for proper cleanup

 // Pure virtual function makes this an abstract class
 virtual void execute(const Bureaucrat& executor) const = 0;

 std::string getName const { return _name; }
 bool isSigned const { return _signed; }
};

// CONCRETE PRODUCTS - Specific implementations
class ShrubberyCreationForm : public AForm {
private:
 std::string _target;
public:
 ShrubberyCreationForm(std::string target)
 : AForm("ShrubberyCreationForm", 145, 137), _target(target) {}

 void execute(const Bureaucrat& executor) const override {
 // Create ASCII trees in file
 std::cout << "Creating shrubbery for " << _target << std::endl;
 }
};

class RobotomyRequestForm : public AForm {
private:
 std::string _target;
public:
 RobotomyRequestForm(std::string target)
 : AForm("RobotomyRequestForm", 72, 45), _target(target) {}

 void execute(const Bureaucrat& executor) const override {
 // 50% chance of success
 std::cout << "Attempting to robotomize " << _target << std::endl;
 }
};

// FACTORY CLASS - Responsible for object creation
class Intern {
public:
 // FACTORY METHOD - Creates objects based on string identifier
 AForm* makeForm(const std::string& formName, const std::string& target) {

 // Array of form names for comparison
 std::string formNames = {
 "shrubbery creation",
 "robotomy request"
 };

 // Array of function pointers to creation methods
 // This avoids messy if/else chains
 AForm* (Intern::*creators)(const std::string&) = {
 &Intern::makeShrubberyForm,
 &Intern::makeRobotomyForm
 };

 // Search for matching form name
 for (int i = 0; i < 2; i++) {
 if (formName == formNames[i]) {
 std::cout << "Intern creates " << formName << std::endl;

 // Call the appropriate creation method using function pointer
 return (this->*creators[i])(target);
 }
 }

 // Form not found - return NULL
 std::cout << "Error: Unknown form '" << formName << "'" << std::endl;
 return NULL;
 }

private:
 // Helper methods for creating specific forms
 AForm* makeShrubberyForm(const std::string& target) {
 return new ShrubberyCreationForm(target); // Dynamic allocation
 }

 AForm* makeRobotomyForm(const std::string& target) {
 return new RobotomyRequestForm(target); // Dynamic allocation
 }
};

// CLIENT CODE - Uses factory without knowing concrete types
int main {
 Intern intern; // Create factory instance

 // Request objects from factory using string identifiers
 AForm* form1 = intern.makeForm("shrubbery creation", "garden");
 AForm* form2 = intern.makeForm("robotomy request", "Bender");
 AForm* form3 = intern.makeForm("invalid form", "test"); // Will return NULL

 // Use objects through abstract interface (polymorphism)
 if (form1) {
 std::cout << "Created: " << form1->getName << std::endl;
 // form1->execute(someExecutor); // Would call ShrubberyCreationForm::execute
 }

 if (form2) {
 std::cout << "Created: " << form2->getName << std::endl;
 // form2->execute(someExecutor); // Would call RobotomyRequestForm::execute
 }

 // CLIENT IS RESPONSIBLE FOR MEMORY CLEANUP
 delete form1; // Calls virtual destructor
 delete form2; // Calls virtual destructor
 // form3 is NULL, no need to delete

 return 0;
}
```

# **Flashcards:**

---
What is the main purpose of the Factory Pattern?;; To encapsulate object creation logic and decouple client code from concrete classes, allowing objects to be created based on runtime parameters without the client knowing the specific types.

What are the key components of the Factory Pattern?;; Abstract Product (common interface), Concrete Products (specific implementations), Factory (creation logic), and Client (uses factory to get objects through abstract interface).

Why use function pointers in Factory Pattern implementation?;; To avoid messy if/else chains when selecting which object to create, making the code more maintainable and following the principle of avoiding repetitive conditional structures.