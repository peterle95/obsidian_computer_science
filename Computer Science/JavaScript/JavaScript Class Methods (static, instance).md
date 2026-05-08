---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date: ""
last-reviewed: 2025-07-23
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
JavaScript Class Methods (both static and instance) solve the fundamental problem of ==encapsulating behavior related to objects and their blueprints (classes).== They provide a structured way to define functions that operate on data within objects or on the class itself, promoting code organization, reusability, and maintainability—key tenets of Object-Oriented Programming (OOP).

Instance methods are crucial for defining the actions that individual objects of a class can perform. For example, a `Car` object might have an `accelerate` method. <mark style="background:

# FF5582A6;">Static methods, on the other hand, are for utility functions that relate to the class as a whole rather than a specific instance.</mark> For example, a `Math` class might have a `Math.max` method.

This concept is important in computer science and JavaScript because it allows for:

- **Encapsulation:** Grouping data (properties) and the functions that operate on that data (methods) within a single unit (the class).
- **Code Reusability:** Defining methods once in a class and having them available to all instances of that class, or as utilities directly on the class.
- **Modularity:** Breaking down complex problems into smaller, manageable, and self-contained units.
- **Clarity and Readability:** Making code easier to understand by clearly defining what an object can do and what utility functions are available on the class itself.

# **Core Explanation:**

---
In JavaScript, a **class** is a blueprint for creating objects. **Methods** are functions defined within a class that define the behavior of objects created from that class. There are two primary types of class methods:

#

#

# 1. Instance Methods

**Definition:** ==Instance methods are functions that belong to an _instance_ (an object) of a class.== *They operate on the data (properties) specific to that particular instance.*

**Key Characteristics:**

- **Access to `this`:** Inside an instance method, `this` refers to the specific object instance on which the method was called. This allows the method to access and modify the instance's properties.
- **Requires an Instance:** You must create an instance of the class before you can call an instance method.
- **Defined without `static` keyword:** They are defined directly within the class body without any special keywords.

**How it Works:** When you create a new object using `new ClassName`, a new instance is created. This instance inherits all the instance methods defined in its class. When an instance method is invoked (e.g., `myObject.methodName`), the JavaScript engine sets the `this` context to `myObject`, allowing the method to interact with `myObject`'s properties.

#

#

# 2. Static Methods

**Definition:** Static methods are functions that belong to the _class itself_, not to any specific instance of the class. They are sometimes referred to as "class methods."

**Key Characteristics:**

- **No Access to `this` (instance context):** Inside a static method, `this` typically refers to the class constructor itself, not an instance. Therefore, static methods cannot directly access or modify instance-specific properties.
- **No Instance Required:** You can call a static method directly on the class name without creating an object instance.
- **Defined with `static` keyword:** They are explicitly defined using the `static` keyword before the method name.
- **Utility Functions:** Often used for utility functions, factory methods (methods that create instances of the class), or methods that operate on class-level data.

**How it Works:** When a static method is invoked (e.g., `ClassName.staticMethodName`), the method operates independently of any specific object. It can access other static properties or call other static methods of the same class. Since they don't operate on instance data, they are ideal for functions that provide general functionality related to the class's purpose.

# **Related Concepts:**

---
- **Classes (`class` keyword):** Class methods are integral to the `class` syntax in JavaScript (introduced in ES6). They are the "behavior" part of a class definition, complementing the properties (data).
- **Objects/Instances:** Instance methods specifically operate on objects created from a class. The concept of an "instance" is fundamental to understanding how instance methods work.
- **`this` keyword:** Understanding how `this` behaves is crucial for both instance and static methods. In instance methods, `this` refers to the instance; in static methods, `this` refers to the class constructor itself.
- **Prototypes:** Behind the scenes, JavaScript classes are built on prototypes. Instance methods are actually added to the `ClassName.prototype` object, making them available to all instances via the prototype chain. Static methods are added directly to the `ClassName` constructor function object.
- **Constructor Functions (ES5):** Before ES6 classes, objects and their methods were primarily created using constructor functions and modifying their `prototype`. Instance methods were added to `Constructor.prototype`, and static methods were added directly to `Constructor`. Classes provide a more concise and syntactically clearer way to achieve the same.
- **Properties (Instance vs. Static):** Just like methods, classes can have instance properties (specific to each object) and static properties (belonging to the class itself). Methods often interact with these properties.

# **Examples:**

---
```js
// Define a simple Car class
class Car
{
 // Constructor method: special instance method called when a new object is created
 constructor(make, model, year)
 {
 this.make = make; // Instance property
 this.model = model; // Instance property
 this.year = year; // Instance property
 }

 //
---
Instance Methods
---
// These methods belong to individual Car objects.
 // They can access 'this' to interact with the object's properties.

 getCarInfo {
 // 'this' refers to the specific Car instance on which the method is called.
 return `This is a ${this.year} ${this.make} ${this.model}.`;
 }

 accelerate {
 console.log(`${this.make} ${this.model} is accelerating!`);
 }

 //
---
Static Methods
---
// These methods belong to the Car class itself, not to an individual Car object.
 // They are called directly on the class (e.g., Car.createDefaultCar).
 // They do not have access to 'this' in the context of an instance.

 static createDefaultCar {
 // Static method to create a standard Car instance.
 // Useful as a "factory method".
 return new Car("Generic", "Model X", 2020);
 }

 static isValidYear(year) {
 // Static utility method to validate a year.
 // It doesn't need any specific Car instance data.
 return year >= 1900 && year <= new Date.getFullYear + 1;
 }

 static compareCars(car1, car2) {
 // Static method that takes instances as arguments but doesn't operate on
 // 'this' as an instance itself. It performs a comparison operation.
 if (car1.year > car2.year) {
 return `${car1.make} is newer than ${car2.make}.`;
 } else if (car2.year > car1.year) {
 return `${car2.make} is newer than ${car1.make}.`;
 } else {
 return `${car1.make} and ${car2.make} are from the same year.`;
 }
 }
}

//
---
Using Instance Methods
---
const myCar = new Car("Toyota", "Camry", 2023);
console.log(myCar.getCarInfo); // Output: This is a 2023 Toyota Camry.
myCar.accelerate; // Output: Toyota Camry is accelerating!

const anotherCar = new Car("Honda", "Civic", 2022);
console.log(anotherCar.getCarInfo); // Output: This is a 2022 Honda Civic.

// Trying to call a static method on an instance will result in an error
// myCar.createDefaultCar; // TypeError: myCar.createDefaultCar is not a function

//
---

Using Static Methods
---
const defaultCar = Car.createDefaultCar;
console.log(defaultCar.getCarInfo); // Output: This is a 2020 Generic Model X.

console.log(Car.isValidYear(2025)); // Output: true
console.log(Car.isValidYear(1899)); // Output: false

console.log(Car.compareCars(myCar, anotherCar)); // Output: Toyota is newer than Honda.

// Trying to call an instance method on the class directly will result in an error
// Car.getCarInfo; // TypeError: Car.getCarInfo is not a function
```

# **Flashcards:**

---
What is an instance method in JavaScript classes?;; An instance method is a function that belongs to an individual object (instance) of a class and operates on that instance's specific data, using `this` to refer to the instance.

What is a static method in JavaScript classes?;; A static method is a function that belongs to the class itself, not to any specific instance. It is called directly on the class name and does not have access to instance-specific data via `this`.

When would you use an instance method versus a static method?;; Use an instance method when the behavior needs to operate on the specific data of an object. Use a static method for utility functions or operations that relate to the class as a whole, not a particular instance.