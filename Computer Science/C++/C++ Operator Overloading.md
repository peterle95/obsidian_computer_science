---
memory: to_finish
tags:
  - mastered
language:
  - C++
review-date: ""
last-reviewed: 2025-08-31
scheda: done
visit-count: 4
confidence-level: 2.5
consecutive-correct: 1
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
# --> Break it up in more notes
# **Core Explanation:**

Operator overloading is a powerful feature in C++ that allows you to redefine the meaning of standard C++ operators (like +, -,\*, /, \==, <<, etc.) when they are applied to objects of your user-defined types (classes). This enables you to make the syntax for working with your objects more intuitive and natural, often mimicking the way these operators work with built-in types.

[[Example of a function overload resolution]]

**Extensive Explanation:**

Imagine you have a class representing a point in a 2D plane. You might want to add two points together to get a new point that is the vector sum of the original two. Instead of writing a function like addPoints(point1, point2), operator overloading allows you to use the familiar + operator: point1 + point2. This makes your code more readable and closer to the mathematical notation you might already be familiar with.

**Key Aspects of Operator Overloading:**

- **Extending Operator Behavior:** Operator overloading doesn't change how operators work for built-in types (like int, float, etc.). It provides a way to define their behavior when one or both of the operands are objects of a user-defined class.
    
- **Making Code More Intuitive:** By giving operators meaningful behavior for your classes, you can make your code easier to understand and write. It allows you to express operations on objects in a more natural and concise way.
    
- **Achieving Polymorphism:** Operator overloading is a form of ad-hoc polymorphism (also known as function overloading for operators). The same operator symbol can have different meanings depending on the types of its operands.
    
- **Syntax:** You overload an operator by defining a special member function or a non-member function (often a friend function) with a specific name: operator \<operator-symbol>.
    

**Rules and Limitations of Operator Overloading:**

- **You cannot create new operators:** You can only overload existing C++ operators. You cannot invent new symbols for operations.
    
- **Precedence and Associativity Remain the Same:** The precedence and associativity of the overloaded operator remain the same as their built-in counterparts. You cannot change whether * has higher precedence than +, for example.
    
- **At least one operand must be of user-defined type:** You cannot overload operators where both operands are of built-in types. This prevents you from changing the fundamental behavior of operators for primitive types.
    
- **Certain operators cannot be overloaded:** Some operators cannot be overloaded, including:
    
    - . (member access operator)
        
    - .* (pointer-to-member dereference operator)
        
    - :: (scope resolution operator)
        
    - sizeof (size of operator)
        
    - typeid (type identification operator)
        
    - ?: (ternary conditional operator)
        
    - # and ## (preprocessor directives)
        
- **You cannot change the number of operands:** The arity (number of operands) of an operator cannot be changed. Unary operators remain unary, and binary operators remain binary.
    

**Syntax and Mechanics of Operator Overloading:**

There are two main ways to overload operators in C++:

**1. Member Function Overloading:**

- The operator is overloaded as a member function of the class for which it is being defined.
    
- For binary operators, the left-hand operand is implicitly the object on which the member function is called (accessed using the this pointer), and the right-hand operand is passed as an argument.
    
- For unary operators, there are no explicit arguments (for prefix/postfix).

```cpp
class Point {
private:
    int x, y;
public:
    Point(int xCoord = 0, int yCoord = 0) : x(xCoord), y(yCoord) {}

    // Overloading the + operator as a member function
    Point operator+(const Point& other) const {
        return Point(x + other.x, y + other.y);
    }

    // Overloading the prefix increment operator ++
    Point& operator++() { // Returns a reference to allow chaining
        ++x;
        ++y;
        return *this;
    }

    // Overloading the postfix increment operator ++ (takes a dummy int)
    Point operator++(int) { // Returns the old value
        Point temp = *this;
        x++;
        y++;
        return temp;
    }

    void display() const {
        std::cout << "Point(" << x << ", " << y << ")" << std::endl;
    }
};

int main() {
    Point p1(1, 2);
    Point p2(3, 4);
    Point p3 = p1 + p2; // Calls p1.operator+(p2)
    p3.display(); // Output: Point(4, 6)

    Point p4 = ++p1; // Calls p1.operator++()
    p1.display(); // Output: Point(2, 3)
    p4.display(); // Output: Point(2, 3)

    Point p5 = p2++; // Calls p2.operator++(0)
    p2.display(); // Output: Point(4, 5)
    p5.display(); // Output: Point(3, 4)

    return 0;
}```
**2. Non-Member Function (Often Friend Function) Overloading:**

- The operator is overloaded as a standalone function that is not a member of the class.
    
- For binary operators, both the left-hand and right-hand operands are passed as arguments.
    
- If the non-member function needs access to the private members of the class, it is often declared as a friend of the class.
```cpp
#include <iostream>

class Point {
private:
    int x, y;
public:
    Point(int xCoord = 0, int yCoord = 0) : x(xCoord), y(yCoord) {}

    friend std::ostream& operator<<(std::ostream& os, const Point& point); // Friend declaration

    // ... other member functions ...
};

// Overloading the output stream operator << as a non-member friend function
std::ostream& operator<<(std::ostream& os, const Point& point) {
    os << "Point(" << point.x << ", " << point.y << ")";
    return os;
}

/*The `operator<<` function returns `std::ostream&` (a reference to an output stream) to allow chaining of output operations (like `std::cout << a << b << c`). Returning by reference avoids creating copies of the stream, ensuring that all output goes to the same stream.*/

int main() {
    Point p(5, 6);
    std::cout << p << std::endl; // Calls operator<<(std::cout, p) - Output: Point(5, 6)
    return 0;
}```
**Commonly Overloaded Operators:**

- **Arithmetic Operators:** +, -, \*, /, %
    
- **Assignment Operators:** =, +=, -=, *=, /=, etc.
    
- **Comparison Operators:** \==, !=, <, >, <=, >=
    
- **Increment and Decrement Operators:** ++, -- (prefix and postfix)
    
- **Input/Output Stream Operators:** << (output), >> (input)
    
- **Subscript Operator:** [] (for container-like classes)
    
- **Function Call Operator:** () (for function objects or functors)
    
- **Dereference Operators:** * (for pointer-like classes, like smart pointers), ->
    

**Choosing Between Member and Non-Member Overloading:**

- **Member functions are usually preferred when:**
    
    - The left-hand operand of a binary operator is always an object of the class.
        
    - The operator modifies the state of the object (e.g., assignment operators).
        
- **Non-member functions (often friend functions) are usually preferred when:**
    
    - The left-hand operand of a binary operator might be of a different type (e.g., overloading << for std::ostream).
        
    - Symmetry is desired for binary operators where both operands are of the same class.
        
    - Conversion on the left-hand operand is needed.
        

**Important Considerations and Best Practices:**

- **Maintain Intuitive Behavior:** Overload operators in a way that is consistent with their expected behavior. For example, + should generally perform addition-like operations. Avoid surprising the user with unexpected semantics.
    
- **Be Consistent with Built-in Types:** Try to mimic the behavior of operators for built-in types where it makes sense for your class.
    
- **Consider Return Types:** Choose appropriate return types for your overloaded operators. For example, arithmetic operators often return a new object representing the result, while assignment operators typically return a reference to the modified object.
    
- **Avoid Ambiguity:** Ensure that your overloaded operators do not create ambiguous situations where the compiler cannot determine which operator overload to use.
    
- **Document Overloaded Operators:** Clearly document the behavior of your overloaded operators so that other developers (and your future self) understand how they work.
    

**In Summary:**

Operator overloading is a powerful tool in C++ that allows you to customize the behavior of operators for your own classes, leading to more expressive and readable code. By understanding the rules, syntax, and best practices, you can effectively leverage this feature to create more natural and intuitive interfaces for your custom types. However, it should be used judiciously and with careful consideration to avoid creating confusing or unexpected behavior.

# **Related Concepts:**

- [[overloading vs overriding]]