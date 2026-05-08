---
memory: to_finish
tags:
  - learned
language:
  - JavaScript
review-date:
last-reviewed: 2025-09-09
scheda: done
visit-count: 6
confidence-level: 3
consecutive-correct: 4
last-struggle-date: 2025-07-19
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
==JavaScript array iteration methods solve the fundamental problem of processing collections of data without writing repetitive loop code. They eliminate the need for manual index management and provide declarative, functional programming approaches to common operations like transforming data, filtering collections, aggregating values, and searching arrays.== These methods are crucial in modern JavaScript development because they lead to more readable, maintainable code and align with functional programming principles that reduce bugs and improve code predictability.

# **Core Explanation:**
---
## *JavaScript provides several higher-order functions for array iteration, each designed for specific use cases:*

**Non-mutating methods (return new values/arrays):**
>- **forEach()**: ==Executes a function== for each array element, returns undefined
>- **map()**: Creates a ==new array by transforming each element with a callback function==
>- **filter()**: Creates a new array with <mark style="background: #FF5582A6;">elements that pass a test function</mark>
>- **reduce()**: Reduces array to a single value by applying a reducer function
>- **some()**: Tests if at least one element passes a test function (returns boolean)
>- **every()**: Tests if all elements pass a test function (returns boolean)
>- **find()**: Returns the first element that passes a test function
>- **findIndex()**: Returns the index of the first element that passes a test function

*Key characteristics: All methods accept callback functions as parameters, most don't mutate the original array, they provide cleaner alternatives to for loops, and they support functional programming patterns. Each callback typically receives (element, index, array) as parameters.*

# **Related Concepts:**
---
## *These iteration methods connect to several important programming concepts:*

- **Higher-Order Functions**: Functions that accept other functions as parameters
- **Functional Programming**: Paradigm emphasizing pure functions and immutability
- **Callback Functions**: Functions passed as arguments to be executed later
- **Array Transformation**: Converting arrays from one form to another
- **Predicates**: Boolean-returning functions used for testing conditions
- **Reduce Pattern**: Folding/accumulating values into a single result
- **Short-Circuit Evaluation**: Methods like some() and every() stop early when condition is met
- **Immutable Data Processing**: Creating new data structures rather than modifying existing ones

- [[C++ Iterators]]
# **Examples:**
---
```javascript
// Sample data for demonstrations
const students = [
  { name: 'Alice', age: 20, grade: 85, active: true },
  { name: 'Bob', age: 22, grade: 92, active: false },
  { name: 'Charlie', age: 19, grade: 78, active: true },
  { name: 'Diana', age: 21, grade: 95, active: true },
  { name: 'Eve', age: 20, grade: 88, active: false }
];

const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// FOREACH - executes function for each element, returns undefined
console.log('=== forEach Example ===');
students.forEach((student, index) => {
  // Performs side effects, doesn't return new array
  console.log(`${index + 1}. ${student.name} - Grade: ${student.grade}`);
});
// Output: Lists all students with their grades
// forEach returns undefined, used for side effects only

// MAP - transforms each element, returns new array of same length
console.log('=== map Example ===');
const studentNames = students.map(student => student.name);
console.log(studentNames); // ['Alice', 'Bob', 'Charlie', 'Diana', 'Eve']

const gradeReports = students.map((student, index) => {
  // Transform each student into a report object
  return {
    id: index + 1,
    name: student.name,
    letterGrade: student.grade >= 90 ? 'A' : student.grade >= 80 ? 'B' : 'C',
    status: student.active ? 'Active' : 'Inactive'
  };
});
console.log(gradeReports); // Array of transformed report objects

// FILTER - creates new array with elements that pass test
console.log('=== filter Example ===');
const activeStudents = students.filter(student => student.active);
console.log(activeStudents); // Only students with active: true

const highAchievers = students.filter(student => student.grade >= 90);
console.log(highAchievers); // Students with grades 90 or above

const evenNumbers = numbers.filter(num => num % 2 === 0);
console.log(evenNumbers); // [2, 4, 6, 8, 10]

// REDUCE - reduces array to single value using accumulator
console.log('=== reduce Example ===');
const totalGrades = students.reduce((sum, student) => {
  // Accumulator pattern: sum starts at 0, adds each grade
  return sum + student.grade;
}, 0); // Initial value is 0
console.log(`Total grades: ${totalGrades}`); // Sum of all grades

const averageGrade = totalGrades / students.length;
console.log(`Average grade: ${averageGrade}`); // Calculate average

// Complex reduce: group students by age
const studentsByAge = students.reduce((groups, student) => {
  // If age group doesn't exist, create empty array
  if (!groups[student.age]) {
    groups[student.age] = [];
  }
  groups[student.age].push(student.name);
  return groups;
}, {}); // Initial value is empty object
console.log(studentsByAge); // { 19: ['Charlie'], 20: ['Alice', 'Eve'], 21: ['Diana'], 22: ['Bob'] }

// SOME - tests if at least one element passes test (short-circuits)
console.log('=== some Example ===');
const hasHighGrade = students.some(student => student.grade >= 95);
console.log(`Has student with 95+ grade: ${hasHighGrade}`); // true (Diana has 95)

const hasYoungStudent = students.some(student => student.age < 18);
console.log(`Has student under 18: ${hasYoungStudent}`); // false

const hasEvenNumber = numbers.some(num => num % 2 === 0);
console.log(`Has even number: ${hasEvenNumber}`); // true (stops at first even number)

// EVERY - tests if all elements pass test (short-circuits on first false)
console.log('=== every Example ===');
const allPassing = students.every(student => student.grade >= 70);
console.log(`All students passing (70+): ${allPassing}`); // true

const allActive = students.every(student => student.active);
console.log(`All students active: ${allActive}`); // false (Bob and Eve are inactive)

const allPositive = numbers.every(num => num > 0);
console.log(`All numbers positive: ${allPositive}`); // true

// FIND - returns first element that passes test (or undefined)
console.log('=== find Example ===');
const topStudent = students.find(student => student.grade >= 95);
console.log(topStudent); // Returns Diana object (first with grade >= 95)

const inactiveStudent = students.find(student => !student.active);
console.log(inactiveStudent.name); // 'Bob' (first inactive student)

const largeNumber = numbers.find(num => num > 5);
console.log(largeNumber); // 6 (first number greater than 5)

// FINDINDEX - returns index of first element that passes test (or -1)
console.log('=== findIndex Example ===');
const topStudentIndex = students.findIndex(student => student.grade >= 95);
console.log(`Top student at index: ${topStudentIndex}`); // 3 (Diana's index)

const inactiveStudentIndex = students.findIndex(student => !student.active);
console.log(`First inactive student at index: ${inactiveStudentIndex}`); // 1 (Bob's index)

const notFoundIndex = students.findIndex(student => student.age > 25);
console.log(`Student over 25 index: ${notFoundIndex}`); // -1 (not found)

// PRACTICAL EXAMPLE: E-commerce order processing
console.log('=== Practical E-commerce Example ===');
const orders = [
  { id: 1, customer: 'John', total: 150, items: ['laptop', 'mouse'], shipped: true },
  { id: 2, customer: 'Jane', total: 75, items: ['book'], shipped: false },
  { id: 3, customer: 'Bob', total: 200, items: ['phone', 'case'], shipped: true },
  { id: 4, customer: 'Alice', total: 50, items: ['headphones'], shipped: false }
];

// Get all customer names (map)
const customerNames = orders.map(order => order.customer);
console.log('Customers:', customerNames);

// Filter unshipped orders (filter)
const unshippedOrders = orders.filter(order => !order.shipped);
console.log('Unshipped orders:', unshippedOrders.length);

// Calculate total revenue (reduce)
const totalRevenue = orders.reduce((sum, order) => sum + order.total, 0);
console.log('Total revenue:', totalRevenue);

// Check if any large orders exist (some)
const hasLargeOrder = orders.some(order => order.total > 100);
console.log('Has large order (>$100):', hasLargeOrder);

// Verify all orders have customers (every)
const allHaveCustomers = orders.every(order => order.customer && order.customer.length > 0);
console.log('All orders have customers:', allHaveCustomers);

// Find specific order (find)
const bobsOrder = orders.find(order => order.customer === 'Bob');
console.log('Bobs order:', bobsOrder);

// Method chaining example: complex data processing
const processedData = students
  .filter(student => student.active)        // Only active students
  .map(student => ({                        // Transform to summary
    name: student.name,
    performance: student.grade >= 90 ? 'Excellent' : 'Good'
  }))
  .reduce((summary, student) => {           // Count by performance
    summary[student.performance] = (summary[student.performance] || 0) + 1;
    return summary;
  }, {});
console.log('Active student performance summary:', processedData);
````

# **Flashcards:**

---

What's the key difference between forEach() and map()?;; forEach() executes a function for each element and returns undefined (used for side effects), while map() transforms each element and returns a new array of the same length

What does the reduce() method do and what are its parameters?;; reduce() reduces an array to a single value using an accumulator function. Parameters: callback(accumulator, currentValue, index, array) and optional initialValue

What's the difference between some() and every()?;; some() returns true if at least one element passes the test (short-circuits on first true), every() returns true only if all elements pass the test (short-circuits on first false)

What's the difference between find() and findIndex()?;; find() returns the first element that passes the test (or undefined), findIndex() returns the index of the first element that passes the test (or -1 if not found)

Which array iteration methods create new arrays vs. modify behavior?;; map() and filter() create new arrays, forEach() returns undefined, reduce() returns a single value, some()/every() return booleans, find()/findIndex() return element/index

What does this code return: [1,2,3,4].filter(x => x > 2).map(x => x * 2)?;; Returns [6, 8] - filter creates [3, 4], then map doubles each element to get [6, 8]