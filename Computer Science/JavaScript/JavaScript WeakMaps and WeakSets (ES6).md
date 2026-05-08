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
confidence-level: 1
consecutive-correct: 0
last-struggle-date: 2025-10-15
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
The fundamental problem that WeakMap and WeakSet solve is **==preventing memory leaks**.== Their primary application is to ==associate secondary data or metadata with an object without preventing that object from being garbage collected when it's no longer in use.==

In JavaScript, memory is managed automatically by a process called [[JavaScript Garbage Collection]]. An object is kept in memory as long as it is "reachable" (i.e., there is a reference to it somewhere in the code). ==Standard Map and Set objects create strong references to the objects used as keys. This means that even if all other references to an object are gone, it will **not** be garbage collected as long as it exists as a key in a Map or Set.==

WeakMap and WeakSet break this behavior by holding "weak" references. ==If an object is only referenced as a key in a WeakMap or WeakSet, the garbage collector can safely destroy it and remove it from memory, along with its associated value in the WeakMap. This is crucial for applications like caching or managing metadata for objects where the lifecycle of the object is controlled by other parts of the application (e.g., a third-party library or the DOM).==

# **Core Explanation:**
---

As we know from the chapter [[JavaScript Garbage Collection]], JavaScript engine keeps a value in memory while it is “reachable” and can potentially be used.

For instance:

```javascript
let john = { name: "John" };

// the object can be accessed, john is the reference to it

// overwrite the reference
john = null;

// the object will be removed from memory
```

Usually, properties of an object or elements of an array or another data structure are considered reachable and kept in memory while that data structure is in memory.

<mark style="background: #FF5582A6;">For instance, if we put an object into an array, then while the array is alive, the object will be alive as well, even if there are no other references to it.</mark>

Like this:

```javascript
let john = { name: "John" };

let array = [ john ];

john = null; // overwrite the reference

// the object previously referenced by john is stored inside the array
// therefore it won't be garbage-collected
// we can get it as array
```

==Similar to that, if we use an object as the key in a regular `Map`, then while the `Map` exists, that object exists as well. It occupies memory and may not be garbage collected.==

For instance:

```javascript
let john = { name: "John" };

let map = new Map;
map.set(john, "...");

john = null; // overwrite the reference

// john is stored inside the map,
// we can get it by using map.keys
```

`WeakMap` <mark style="background: #FF5582A6;">is fundamentally different in this aspect. It doesn’t prevent garbage-collection of key objects.</mark>

Let’s see what it means on examples.

## WeakMap

> The first difference between `Map` and `WeakMap` is that keys must be objects, not primitive values:

```javascript
let weakMap = new WeakMap;

let obj = {};

weakMap.set(obj, "ok"); // works fine (object key)

// can't use a string as the key
weakMap.set("test", "Whoops"); // Error, because "test" is not an object
```

<mark style="background: #BBFABBA6;">Now, if we use an object as the key in it, and there are no other references to that object – it will be removed from memory (and from the map) automatically.</mark>

```javascript
let john = { name: "John" };

let weakMap = new WeakMap;
weakMap.set(john, "...");

john = null; // overwrite the reference

// john is removed from memory!
```

Compare it with the regular `Map` example above. Now if `john` only exists as the key of `WeakMap` – it will be automatically deleted from the map (and memory).

`WeakMap` <mark style="background: #D2B3FFA6;">does not support iteration and methods</mark> `keys`, `values`, `entries`, so there’s no way to get all keys or values from it.

`WeakMap` has only the following methods:

- `weakMap.set(key, value)`
- `weakMap.get(key)`
- `weakMap.delete(key)`
- `weakMap.has(key)`

Why such a limitation? That’s for technical reasons. If an object has lost all other references (like `john` in the code above), then it is to be garbage-collected automatically. But technically it’s not exactly specified _when the cleanup happens_.

The JavaScript engine decides that. It may choose to perform the memory cleanup immediately or to wait and do the cleaning later when more deletions happen. So, technically, the current element count of a `WeakMap` is not known. The engine may have cleaned it up or not, or did it partially. For that reason, methods that access all keys/values are not supported.

Now, where do we need such a data structure?

### Use case: additional data

==The main area of application for `WeakMap` is an _additional data storage_.==

==If we’re working with an object that “belongs” to another code, maybe even a third-party library, and would like to store some data associated with it, that should only exist while the object is alive – then `WeakMap` is exactly what’s needed.==

We put the data to a `WeakMap`, using the object as the key, and when the object is garbage collected, that data will automatically disappear as well.

```javascript
weakMap.set(john, "secret documents");
// if john dies, secret documents will be destroyed automatically
```

Let’s look at an example.

For instance, we have code that keeps a visit count for users. The information is stored in a map: a user object is the key and the visit count is the value. When a user leaves (its object gets garbage collected), we don’t want to store their visit count anymore.

Here’s an example of a counting function with `Map`:

```javascript
// 📁 visitsCount.js
let visitsCountMap = new Map; // map: user => visits count

// increase the visits count
function countUser(user) {
 let count = visitsCountMap.get(user) || 0;
 visitsCountMap.set(user, count + 1);
}
```

And here’s another part of the code, maybe another file using it:

```javascript
// 📁 main.js
let john = { name: "John" };

countUser(john); // count his visits

// later john leaves us
john = null;
```

Now, `john` object should be garbage collected, but remains in memory, as it’s a key in `visitsCountMap`.

We need to clean `visitsCountMap` when we remove users, otherwise it will grow in memory indefinitely. Such cleaning can become a tedious task in complex architectures.

We can avoid it by switching to `WeakMap` instead:

```javascript
// 📁 visitsCount.js
let visitsCountMap = new WeakMap; // weakmap: user => visits count

// increase the visits count
function countUser(user) {
 let count = visitsCountMap.get(user) || 0;
 visitsCountMap.set(user, count + 1);
}
```

Now we don’t have to clean `visitsCountMap`. After `john` object becomes unreachable, by all means except as a key of `WeakMap`, it gets removed from memory, along with the information by that key from `WeakMap`.

### Use case: caching

Another common example is caching. ==We can store (“cache”) results from a function, so that future calls on the same object can reuse it.==

To achieve that, we can use `Map` (not optimal scenario):

```javascript
// 📁 cache.js
let cache = new Map;

// calculate and remember the result
function process(obj) {
 if (!cache.has(obj)) {
 let result = /* calculations of the result for */ obj;

 cache.set(obj, result);
 return result;
 }

 return cache.get(obj);
}

// Now we use process in another file:

// 📁 main.js
let obj = {/* let's say we have an object */};

let result1 = process(obj); // calculated

// ...later, from another place of the code...
let result2 = process(obj); // remembered result taken from cache

// ...later, when the object is not needed any more:
obj = null;

alert(cache.size); // 1 (Ouch! The object is still in cache, taking memory!)
```

For multiple calls of `process(obj)` with the same object, it only calculates the result the first time, and then just takes it from `cache`. ==The downside is that we need to clean `cache` when the object is not needed any more.==

If we replace `Map` with `WeakMap`, then this problem disappears. The cached result will be removed from memory automatically after the object gets garbage collected.

```javascript
// 📁 cache.js
let cache = new WeakMap;

// calculate and remember the result
function process(obj) {
 if (!cache.has(obj)) {
 let result = /* calculate the result for */ obj;

 cache.set(obj, result);
 return result;
 }

 return cache.get(obj);
}

// 📁 main.js
let obj = {/* some object */};

let result1 = process(obj);
let result2 = process(obj);

// ...later, when the object is not needed any more:
obj = null;

// Can't get cache.size, as it's a WeakMap,
// but it's 0 or soon be 0
// When obj gets garbage collected, cached data will be removed as well
```

## WeakSet

`WeakSet` behaves similarly:

- It is analogous to `Set`, but we may only add objects to `WeakSet` (not primitives).
- An object exists in the set while it is reachable from somewhere else.
- Like `Set`, it supports `add`, `has` and `delete`, but not `size`, `keys` and no iterations.

Being “weak”, it also serves as additional storage. But not for arbitrary data, rather for “yes/no” facts. A membership in `WeakSet` may mean something about the object.

For instance, we can add users to `WeakSet` to keep track of those who visited our site:

```javascript
let visitedSet = new WeakSet;

let john = { name: "John" };
let pete = { name: "Pete" };
let mary = { name: "Mary" };

visitedSet.add(john); // John visited us
visitedSet.add(pete); // Then Pete
visitedSet.add(john); // John again

// visitedSet has 2 users now

// check if John visited?
alert(visitedSet.has(john)); // true

// check if Mary visited?
alert(visitedSet.has(mary)); // false

john = null;

// visitedSet will be cleaned automatically
```

The most notable limitation of `WeakMap` and `WeakSet` is the absence of iterations, and the inability to get all current content. That may appear inconvenient, but does not prevent `WeakMap/WeakSet` from doing their main job – be an “additional” storage of data for objects which are stored/managed at another place.

## Summary

`WeakMap` is `Map`-like collection that allows only objects as keys and removes them together with associated value once they become inaccessible by other means.

`WeakSet` is `Set`-like collection that stores only objects and removes them once they become inaccessible by other means.

Their main advantages are that they have weak reference to objects, so they can easily be removed by garbage collector.

That comes at the cost of not having support for `clear`, `size`, `keys`, `values`…

`WeakMap` and `WeakSet` are used as “secondary” data structures in addition to the “primary” object storage. Once the object is removed from the primary storage, if it is only found as the key of `WeakMap` or in a `WeakSet`, it will be cleaned up automatically.

# **Related Concepts:**
---

- **Garbage Collection:** This is the foundational mechanism that makes WeakMap and WeakSet useful. Garbage collection is the process by which the JavaScript engine automatically identifies and frees up memory that is no longer being used by the application. WeakMap and WeakSet leverage this process by not preventing the collection of their keys.
    
- **Strong vs. Weak References:** This is the core difference.
    
    - **Strong Reference**: A standard reference from one object to another. As long as a strong reference to an object exists, it cannot be garbage collected. Map and Set keys are strongly referenced.
        
    - **Weak Reference**: A reference that does not protect an object from garbage collection. If an object's only remaining references are weak, it can be collected. This is the type of reference WeakMap and WeakSet use for their keys.
        
- **[[JavaScript Map and Set (ES6)]]:** These are the direct, strongly-referenced counterparts to WeakMap and WeakSet.
    
    - **Difference:** Map/Set can hold keys of any type (including primitives), are iterable, and have a .size property. WeakMap/WeakSet only accept objects as keys, are not iterable, and have no .size property, precisely because their contents can disappear at any time due to garbage collection.
        
- **WeakRef and FinalizationRegistry:** These are lower-level JavaScript features that provide more direct control over weak referencing and cleanup.
    
    - **WeakRef**: An object that lets you create a direct weak reference to another object. You must use a .deref() method to access the object, which may return undefined if it has been garbage collected.
        
    - **FinalizationRegistry**: Lets you register a callback to be invoked after an object has been garbage collected.
        
    - **Connection:** WeakMap and WeakSet can be thought of as high-level, optimized abstractions built upon the same principles that govern WeakRef and FinalizationRegistry.
        

# **Examples:**
---

```js
//== Use Case: Associating Metadata with DOM Elements ==
// We want to attach some metadata to DOM elements. If we use a regular Map,
// and later remove a DOM element from the page, the Map will still hold a
// reference to it, preventing it from being garbage collected and creating a memory leak.
// A WeakMap solves this perfectly.

// Create a WeakMap to store metadata for our DOM elements.
const elementMetadata = new WeakMap();

// Select two button elements from the DOM.
let button1 = document.querySelector('#button1');
let button2 = document.querySelector('#button2');

// Check if the buttons exist before proceeding.
if (button1 && button2) {
    // Set some metadata for each button. The key is the DOM element object itself.
    elementMetadata.set(button1, { clickCount: 0, id: 'button1' });
    elementMetadata.set(button2, { lastClick: null, id: 'button2' });

    // Simulate a click on button 1.
    // We can retrieve the metadata object and update it.
    let meta1 = elementMetadata.get(button1);
    meta1.clickCount++;
    console.log(elementMetadata.get(button1)); // { clickCount: 1, id: 'button1' }

    // Now, let's simulate removing button1 from the DOM.
    // In a real application, this could happen due to user interaction or a re-render.
    console.log("Removing button 1 from the DOM...");
    button1.remove();

    // Overwrite the variable that was holding a direct reference to the button.
    // Now, the only remaining reference to the button object is the *weak* one inside our WeakMap.
    button1 = null;

    // At this point, the garbage collector is free to destroy the button1 object.
    // When it does, the key-value pair will also be automatically removed from the elementMetadata WeakMap.
    // We cannot directly check the size of the WeakMap, but we can be sure that the memory
    // will be reclaimed, preventing a leak. If we had used a regular Map, the entry for button1
    // would persist in memory indefinitely.
}
```

# **Flashcards:**
---
What is the fundamental difference between Map and WeakMap?;;Map holds strong references to its keys, preventing garbage collection, while WeakMap holds weak references, allowing keys to be garbage collected.  

What types of keys are permitted in a WeakMap?;;Only objects are allowed as keys; primitive values will cause an error.  

Why can't you iterate over a WeakMap or check its size?;;Because the garbage collector can remove entries at any time, making the contents unpredictable and the size unreliable.  

What happens to a value in a WeakMap when its corresponding key object is garbage collected?;;The value is also removed from the WeakMap automatically, preventing memory leaks. 

What is a primary use case for WeakMap?;;Storing metadata or caching results for an object without preventing the object from being removed from memory.  

How does WeakSet differ from Set?;;WeakSet can only store objects, holds weak references to them, and is not iterable, whereas Set can store any value, holds strong references, and is iterable.
