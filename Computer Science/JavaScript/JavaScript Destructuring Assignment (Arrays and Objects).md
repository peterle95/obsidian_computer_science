---
memory: to_finish
tags:
  - to_learn
language:
  - JavaScript
review-date: 2025-11-20
last-reviewed: ""
scheda: done
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
Destructuring assignment provides a more concise and readable syntax for extracting data from arrays and objects. Its primary application is to "unpack" values from data structures into distinct variables, which simplifies code and avoids repetitive property access (e.g., object.property).

This is particularly important for a few key reasons:

- **Improved Readability and Maintainability:** It makes code cleaner and easier to understand by clearly stating what data is being extracted at the beginning of a statement.

- **Efficient Data Handling:** It allows functions to selectively extract only the required fields from an object or elements from an array, rather than processing the entire structure.

- **Simplified Function Arguments:** It is highly effective for handling function parameters, especially when dealing with functions that have many optional arguments. By passing a single "options" object, the function can destructure it to access the parameters it needs, assign default values, and ignore the order of arguments, making the function's signature cleaner and the call sites less error-prone.

# **Core Explanation:**
---

The two most used data structures in JavaScript are `Object` and `Array`.

- Objects allow us to create a single entity that stores data items by key.
- Arrays allow us to gather data items into an ordered list.

However, when we pass these to a function, we may not need all of it. The function might only require certain elements or properties.

_Destructuring assignment_ is a special syntax that allows us to “unpack” arrays or objects into a bunch of variables, as sometimes that’s more convenient.

Destructuring also works well with complex functions that have a lot of parameters, default values, and so on. Soon we’ll see that.

## Array destructuring

Here’s an example of how an array is destructured into variables:

```javascript
// we have an array with a name and surname
let arr = ["John", "Smith"]

// destructuring assignment
// sets firstName = arr
// and surname = arr
let [firstName, surname] = arr;

alert(firstName); // John
alert(surname); // Smith
```

Now we can work with variables instead of array members.

It looks great when combined with `split` or other array-returning methods:

```javascript
let [firstName, surname] = "John Smith".split(' ');
alert(firstName); // John
alert(surname); // Smith
```

As you can see, the syntax is simple. There are several peculiar details though. Let’s see more examples to understand it better.

“Destructuring” does not mean “destructive”.

It’s called “destructuring assignment,” because it “destructurizes” by copying items into variables. However, the array itself is not modified.

It’s just a shorter way to write:

```javascript
// let [firstName, surname] = arr;
let firstName = arr;
let surname = arr;
```

Ignore elements using commas

Unwanted elements of the array can also be thrown away via an extra comma:

```javascript
// second element is not needed
let [firstName, , title] = ["Julius", "Caesar", "Consul", "of the Roman Republic"];

alert( title ); // Consul
```

In the code above, the second element of the array is skipped, the third one is assigned to `title`, and the rest of the array items are also skipped (as there are no variables for them).

Works with any iterable on the right-side

…Actually, we can use it with any iterable, not only arrays:

```javascript
let [a, b, c] = "abc"; // ["a", "b", "c"]
let [one, two, three] = new Set([1, 2, 3]);
```

That works, because internally a destructuring assignment works by iterating over the right value. It’s a kind of syntax sugar for calling `for..of` over the value to the right of `=` and assigning the values.

Assign to anything at the left-side

We can use any “assignables” on the left side.

For instance, an object property:

```javascript
let user = {};
[user.name, user.surname] = "John Smith".split(' ');

alert(user.name); // John
alert(user.surname); // Smith
```

Looping with .entries

In the previous chapter, we saw the Object.entries(obj) method.

We can use it with destructuring to loop over the keys-and-values of an object:

```javascript
let user = {
 name: "John",
 age: 30
};

// loop over the keys-and-values
for (let [key, value] of Object.entries(user)) {
 alert(`${key}:${value}`); // name:John, then age:30
}
```

The similar code for a `Map` is simpler, as it’s iterable:

```javascript
let user = new Map;
user.set("name", "John");
user.set("age", "30");

// Map iterates as [key, value] pairs, very convenient for destructuring
for (let [key, value] of user) {
 alert(`${key}:${value}`); // name:John, then age:30
}
```

Swap variables trick

There’s a well-known trick for swapping values of two variables using a destructuring assignment:

```javascript
let guest = "Jane";
let admin = "Pete";

// Let's swap the values: make guest=Pete, admin=Jane
[guest, admin] = [admin, guest];

alert(`${guest} ${admin}`); // Pete Jane (successfully swapped!)
```

Here we create a temporary array of two variables and immediately destructure it in swapped order.

We can swap more than two variables this way.

## The rest ‘…’

Usually, if the array is longer than the list at the left, the “extra” items are omitted.

For example, here only two items are taken, and the rest is just ignored:

```javascript
let [name1, name2] = ["Julius", "Caesar", "Consul", "of the Roman Republic"];

alert(name1); // Julius
alert(name2); // Caesar
// Further items aren't assigned anywhere
```

If we’d like also to gather all that follows – we can add one more parameter that gets “the rest” using three dots `"..."`:

```javascript
let [name1, name2, ...rest] = ["Julius", "Caesar", "Consul", "of the Roman Republic"];

// rest is an array of items, starting from the 3rd one
alert(rest); // Consul
alert(rest); // of the Roman Republic
alert(rest.length); // 2
```

The value of `rest` is the array of the remaining array elements.

We can use any other variable name in place of `rest`, just make sure it has three dots before it and goes last in the destructuring assignment.

```javascript
let [name1, name2, ...titles] = ["Julius", "Caesar", "Consul", "of the Roman Republic"];
// now titles = ["Consul", "of the Roman Republic"]
```

## Default values

If the array is shorter than the list of variables on the left, there will be no errors. Absent values are considered undefined:

```javascript
let [firstName, surname] = ;

alert(firstName); // undefined
alert(surname); // undefined
```

If we want a “default” value to replace the missing one, we can provide it using `/=`:

```javascript
// default values
let [name = "Guest", surname = "Anonymous"] = ["Julius"];

alert(name); // Julius (from array)
alert(surname); // Anonymous (default used)
```

Default values can be more complex expressions or even function calls. They are evaluated only if the value is not provided.

For instance, here we use the `prompt` function for two defaults:

```javascript
// runs only prompt for surname
let [name = prompt('name?'), surname = prompt('surname?')] = ["Julius"];

alert(name); // Julius (from array)
alert(surname); // whatever prompt gets
```

Please note: the `prompt` will run only for the missing value (`surname`).

## Object destructuring

The destructuring assignment also works with objects.

The basic syntax is:

```javascript
let {var1, var2} = {var1:…, var2:…}
```

We should have an existing object on the right side, that we want to split into variables. The left side contains an object-like “pattern” for corresponding properties. In the simplest case, that’s a list of variable names in `{...}`.

For instance:

```javascript
let options = {
 title: "Menu",
 width: 100,
 height: 200
};

let {title, width, height} = options;

alert(title); // Menu
alert(width); // 100
alert(height); // 200
```

Properties `options.title`, `options.width` and `options.height` are assigned to the corresponding variables.

The order does not matter. This works too:

```javascript
// changed the order in let {...}
let {height, width, title} = { title: "Menu", height: 200, width: 100 }
```

The pattern on the left side may be more complex and specify the mapping between properties and variables.

If we want to assign a property to a variable with another name, for instance, make `options.width` go into the variable named `w`, then we can set the variable name using a colon:

```javascript
let options = {
 title: "Menu",
 width: 100,
 height: 200
};

// { sourceProperty: targetVariable }
let {width: w, height: h, title} = options;

// width -> w
// height -> h
// title -> title

alert(title); // Menu
alert(w); // 100
alert(h); // 200
```

The colon shows “what : goes where”. In the example above the property `width` goes to `w`, property `height` goes to `h`, and `title` is assigned to the same name.

For potentially missing properties we can set default values using `"="`, like this:

```javascript
let options = {
 title: "Menu"
};

let {width = 100, height = 200, title} = options;

alert(title); // Menu
alert(width); // 100
alert(height); // 200
```

Just like with arrays or function parameters, default values can be any expressions or even function calls. They will be evaluated if the value is not provided.

In the code below `prompt` asks for `width`, but not for `title`:

```javascript
let options = {
 title: "Menu"
};

let {width = prompt("width?"), title = prompt("title?")} = options;

alert(title); // Menu
alert(width); // (whatever the result of prompt is)
```

We also can combine both the colon and equality:

```javascript
let options = {
 title: "Menu"
};

let {width: w = 100, height: h = 200, title} = options;

alert(title); // Menu
alert(w); // 100
alert(h); // 200
```

If we have a complex object with many properties, we can extract only what we need:

```javascript
let options = {
 title: "Menu",
 width: 100,
 height: 200
};

// only extract title as a variable
let { title } = options;

alert(title); // Menu
```

## The rest pattern “…”

What if the object has more properties than we have variables? Can we take some and then assign the “rest” somewhere?

We can use the rest pattern, just like we did with arrays. It’s not supported by some older browsers (IE, use Babel to polyfill it), but works in modern ones.

It looks like this:

```javascript
let options = {
 title: "Menu",
 height: 200,
 width: 100
};

// title = property named title
// rest = object with the rest of properties
let {title, ...rest} = options;

// now title="Menu", rest={height: 200, width: 100}
alert(rest.height); // 200
alert(rest.width); // 100
```

Gotcha if there’s no `let`

In the examples above variables were declared right in the assignment: `let {…} = {…}`. Of course, we could use existing variables too, without `let`. But there’s a catch.

This won’t work:

```javascript
let title, width, height;

// error in this line
{title, width, height} = {title: "Menu", width: 200, height: 100};
```

The problem is that JavaScript treats `{...}` in the main code flow (not inside another expression) as a code block. Such code blocks can be used to group statements, like this:

```javascript
{
 // a code block
 let message = "Hello";
 // ...
 alert( message );
}
```

So here JavaScript assumes that we have a code block, that’s why there’s an error. We want destructuring instead.

To show JavaScript that it’s not a code block, we can wrap the expression in parentheses `(...)`:

```javascript
let title, width, height;

// okay now
({title, width, height} = {title: "Menu", width: 200, height: 100});

alert( title ); // Menu
```

## Nested destructuring

If an object or an array contains other nested objects and arrays, we can use more complex left-side patterns to extract deeper portions.

In the code below `options` has another object in the property `size` and an array in the property `items`. The pattern on the left side of the assignment has the same structure to extract values from them:

```javascript
let options = {
 size: {
 width: 100,
 height: 200
 },
 items: ["Cake", "Donut"],
 extra: true
};

// destructuring assignment split in multiple lines for clarity
let {
 size: { // put size here
 width,
 height
 },
 items: [item1, item2], // assign items here
 title = "Menu" // not present in the object (default value is used)
} = options;

alert(title); // Menu
alert(width); // 100
alert(height); // 200
alert(item1); // Cake
alert(item2); // Donut
```

All properties of `options` object except `extra` which is absent in the left part, are assigned to corresponding variables:

Finally, we have `width`, `height`, `item1`, `item2` and `title` from the default value.

Note that there are no variables for `size` and `items`, as we take their content instead.

## Smart function parameters

There are times when a function has many parameters, most of which are optional. That’s especially true for user interfaces. Imagine a function that creates a menu. It may have a width, a height, a title, an item list and so on.

Here’s a bad way to write such a function:

```javascript
function showMenu(title = "Untitled", width = 200, height = 100, items = ) {
 // ...
}
```

In real-life, the problem is how to remember the order of arguments. Usually, IDEs try to help us, especially if the code is well-documented, but still… Another problem is how to call a function when most parameters are ok by default.

Like this?

```javascript
// undefined where default values are fine
showMenu("My Menu", undefined, undefined, ["Item1", "Item2"])
```

That’s ugly. And becomes unreadable when we deal with more parameters.

Destructuring comes to the rescue!

We can pass parameters as an object, and the function immediately destructurizes them into variables:

```javascript
// we pass object to function
let options = {
 title: "My menu",
 items: ["Item1", "Item2"]
};

// ...and it immediately expands it to variables
function showMenu({title = "Untitled", width = 200, height = 100, items = }) {
 // title, items – taken from options,
 // width, height – defaults used
 alert( `${title} ${width} ${height}` ); // My Menu 200 100
 alert( items ); // Item1, Item2
}

showMenu(options);
```

We can also use more complex destructuring with nested objects and colon mappings:

```javascript
let options = {
 title: "My menu",
 items: ["Item1", "Item2"]
};

function showMenu({
 title = "Untitled",
 width: w = 100, // width goes to w
 height: h = 200, // height goes to h
 items: [item1, item2] // items first element goes to item1, second to item2
}) {
 alert( `${title} ${w} ${h}` ); // My Menu 100 200
 alert( item1 ); // Item1
 alert( item2 ); // Item2
}

showMenu(options);
```

The full syntax is the same as for a destructuring assignment:

```javascript
function({
 incomingProperty: varName = defaultValue
 ...
})
```

Then, for an object of parameters, there will be a variable `varName` for the property `incomingProperty`, with `defaultValue` by default.

Please note that such destructuring assumes that `showMenu` does have an argument. If we want all values by default, then we should specify an empty object:

```javascript
showMenu({}); // ok, all values are default

showMenu; // this would give an error
```

We can fix this by making `{}` the default value for the whole object of parameters:

```javascript
function showMenu({ title = "Menu", width = 100, height = 200 } = {}) {
 alert( `${title} ${width} ${height}` );
}

showMenu; // Menu 100 200
```

In the code above, the whole arguments object is `{}` by default, so there’s always something to destructurize.

## Summary

- Destructuring assignment allows for instantly mapping an object or array onto many variables.

- The full object syntax:

 ```javascript
 let {prop : varName = defaultValue, ...rest} = object
 ```

 This means that property `prop` should go into the variable `varName` and, if no such property exists, then the `default` value should be used.

 Object properties that have no mapping are copied to the `rest` object.

- The full array syntax:

 ```javascript
 let [item1 = defaultValue, item2, ...rest] = array
 ```

 The first item goes to `item1`; the second goes into `item2`, and all the rest makes the array `rest`.

- It’s possible to extract data from nested arrays/objects, for that the left side must have the same structure as the right one.

# **Related Concepts:**

---
- **Spread Syntax (...):** Spread syntax uses the same three-dot (...) notation as the rest parameter in destructuring but performs the opposite operation. While the rest syntax collects multiple elements into a single array or object, the spread syntax "expands" an iterable's elements into places like function arguments, array literals, or object literals. For example, let newArr = [...oldArr, 4, 5]; expands oldArr into newArr.

- **Iterables and the Iterator Protocol:** Destructuring isn't limited to just arrays; it works on any iterable object (e.g., Strings, Sets, Maps). An object is considered iterable if it implements the [Symbol.iterator] method. This method returns an iterator, which is an object with a next method. The next method returns an object with value and done properties. Destructuring uses this protocol to sequentially access the values to be assigned to variables.

- **Function Parameters and Arguments:** Destructuring is frequently used to handle function parameters. It allows a function to accept a single object as an argument and unpack its properties into named local variables. This simulates named parameters, making function calls more readable and independent of the order of arguments. The native arguments object in JavaScript is array-like but is not a true array and is not available in arrow functions; rest parameters (...args) are the modern and more flexible alternative for capturing multiple function arguments.

# **Flashcards:**

---
What is destructuring assignment in JavaScript?;;A syntax that allows unpacking values from arrays or properties from objects into distinct variables.

How do you destructure an array into variables a and b?;;let [a, b] = yourArray;

How do you destructure the name and age properties from a user object?;;let { name, age } = user;

How do you assign a destructured object property width to a new variable name w?;;let { width: w } = yourObject;

What is the purpose of the ...rest syntax in destructuring?;;It collects all remaining elements of an array or properties of an object into a new array or object.

How do you provide a default value for a destructured variable?;;Use the equals sign (=), for example: let [name = "Anonymous"] = ; or let { title = "Untitled" } = {};