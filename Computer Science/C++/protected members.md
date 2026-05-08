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
consecutive-correct: 2
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

Protected access enables direct member access in derived classes while maintaining encapsulation from external code. This design choice:

* ==Allows derived classes to modify attributes directly without getters/setters==
* Maintains proper encapsulation hierarchy - only immediate descendants get access
* Reduces boilerplate code while keeping implementation details hidden from unrelated classes

Tradeoff: Weakens strict encapsulation but is appropriate for tightly coupled inheritance hierarchies where derived classes need intimate knowledge of base implementation

Let's look at a practical example comparing protected and private access modifiers in a base class and a derived class scenario, using a simplified representation (like in Python, C++, or Java-like pseudocode):

**Example: Shape Hierarchy (Illustrating Protected vs. Private)**

Imagine we are building a system to represent geometric shapes.

**Scenario 1: Using private members in the base class Shape**

```cpp
class Shape 
{   
	private int x;       // x-coordinate of the shape's position   
	private int y;       // y-coordinate of the shape's position    
	public Shape(int x, int y) {     
		this.x = x;     
		this.y = y;   
	}    
	public int getX() { return x; }   // Getter for x   
	public int getY() { return y; }   // Getter for y   
	public void setPosition(int newX, int newY) { // Setter for position     
	this.x = newX;     
	this.y = newY;   
	}    

	public virtual void draw() {     
		print("Drawing a generic shape at (" + x + ", " + y + ")");   
	} 
} 

class Circle extends Shape {   
	private int radius;    
	public Circle(int x, int y, int radius) : base(x, y) 
	{     
		this.radius = radius;   
	}    
	public void adjustRadiusBasedOnPosition() {     
		// Problem! Cannot directly access x and y from Shape     
		// We MUST use getters (getX(), getY())     
		int currentX = getX();     
		int currentY = getY();     
		radius = radius + (currentX + currentY) / 10; // Example logic   
	}    
	public override void draw() {     
		print("Drawing a circle at (" + getX() + ", " + getY() + ") with radius " + radius);   
	} 
}  
// Example Usage: 
Shape genericShape = new Shape(10, 20); 
Circle myCircle = new Circle(30, 40, 5); 
myCircle.adjustRadiusBasedOnPosition(); 
myCircle.draw(); 
// Output: Drawing a circle at (30, 40) with radius ... (radius will be adjusted) 
genericShape.draw(); 
// Output: Drawing a generic shape at (10, 20)`
```

**In Scenario 1 (using private):**

- **Strict Encapsulation:** x and y in Shape are strictly encapsulated. Only Shape's methods can directly access them.
    
- **Derived Class Limitation:** Circle (and any other derived shape like Rectangle, Triangle, etc.) cannot directly access x and y. It must use the public getters (getX(), getY()) provided by the Shape class.
    
- **Boilerplate in Base Class:** The Shape class needs to provide explicit getters and setters if derived classes (or external code) need to interact with the position.
    

**Scenario 2: Using protected members in the base class Shape**

```cpp

class Shape {   
	protected int x;       // x-coordinate of the shape's position (PROTECTED)   
	protected int y;       // y-coordinate of the shape's position (PROTECTED)    
	public Shape(int x, int y) {     this.x = x;     this.y = y;   }    
// Getters and setters are now optional, depending on design needs.   
// We might still have some public methods for controlled interaction    
	public virtual void draw() 
	{     
		print("Drawing a generic shape at (" + x + ", " + y + ")");   
	} 
}

class Circle extends Shape {  
	private int radius;    
	public Circle(int x, int y, int radius) : base(x, y) 
	{     
		this.radius = radius;   
	}    
	public void adjustRadiusBasedOnPosition() 
	{     
		// Now we can directly access x and y!     
		x += 5; // Directly modify x!  (Example - might not be good design in all cases)
		y -= 2; // Directly modify y!     
		radius = radius + (x + y) / 10; // Example logic using direct access   
	}    
	public override void draw() 
	{     
		print("Drawing a circle at (" + x + ", " + y + ") with radius " + radius);   
	} 
}  

// Example Usage (same as before, but behavior might be slightly different due to direct modification): 
Shape genericShape = new Shape(10, 20); 
Circle myCircle = new Circle(30, 40, 5);  
myCircle.adjustRadiusBasedOnPosition(); 
myCircle.draw(); // Output: Drawing a circle at (35, 38) with radius ... (radius and position adjusted directly) 
genericShape.draw(); // Output: Drawing a generic shape at (10, 20) (position of genericShape is unaffected)
```

**In Scenario 2 (using protected):**

- **Relaxed Encapsulation (within hierarchy):** x and y are still encapsulated from outside classes. However, Circle (and other derived classes) can directly access and modify them.
    
- **Derived Class Convenience:** Circle can directly work with x and y without the need for getters and setters, leading to potentially cleaner and more efficient code within the inheritance hierarchy.
    
- **Reduced Boilerplate (potentially):** The Shape class might not need to provide as many getters and setters if direct access by derived classes is intended and considered safe within the design.
    

**Key Takeaways from the Example:**

- **private enforces strict encapsulation:** Even derived classes are treated as "external" in terms of direct access. This is the strongest form of encapsulation.
    
- **protected relaxes encapsulation for inheritance:** It allows derived classes to have "intimate knowledge" and direct access to base class members, while still protecting those members from unrelated classes.
    
- **Tradeoff is real:** protected makes derived classes more dependent on the internal implementation of the base class. If you change x or y in Shape (e.g., change their data type or meaning), you might break derived classes that directly access them. With private and getters/setters, you have more flexibility to change the internal representation of Shape without necessarily breaking derived classes (as long as the getter/setter interface remains consistent).
    

**When is protected appropriate?**

As your Obsidian note suggests, protected is most appropriate when:

- **Tightly Coupled Inheritance Hierarchy:** You are designing a set of classes that are intrinsically related and will evolve together. Derived classes are designed to be deeply intertwined with the base class's implementation.
    
- **Performance or Convenience Matters within the Hierarchy:** Avoiding getters/setters can sometimes improve performance slightly, and it can definitely make code within the derived classes more concise and easier to read when dealing with core base class attributes.
    
- **Controlled Exposure:** You want to give derived classes more freedom but still maintain a level of encapsulation from the outside world. You want to prevent unrelated classes from messing with the internal state of the base class.
    

**When to prefer private?**

- **Loose Coupling and Reusability:** When you want to create base classes that are more independent and reusable in different contexts. You want to minimize the dependencies of derived classes on the internal details of the base class.
    
- **Strong Encapsulation is Paramount:** When you prioritize strict encapsulation and want to control all access to internal state through well-defined public interfaces (getters/setters).
    
- **Anticipating Evolution and Change:** When you expect the base class's implementation to change significantly over time, and you want to minimize the risk of breaking derived classes.
    

**In conclusion, the choice between protected and private is a design decision based on the specific needs of your inheritance hierarchy and the balance you want to strike between encapsulation, convenience, and maintainability.** protected is a tool for specific scenarios where a looser form of encapsulation within the inheritance hierarchy is beneficial, but it should be used thoughtfully and with an understanding of its implications.

