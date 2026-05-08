---
memory: to_finish
tags:
  - learned
language:
  - C++
review-date: ""
last-reviewed: 2025-10-11
scheda: done
visit-count: 2
confidence-level: 2
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

## Overview
Fixed-point numbers represent a different approach to handling numerical values in programming, distinct from both floating-point numbers and integers. They offer a unique balance between precision and performance.

## What are Fixed-Point Numbers?
Fixed-point numbers represent values with a **fixed number of digits before and after the decimal point**. This contrasts with:
- Floating-point numbers where the decimal point can "float"
- Integers which only handle whole numbers

## Comparison of Number Types

| Number Type    | Representation                                  | Range               | Precision                                 |
| -------------- | ----------------------------------------------- | ------------------- | ----------------------------------------- |
| Fixed-Point    | Fixed number of digits before and after decimal | Limited             | Constant; determined by fractional digits |
| Floating-Point | Decimal point "floats"                          | Wide                | Variable; better for smaller numbers      |
| Integer        | Whole numbers only                              | Limited to integers | Exact within range                        |

## Advantages of Fixed-Point Numbers

Fixed-point numbers bridge the gap between integers and floating-point numbers, offering several benefits:

1. **Performance**
   - Faster arithmetic operations compared to floating-point
   - More efficient memory usage

2. **Hardware Compatibility**
   - Useful for platforms without floating-point support
   - Common in [[embedded systems]]

3. **Precision Control**
   - Consistent precision level
   - Predictable rounding behavior

## Limitations

The main tradeoffs when using fixed-point numbers:

- Limited range compared to floating-point
- Must choose between:
    - More digits before decimal (larger numbers)
    - More digits after decimal (higher precision)


**1. The "Why": The Problem with Floating-Point Numbers**

Before understanding fixed-point, it's helpful to appreciate why we sometimes need alternatives to the more common floating-point numbers (like float and double).

- **Performance:** Floating-point operations can be computationally expensive, especially on embedded systems or platforms without dedicated Floating-Point Units (FPUs).
    
- **Determinism and Reproducibility:** [[Floating-Point Numbers]] arithmetic can sometimes lead to subtle variations in results across different platforms or even different compiler optimizations. This can be problematic in areas like control systems, simulations, or financial calculations where precise and repeatable results are critical.
    
- **Memory Usage:** float and double typically occupy 4 or 8 bytes, respectively. In memory-constrained environments, you might need a more compact representation for fractional values.
    
- **Exact Representation:** Many decimal fractions (like 0.1) cannot be represented exactly in binary floating-point. This can lead to small rounding errors accumulating over multiple calculations.
    

**2. What are Fixed-Point Values? (The Basic Idea)**

Fixed-point numbers offer a way to represent fractional values using integers. The core idea is to allocate a fixed number of bits for the integer part and a fixed number of bits for the fractional part of a number. Think of it like this:

Imagine you have a 16-bit number. You could decide that the first 8 bits represent the integer part, and the last 8 bits represent the fractional part.

```
[ Integer Part (8 bits) ] [ Fractional Part (8 bits) ]
```


**Key Concepts:**

- **Integer Representation:** The underlying storage is still an integer type (like int, short, or long long).
    
- **Implicit Decimal Point:** The position of the decimal point is implied or fixed based on your chosen bit allocation. It's not physically stored.
    
- **Scaling Factor:** You essentially multiply the "true" value by a scaling factor to represent it as an integer. The scaling factor is a power of 2 (e.g., 2<sup>8</sup> if you have 8 fractional bits).
    

**Example:**

Let's say you have 8 fractional bits. This means your scaling factor is 2<sup>8</sup> = 256.

- To represent the value 1.5:
    
    - Multiply by the scaling factor: 1.5 * 256 = 384
        
    - Store the integer value 384.
        
- To interpret the stored integer 384:
    
    - Divide by the scaling factor: 384 / 256 = 1.5
        

**3. Why Use Fixed-Point? (The Benefits Revisited)**

- **Performance:** Integer arithmetic is generally much faster than floating-point arithmetic, especially on systems without FPUs.
    
- **Determinism:** Fixed-point arithmetic is deterministic. Given the same inputs and operations, you'll always get the same output.
    
- **Memory Efficiency:** You can choose the size of your integer type to match your precision requirements, potentially saving memory.
    
- **Exact Representation of Certain Fractions:** You can exactly represent fractions that can be expressed as a power of 2 divided by your scaling factor.
    

**4. The Trade-offs (When Fixed-Point Might Not Be Ideal)**

- **Limited Range:** The range of values you can represent is limited by the number of bits you allocate for the integer part. If your values get too large or too small, you'll experience overflow or underflow.
    
- **Manual Scaling and Precision Management:** You need to be mindful of the scaling factor during calculations. Incorrect scaling can lead to loss of precision or incorrect results.
    
- **More Complex Implementation:** You don't get the built-in convenience of floating-point types. You'll need to implement or use libraries for fixed-point operations.
    

**5. Representing Fixed-Point Values in C++**

C++ doesn't have a built-in fixed_point data type like it has float and double. You have a few common approaches:

**a) Using Standard Integer Types with Explicit Scaling:**

This is the most basic approach. You use regular integer types and handle the scaling explicitly in your code.

```cpp
#include <iostream>

using namespace std;

int main() {
  // Let's use 8 fractional bits (scaling factor 2^8 = 256)
  const int fractionalBits = 8;
  const int scalingFactor = 1 << fractionalBits;

  // Represent 1.5
  int fixedPoint_1_5 = static_cast<int>(1.5 * scalingFactor); // fixedPoint_1_5 will be 384

  // Represent 0.75
  int fixedPoint_0_75 = static_cast<int>(0.75 * scalingFactor); // fixedPoint_0_75 will be 192

  // Perform addition (needs scaling adjustment after)
  int sum = fixedPoint_1_5 + fixedPoint_0_75; // sum is 576

  // Convert back to the actual value
  double actualSum = static_cast<double>(sum) / scalingFactor; // actualSum is 2.25

  cout << "Fixed-point 1.5: " << fixedPoint_1_5 << endl;
  cout << "Fixed-point 0.75: " << fixedPoint_0_75 << endl;
  cout << "Sum (fixed-point): " << sum << endl;
  cout << "Actual Sum: " << actualSum << endl;

  return 0;
}
```


**Important Considerations with Basic Integer Scaling:**

- **Multiplication:** When multiplying two fixed-point numbers, you effectively multiply their scaling factors as well. You'll need to divide by the scaling factor once to get the correct result.
    
- **Division:** When dividing, you'll need to multiply by the scaling factor.
    
- **Overflow/Underflow:** Be very careful about potential overflows or underflows during intermediate calculations. You might need to use larger integer types for intermediate results.
    

**b) Creating a Fixed-Point Class:**

For more complex applications, it's often beneficial to create a custom fixed-point class. This encapsulates the scaling factor and provides overloaded operators for easier arithmetic.

```cpp
#include <iostream>

class FixedPoint {
private:
  int value;
  static const int fractionalBits = 8;
  static const int scalingFactor = 1 << fractionalBits;

public:
  // Constructor from double
  FixedPoint(double val = 0.0) : value(static_cast<int>(val * scalingFactor)) {}

  // Constructor from integer
  FixedPoint(int val) : value(val << fractionalBits) {}

  // Convert to double
  double toDouble() const {
    return static_cast<double>(value) / scalingFactor;
  }

  // Overload addition
  FixedPoint operator+(const FixedPoint& other) const {
    return FixedPoint(static_cast<double>(value + other.value) / scalingFactor);
  }

  // Overload subtraction
  FixedPoint operator-(const FixedPoint& other) const {
    return FixedPoint(static_cast<double>(value - other.value) / scalingFactor);
  }

  // Overload multiplication
  FixedPoint operator*(const FixedPoint& other) const {
    // Need to divide by the scaling factor after multiplication
    return FixedPoint(static_cast<double>(static_cast<long long>(value) * other.value) / (scalingFactor * scalingFactor));
  }

  // Overload division
  FixedPoint operator/(const FixedPoint& other) const {
    // Need to multiply by the scaling factor before division
    return FixedPoint(static_cast<double>(static_cast<long long>(value) * scalingFactor) / other.value);
  }

  friend std::ostream& operator<<(std::ostream& os, const FixedPoint& fp) {
    os << fp.toDouble();
    return os;
  }
};

int main() {
  FixedPoint fp1(1.5);
  FixedPoint fp2(0.75);

  FixedPoint sum = fp1 + fp2;
  FixedPoint product = fp1 * fp2;

  cout << "fp1: " << fp1 << endl;
  cout << "fp2: " << fp2 << endl;
  cout << "Sum: " << sum << endl;
  cout << "Product: " << product << endl;

  return 0;
}
```


**Key Improvements with the Class Approach:**

- **[[C++ Encapsulation]]:** The scaling factor is hidden within the class, reducing the chance of errors.
    
- **[[C++ Operator Overloading]]:** Makes the code more readable and natural, as you can use standard arithmetic operators.
    
- **Type Safety:** Helps prevent mixing regular integers with fixed-point values unintentionally.
    

**c) Using Existing Fixed-Point Libraries:**

Several libraries provide robust and well-tested fixed-point implementations in C++. These libraries often handle more advanced features like different rounding modes, saturation arithmetic (preventing overflow by clamping values), and various fixed-point formats. Some popular options include:

- **Boost.Fixed_Point:** Part of the Boost C++ Libraries, offering a comprehensive fixed-point implementation.
    
- **libfixmath:** A lightweight library specifically designed for fixed-point math.
    

Using a library is generally recommended for complex or critical applications, as they often have better performance and error handling than basic manual implementations.

**6. Performing Common Operations:**

- **Addition and Subtraction:** These are relatively straightforward. If both operands have the same scaling factor, you can directly add or subtract the underlying integer values. If the scaling factors differ, you need to scale one of the operands before performing the operation.
    
- **Multiplication:** When multiplying two fixed-point numbers, the number of fractional bits in the result is the sum of the fractional bits of the operands. You'll typically need to shift the result to maintain the original scaling factor.
    
- **Division:** Division involves similar scaling adjustments to maintain the correct number of fractional bits in the result.
    
- **Comparisons:** You can directly compare the underlying integer values of fixed-point numbers with the same scaling factor.
    

**7. When to Use Fixed-Point Values**

- **Embedded Systems and Microcontrollers:** Where processing power and memory are limited.
    
- **Digital Signal Processing (DSP):** Many DSP algorithms benefit from the performance and determinism of fixed-point arithmetic.
    
- **Game Development:** For certain calculations like physics or AI, fixed-point can offer performance advantages.
    
- **Financial Applications:** Where exact results and reproducibility are crucial.
    
- **Simulations:** In some simulations, fixed-point arithmetic can be more efficient and predictable.
    

**8. Important Considerations and Best Practices**

- **Choose the Right Number of Fractional Bits:** This determines the precision of your representation. More fractional bits mean higher precision but also a smaller integer range.
    
- **Be Aware of Overflow and Underflow:** Carefully consider the potential range of your calculations and choose integer types and bit allocations that prevent overflow or underflow. Saturation arithmetic (offered by some libraries) can be helpful here.
    
- **Scaling Factor Consistency:** Ensure you are consistent with your scaling factors throughout your calculations.
    
- **Rounding:** Be mindful of rounding errors when converting between floating-point and fixed-point or during scaling adjustments. Consider different rounding modes (e.g., rounding to nearest, truncation).
    
- **Consider Using Libraries:** For complex scenarios or when correctness is critical, using a well-tested fixed-point library is often the best approach.
    

**In Summary:**

Fixed-point values offer a way to represent fractional numbers using integers, providing potential benefits in performance, determinism, and memory efficiency. However, they require careful management of scaling and precision. While C++ doesn't have a built-in fixed-point type, you can implement them manually or leverage existing libraries. Understanding the trade-offs and choosing the right approach based on your application's needs is crucial.
# **Related Concepts:**

# **Examples:**

