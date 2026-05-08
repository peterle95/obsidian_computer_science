---
memory: to_finish
tags:
  - learned
language:
  - JavaScript
review-date:
last-reviewed: 2025-09-15
scheda: done
visit-count: 5
confidence-level: 2.5
consecutive-correct: 3
last-struggle-date: 2025-08-16
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
The **optional chaining operator (`?.`)** allows ==safe access to nested object properties without throwing errors if intermediate properties are `null` or `undefined`==. Introduced in ES2020, it provides a concise way to handle ==potentially missing properties in object chains==.

**Syntax Variations:**
- **Property access**: `obj?.prop` or `obj?.['prop']`
- **Method calls**: `obj?.method?.()`
- **Array/computed access**: `obj?.[index]` or `obj?.[expression]`

**Behavior:**
- Returns `undefined` if any part of the chain is `null` or `undefined`
- Stops evaluation immediately when encountering `null`/`undefined` (short-circuiting)
- Does NOT throw TypeError for missing properties
- Can be chained multiple times: `obj?.prop?.subProp?.value`

**Use Cases:**
- API responses with optional nested data
- Accessing properties on potentially null objects
- Safe method calls on objects that might not exist
- Avoiding lengthy null checks
# **Related Concepts:**
---
- [[JavaScript Object Properties (Accessing, Modifying, Deleting)]]
- [[JavaScript Object Methods]]
- [[JavaScript - Nullish Coalescing Operator (？？)]]
- Error handling and defensive programming
- TypeScript optional properties
- Logical AND (`&&`) short-circuiting patterns
# **Examples:**
---
```javascript
// Basic object with nested properties
const user = {
  name: 'Alice',
  address: {
    street: '123 Main St',
    city: 'New York',
    coordinates: {
      lat: 40.7128,
      lng: -74.0060
    }
  },
  preferences: {
    theme: 'dark'
  }
};

// Without optional chaining (traditional approach)
let city1;
if (user && user.address && user.address.city) {
  city1 = user.address.city;
}
console.log(city1); // 'New York'

// With optional chaining
const city2 = user?.address?.city;
console.log(city2); // 'New York'

// The first examples show the traditional null-checking approach versus optional chaining - the optional chaining version is much more concise and readable while providing the same safety.

// Accessing deeply nested properties
const latitude = user?.address?.coordinates?.lat;
console.log(latitude); // 40.7128

// The deeply nested property access demonstrates how optional chaining can safely traverse multiple levels without throwing errors, returning `undefined` when any intermediate property is missing.

// Handling missing properties
const user2 = { name: 'Bob' }; // No address property

const missingCity = user2?.address?.city;
console.log(missingCity); // undefined (no error thrown)

// Traditional approach would throw error
// console.log(user2.address.city); // TypeError: Cannot read properties of undefined

// Optional chaining with array access
const users = [
  { name: 'Alice', posts: [{ title: 'Hello World' }] },
  { name: 'Bob' } // No posts array
];

console.log(users[0]?.posts?.[0]?.title); // 'Hello World'
console.log(users[1]?.posts?.[0]?.title); // undefined

// Array access examples show how optional chaining works with bracket notation, useful for accessing array elements or computed properties safely.

// Optional method calls
const api = {
  getData: function() {
    return { status: 'success' };
  }
};

const emptyApi = {};

// Safe method calls
const result1 = api?.getData?.();
console.log(result1); // { status: 'success' }

const result2 = emptyApi?.getData?.();
console.log(result2); // undefined (method doesn't exist)

// Method call examples demonstrate the `?.()` syntax for safely calling methods that might not exist, preventing "method is not a function" errors.

// Combining with nullish coalescing (??)
const userTheme = user?.preferences?.theme ?? 'light';
console.log(userTheme); // 'dark'

const missingTheme = user2?.preferences?.theme ?? 'light';
console.log(missingTheme); // 'light' (fallback value)

// The combination with nullish coalescing (`??`) shows a common pattern for providing fallback values when properties are missing.

// Complex real-world example
const apiResponse = {
  data: {
    user: {
      profile: {
        avatar: 'avatar.jpg',
        social: {
          twitter: '@alice'
        }
      }
    }
  }
};

// Multiple optional chains
const twitterHandle = apiResponse?.data?.user?.profile?.social?.twitter;
const avatarUrl = apiResponse?.data?.user?.profile?.avatar;
const missingField = apiResponse?.data?.user?.profile?.social?.instagram;

console.log(twitterHandle); // '@alice'
console.log(avatarUrl); // 'avatar.jpg'
console.log(missingField); // undefined

// The complex API response example illustrates real-world usage where deeply nested optional data is common.

// Function parameters with optional chaining
function displayUserInfo(userObj) {
  const name = userObj?.name ?? 'Anonymous';
  const email = userObj?.contact?.email ?? 'No email provided';
  const phone = userObj?.contact?.phone ?? 'No phone provided';
  
  console.log(`Name: ${name}`);
  console.log(`Email: ${email}`);
  console.log(`Phone: ${phone}`);
}

displayUserInfo({ name: 'Alice', contact: { email: 'alice@example.com' } });
displayUserInfo({ name: 'Bob' });
displayUserInfo(null);

// The function parameter example shows how optional chaining simplifies defensive programming in functions.
```
# **Flashcards:**
---
What does the optional chaining operator (`?.`) do and when does it return `undefined`?;; Optional chaining safely accesses nested object properties without throwing errors. It returns `undefined` if any part of the property chain is `null` or `undefined`, and uses short-circuiting to stop evaluation immediately when encountering these values.

What are the three main syntax variations of optional chaining and when would you use each?;; 1) Property access: `obj?.prop` for accessing object properties, 2) Method calls: `obj?.method?.()` for safely calling methods that might not exist, 3) Computed/array access: `obj?.[key]` or `obj?.[0]` for dynamic property names or array elements.

How does optional chaining work with the nullish coalescing operator (`??`) and why is this combination useful?;; Optional chaining returns `undefined` for missing properties, while nullish coalescing provides fallback values when the left side is `null` or `undefined`. Combined as `obj?.prop ?? 'default'`, they provide safe property access with fallback values in a single, readable expression.