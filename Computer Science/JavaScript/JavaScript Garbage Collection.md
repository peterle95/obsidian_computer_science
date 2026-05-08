---
memory: to_finish
tags:
  - learned
language:
  - JavaScript
review-date:
last-reviewed: 2025-08-28
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-08-07
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
# **Core Explanation:**
---
Memory management in JavaScript is ==performed automatically and invisibly to us==. We create primitives, objects, functions… All that takes memory.

What happens when something is not needed any more? How does the JavaScript engine discover it and clean it up?

## Reachability

The main concept of memory management in JavaScript is _reachability_.

Simply put, “<mark style="background: #FF5582A6;">reachable</mark>” values are those that are accessible or usable somehow. They are <mark style="background: #FF5582A6;">guaranteed to be stored in memory</mark>.

1. There’s a base set of inherently reachable values, that cannot be deleted for obvious reasons.
    
    For instance:
    
    - The currently executing function, its local variables and parameters.
    - Other functions on the current chain of nested calls, their local variables and parameters.
    - Global variables.
    - (there are some other, internal ones as well)
    
    ==These values are called _roots_.==
    
2. Any other value is considered reachable if it’s reachable from a root by a reference or by a chain of references.
    
    For instance, if there’s an object in a global variable, and that object has a property referencing another object, _that_ object is considered reachable. And those that it references are also reachable. Detailed examples to follow.
    

There’s a background process in the JavaScript engine that is called [garbage collector](https://en.wikipedia.org/wiki/Garbage_collection_\(computer_science\)). It monitors all objects and removes those that have become unreachable.

## A simple example

Here’s the simplest example:

```javascript
// user has a reference to the object
let user = {
  name: "John"
};
```

<img src="assets/images/Screenshot 2025-07-23 113114.png" alt="JavaScript Garbage Collection example" style="width: 150px; height: auto;" />

Here the arrow depicts an object reference. The global variable `"user"` references the object `{name: "John"}` (we’ll call it John for brevity). The `"name"` property of John stores a primitive, so it’s painted inside the object.

If the value of `user` is overwritten, the reference is lost:

```javascript
user = null;
```


<img src="assets/images/Screenshot 2025-07-23 113552.png" alt="JavaScript Garbage Collection example" style="width: 200px; height: auto;" />

Now John becomes unreachable. There’s no way to access it, no references to it. Garbage collector will junk the data and free the memory.

## Two references

Now let’s imagine we copied the reference from `user` to `admin`:

```javascript
// user has a reference to the object
let user = {
  name: "John"
};

let admin = user;
```


<img src="assets/images/Screenshot 2025-07-23 113734.png" alt="JavaScript Garbage Collection example" style="width: 200px; height: auto;" />

Now if we do the same:

```javascript
user = null;
```

…Then the object is still reachable via `admin` global variable, so it must stay in memory. If we overwrite `admin` too, then it can be removed.

## Interlinked objects

Now a more complex example. The family:

```javascript
function marry(man, woman) {
  woman.husband = man;
  man.wife = woman;

  return {
    father: man,
    mother: woman
  }
}

let family = marry({
  name: "John"
}, {
  name: "Ann"
});
```

Function `marry` “marries” two objects by giving them references to each other and returns a new object that contains them both.

The resulting memory structure:

<img src="assets/images/Screenshot 2025-07-23 113845.png" alt="JavaScript Garbage Collection example" style="width: 350px; height: auto;" />

As of now, all objects are reachable.

Now let’s remove two references:

```javascript
delete family.father;
delete family.mother.husband;
```


<img src="assets/images/Screenshot 2025-07-23 114007.png" alt="JavaScript Garbage Collection example" style="width: 350px; height: auto;" />

It’s not enough to delete only one of these two references, because all objects would still be reachable.

But if we delete both, then we can see that John has no incoming reference any more:


<img src="assets/images/Screenshot 2025-07-23 114106.png" alt="JavaScript Garbage Collection example" style="width: 350px; height: auto;" />

Outgoing references do not matter. Only incoming ones can make an object reachable. So, John is now unreachable and will be removed from the memory with all its data that also became unaccessible.

After garbage collection:


<img src="assets/images/Screenshot 2025-07-23 114221.png" alt="JavaScript Garbage Collection example" style="width: 200px; height: auto;" />

## Unreachable island

It is possible that the whole island of interlinked objects becomes unreachable and is removed from the memory.

The source object is the same as above. Then:

```javascript
family = null;
```

The in-memory picture becomes:


<img src="assets/images/Screenshot 2025-07-23 114310.png" alt="JavaScript Garbage Collection example" style="width: 350px; height: auto;" />

This example demonstrates how important the concept of reachability is.

It’s obvious that John and Ann are still linked, both have incoming references. But that’s not enough.

The former `"family"` object has been unlinked from the root, there’s no reference to it any more, so the whole island becomes unreachable and will be removed.

## Internal algorithms

<mark style="background: #FF5582A6;">The basic garbage collection algorithm is called “mark-and-sweep”</mark>.

The following “garbage collection” steps are regularly performed:

- The garbage collector takes roots and “marks” (remembers) them.
- Then it visits and “marks” all references from them.
- Then it visits marked objects and marks _their_ references. All visited objects are remembered, so as not to visit the same object twice in the future.
- …And so on until every reachable (from the roots) references are visited.
- All objects except marked ones are removed.

For instance, let our object structure look like this:

<img src="assets/images/Screenshot 2025-07-23 114417.png" alt="JavaScript Garbage Collection example" style="width: 450px; height: auto;" />

We can clearly see an “unreachable island” to the right side. Now let’s see how “mark-and-sweep” garbage collector deals with it.

The first step marks the roots:

<img src="assets/images/Screenshot 2025-07-23 114522.png" alt="JavaScript Garbage Collection example" style="width: 450px; height: auto;" />

Then we follow their references and mark referenced objects:

<img src="assets/images/Screenshot 2025-07-23 114717.png" alt="JavaScript Garbage Collection example" style="width: 450px; height: auto;" />

…And continue to follow further references, while possible:

<img src="assets/images/Screenshot 2025-07-23 114830.png" alt="JavaScript Garbage Collection example" style="width: 450px; height: auto;" />

Now the objects that could not be visited in the process are considered unreachable and will be removed:

<img src="assets/images/Screenshot 2025-07-23 114917.png" alt="JavaScript Garbage Collection example" style="width: 450px; height: auto;" />

We can also imagine the process as spilling a huge bucket of paint from the roots, that flows through all references and marks all reachable objects. The unmarked ones are then removed.

That’s the concept of how garbage collection works. JavaScript engines apply many optimizations to make it run faster and not introduce any delays into the code execution.

Some of the optimizations:

- **Generational collection** – objects are split into two sets: “new ones” and “old ones”. In typical code, many objects have a short life span: they appear, do their job and die fast, so it makes sense to track new objects and clear the memory from them if that’s the case. Those that survive for long enough, become “old” and are examined less often.
- **Incremental collection** – if there are many objects, and we try to walk and mark the whole object set at once, it may take some time and introduce visible delays in the execution. So the engine splits the whole set of existing objects into multiple parts. And then clear these parts one after another. There are many small garbage collections instead of a total one. That requires some extra bookkeeping between them to track changes, but we get many tiny delays instead of a big one.
- **Idle-time collection** – the garbage collector tries to run only while the CPU is idle, to reduce the possible effect on the execution.

There exist other optimizations and flavours of garbage collection algorithms. As much as I’d like to describe them here, I have to hold off, because different engines implement different tweaks and techniques. And, what’s even more important, things change as engines develop, so studying deeper “in advance”, without a real need is probably not worth that. Unless, of course, it is a matter of pure interest, then there will be some links for you below.

## Summary

The main things to know:

- Garbage collection is performed automatically. We cannot force or prevent it.
- Objects are retained in memory while they are reachable.
- Being referenced is not the same as being reachable (from a root): a pack of interlinked objects can become unreachable as a whole, as we’ve seen in the example above.
# **Related Concepts:**
---

[[Javascript Lexical Environment]]

**Memory leaks in JavaScript**: Memory leaks occur when objects that are no longer needed remain in memory due to unintended references. ==Common causes include forgotten event listeners, closures capturing large objects, and circular references not handled by garbage collection.==

**Reference counting vs Mark-and-sweep**: Reference counting tracks how many references point to each object and removes objects with zero references. Mark-and-sweep (used by JavaScript) marks all reachable objects from roots and sweeps away unmarked ones. Reference counting fails with circular references, while mark-and-sweep handles them correctly.

**Weak references in JavaScript (WeakMap, WeakSet)**: Weak references don't prevent garbage collection of their target objects. WeakMap and WeakSet allow references to objects without forcing them to stay in memory, useful for caches, metadata storage, and preventing memory leaks in observer patterns.

**Memory profiling and heap snapshots**: Tools like Chrome DevTools allow developers to take heap snapshots to analyze memory usage, identify memory leaks, and understand object retention patterns. This helps optimize applications by finding objects that unexpectedly remain in memory.

**Generational garbage collection**: A GC optimization that divides objects into generations (young and old) based on their lifetime. New objects are frequently checked for collection (young generation), while objects that survive multiple collections are moved to the old generation and checked less frequently.

**Event listener cleanup and memory management**: Improper handling of event listeners can cause memory leaks when listeners remain attached to DOM elements even after they're removed from the document. Always removing event listeners when components are destroyed prevents these leaks.

# **Examples:**
---

```javascript
// EXAMPLE 1: Basic reachability demonstration
// This example shows how objects become unreachable and eligible for garbage collection

// Create an object that is reachable through the 'user' variable
let user = {
  name: "Alice",
  data: new Array(10000).fill("some data") // Large data to make memory usage noticeable
};

// At this point, the object is reachable from the 'user' variable (a root)
console.log(user.name); // "Alice"

// Now we make the object unreachable by removing all references to it
user = null;

// The large object is now unreachable and will be garbage collected
// We can't access it anymore, and its memory will be freed
// console.log(user.name); // Would throw an error: Cannot read property 'name' of null


// EXAMPLE 2: Multiple references to the same object
// This example shows that an object remains in memory as long as at least one reference exists

// Create an object with multiple references
let user1 = { 
  name: "Bob",
  details: { age: 30, job: "developer" }
};

// Create a second reference to the same object
let admin = user1;

// Even if we remove the first reference
user1 = null;

// The object is still reachable through 'admin' and won't be garbage collected
console.log(admin.name); // "Bob"

// Only when all references are removed will the object become eligible for garbage collection
admin = null;


// EXAMPLE 3: Circular references
// This example demonstrates how modern garbage collectors handle circular references

function createCircularReference() {
  let object1 = {};
  let object2 = {};
  
  // Create circular references between the objects
  object1.ref = object2;  // object1 references object2
  object2.ref = object1;  // object2 references object1
  
  // Return references to both objects
  return { obj1: object1, obj2: object2 };
}

// Create objects with circular references
let result = createCircularReference();

// Even though these objects reference each other, they're still reachable from 'result'
console.log(result.obj1.ref === result.obj2); // true
console.log(result.obj2.ref === result.obj1); // true

// Remove the external references
result = null;

// Now both objects form an "unreachable island" despite their circular references
// The garbage collector will identify and remove both objects


// EXAMPLE 4: Using WeakMap to prevent memory leaks
// This example shows how WeakMap helps prevent memory leaks when associating metadata with objects

// Regular Map would keep references to key objects even if they're no longer used elsewhere
let cache = new WeakMap();

function processUser(user) {
  // Check if we've processed this user before
  if (cache.has(user)) {
    console.log("Using cached data for", user.name);
    return cache.get(user);
  }
  
  // Process the user (expensive operation)
  let result = { processed: true, score: Math.random() * 100 };
  
  // Store the result in the cache
  // The user object is the key, but WeakMap won't prevent it from being garbage collected
  cache.set(user, result);
  
  return result;
}

// Create a user
let user2 = { name: "Charlie" };

// Process the user
let processedData = processUser(user2);
console.log(processedData);

// Process again - should use cache
processedData = processUser(user2);

// If we remove all references to the user
user2 = null;

// The WeakMap won't prevent garbage collection of the user object
// Both the user object and its associated data in the WeakMap become eligible for collection


// EXAMPLE 5: Memory leak through closures
// This example shows how closures can unintentionally keep large objects in memory

function createLeakyFunction() {
  // Large array that would be useful to garbage collect when no longer needed
  const largeData = new Array(1000000).fill("potentially large data");
  
  // This function captures the largeData in its closure
  function leakyFunction() {
    // Do something with one item
    console.log(largeData[0]);
  }
  
  return leakyFunction;
}

// This function only needs to access a single item from largeData
// but it keeps the entire array in memory due to the closure
const leaky = createLeakyFunction();

// Better approach: only keep what you need
function createEfficientFunction() {
  // Large array
  const largeData = new Array(1000000).fill("potentially large data");
  
  // Only keep the specific data we need
  const firstItem = largeData[0];
  
  function efficientFunction() {
    // Only use the specific item we saved
    console.log(firstItem);
  }
  
  return efficientFunction;
}

// This function only keeps the one item it needs in memory
const efficient = createEfficientFunction();
```

# **Flashcards:**
---

What is the main concept of memory management in JavaScript?;; Reachability - objects are kept in memory as long as they're reachable from roots through references or a chain of references.

What are considered "roots" in JavaScript garbage collection?;; Roots include currently executing functions and their variables, functions in the call stack, global variables, and internal JavaScript engine references.

How does the mark-and-sweep algorithm work?;; It starts by marking all roots, then follows and marks all references from them recursively. Any objects not marked after this process are considered unreachable and are removed from memory.

What happens to circular references in modern JavaScript engines?;; Modern JavaScript engines using mark-and-sweep can properly identify and collect objects with circular references when they become unreachable from roots.

What is an "unreachable island" in garbage collection?;; An unreachable island is a group of objects that reference each other but have no incoming references from roots. The entire island becomes eligible for garbage collection.

What are some optimizations used in JavaScript garbage collection?;; Optimizations include generational collection (separating new and old objects), incremental collection (processing objects in parts), and idle-time collection (running during CPU idle time).