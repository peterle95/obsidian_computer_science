---
tags:
 - learned
language:
 - JavaScript
review-date: ""
last-reviewed: 2025-08-18
scheda: done
visit-count: 5
confidence-level: 2
consecutive-correct: 3
last-struggle-date: 2025-07-11

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

# **Core Explanation:**

---
The `this` keyword in JavaScript refers to the execution context of a function. ==Its value is determined by *how* a function is called, not where it is defined. ==This dynamic behavior makes `this` a frequent source of confusion for developers. Understanding `this` is crucial for working with objects, events, and modern JavaScript features like [[JavaScript Arrow Functions (ES6)]]

# Difference between `this` in C++ vs JavaScript
---

# C++ `this`
- **Static binding**: Value is determined <mark style="background:

# FF5582A6;">at compile time</mark>
- **Always refers to**: The current instance of the class
- **Scope**: Only available within non-static member functions
- **Implementation**: Implicit pointer to the object on which the method is called
- **Consistency**: Always points to the same object throughout a method's execution

#

# JavaScript `this`
- **Dynamic binding**: Value is determined <mark style="background:

# FF5582A6;">at runtime</mark>
- **Context-dependent**: Value depends on how a function is called
- **Binding rules**:
> - In global context: References global object (window in browsers, global in Node.js)
> - In object methods: References the object that owns the method
> - In event handlers: References the element that received the event
> - In arrow functions: Inherits `this` from the enclosing lexical context
> - With call/apply/bind: Can be explicitly set
- **Flexibility**: Can change even within the same function based on execution context

#

# Key Differences

>1. C++ `this` is predictable and static; JavaScript `this` is dynamic and context-dependent
>2. C++ `this` is always a pointer; JavaScript `this` is a reference
>3. C++ `this` cannot be reassigned; JavaScript `this` can change based on how a function is invoked
>4. C++ `this` only exists in class methods; JavaScript `this` exists in any function

# Execution Context in JavaScript

---
Execution context is the environment in which JavaScript code is evaluated and executed. It consists of:

1. **Variable Environment**: Storage for variables and function declarations
2. **Scope Chain**: Access to variables outside the current function
3. **`this` Value**: Reference determined by how a function is called

#

# Types of Execution Contexts

1. **Global Execution Context**:
 - Created when script first loads
 - Provides the global object (window in browsers, global in Node.js)
 - Sets `this` to the global object in non-strict mode

2. **Function Execution Context**:
 - Created when a function is called
 - Each function call gets its own execution context
 - Determines the value of `this` based on invocation pattern

3. **Eval Execution Context**:
 - Created when code is executed within an eval function

#

# How Execution Context Affects `this`

The execution context directly determines what `this` references:
- In<mark style="background:

# FFF3A3A6;"> global context:</mark> `this` refers to the global object
- In <mark style="background:

# FF5582A6;">object methods:</mark> `this` refers to the object that called the method
- In <mark style="background:

# FFB86CA6;">event handlers:</mark> `this` refers to the element that received the event
- In <mark style="background:

# BBFABBA6;">constructors:</mark> `this` refers to the newly created instance
- In <mark style="background:

# D2B3FFA6;">arrow functions:</mark> `this` is lexically bound (inherited from parent scope)

Execution context is what makes JavaScript's `this` behavior so dynamic compared to C++'s static `this` pointer.

# **Related Concepts:**

---
* **Execution Context:** The environment in which a function is executed, which includes the value of `this`.
* **Global Object:** In web browsers, the `window` object; in Node.js, `global`. When `this` is used in the global scope or in a regular function call (non-strict mode), it refers to the global object.
* **Method Invocation:** When a function is called as a method of an object (e.g., `obj.method`), `this` inside the method refers to the object itself (`obj`).
* **Constructor Invocation:** When a function is called with the `new` keyword (e.g., `new MyObject`), `this` inside the constructor refers to the newly created instance of the object.
* **Explicit Binding:** Methods like `call`, `apply`, and `bind` allow you to explicitly set the value of `this` for a function.
 * `call` and `apply`: Immediately invoke the function with a specified `this` value. `call` takes arguments individually, while `apply` takes them as an array.
 * `bind`: Returns a *new function* with `this` permanently bound to a specified value, without immediately invoking it.
* **Arrow Functions:** Arrow functions (`\=>`) behave differently regarding `this`. They do not have their own `this` context; instead, they lexically inherit `this` from their enclosing scope (the scope in which they were defined). This makes them very useful for callbacks and methods where you want `this` to consistently refer to the surrounding object.
* **Strict Mode:** In strict mode (`'use strict';`), if `this` is not explicitly set, it will be `undefined` in regular function calls (rather than the global object).

# **Examples:**

---
```Javascript
// The 'this' keyword in JavaScript is highly dynamic and its value depends entirely
// on how the function containing 'this' is called (its execution context).

// 1. Global Context
// When 'this' is used in the global scope (outside of any function),
// it refers to the global object.
// In a web browser environment, the global object is `window`.
// In a Node.js environment, the global object is `global`.
console.log("
---
1. Global Context
---
");
console.log(this === window); // In a browser, this will log 'true'.
 // In Node.js, 'this' at the top level of a module is usually an empty object {}
 // or `exports`, not the global `global` object, due to module encapsulation.
 // For simplicity, most 'this' examples below will assume a browser context
 // where 'window' is the global object.

// 2. Function Context (Regular Function Call)
// When a function is called as a standalone function (not as a method of an object),
// 'this' refers to the global object (e.g., `window` in browsers) in non-strict mode.
// In strict mode (`'use strict';`), 'this' will be `undefined`.
console.log("\n
---
2. Function Context
---
");
function showThis {
 console.log("Inside showThis (non-strict):", this);
}
showThis; // Calling showThis directly. 'this' will be the global object (window in browsers).

function showThisStrict {
 'use strict'; // Enable strict mode for this function
 console.log("Inside showThisStrict (strict):", this);
}
showThisStrict; // In strict mode, when called as a standalone function, 'this' is `undefined`.

// 3. Method Context (Function as an Object Method)
// When a function is called as a method of an object, 'this' refers to the object
// that the method belongs to (the object on the left side of the dot operator).
console.log("\n
---
3. Method Context
---
");
const person = {
 name: "Alice", // A property of the person object
 greet: function { // A regular function defined as a method
 // When 'person.greet' is called, 'this' inside 'greet' will refer to 'person'.
 console.log("Hello, my name is", this.name);
 },
 // Arrow functions behave differently for 'this'. They do NOT have their own 'this'.
 // Instead, they lexically inherit 'this' from their *enclosing scope* (where they were defined).
 // In this case, 'introduce' is defined in the global scope (or module scope in Node.js),
 // so 'this' inside 'introduce' will refer to the global object (window).
 introduce: => {
 console.log("Introducing:", this); // 'this' here is the global object (window), NOT 'person'.
 // This is a common pitfall when using arrow functions as methods. It is inside an object, which is in the global scope. If it would be inside a function it would be different
 }
};
person.greet; // Output: "Hello, my name is Alice" ('this' is 'person')
person.introduce; // Output: "Introducing: Window" (or global object)

const anotherPerson = {
 name: "Bob"
};
// Assigning the 'greet' method from 'person' to 'anotherPerson'.
// The value of 'this' depends on *how* the function is called, not where it was originally defined.
anotherPerson.greet = person.greet;
anotherPerson.greet; // When 'anotherPerson.greet' is called, 'this' will refer to 'anotherPerson'.
 // Output: "Hello, my name is Bob"

// 4. Constructor Context (Using the 'new' Keyword)
// When a function is called with the `new` keyword, it acts as a constructor.
// A new empty object is created, and 'this' inside the constructor function
// refers to this newly created object.
console.log("\n
---
4. Constructor Context
---
");
function Car(make, model) {
 // 'this' refers to the new object being created (e.g., myCar).
 this.make = make; // Adds a 'make' property to the new object.
 this.model = model; // Adds a 'model' property to the new object.
 this.displayInfo = function {
 // 'this' here refers to the specific instance of the Car object (e.g., myCar).
 console.log(`Car: ${this.make} ${this.model}`);
 };
}
const myCar = new Car("Toyota", "Camry"); // 'new' keyword invokes Car as a constructor
myCar.displayInfo; // Output: "Car: Toyota Camry"

// 5. Explicit Binding (call, apply, bind)
// These methods allow you to explicitly set the value of 'this' for a function call.
console.log("\n
---
5. Explicit Binding
---
");
const dog = {
 name: "Fido",
 bark: function(sound) {
 console.log(`${this.name} barks ${sound}`);
 }
};

const cat = {
 name: "Whiskers"
};

// call: Invokes the function immediately. The first argument is the value for 'this',
// and subsequent arguments are passed individually to the function.
dog.bark.call(cat, "meow"); // 'this' inside 'bark' will be 'cat'. Output: "Whiskers barks meow"

// apply: Invokes the function immediately. The first argument is the value for 'this',
// and the second argument is an array (or array-like object) of arguments to be passed to the function.
dog.bark.apply(cat, ["purr"]); // 'this' inside 'bark' will be 'cat'. Output: "Whiskers barks purr"

// bind: Returns a *new function* with 'this' permanently bound to a specified value.
// It does not immediately invoke the function.
const catBark = dog.bark.bind(cat); // 'catBark' is now a new function where 'this' is always 'cat'.
catBark("hiss"); // When 'catBark' is called, 'this' inside the original 'bark' function (now 'catBark')
 // will always be 'cat'. Output: "Whiskers barks hiss"

// 6. Arrow Functions vs. Regular Functions with 'this' in Callbacks
// This is a crucial distinction, especially with asynchronous code like setTimeout.
// Arrow functions do *not* define their own 'this'. They inherit 'this' from the scope
// in which they were created (lexical 'this').
// Regular functions, however, define their own 'this' based on how they are called.
console.log("\n
---
6. Arrow Functions vs. Regular Functions
---
");
const counter = {
 count: 0,
 // Regular function method
 startRegular: function {
 console.log("Starting regular function counter...");
 // When this 'setTimeout' callback (a regular function) is executed,
 // it's called as a standalone function by the `window` (or `global`) object.
 // Therefore, 'this' inside this callback will refer to the global object.
 setTimeout(function {
 // console.log(this); // In browser, this would be 'Window'
 this.count++; // This attempts to increment a 'count' property on the global object,
 // not on the 'counter' object. This will likely lead to unexpected behavior
 // or an error if 'window.count' is not a number.
 console.log("Regular function counter (incorrect 'this'):", this.count);
 }, 100); // Small delay to illustrate asynchronous nature
 },
 // Arrow function method
 startArrow: function {
 console.log("Starting arrow function counter...");
 // An arrow function does not have its own 'this'. It captures 'this' from
 // its surrounding lexical scope. In this case, the surrounding scope is
 // the 'startArrow' method, where 'this' correctly refers to the 'counter' object.
 setTimeout( => {
 this.count++; // 'this' correctly refers to the 'counter' object.
 console.log("Arrow function counter (correct 'this'):", this.count);
 }, 200); // Slightly longer delay
 }
};

// To clearly see the difference, let's reset and run.
// Run this in a browser console for the clearest demonstration of 'window.count'.
// If run in Node.js, 'this' in the regular function might be `Timeout` object or `undefined` in strict mode.

// Demonstrate the problematic regular function behavior:
counter.count = 0; // Reset counter for this test
counter.startRegular; // This will modify 'window.count' (or similar global property), not 'counter.count'

// To fix the 'this' issue with regular functions in callbacks, you can use 'bind':
const counterBound = {
 count: 0,
 startBound: function {
 console.log("Starting bound regular function counter...");
 // Here, we explicitly 'bind' the 'this' context of the setTimeout callback
 // to the 'this' of the 'startBound' method (which is the 'counterBound' object).
 setTimeout(function {
 this.count++;
 console.log("Bound function counter (correct 'this'):", this.count);
 }.bind(this), 300); // '.bind(this)' creates a new function where 'this' is permanently 'counterBound'.
 }
};
counterBound.count = 0; // Reset for this test
counterBound.startBound; // This will correctly increment 'counterBound.count'

// Demonstrate the correct arrow function behavior:
counter.count = 0; // Reset counter for this test
counter.startArrow; // This will correctly increment 'counter.count' because arrow functions lexically bind 'this'.
```

# **Flashcards:**

---
What does the this keyword refer to in JavaScript?;; The this keyword refers to the execution context of a function; its value is determined by how the function is called.

How does (this) behave in a regular function call (non-strict mode)?;; In a regular function call (non-strict mode), this refers to the global object (e.g., window in browsers).

How does (this) behave in an arrow function?;; Arrow functions do not have their own this; they lexically inherit this from their enclosing scope.