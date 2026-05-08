---
memory: to_finish
tags:
  - learning
language:
  - JavaScript
review-date: 2025-11-25
last-reviewed: 2025-10-15
scheda: done
visit-count: 1
confidence-level: 1.5
consecutive-correct: 1
last-struggle-date:
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

# Core Explanation:
---

Let’s step away from the individual data structures and talk about the iterations over them.

In the previous chapter we saw methods `map.keys`, `map.values`, `map.entries`.

These methods are generic, there is a common agreement to use them for data structures. If we ever create a data structure of our own, we should implement them too.

They are supported for:

- `Map`
- `Set`
- `Array`

Plain objects also support similar methods, but the syntax is a bit different.

## Object.keys, values, entries

For plain objects, the following methods are available:

- Object.keys(obj) – returns an array of keys.
- Object.values(obj) – returns an array of values.
- Object.entries(obj) – returns an array of `[key, value]` pairs.

Please note the distinctions (compared to map for example):

|                 | Map          | Object                                   |
| --------------- | ------------ | ---------------------------------------- |
| **Call syntax** | `map.keys()` | `Object.keys(obj)`, but not `obj.keys()` |
| **Returns**     | Iterable     | "real" Array                             |

The first difference is that we have to call `Object.keys(obj)`, and not `obj.keys()`.

Why so? The main reason is flexibility. Remember, objects are a base of all complex structures in JavaScript. So we may have an object of our own like `data` that implements its own `data.values` method. And we still can call `Object.values(data)` on it.

The second difference is that `Object.*` methods return “real” array objects, not just an iterable. That’s mainly for historical reasons.

For instance:

```javascript
let user = {
 name: "John",
 age: 30
};
```

- `Object.keys(user) = ["name", "age"]`
- `Object.values(user) = ["John", 30]`
- `Object.entries(user) = [ ["name","John"], ["age",30] ]`

Here’s an example of using `Object.values` to loop over property values:

```javascript
let user = {
 name: "John",
 age: 30
};

// loop over values
for (let value of Object.values(user)) {
 alert(value); // John, then 30
}
```
### Object.\keys/values/entries ignore symbolic properties

Just like a `for..in` loop, ==these methods ignore properties that use `Symbol(...)` as keys==.

Usually that’s convenient. But if we want symbolic keys too, then there’s a separate method Object.getOwnPropertySymbols that returns an array of only symbolic keys. Also, there exist a method Reflect.ownKeys(obj) that returns _all_ keys.

## Transforming objects

Objects lack many methods that exist for arrays, e.g. `map`, `filter` and others.

If we’d like to apply them, then we can use `Object.entries` followed by `Object.fromEntries`:

1. Use `Object.entries(obj)` to get an array of key/value pairs from `obj`.
2. Use array methods on that array, e.g. `map`, to transform these key/value pairs.
3. Use `Object.fromEntries(array)` on the resulting array to turn it back into an object.

For example, we have an object with prices, and would like to double them:

```javascript
let prices = {
 banana: 1,
 orange: 2,
 meat: 4,
};

let doublePrices = Object.fromEntries(
 // convert prices to array, map each key/value pair into another pair
 // and then fromEntries gives back the object
 Object.entries(prices).map(entry => [entry, entry * 2])
);

alert(doublePrices.meat); // 8
```

It may look difficult at first sight, but becomes easy to understand after you use it once or twice. We can make powerful chains of transforms this way.

# **Related Concepts:**
---
- [[JavaScript Object Properties (Accessing, Modifying, Deleting)]]
- [[JavaScript Object `hasOwnProperty`]]
- [[JavaScript Object Prototypes and Prototypal Inheritance]]
- `for...in` loops vs array iteration
- Object property descriptors and enumerability
- Map constructor and object conversion

# **Examples:**
---
```javascript
const person = {
 name: 'Alice',
 age: 30,
 city: 'New York',
 occupation: 'Developer'
 // Number of legs: 4, <-- this is not allowed
 // "Number of legs": 4 <-- this is allowed
 // numberOfLegs: 4 <-- this is allowed
};

// Object.keys - Get all property names
const keys = Object.keys(person);
console.log(keys); // ['name', 'age', 'city', 'occupation']

// Object.values - Get all property values
const values = Object.values(person);
console.log(values); // ['Alice', 30, 'New York', 'Developer']

// Object.entries - Get key-value pairs
const entries = Object.entries(person);
console.log(entries);
// [['name', 'Alice'], ['age', 30], ['city', 'New York'], ['occupation', 'Developer']]

// Iterating with array methods
Object.keys(person).forEach(key => {
 console.log(`${key}: ${person[key]}`);
});

// Using entries with destructuring
Object.entries(person).forEach(([key, value]) => {
 console.log(`${key}: ${value}`);
});

// Filtering and transforming
const numbers = { a: 1, b: 2, c: 3, d: 4 };

// Filter values greater than 2
const filtered = Object.entries(numbers)
 .filter(([key, value]) => value > 2)
 .reduce((obj, [key, value]) => ({ ...obj, [key]: value }), {});
console.log(filtered); // { c: 3, d: 4 }

// Transform values
const doubled = Object.fromEntries(
 Object.entries(numbers).map(([key, value]) => [key, value * 2])
);
console.log(doubled); // { a: 2, b: 4, c: 6, d: 8 }

// Converting to Map
const personMap = new Map(Object.entries(person));
console.log(personMap.get('name')); // 'Alice'

// Getting property count
const propertyCount = Object.keys(person).length;
console.log(propertyCount); // 4

// With inheritance - only own properties
function Animal(name) {
 this.name = name;
}
Animal.prototype.species = 'Unknown';

const dog = new Animal('Rex');
dog.age = 3;

console.log(Object.keys(dog)); // ['name', 'age'] - no 'species'
console.log(Object.values(dog)); // ['Rex', 3] - no 'Unknown'
```

**Explanations:**

_The first three examples demonstrate the basic functionality: `Object.keys` extracts all property names into an array, `Object.values` extracts all property values, and `Object.entries` creates an array of [key, value] pairs._

_The iteration examples show two approaches: using keys to access values manually, versus using destructuring with entries for cleaner code when you need both key and value._

_The filtering and transforming examples demonstrate how to filter object properties based on values and transform all values while preserving the object structure using `Object.fromEntries`._

_The Map conversion uses entries directly since Map constructor accepts [key, value] pairs. Property count is easily obtained from the keys array length._

_The inheritance example demonstrates that these methods only return own properties, excluding inherited properties like 'species' from the prototype._

# **Flashcards:**

---
What do `Object.keys`, `Object.values`, and `Object.entries` return, and what type of properties do they include?;; `Object.keys` returns an array of property names, `Object.values` returns an array of property values, and `Object.entries` returns an array of [key, value] pairs. All three only include **own enumerable properties**, excluding inherited properties from the prototype chain.

How can you convert an object to a Map using these methods, and why is `Object.entries` particularly useful?;; Use `new Map(Object.entries(obj))` to convert an object to a Map. `Object.entries` is particularly useful because it returns [key, value] pairs that can be directly consumed by Map constructor, used with destructuring, or converted back to objects with `Object.fromEntries`.
<!--SR:!2025-06-15,1,230-->

How would you filter an object's properties based on their values and recreate the object with only the filtered properties?;; Use `Object.entries(obj).filter(([key, value]) => condition).reduce((newObj, [key, value]) => ({...newObj, [key]: value}), {})` or the more modern `Object.fromEntries(Object.entries(obj).filter(([key, value]) => condition))` to filter and reconstruct the object.
<!--SR:!2025-06-15,1,230-->