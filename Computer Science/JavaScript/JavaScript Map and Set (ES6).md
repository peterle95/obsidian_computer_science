---
memory: to_finish
tags:
  - learning
language:
  - JavaScript
review-date: 2025-11-25
last-reviewed: 2025-10-18
scheda: done
visit-count: 3
confidence-level: 1.5
consecutive-correct: 1
last-struggle-date: 2025-10-07
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

While Objects and Arrays are versatile, they have inherent limitations. ==**Object keys must be strings or symbols, and checking for the existence of an item in a large array can be slow. Map and Set were introduced in ES6 to provide more specialized and efficient data structures for handling keyed collections and unique values, respectively.**== They offer greater flexibility with key types, improved performance for frequent additions and deletions, and more straightforward methods for common operations.

# **Core Explanation:**
---

Till now, we’ve learned about the following complex data structures:

- Objects are used for storing keyed collections.
- Arrays are used for storing ordered collections.

But that’s not enough for real life. That’s why `Map` and `Set` also exist.
## Map

Map is a collection of keyed data items, just like an `Object`. ==But the main difference is that `Map` allows keys of any type.==

Methods and properties are:

- `new Map` – creates the map.
- `map.set(key, value)` – stores the value by the key.
- `map.get(key)` – returns the value by the key, `undefined` if `key` doesn’t exist in map.
- `map.has(key)` – returns `true` if the `key` exists, `false` otherwise.
- `map.delete(key)` – removes the element (the key/value pair) by the key.
- `map.clear` – removes everything from the map.
- `map.size` – returns the current element count.

For instance:
```javascript
let map = new Map;

map.set('1', 'str1'); // a string key
map.set(1, 'num1'); // a numeric key
map.set(true, 'bool1'); // a boolean key

// remember the regular Object? it would convert keys to string
// Map keeps the type, so these two are different:
alert( map.get(1) ); // 'num1'
alert( map.get('1') ); // 'str1'

alert( map.size ); // 3
```

As we can see, unlike objects, ==keys are not converted to strings. Any type of key is possible.==

`map[key]` isn’t the right way to use a `Map`

Although `map[key]` also works, e.g. we can set `map[key] = 2`, this is treating `map` as a plain JavaScript object, so it implies all corresponding limitations (only string/symbol keys and so on).

So we should use `map` methods: `set`, `get` and so on.

**Map can also use objects as keys**

For instance:

```javascript
let john = { name: "John" };

// for every user, let's store their visits count
let visitsCountMap = new Map;

// john is the key for the map
visitsCountMap.set(john, 123);

alert( visitsCountMap.get(john) ); // 123
```

Using objects as keys is one of the most notable and important `Map` features. The same does not count for `Object`. String as a key in `Object` is fine, but we can’t use another `Object` as a key in `Object`.

Let’s try:

```javascript
let john = { name: "John" };
let ben = { name: "Ben" };

let visitsCountObj = {}; // try to use an object

visitsCountObj[ben] = 234; // try to use ben object as the key
visitsCountObj[john] = 123; // try to use john object as the key, ben object will get replaced

// That's what got written!
alert( visitsCountObj["[object Object]"] ); // 123
```

As `visitsCountObj` is an object, it converts all `Object` keys, such as `john` and `ben` above, to same string `"[object Object]"`. Definitely not what we want.

How `Map` compares keys

To test keys for equivalence, `Map` uses the algorithm SameValueZero. It is roughly the same as strict equality `\===`, but the difference is that `NaN` is considered equal to `NaN`. So `NaN` can be used as the key as well.

This algorithm can’t be changed or customized.

Chaining

Every `map.set` call returns the map itself, so we can “chain” the calls:

```javascript
map.set('1', 'str1')
 .set(1, 'num1')
 .set(true, 'bool1');
```

### Iteration over Map

For looping over a `map`, there are 3 methods:

- `map.keys` – returns an iterable for keys,
- `map.values` – returns an iterable for values,
- `map.entries` – returns an iterable for entries `[key, value]`, it’s used by default in `for..of`.

For instance:

```javascript
let recipeMap = new Map([
 ['cucumber', 500],
 ['tomatoes', 350],
 ['onion', 50]
]);

// iterate over keys (vegetables)
for (let vegetable of recipeMap.keys) {
 alert(vegetable); // cucumber, tomatoes, onion
}

// iterate over values (amounts)
for (let amount of recipeMap.values) {
 alert(amount); // 500, 350, 50
}

// iterate over [key, value] entries
for (let entry of recipeMap) { // the same as of recipeMap.entries
 alert(entry); // cucumber,500 (and so on)
}
```

The insertion order is used

The iteration goes in the same order as the values were inserted. `Map` preserves this order, unlike a regular `Object`.

Besides that, `Map` has a built-in `forEach` method, similar to `Array`:

```javascript
// runs the function for each (key, value) pair
recipeMap.forEach( (value, key, map) => {
 alert(`${key}: ${value}`); // cucumber: 500 etc
});
```

### Object.entries: Map from Object

When a `Map` is created, we can pass an array (or another iterable) with key/value pairs for initialization, like this:

```javascript
// array of [key, value] pairs
let map = new Map([
 ['1', 'str1'],
 [1, 'num1'],
 [true, 'bool1']
]);

alert( map.get('1') ); // str1
```

If we have a plain object, and we’d like to create a `Map` from it, then we can use built-in method Object.entries(obj) that returns an array of key/value pairs for an object exactly in that format.

So we can create a map from an object like this:

```javascript
let obj = {
 name: "John",
 age: 30
};

let map = new Map(Object.entries(obj));

alert( map.get('name') ); // John
```

Here, `Object.entries` returns the array of key/value pairs: `[ ["name","John"], ["age", 30] ]`. That’s what `Map` needs.

### Object.fromEntries: Object from Map

We’ve just seen how to create `Map` from a plain object with `Object.entries(obj)`.

There’s `Object.fromEntries` method that does the reverse: given an array of `[key, value]` pairs, it creates an object from them:

```javascript
let prices = Object.fromEntries([
 ['banana', 1],
 ['orange', 2],
 ['meat', 4]
]);

// now prices = { banana: 1, orange: 2, meat: 4 }

alert(prices.orange); // 2
```

We can use `Object.fromEntries` to get a plain object from `Map`.

E.g. we store the data in a `Map`, but we need to pass it to a 3rd-party code that expects a plain object.

Here we go:

```javascript
let map = new Map;
map.set('banana', 1);
map.set('orange', 2);
map.set('meat', 4);

let obj = Object.fromEntries(map.entries); // make a plain object (*)

// done!
// obj = { banana: 1, orange: 2, meat: 4 }

alert(obj.orange); // 2
```

A call to `map.entries` returns an iterable of key/value pairs, exactly in the right format for `Object.fromEntries`.

We could also make line `(*)` shorter:

```javascript
let obj = Object.fromEntries(map); // omit .entries
```

That’s the same, because `Object.fromEntries` expects an iterable object as the argument. Not necessarily an array. And the standard iteration for `map` returns same key/value pairs as `map.entries`. So we get a plain object with same key/values as the `map`.

## Set

A `Set` is a special type collection – “set of values” (without keys), ==where each value may occur only once.==

Its main methods are:

- `new Set([iterable])` – creates the set, and if an `iterable` object is provided (usually an array), copies values from it into the set.
- `set.add(value)` – adds a value, returns the set itself.
- `set.delete(value)` – removes the value, returns `true` if `value` existed at the moment of the call, otherwise `false`.
- `set.has(value)` – returns `true` if the value exists in the set, otherwise `false`.
- `set.clear` – removes everything from the set.
- `set.size` – is the elements count.

The main feature is that repeated calls of `set.add(value)` with the same value don’t do anything. That’s the reason why each value appears in a `Set` only once.

For example, we have visitors coming, and we’d like to remember everyone. But repeated visits should not lead to duplicates. A visitor must be “counted” only once.

`Set` is just the right thing for that:

```javascript
let set = new Set;

let john = { name: "John" };
let pete = { name: "Pete" };
let mary = { name: "Mary" };

// visits, some users come multiple times
set.add(john);
set.add(pete);
set.add(mary);
set.add(john);
set.add(mary);

// set keeps only unique values
alert( set.size ); // 3

for (let user of set) {
 alert(user.name); // John (then Pete and Mary)
}
```

The alternative to `Set` could be an array of users, and the code to check for duplicates on every insertion using arr.find. But the performance would be much worse, because this method walks through the whole array checking every element. `Set` is much better optimized internally for uniqueness checks.

### Iteration over Set

We can loop over a set either with `for..of` or using `forEach`:

```javascript
let set = new Set(["oranges", "apples", "bananas"]);

for (let value of set) alert(value);

// the same with forEach:
set.forEach((value, valueAgain, set) => {
 alert(value);
});
```

Note the funny thing. The callback function passed in `forEach` has 3 arguments: a `value`, then _the same value_ `valueAgain`, and then the target object. Indeed, the same value appears in the arguments twice.

That’s for compatibility with `Map` where the callback passed `forEach` has three arguments. Looks a bit strange, for sure. But this may help to replace `Map` with `Set` in certain cases with ease, and vice versa.

The same methods `Map` has for iterators are also supported:

- `set.keys` – returns an iterable object for values,
- `set.values` – same as `set.keys`, for compatibility with `Map`,
- `set.entries` – returns an iterable object for entries `[value, value]`, exists for compatibility with `Map`.

## Summary

`Map` – is a collection of keyed values.

Methods and properties:

- `new Map([iterable])` – creates the map, with optional `iterable` (e.g. array) of `[key,value]` pairs for initialization.
- `map.set(key, value)` – stores the value by the key, returns the map itself.
- `map.get(key)` – returns the value by the key, `undefined` if `key` doesn’t exist in map.
- `map.has(key)` – returns `true` if the `key` exists, `false` otherwise.
- `map.delete(key)` – removes the element by the key, returns `true` if `key` existed at the moment of the call, otherwise `false`.
- `map.clear` – removes everything from the map.
- `map.size` – returns the current element count.

The differences from a regular `Object`:

- Any keys, objects can be keys.
- Additional convenient methods, the `size` property.

`Set` – is a collection of unique values.

Methods and properties:

- `new Set([iterable])` – creates the set, with optional `iterable` (e.g. array) of values for initialization.
- `set.add(value)` – adds a value (does nothing if `value` exists), returns the set itself.
- `set.delete(value)` – removes the value, returns `true` if `value` existed at the moment of the call, otherwise `false`.
- `set.has(value)` – returns `true` if the value exists in the set, otherwise `false`.
- `set.clear` – removes everything from the set.
- `set.size` – is the elements count.

Iteration over `Map` and `Set` is always in the insertion order, so we can’t say that these collections are unordered, but we can’t reorder elements or directly get an element by its number.

# **Related Concepts:**
---

- **[[JavaScript WeakMaps and WeakSets (ES6)]]**: These are special versions of Map and Set that hold "weak" references to their keys (for WeakMap) or values (for WeakSet), which must be objects.If an object stored in a WeakMap or WeakSet has no other references pointing to it, the garbage collector can remove it, which helps prevent memory leaks. This makes them ideal for associating metadata with objects without preventing their cleanup. They have a limited API and are not iterable, meaning they don't have methods like size, keys, or forEach.
    
- **Objects vs. Maps**:
    - **Keys**: Map keys can be of any type (including objects), whereas Object keys are limited to strings and symbols.
        
    - **Performance**: Map is generally more performant for frequent additions and deletions, especially with large datasets.Object may be slightly faster for a fixed number of properties that are only read frequently.
        
    - **Order**: Map preserves the insertion order of its elements, while the order of Object properties is not guaranteed.[
        
    - **Ergonomics**: Map has a size property and built-in methods like get, set, and has, while for Object you need to use methods like Object.keys() to get the size and you risk clashes with prototype properties.
        
- **Arrays vs. Sets**:
    - **Uniqueness**: A Set only stores unique values; it automatically handles duplicates. An Array can contain duplicate values.
        
    - **Performance**: Set is highly optimized for checking if a value exists using the .has() method (O(1) time complexity). In an Array, checking for existence with .includes() requires scanning the elements (O(n) time complexity), which is much slower for large arrays.
        
    - **Element Access**: Array elements are accessed by a numeric index. Set elements cannot be accessed by index.

# **Examples:**
---

```js
// --- Map Example: Caching Function Results ---

// A function that performs a "costly" operation, like a complex calculation or network request.
function slowCalculation(num) {
  // Simulate a delay
  for (let i = 0; i < 1e9; i++) {}
  return num * 2;
}

// Create a Map to act as a cache.
// The keys will be the function arguments, and the values will be the results.
const cache = new Map();

// A function that uses the cache to avoid re-calculating results.
function cachedCalculation(num) {
  // Check if the result for this 'num' is already in the cache.
  if (cache.has(num)) {
    // If it exists, return the cached value instantly.
    console.log(`Fetching result for ${num} from cache.`);
    return cache.get(num);
  }

  // If not in the cache, perform the slow calculation.
  console.log(`Performing slow calculation for ${num}...`);
  const result = slowCalculation(num);
  
  // Store the new result in the cache for future use.
  cache.set(num, result);
  
  // Return the newly calculated result.
  return result;
}

// First call: will be slow and perform the calculation.
console.log(cachedCalculation(5)); 
// Second call with the same argument: will be instantaneous as it hits the cache.
console.log(cachedCalculation(5)); 
// Third call with a new argument: will be slow again.
console.log(cachedCalculation(10));
```

```js
// --- Set Example: Removing Duplicates from an Array ---

// An array containing duplicate values.
const numbersWithDuplicates = [2, 3, 4, 4, 2, 3, 3, 4, 4, 5, 5, 6, 6, 7];

// To get an array with only unique values, we can convert this array into a Set.
// The Set constructor accepts an iterable (like an array) and automatically discards duplicates.
const uniqueNumbersSet = new Set(numbersWithDuplicates);

// The result is a Set object containing only the unique numbers.
console.log(uniqueNumbersSet); // Set { 2, 3, 4, 5, 6, 7 }

// It's a common pattern to convert the Set back into an Array if you need to use array methods.
// This can be done easily using the spread syntax (...) or Array.from().
const uniqueNumbersArray = [...uniqueNumbersSet];

// Now we have a new array with all duplicates removed.
console.log(uniqueNumbersArray); // [ 2, 3, 4, 5, 6, 7 ]
```
# **Flashcards:**
---
What is the main difference between Object and Map keys?;;Map allows keys of any data type, including objects, while Object keys can only be strings or symbols.

What is the primary purpose of a Set?;;To store a collection of unique values, where each value can only occur once.

How do you get the number of elements in a Map or a Set?;;Using the `.size` property.
Does a Map preserve the order of its elements?;;Yes, a Map iterates through elements in the order of their insertion.

How can you create a Map from an existing Object?;;By using `new Map(Object.entries(obj))`.
What method is used to check for the existence of a value in a Set?;;The `.has(value)` method, which returns true or false.