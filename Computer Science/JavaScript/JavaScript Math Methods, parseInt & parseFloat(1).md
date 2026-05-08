---
memory: to_finish
tags:
  - learning
language:
  - JavaScript
review-date: 2025-11-25
last-reviewed: 2025-10-19
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-09-19
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

# Core Explanation:
---
## Rounding

One of the most used operations when working with numbers is rounding.

There are several built-in functions for rounding:

### `Math.floor`

Rounds down: `3.1` becomes `3`, and `-1.1` becomes `-2`.

### `Math.ceil`

Rounds up: `3.1` becomes `4`, and `-1.1` becomes `-1`.

### `Math.round`

Rounds to the nearest integer: `3.1` becomes `3`, `3.6` becomes `4`. In the middle cases `3.5` rounds up to `4`, and `-3.5` rounds up to `-3`.

### `Math.trunc` (not supported by Internet Explorer)

Removes anything after the decimal point without rounding: `3.1` becomes `3`, `-1.1` becomes `-1`.

### Here’s the table to summarize the differences between them:

|        | `Math.floor` | `Math.ceil` | `Math.round` | `Math.trunc` |
| ------ | ------------ | ----------- | ------------ | ------------ |
| `3.1`  | `3`          | `4`         | `3`          | `3`          |
| `3.5`  | `3`          | `4`         | `4`          | `3`          |
| `3.6`  | `3`          | `4`         | `4`          | `3`          |
| `-1.1` | `-2`         | `-1`        | `-1`         | `-1`         |
| `-1.5` | `-2`         | `-1`        | `-1`         | `-1`         |
| `-1.6` | `-2`         | `-1`        | `-2`         | `-1`         |

These functions cover all of the possible ways to deal with the decimal part of a number. But what if we’d like to round the number to `n-th` digit after the decimal?

For instance, we have `1.2345` and want to round it to 2 digits, getting only `1.23`.

There are two ways to do so:

1. Multiply-and-divide.

 For example, to round the number to the 2nd digit after the decimal, we can multiply the number by `100`, call the rounding function and then divide it back.


 ```javascript
 let num = 1.23456;

 alert( Math.round(num * 100) / 100 ); // 1 -> 123 -> 123 -> 1
 ```

2. The method toFixed(n) rounds the number to `n` digits after the point and returns a string representation of the result.

 ```javascript
 let num = 12.34;
 alert( num.toFixed(1) ); // "12.3"
 ```

 This rounds up or down to the nearest value, similar to `Math.round`:

 ```javascript
 let num = 12.36;
 alert( num.toFixed(1) ); // "12.4"
 ```

 Please note that the result of `toFixed` is a string. If the decimal part is shorter than required, zeroes are appended to the end:

 ```javascript
 let num = 12.34;
 alert( num.toFixed(5) ); // "12.34000", added zeroes to make exactly 5 digits
 ```

 We can convert it to a number using the unary plus or a `Number` call, e.g. write `+num.toFixed(5)`.

## Imprecise calculations

Internally, a number is represented in 64-bit format IEEE-754, so there are exactly 64 bits to store a number: 52 of them are used to store the digits, 11 of them store the position of the decimal point, and 1 bit is for the sign.

If a number is really huge, it may overflow the 64-bit storage and become a special numeric value `Infinity`:

```javascript
alert( 1e500 ); // Infinity
```

What may be a little less obvious, but happens quite often, is the loss of precision.

Consider this (falsy!) equality test:

```javascript
alert( 0 + 0 == 0 ); // false
```

That’s right, if we check whether the sum of `0.1` and `0.2` is `0.3`, we get `false`.

Strange! What is it then if not `0.3`?

```javascript
alert( 0 + 0 ); // 0
```

Ouch! Imagine you’re making an e-shopping site and the visitor puts `$0.10` and `$0.20` goods into their cart. The order total will be `$0.30000000000000004`. That would surprise anyone.

But why does this happen?

A number is stored in memory in its binary form, a sequence of bits – ones and zeroes. But fractions like `0.1`, `0.2` that look simple in the decimal numeric system are actually unending fractions in their binary form.

```javascript
alert(0.toString(2)); // 0
alert(0.toString(2)); // 0
alert((0 + 0.2).toString(2)); // 0
```

What is `0.1`? It is one divided by ten `1/10`, one-tenth. In the decimal numeral system, such numbers are easily representable. Compare it to one-third: `1/3`. It becomes an endless fraction `0.33333(3)`.

So, division by powers `10` is guaranteed to work well in the decimal system, but division by `3` is not. For the same reason, in the binary numeral system, the division by powers of `2` is guaranteed to work, but `1/10` becomes an endless binary fraction.

There’s just no way to store _exactly 0.1_ or _exactly 0.2_ using the binary system, just like there is no way to store one-third as a decimal fraction.

The numeric format IEEE-754 solves this by rounding to the nearest possible number. These rounding rules normally don’t allow us to see that “tiny precision loss”, but it exists.

We can see this in action:

```javascript
alert( 0.toFixed(20) ); // 0
```

And when we sum two numbers, their “precision losses” add up.

That’s why `0 + 0.2` is not exactly `0.3`.

### Not only JavaScript

The same issue exists in many other programming languages.

PHP, Java, C, Perl, and Ruby give exactly the same result, because they are based on the same numeric format.

Can we work around the problem? Sure, the most reliable method is to round the result with the help of a method toFixed(n):

```javascript
let sum = 0 + 0.2;
alert( sum.toFixed(2) ); // "0.30"
```

Please note that `toFixed` always returns a string. It ensures that it has 2 digits after the decimal point. That’s actually convenient if we have an e-shopping and need to show `$0.30`. For other cases, we can use the unary plus to coerce it into a number:

```javascript
let sum = 0 + 0.2;
alert( +sum.toFixed(2) ); // 0
```

We also can temporarily multiply the numbers by 100 (or a bigger number) to turn them into integers, do the maths, and then divide back. Then, as we’re doing maths with integers, the error somewhat decreases, but we still get it on division:

```javascript
alert( (0 * 10 + 0 * 10) / 10 ); // 0
alert( (0 * 100 + 0 * 100) / 100); // 0
```

So, the multiply/divide approach reduces the error, but doesn’t remove it totally.

Sometimes we could try to evade fractions at all. Like if we’re dealing with a shop, then we can store prices in cents instead of dollars. But what if we apply a discount of 30%? In practice, totally evading fractions is rarely possible. Just round them to cut “tails” when needed.

### The funny thing

Try running this:

```javascript
// Hello! I'm a self-increasing number!
alert( 9999999999999999 ); // shows 10000000000000000
```

This suffers from the same issue: a loss of precision. There are 64 bits for the number, 52 of them can be used to store digits, but that’s not enough. So the least significant digits disappear.

JavaScript doesn’t trigger an error in such events. It does its best to fit the number into the desired format, but unfortunately, this format is not big enough.

### Two zeroes

Another funny consequence of the internal representation of numbers is the existence of two zeroes: `0` and `-0`.

That’s because a sign is represented by a single bit, so it can be set or not set for any number including a zero.

In most cases, the distinction is unnoticeable, because operators are suited to treat them as the same.

## Tests: isFinite and isNaN

Remember these two special numeric values?

- `Infinity` (and `-Infinity`) is a special numeric value that is greater (less) than anything.
- `NaN` represents an error.

They belong to the type `number`, but are not “normal” numbers, so there are special functions to check for them:

- `isNaN(value)` converts its argument to a number and then tests it for being `NaN`:

 ```javascript
 alert( isNaN(NaN) ); // true
 alert( isNaN("str") ); // true
 ```

 But do we need this function? Can’t we just use the comparison `\=== NaN`? Unfortunately not. The value `NaN` is unique in that it does not equal anything, including itself:

 ```javascript
 alert( NaN === NaN ); // false
 ```

- `isFinite(value)` converts its argument to a number and returns `true` if it’s a regular number, not `NaN/Infinity/-Infinity`:

 ```javascript
 alert( isFinite("15") ); // true
 alert( isFinite("str") ); // false, because a special value: NaN
 alert( isFinite(Infinity) ); // false, because a special value: Infinity
 ```
Sometimes `isFinite` is used to validate whether a string value is a regular number:
```javascript
let num = +prompt("Enter a number", '');

// will be true unless you enter Infinity, -Infinity or not a number
alert( isFinite(num) );
```
Please note that an empty or a space-only string is treated as `0` in all numeric functions including `isFinite`.

### `Number.isNaN` and `Number.isFinite`

Number.isNaN and Number.isFinite methods are the more “strict” versions of `isNaN` and `isFinite` functions. They do not autoconvert their argument into a number, but check if it belongs to the `number` type instead.

- `Number.isNaN(value)` returns `true` if the argument belongs to the `number` type and it is `NaN`. In any other case, it returns `false`.

 ```javascript
 alert( Number.isNaN(NaN) ); // true
 alert( Number.isNaN("str" / 2) ); // true

 // Note the difference:
 alert( Number.isNaN("str") ); // false, because "str" belongs to the string type, not the number type
 alert( isNaN("str") ); // true, because isNaN converts string "str" into a number and gets NaN as a result of this conversion
 ```

- `Number.isFinite(value)` returns `true` if the argument belongs to the `number` type and it is not `NaN/Infinity/-Infinity`. In any other case, it returns `false`.

 ```javascript
 alert( Number.isFinite(123) ); // true
 alert( Number.isFinite(Infinity) ); // false
 alert( Number.isFinite(2 / 0) ); // false

 // Note the difference:
 alert( Number.isFinite("123") ); // false, because "123" belongs to the string type, not the number type
 alert( isFinite("123") ); // true, because isFinite converts string "123" into a number 123
 ```

In a way, `Number.isNaN` and `Number.isFinite` are simpler and more straightforward than `isNaN` and `isFinite` functions. In practice though, `isNaN` and `isFinite` are mostly used, as they’re shorter to write.

### Comparison with `Object.is`

There is a special built-in method `Object.is` that compares values like `\===`, but is more reliable for two edge cases:

1. It works with `NaN`: `Object.is(NaN, NaN) === true`, that’s a good thing.
2. Values `0` and `-0` are different: `Object.is(0, -0) === false`, technically that’s correct because internally the number has a sign bit that may be different even if all other bits are zeroes.

In all other cases, `Object.is(a, b)` is the same as `a === b`.

We mention `Object.is` here, because it’s often used in JavaScript specification. When an internal algorithm needs to compare two values for being exactly the same, it uses `Object.is` (internally called SameValue).
## parseInt and parseFloat

Numeric conversion using a plus `+` or `Number` is strict. If a value is not exactly a number, it fails:

```javascript
alert( +"100px" ); // NaN
```

The sole exception is spaces at the beginning or at the end of the string, as they are ignored.

But in real life, we often have values in units, like `"100px"` or `"12pt"` in CSS. Also in many countries, the currency symbol goes after the amount, so we have `"19€"` and would like to extract a numeric value out of that.

That’s what `parseInt` and `parseFloat` are for.

They “read” a number from a string until they can’t. In case of an error, the gathered number is returned. The function `parseInt` returns an integer, whilst `parseFloat` will return a floating-point number:

```javascript
alert( parseInt('100px') ); // 100
alert( parseFloat('12.5em') ); // 12

alert( parseInt('12.3') ); // 12, only the integer part is returned
alert( parseFloat('12.4') ); // 12.3, the second point stops the reading
```

There are situations when `parseInt/parseFloat` will return `NaN`. It happens when no digits could be read:

```javascript
alert( parseInt('a123') ); // NaN, the first symbol stops the process
```

The second argument of `parseInt(str, radix)`

The `parseInt` function has an optional second parameter. It specifies the base of the numeral system, so `parseInt` can also parse strings of hex numbers, binary numbers and so on:

```javascript
alert( parseInt('0xff', 16) ); // 255
alert( parseInt('ff', 16) ); // 255, without 0x also works

alert( parseInt('2n9c', 36) ); // 123456
```

## Other math functions

JavaScript has a built-in Math object which contains a small library of mathematical functions and constants.

A few examples:

`Math.random`

Returns a random number from 0 to 1 (not including 1).

```javascript
alert( Math.random ); // 0
alert( Math.random ); // 0
alert( Math.random ); // ... (any random numbers)
```

`Math.max(a, b, c...)` and `Math.min(a, b, c...)`

Returns the greatest and smallest from the arbitrary number of arguments.

```javascript
alert( Math.max(3, 5, -10, 0, 1) ); // 5
alert( Math.min(1, 2) ); // 1
```

`Math.pow(n, power)`

Returns `n` raised to the given power.

```javascript
alert( Math.pow(2, 10) ); // 2 in power 10 = 1024
```

There are more functions and constants in `Math` object, including trigonometry, which you can find in the docs for the Math object.
## Summary

To write numbers with many zeroes:

- Append `"e"` with the zeroes count to the number. Like: `123e6` is the same as `123` with 6 zeroes `123000000`.
- A negative number after `"e"` causes the number to be divided by 1 with given zeroes. E.g. `123e-6` means `0.000123` (`123` millionths).

For different numeral systems:

- Can write numbers directly in hex (`0x`), octal (`0o`) and binary (`0b`) systems.
- `parseInt(str, base)` parses the string `str` into an integer in numeral system with given `base`, `2 ≤ base ≤ 36`.
- `num.toString(base)` converts a number to a string in the numeral system with the given `base`.

For regular number tests:

- `isNaN(value)` converts its argument to a number and then tests it for being `NaN`
- `Number.isNaN(value)` checks whether its argument belongs to the `number` type, and if so, tests it for being `NaN`
- `isFinite(value)` converts its argument to a number and then tests it for not being `NaN/Infinity/-Infinity`
- `Number.isFinite(value)` checks whether its argument belongs to the `number` type, and if so, tests it for not being `NaN/Infinity/-Infinity`

For converting values like `12pt` and `100px` to a number:

- Use `parseInt/parseFloat` for the “soft” conversion, which reads a number from a string and then returns the value they could read before the error.

For fractions:

- Round using `Math.floor`, `Math.ceil`, `Math.trunc`, `Math.round` or `num.toFixed(precision)`.
- Make sure to remember there’s a loss of precision when working with fractions.

More mathematical functions:

- See the Math object when you need them. The library is very small but can cover basic needs.

# **Related Concepts:**

---
- **Number Type Conversion**

JavaScript has several ways to convert values to numbers: `Number`, unary `+`, and `parseInt/parseFloat`. While `Number` and `+` require exact numeric formats, `parseInt/parseFloat` are more flexible with mixed content.

- **Floating Point Precision**

JavaScript numbers are stored as 64-bit floating point, which can lead to precision issues. Rounding functions help manage these precision problems, especially when dealing with decimal calculations.

- **String Methods for Numbers**

Methods like `toFixed`, `toPrecision`, and `toExponential` convert numbers to strings with specific formatting, complementing the rounding functions for display purposes.

- **[[JavaScript Type Conversion (Coercion)]]**

JavaScript's automatic type conversion can interact with these number functions in unexpected ways, making explicit conversion with these methods important for predictable results.

# **Examples:**

---
```javascript
// Basic rounding examples
let price = 19.99;
let discountedPrice = 17.234;

// Math.floor - always rounds down (useful for pricing)
console.log(Math.floor(price)); // 19 - removes cents completely
console.log(Math.floor(-2.8)); // -3 - rounds toward negative infinity

// Math.ceil - always rounds up (useful for pagination)
let itemsPerPage = 10;
let totalItems = 47;
let totalPages = Math.ceil(totalItems / itemsPerPage); // 5 pages needed
console.log(totalPages);

// Math.round - rounds to nearest integer
console.log(Math.round(discountedPrice)); // 17 - standard rounding rules
console.log(Math.round(2.5)); // 3 - ties round up
console.log(Math.round(-2.5)); // -2 - negative ties round toward zero

// Math.trunc - simply removes decimal part
console.log(Math.trunc(discountedPrice)); // 17 - just cuts off decimals
console.log(Math.trunc(-2.8)); // -2 - doesn't round, just truncates

// Rounding to specific decimal places
let measurement = 3.14159;

// Method 1: Multiply, round, divide
let rounded2Places = Math.round(measurement * 100) / 100; // 3
let rounded3Places = Math.round(measurement * 1000) / 1000; // 3

// Method 2: toFixed (returns string)
let fixedString = measurement.toFixed(2); // "3.14"
let fixedNumber = +measurement.toFixed(2); // 3 (converted back to number)

// parseInt and parseFloat for extracting numbers from strings
let cssValue = "250px";
let fontSizeValue = "1.5em";
let priceString = "USD 29.99";

// parseInt extracts integer from beginning of string
let pixelValue = parseInt(cssValue); // 250
console.log(pixelValue + 50); // 300 - can do math operations

// parseFloat extracts decimal number from beginning
let fontSize = parseFloat(fontSizeValue); // 1
console.log(fontSize * 16); // 24 - convert em to pixels

// parseInt with different bases (radix parameter)
let binaryString = "1010";
let hexString = "FF";
console.log(parseInt(binaryString, 2)); // 10 - binary to decimal
console.log(parseInt(hexString, 16)); // 255 - hex to decimal

// Handling edge cases
console.log(parseInt("abc123")); // NaN - no number at start
console.log(parseFloat("12.56")); // 12 - stops at second decimal
console.log(parseInt(" 42 ")); // 42 - ignores whitespace
```

# **Flashcards:**

---
What does Math.floor do and how does it handle negative numbers?;; Math.floor always rounds DOWN to the nearest integer. For positive numbers like 3.9, it becomes 3. For negative numbers like -1.1, it becomes -2 (further from zero).

What's the difference between parseInt and parseFloat?;; parseInt extracts only the integer portion from a string and stops at the first decimal point. parseFloat extracts the full decimal number. For "12.34px": parseInt returns 12, parseFloat returns 12.

How do you round a number to 2 decimal places in JavaScript?;; Two methods: 1) Math.round(num * 100) / 100, or 2) +num.toFixed(2). The toFixed method returns a string, so use + to convert back to number.

When does Math.round round up vs down for values?;; Math.round always rounds UP for positive numbers (2 becomes 3) but rounds toward zero for negative numbers (-2 becomes -2, not -3).

What happens when parseInt encounters non-numeric characters?;; parseInt reads from left to right and stops at the first non-numeric character. "123abc" returns 123, but "abc123" returns NaN because it can't start reading a number.

What's the key difference between Math.trunc and Math.floor?;; Math.trunc simply removes the decimal part without rounding (3 becomes 3, -3 becomes -3). Math.floor always rounds down (-3 becomes -4). They differ only with negative numbers.