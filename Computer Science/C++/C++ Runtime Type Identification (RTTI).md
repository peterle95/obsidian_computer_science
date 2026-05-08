---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date: ""
last-reviewed: 2025-07-17
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 3
last-struggle-date: ""

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
RTTI solves the fundamental problem of ==**type uncertainty in polymorphic hierarchies**==. When you have a base pointer or reference that could point to any derived class, you often need to know the actual runtime type to perform type-specific operations safely.

**Key Problems RTTI Addresses:**
- **Safe Downcasting**: Converting from base to derived types without undefined behavior
- **Type-Specific Behavior**: Executing different code paths based on actual object type
- **Interface Limitations**: When base class interface is insufficient for specific derived functionality
- **Runtime Decisions**: Making type-dependent choices when compile-time type is unknown

**Why Critical in C++:**
- C++ allows storing derived objects as base pointers/references (polymorphism)
- Static typing means compiler can't always determine runtime types
- Alternative approaches (void* casting, C-style casts) are unsafe and error-prone
- Essential for frameworks, serialization, and complex object hierarchies

# **Core Explanation:**

---
**Runtime Type Identification (RTTI)** is a C++ mechanism that ==allows programs to determine the actual type of polymorphic objects during execution==, rather than at compile time.

**Key Components:**
>1. **`dynamic_cast`**: Safe casting operator for polymorphic types
>2. **`typeid`**: Operator returning type information
>3. **`std::type_info`**: Class providing type metadata
 --> what is type metadata?

**How ==dynamic_cast== Works:**
- ==**With Pointers**: Returns `nullptr` if cast fails==
- ==**With References**: Throws `std::bad_cast` exception if cast fails==
- **Requirements**: Source type must be polymorphic (have virtual functions)

**Underlying Mechanism:**
- Compiler generates **Virtual Table (vtable)** for polymorphic classes
- Each object contains **vptr** (virtual pointer) to its class's vtable
- RTTI data is stored alongside vtable information
- `dynamic_cast` queries this runtime information to verify type compatibility

**Performance Considerations:**
- Small runtime overhead due to type checking
- Memory overhead for storing type information
- Can be disabled with `-fno-rtti` compiler flag if not needed

# **Related Concepts:**

---
**Direct Dependencies:**
- **Polymorphism**: RTTI only works with polymorphic types (classes with virtual functions)
- **Virtual Functions**: Enable polymorphism and provide the infrastructure RTTI relies on
- **Inheritance**: RTTI operates within class hierarchies

**Related Casting Operations:**
- **`static_cast`**: Compile-time casting, no runtime checks, faster but less safe
- **`const_cast`**: Removes/adds const qualifier, not type-related
- **`reinterpret_cast`**: Low-level bit reinterpretation, no safety guarantees
- **C-style casts**: Combines multiple cast types, generally unsafe

**Alternative Approaches:**
- **Virtual Functions**: Often preferred over RTTI for type-specific behavior
- **Visitor Pattern**: Design pattern that avoids need for type checking
- **Template Specialization**: Compile-time type differentiation
- **Tagged Unions/Variants**: Explicit type tracking in data structures

**Language Comparisons:**
- **Java**: `instanceof` operator and reflection API
- **Python**: Built-in `isinstance` and `type` functions
- **C

# **: `is` and `as` operators for type testing

# **Examples:**

---
```cpp

# include <iostream>

# include <typeinfo>

# include <memory>

// Base class must be polymorphic (have virtual functions) for RTTI to work
class Shape {
public:
 virtual ~Shape = default; // Virtual destructor makes class polymorphic
 virtual void draw = 0; // Pure virtual function
};

class Circle : public Shape {
private:
 double radius;
public:
 Circle(double r) : radius(r) {}
 void draw override { std::cout << "Drawing circle\n"; }
 double getRadius const { return radius; } // Circle-specific method
};

class Rectangle : public Shape {
private:
 double width, height;
public:
 Rectangle(double w, double h) : width(w), height(h) {}
 void draw override { std::cout << "Drawing rectangle\n"; }
 double getArea const { return width * height; } // Rectangle-specific method
};

// Function demonstrating dynamic_cast with pointers
void identifyShapeByPointer(Shape* shape) {
 std::cout << "\n=== Pointer-based RTTI ===\n";

 // Try to cast to Circle - returns nullptr if shape is not a Circle
 Circle* circle = dynamic_cast<Circle*>(shape);
 if (circle != nullptr) {
 std::cout << "Shape is a Circle with radius: " << circle->getRadius << "\n";
 return; // Early return since we found the type
 }

 // Try to cast to Rectangle - returns nullptr if shape is not a Rectangle
 Rectangle* rectangle = dynamic_cast<Rectangle*>(shape);
 if (rectangle != nullptr) {
 std::cout << "Shape is a Rectangle with area: " << rectangle->getArea << "\n";
 return;
 }

 // If we reach here, it's neither Circle nor Rectangle
 std::cout << "Unknown shape type\n";
}

// Function demonstrating dynamic_cast with references
void identifyShapeByReference(Shape& shape) {
 std::cout << "\n=== Reference-based RTTI ===\n";

 // With references, dynamic_cast throws std::bad_cast on failure
 // So we must use try-catch blocks

 try {
 // Attempt to cast reference to Circle
 Circle& circle = dynamic_cast<Circle&>(shape);
 std::cout << "Shape is a Circle with radius: " << circle.getRadius << "\n";
 return; // Success - exit function
 }
 catch (const std::bad_cast&) {
 // Not a Circle, continue trying other types
 // Note: We don't print anything here, just continue
 }

 try {
 // Attempt to cast reference to Rectangle
 Rectangle& rectangle = dynamic_cast<Rectangle&>(shape);
 std::cout << "Shape is a Rectangle with area: " << rectangle.getArea << "\n";
 return; // Success - exit function
 }
 catch (const std::bad_cast&) {
 // Not a Rectangle either
 std::cout << "Unknown shape type\n";
 }
}

// Function demonstrating typeid operator
void demonstrateTypeid(Shape* shape) {
 std::cout << "\n=== typeid Demonstration ===\n";

 // typeid returns std::type_info object with type information
 const std::type_info& shapeType = typeid(*shape); // Dereference to get actual object type
 const std::type_info& circleType = typeid(Circle);
 const std::type_info& rectangleType = typeid(Rectangle);

 std::cout << "Object type name: " << shapeType.name << "\n";

 // Compare types using == operator
 if (shapeType == circleType) {
 std::cout << "Type matches Circle\n";
 }
 else if (shapeType == rectangleType) {
 std::cout << "Type matches Rectangle\n";
 }
 else {
 std::cout << "Unknown type\n";
 }
}

// Practical example: Processing different shapes with type-specific operations
void processShapes {
 std::cout << "\n=== Practical Shape Processing ===\n";

 // Create different shapes stored as base pointers
 std::unique_ptr<Shape> shapes = {
 std::make_unique<Circle>(5.0),
 std::make_unique<Rectangle>(3.0, 4.0),
 std::make_unique<Circle>(2.5)
 };

 // Process each shape with type-specific logic
 for (auto& shape : shapes) {
 shape->draw; // Polymorphic call - works for all shapes

 // Use RTTI for type-specific operations
 if (Circle* circle = dynamic_cast<Circle*>(shape.get)) {
 // Only circles have radius - safe to call circle-specific methods
 std::cout << " Circumference: " << 2 * 3 * circle->getRadius << "\n";
 }
 else if (Rectangle* rect = dynamic_cast<Rectangle*>(shape.get)) {
 // Only rectangles have area calculation - safe to call rectangle-specific methods
 std::cout << " Area: " << rect->getArea << "\n";
 }
 }
}

int main {
 // Create test objects
 Circle circle(3.0);
 Rectangle rectangle(4.0, 5.0);

 // Demonstrate RTTI with different approaches
 Shape* shapePtr1 = &circle;
 Shape* shapePtr2 = &rectangle;

 // Test pointer-based RTTI
 identifyShapeByPointer(shapePtr1);
 identifyShapeByPointer(shapePtr2);

 // Test reference-based RTTI
 identifyShapeByReference(circle);
 identifyShapeByReference(rectangle);

 // Test typeid operator
 demonstrateTypeid(shapePtr1);
 demonstrateTypeid(shapePtr2);

 // Practical usage example
 processShapes;

 return 0;
}

/*
Expected Output:
=== Pointer-based RTTI ===
Shape is a Circle with radius: 3

=== Pointer-based RTTI ===
Shape is a Rectangle with area: 20

=== Reference-based RTTI ===
Shape is a Circle with radius: 3

=== Reference-based RTTI ===
Shape is a Rectangle with area: 20

=== typeid Demonstration ===
Object type name: 6Circle (implementation-specific)
Type matches Circle

=== typeid Demonstration ===
Object type name: 9Rectangle (implementation-specific)
Type matches Rectangle

=== Practical Shape Processing ===
Drawing circle
 Circumference: 31
Drawing rectangle
 Area: 12
Drawing circle
 Circumference: 15
*/
```

# **Flashcards:**

---
What is the primary purpose of RTTI in C++?;; RTTI (Runtime Type Identification) allows programs to determine the actual type of polymorphic objects during execution, enabling safe downcasting and type-specific operations when only base pointers/references are available.

What happens when dynamic_cast fails with pointers vs references?;; With pointers: dynamic_cast returns nullptr on failure. With references: dynamic_cast throws std::bad_cast exception on failure, requiring try-catch blocks.

What are the requirements for RTTI to work in C++?;; The class hierarchy must be polymorphic, meaning the base class must have at least one virtual function (typically virtual destructor). Non-polymorphic types cannot use RTTI.

How does dynamic_cast differ from static_cast in terms of safety?;; dynamic_cast performs runtime type checking and fails safely (nullptr/exception), while static_cast performs compile-time casting with no runtime checks, making it faster but potentially unsafe for downcasting.

What are the two main RTTI operators in C++ and their purposes?;; dynamic_cast: Safe casting operator that checks type compatibility at runtime. typeid: Returns std::type_info object containing type metadata for comparison and identification.

What is the performance cost of using RTTI?;; RTTI adds small runtime overhead for type checking, memory overhead for storing type information in vtables, and can be disabled with -fno-rtti compiler flag when not needed.