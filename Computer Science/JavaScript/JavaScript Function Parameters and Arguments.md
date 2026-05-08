---
memory: to_finish
tags:
  - learned
language:
  - JavaScript
review-date: ""
last-reviewed: 2025-09-04
scheda: done
visit-count: 2
confidence-level: 1.5
consecutive-correct: 2
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
Function parameters and arguments are essential because they enable **functions to be flexible and dynamic**. They solve the problem of making functions operate on different sets of data without needing to rewrite the function's core logic each time. By allowing external data to be passed into a function, they empower functions to be truly **reusable** and adaptable to various situations, transforming static code blocks into versatile tools that can process diverse inputs. This is critical for building modular, generalized, and efficient programs where functions can perform the same operation on different pieces of information.

# **Core Explanation:**

---

In JavaScript functions, **parameters** and **arguments** are terms that describe the inputs a function receives. While often used interchangeably in casual conversation, they refer to different aspects of the input mechanism:

* **Parameters**: These are named variables listed in the function's definition. They act as **placeholders** for the values that the function expects to receive when it is called. Parameters are defined within the parentheses `` of the `function` declaration. Inside the function body, parameters behave like local variables, holding the values passed in.

 ```javascript
 // 'param1' and 'param2' are parameters
 function myFunction(param1, param2) {
 // ...
 }
 ```

* **Arguments**: These are the actual **values** that are passed to the function when it is invoked or called. Arguments are supplied within the parentheses `` during the function call, and they are mapped to the corresponding parameters in the order they are listed.

 ```javascript
 // 'valueA' and 'valueB' are arguments
 myFunction(valueA, valueB);
 ```

**How they work:**
1. When a function is called, JavaScript takes the arguments provided and assigns them to the function's parameters based on their position. The first argument maps to the first parameter, the second argument to the second parameter, and so on.
2. Inside the function, these parameters behave as local variables, initialized with the values of their corresponding arguments.
3. **Passing by Value vs. Passing by Reference**:
 * **Primitives** (numbers, strings, booleans, null, undefined, symbols, bigints) are **passed by value**. This means a copy of the argument's value is passed to the parameter. Changes to the parameter inside the function do not affect the original variable outside the function.
 * **Objects** (and arrays, which are a type of object) are **passed by reference** (or more accurately, by *sharing*). This means a copy of the *reference* (memory address) to the object is passed to the parameter. If the object's properties are modified inside the function, these changes will affect the original object outside the function because both the original variable and the parameter point to the same object in memory. However, reassigning the parameter to a *new* object inside the function will not affect the original variable.
4. **Missing Arguments**: If fewer arguments are provided than there are parameters, the missing parameters will be initialized with the value `undefined`.
5. **Extra Arguments**: If more arguments are provided than there are parameters, the extra arguments are ignored by the named parameters, but they can still be accessed inside the function using the special `arguments` object (though newer features like Rest Parameters are generally preferred).

# **Related Concepts:**

---
* **[[JavaScript Functions]]**: Parameters and arguments are fundamental to functions, enabling their reusability and ability to process dynamic data. They are the primary means by which data flows *into* a function.
* **Variables**: Parameters themselves act as local variables within the function's scope. Understanding how variables are declared, assigned, and scoped is essential for effectively using parameters and arguments.
* **Scope**: Parameters exist within the local scope of the function. Their values are accessible only within that function's body, and they do not conflict with variables of the same name outside the function.
* **Data Types**: The type of data being passed as an argument (e.g., number, string, object) influences how it behaves inside the function, particularly regarding passing by value vs. passing by reference.
* **[[JavaScript Default Parameters (ES6)]]**: This modern feature allows you to specify a default value for a parameter directly in the function's definition. This value is used if the argument for that parameter is `undefined` or omitted when the function is called, providing a cleaner alternative to manual `if` checks inside the function.
* **[[JavaScript Rest Parameters and Spread Syntax (ES6)]]**: Rest parameters (`...rest`) allow a function to accept an indefinite number of arguments as an array. This is a modern and preferred way to handle functions that take a variable number of inputs, largely replacing the older `arguments` object. Spread syntax (`...array`) is related but used to expand an iterable (like an array) into individual elements.

# **Examples:**

---
```javascript
//
---
Basic Parameters and Arguments
---
// Define a function 'greetUser' with one parameter: 'userName'
function greetUser(userName) {
 console.log(`Hello, ${userName}!`); // 'userName' is a local variable holding the passed argument
}

// Call 'greetUser' with "Sarah" as the argument
greetUser("Sarah"); // Output: Hello, Sarah!

// Call 'greetUser' with "Mike"
greetUser("Mike"); // Output: Hello, Mike!

//
---
Multiple Parameters and Argument Order
---
// Define a function 'calculateArea' with two parameters: 'length' and 'width'
function calculateArea(length, width) {
 let area = length * width; // Uses the values of length and width
 console.log(`The area is: ${area}`);
}

// Arguments are mapped by their position
calculateArea(10, 5); // Output: The area is: 50 (length=10, width=5)
calculateArea(7, 8); // Output: The area is: 56 (length=7, width=8)

//
---
What happens with Missing or Extra Arguments
---
// Function with three parameters
function describePerson(name, age, city) {
 console.log(`${name} is ${age} years old and lives in ${city}.`);
}

// Too few arguments: 'city' will be undefined
describePerson("Emma", 28); // Output: Emma is 28 years old and lives in undefined.

// Too many arguments: 'occupation' argument is ignored by the defined parameters
describePerson("David", 40, "New York", "Engineer"); // Output: David is 40 years old and lives in New York.

//
---
Passing by Value (for Primitives)
---
let score = 100; // A primitive number

function modifyScore(s) {
 s = s + 50; // 's' is a copy of 'score'. Modifying 's' does not affect 'score'.
 console.log(`Inside function, score is: ${s}`);
}

modifyScore(score); // Output: Inside function, score is: 150
console.log(`Outside function, score is: ${score}`); // Output: Outside function, score is: 100 (original 'score' remains unchanged)

//
---
Passing by Reference (for Objects/Arrays)
---
let userProfile = {
 name: "Jane",
 level: 1
}; // An object

function updateProfile(profile) {
 profile.level++; // Modifies a property of the object pointed to by 'profile'
 profile.isActive = true; // Adds a new property to the object
 console.log("Inside function, profile:", profile);
}

updateProfile(userProfile);
// Output: Inside function, profile: { name: 'Jane', level: 2, isActive: true }

console.log("Outside function, userProfile:", userProfile);
// Output: Outside function, userProfile: { name: 'Jane', level: 2, isActive: true }
// The original object 'userProfile' was modified because 'profile' and 'userProfile' refer to the same object.

// What happens if you reassign the parameter inside the function
function reassignObject(obj) {
 obj = {
 newProperty: "new value"
 }; // 'obj' now points to a NEW object, not the original 'myObj'
 console.log("Inside reassignObject, obj:", obj);
}

let myObj = {
 oldProperty: "old value"
};
reassignObject(myObj); // Output: Inside reassignObject, obj: { newProperty: 'new value' }
console.log("Outside reassignObject, myObj:", myObj); // Output: Outside reassignObject, myObj: { oldProperty: 'old value' }
// 'myObj' was not affected because the parameter 'obj' was reassigned to a different object.

//
---

The 'arguments' Object (Older approach for variable arguments)
---
// 'arguments' is an array-like object (not a true array) containing all arguments passed to the function
function sumAllArguments {
 let total = 0;
 for (let i = 0; i < arguments.length; i++) {
 total += arguments[i]; // Access arguments by index
 }
 console.log(`Sum using arguments object: ${total}`);
}

sumAllArguments(1, 2, 3); // Output: Sum using arguments object: 6
sumAllArguments(10, 20, 30, 40); // Output: Sum using arguments object: 100

// Note: For modern JavaScript, Rest Parameters (...args) are preferred for this use case.
````

# **Flashcards:**

---
What is the difference between a function parameter and an argument?;;Parameters are placeholders in the function definition; arguments are the actual values passed during the function call.

Are primitive data types passed by value or by reference in JavaScript functions?;;They are passed by value, meaning a copy of the value is made, so changes inside the function do not affect the original.

Are objects passed by value or by reference in JavaScript functions?;;Objects are passed by reference (more accurately, by sharing). Modifying object properties inside the function affects the original object outside.