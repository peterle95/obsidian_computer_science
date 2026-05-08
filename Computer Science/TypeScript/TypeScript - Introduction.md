---
tags:
  - to_learn
language:
  - TypeScript
review-date:
last-reviewed: ""
scheda: to_finish
visit-count: 0
confidence-level: 1
consecutive-correct: 0
last-struggle-date: ""
cssclasses:
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
TypeScript stands in an unusual relationship to [[JavaScript]]. TypeScript offers all of JavaScript’s features, and an additional layer on top of these: TypeScript’s type system.

For example, JavaScript provides language primitives like `string` and `number`, but it doesn’t check that you’ve consistently assigned these. TypeScript does.

This means that your existing working JavaScript code is also TypeScript code. The main benefit of TypeScript is that it can highlight unexpected behavior in your code, lowering the chance of bugs.

This tutorial provides a brief overview of TypeScript, focusing on its type system.
 
# **Core Explanation:**
---
## Types by Inference

TypeScript knows the JavaScript language and will generate types for you in many cases. For example in creating a variable and assigning it to a particular value, TypeScript will use the value as its type.

```typescript
let helloWorld = "Hello World";           
	let helloWorld: string
```

By understanding how JavaScript works, TypeScript can build a type-system that accepts JavaScript code but has types. This offers a type-system without needing to add extra characters to make types explicit in your code. That’s how TypeScript knows that `helloWorld` is a `string` in the above example.

You may have written JavaScript in Visual Studio Code, and had editor auto-completion. ==Visual Studio Code uses TypeScript under the hood to make it easier to work with JavaScript.==

## Defining Types

==You can use a wide variety of design patterns in JavaScript. However, some design patterns make it difficult for types to be inferred automatically (for example, patterns that use dynamic programming). To cover these cases, TypeScript supports an extension of the JavaScript language, which offers places for you to tell TypeScript what the types should be.==

For example, to create an object with an inferred type which includes `name: string` and `id: number`, you can write:

```typescript
const user = {    
name: "Hayes",    
id: 0,  }; 
```

You can ==explicitly describe this object’s shape using an `interface` declaration:==

```typescript
interface User {    
name: string;  
  id: number;  } 
```

You can then <mark style="background: #FF5582A6;">declare that a JavaScript object conforms to the shape</mark> of your new `interface` by using syntax like `: TypeName` after a variable declaration:

```typescript
const user: User = {    
	name: "Hayes",    
	id: 0,  };
```

If you provide an object that doesn’t match the interface you have provided, TypeScript will warn you:

```typescript
interface User {    
	name: string;   
	 id: number; 
 }  
 
 const user: User = {    
 username: "Hayes",  // Error
 ---> Object literal may only specify known properties, and 'username' does not exist in type 'User'.    
 id: 0,  
 }; ```

Since JavaScript supports classes and object-oriented programming, so does TypeScript. You can use an interface declaration with classes:

```typescript

interface User {   
 name: string;   
  id: number;  
  } 
   
class UserAccount {    
name: string;    
id: number;    
	constructor(name: string, id: number) {      
		this.name = name;      
		this.id = id;   
		 }  
} 

const user: User = new UserAccount("Murphy", 1);
```

You can use interfaces to annotate parameters and return values to functions:

```typescript
function deleteUser(user: User) {   
 // ... 
}  

function getAdminUser(): User {    
//...  
}   
```

There is already a small set of primitive types available in JavaScript: `boolean`, `bigint`, `null`, `number`, `string`, `symbol`, and `undefined`, which you can use in an interface. ==TypeScript extends this list with a few more, such as `any` (allow anything), [`unknown`](https://www.typescriptlang.org/play#example/unknown-and-never) (ensure someone using this type declares what the type is), [`never`](https://www.typescriptlang.org/play#example/unknown-and-never) (it’s not possible that this type could happen), and `void` (a function which returns `undefined` or has no return value).==

You’ll see that there are two syntaxes for building types: [Interfaces and Types](https://www.typescriptlang.org/play/?e=83#example/types-vs-interfaces). You should prefer `interface`. Use `type` when you need specific features.

## Composing Types

With TypeScript, <mark style="background: #FFB8EBA6;">you can create complex types by combining simple ones</mark>. There are two popular ways to do so: <mark style="background: #FFB8EBA6;">unions and generics</mark>.

### Unions

With a union, <mark style="background: #BBFABBA6;">you can declare that a type could be one of many types</mark>. For example, you can describe a `boolean` type as being either `true` or `false`:

```typescript 
type MyBool = true | false;
```

_Note:_ If you hover over `MyBool` above, you’ll see that it is classed as `boolean`. That’s a property of the Structural Type System. More on this below.

==A popular use-case for union types is to describe the set of `string` or `number` [literals](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#literal-types) that a value is allowed to be:==

```typescript
type WindowStates = "open" | "closed" | "minimized";  
type LockStates = "locked" | "unlocked";  
type PositiveOddNumbersUnderTen = 1 | 3 | 5 | 7 | 9;  
```

Unions provide a way to handle different types too. For example, you may have a function that takes an `array` or a `string`:

```typescript
function getLength(obj: string | string[]) {    
	return obj.length;  
} 
```

To learn the type of a variable, use `typeof`:

|Type|Predicate|
|---|---|
|string|`typeof s === "string"`|
|number|`typeof n === "number"`|
|boolean|`typeof b === "boolean"`|
|undefined|`typeof undefined === "undefined"`|
|function|`typeof f === "function"`|
|array|`Array.isArray(a)`|

For example, you can make a function return different values depending on whether it is passed a string or an array:

```typescript
function wrapInArray(obj: string | string[]) {    
	if (typeof obj === "string") {     
			 return [obj];                
			 --> (parameter) obj: string    
			 }    
	return obj;  
}  
```

### Generics

==Generics provide variables to types==. A common example is an array. ==An array without generics could contain anything. An array with generics can describe the values that the array contains.==

```typescript
type StringArray = Array<string>;  
type NumberArray = Array<number>;  
type ObjectWithNameArray = Array<{ name: string }>; 
```

You can declare your own types that use generics:

```typescript
interface Backpack<Type> {    
	add: (obj: Type) => void;    
	get: () => Type;  
}  
// This line is a shortcut to tell TypeScript there is a  
// constant called `backpack`, and to not worry about where it came from.  declare const backpack: 
Backpack<string>;  
// object is a string, because we declared it above as the variable part of Backpack.  
const object = backpack.get(); 
 // Since the backpack variable is a string, you can't pass a number to the add function.  
 backpack.add(23);  
 --->Argument of type 'number' is not assignable to parameter of type 'string'.
```

## Structural Type System

One of TypeScript’s core principles is that type checking focuses on the ==_shape_ that values have. This is sometimes called “duck typing” or “structural typing”.==

In a structural type system, if two objects have the same shape, they are considered to be of the same type.

```typescript
interface Point {    
	x: number;    
	y: number;  
}  

function logPoint(p: Point) {    
	console.log(`${p.x}, ${p.y}`);  
}  
// logs "12, 26"  
const point = { 
	x: 12, 
	y: 26 
};  
logPoint(point);   
```

==The `point` variable is never declared to be a `Point` type. However, TypeScript compares the shape of `point` to the shape of `Point` in the type-check. They have the same shape, so the code passes.==

The shape-matching only requires a subset of the object’s fields to match.

```typescript

const point3 = { 
	x: 12, 
	y: 26, 
	z: 89 
};  
logPoint(point3); 
// logs "12, 26" 
const rect = { 
	x: 33, 
	y: 3, 
	width: 30, 
	height: 80 
}; 
 logPoint(rect); 
 // logs "33, 3"  
 const color = { 
	 hex: "#187ABF" 
};  
 logPoint(color);  
 --> Argument of type '{ hex: string; }' is not assignable to parameter of type 'Point'.   Type '{ hex: string; }' is missing the following properties from type 'Point': x, y`
 
 ```

There is no difference between how classes and objects conform to shapes:

```typescript

class VirtualPoint {    
	x: number;    
	y: number;    
		constructor(x: number, y: number) {      
			this.x = x;      
			this.y = y;    
		}  
}  
const newVPoint = new VirtualPoint(13, 56);  
logPoint(newVPoint); // logs "13, 56"  
```

If the object or class has all the required properties, TypeScript will say they match, regardless of the implementation details.

# **Memory Palace**
---
## **1. Chosen Location / Room**
_What palace location are you using (kitchen, hallway, bus stop, school entrance…)_

## **2. Loci (Memory Spots)**
_List 1–3 physical “spots” in that location that you will attach information to._
- Spot 1:  
- Spot 2:  
- Spot 3:  

## **3. Encoded Imagery / Story (Visual OR Non-Visual)**
_Describe the mnemonic you attach to each spot. This can be visual, verbal, symbolic, conceptual, or sensory._
- Spot 1 mnemonic:  
- Spot 2 mnemonic:  
- Spot 3 mnemonic:  

## **4. Retrieval Path**
_Write a clear retrieval route (e.g., “enter kitchen → sink → fridge → window”)._

# **Related Concepts:**
---
 _Identify and briefly explain any concepts that are closely related, dependent on, or analogous to the current topic. How do they connect or differ?_
# **Examples:**
---
_Provide practical code examples that illustrate the concept's application. Ensure the code is well-commented to explain each step and its relevance. The explanation has to be as comments in the code_

# **Flashcards:**
---
CREATE 6 FLASHCARDS REGARDING THIS NOTE
This is the format: FRONT TEXT;; BACK TEXT

**CRITICAL FLASHCARD FORMAT SPECIFICATION**

When creating flashcards, you MUST follow this exact format without exception:

**FORMAT: FRONT_TEXT;; BACK_TEXT**

**MANDATORY REQUIREMENTS:**
- Use exactly TWO semicolons (;;) as the separator between front and back text
- NO spaces around the semicolons
- Each flashcard must be on a single line
- Front text comes first, followed by ;;, then back text
- Do not add any additional formatting, bullets, numbers, or prefixes
- Do not wrap in quotes or code blocks

**EXAMPLES:**
What is React?;; A JavaScript library for building user interfaces
How do you import a component in React?;; Using import statement: import ComponentName from './path'
What does JSX stand for?;; JavaScript XML - a syntax extension for JavaScript

**WHAT NOT TO DO:**
❌ Front text: Back text
❌ Front text; Back text  
❌ 1. Front text;; Back text
❌ - Front text;; Back text
❌ "Front text";; "Back text"

**REMEMBER:** The double semicolon (;;) is the ONLY acceptable separator. Any other format will break the flashcard system. This format is non-negotiable and must be followed precisely for proper parsing and functionality.
