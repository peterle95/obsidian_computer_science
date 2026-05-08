---
memory: to_finish
tags:
  - learned
language:
  - C++
review-date: ""
last-reviewed: 2025-10-21
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

The "cmath" header provides a wide array of mathematical functions that perform common calculations, from basic arithmetic to more advanced trigonometric and exponential operations.

**Extensive Explanation:**

The "cmath" library offers a collection of functions designed to perform standard mathematical operations. These functions are generally efficient and well-optimized, making them suitable for a wide range of numerical computations.

**Key Categories and Functions:**

- **Trigonometric Functions:** These functions operate on angles typically expressed in radians.
    
    - **std::sin(double angle):** Calculates the sine of an angle.
        
    - **std::cos(double angle):** Calculates the cosine of an angle.
        
    - **std::tan(double angle):** Calculates the tangent of an angle.
        
    - **std::asin(double x):** Calculates the arcsine (inverse sine) of x, returning the angle in radians. The argument x must be in the range [-1, 1].
        
    - **std::acos(double x):** Calculates the arccosine (inverse cosine) of x, returning the angle in radians. The argument x must be in the range [-1, 1].
        
    - **std::atan(double x):** Calculates the arctangent (inverse tangent) of x, returning the angle in radians.
        
    - **std::atan2(double y, double x):** Calculates the arctangent of y/x, but uses the signs of both arguments to determine the quadrant of the result, returning the angle in radians in the range [-π, π].
        
- **Hyperbolic Functions:** These functions are analogous to trigonometric functions but defined using hyperbolas.
    
    - **std::sinh(double x):** Calculates the hyperbolic sine of x.
        
    - **std::cosh(double x):** Calculates the hyperbolic cosine of x.
        
    - **std::tanh(double x):** Calculates the hyperbolic tangent of x.
        
    - **std::asinh(double x):** Calculates the inverse hyperbolic sine of x.
        
    - **std::acosh(double x):** Calculates the inverse hyperbolic cosine of x.
        
    - **std::atanh(double x):** Calculates the inverse hyperbolic tangent of x.
        
- **Exponential and Logarithmic Functions:**
    
    - **std::exp(double x):** Calculates the exponential function e<sup>x</sup>.
        
    - **std::log(double x):** Calculates the natural logarithm (base e) of x. The argument x must be positive.
        
    - **std::log10(double x):** Calculates the base-10 logarithm of x. The argument x must be positive.
        
    - **std::log2(double x):** Calculates the base-2 logarithm of x. The argument x must be positive.
        
    - **std::expm1(double x):** Calculates e<sup>x</sup> - 1, providing better precision for values of x close to zero.
        
    - **std::log1p(double x):** Calculates log(1 + x), providing better precision for values of x close to zero.
        
- **Power Functions:**
    
    - **std::pow(double base, double exponent):** Calculates base raised to the power of exponent.
        
    - **std::sqrt(double x):** Calculates the square root of x. The argument x must be non-negative.
        
    - **std::cbrt(double x):** Calculates the cube root of x.
        
- **Rounding and Remainder Functions:**
    
    - **std::ceil(double x):** Returns the smallest integer not less than x.
        
    - **std::floor(double x):** Returns the largest integer not greater than x.
        
    - **std::round(double x):** Returns the nearest integer to x. Rounds halfway cases away from zero (e.g., 2.5 rounds to 3, -2.5 rounds to -3).
        
    - **std::trunc(double x):** Returns the integer part of x (rounds towards zero).
        
    - **std::fmod(double numer, double denom):** Calculates the floating-point remainder of numer divided by denom. The result has the same sign as numer.
        
    - **std::remainder(double numer, double denom):** Calculates the floating-point remainder of numer divided by denom. The result is the value numer - n * denom, where n is the integer nearest to the exact value of numer / denom.
        
- **Absolute Value and Sign Functions:**
    
    - **std::fabs(double x):** Calculates the absolute value of a floating-point number x.
        
    - **std::copysign(double magnitude, double sign):** Returns a value with the magnitude of magnitude and the sign of sign.
        
- **Classification Functions:** These functions help determine the nature of floating-point numbers.
    
    - **std::fpclassify(double x):** Returns an integer value representing the classification of x (e.g., FP_INFINITE, FP_NAN, FP_NORMAL, FP_SUBNORMAL, FP_ZERO).
        
    - **std::isfinite(double x):** Checks if x is a finite number (not infinity or NaN).
        
    - **std::isinf(double x):** Checks if x is infinite.
        
    - **std::isnan(double x):** Checks if x is "Not a Number" (NaN).
        
    - **std::isnormal(double x):** Checks if x is a normal number (not zero, subnormal, infinite, or NaN).
        
    - **std::signbit(double x):** Checks if the sign bit of x is set (i.e., if x is negative).
        
- **Constants:** "cmath" also defines some useful mathematical constants (often as macros, but accessible within the std namespace in C++11 and later):
    
    - **M_PI (or std::numbers::pi in C++20):** The value of pi (approximately 3.14159...).
        

**Complete Explanation with Examples:**
 ```cpp
#include <iostream>
#include <cmath>
#include <numbers> // For C++20 standard constants

int main() {
    double angle_degrees = 45.0;
    double angle_radians = angle_degrees * std::numbers::pi / 180.0; // Convert degrees to radians

    // Trigonometric functions
    std::cout << "sin(" << angle_degrees << " degrees): " << std::sin(angle_radians) << std::endl;
    std::cout << "cos(" << angle_degrees << " degrees): " << std::cos(angle_radians) << std::endl;
    std::cout << "tan(" << angle_degrees << " degrees): " << std::tan(angle_radians) << std::endl;

    double value = 0.5;
    std::cout << "asin(" << value << "): " << std::asin(value) << " radians" << std::endl;

    // Exponential and logarithmic functions
    double x = 2.0;
    std::cout << "exp(" << x << "): " << std::exp(x) << std::endl;
    std::cout << "log(" << x << "): " << std::log(x) << std::endl;
    std::cout << "log10(" << x << "): " << std::log10(x) << std::endl;

    // Power functions
    double base = 3.0;
    double exponent = 2.5;
    std::cout << "pow(" << base << ", " << exponent << "): " << std::pow(base, exponent) << std::endl;
    std::cout << "sqrt(" << 9.0 << "): " << std::sqrt(9.0) << std::endl;
    std::cout << "cbrt(" << 27.0 << "): " << std::cbrt(27.0) << std::endl;

    // Rounding functions
    double num_to_round = 3.7;
    std::cout << "ceil(" << num_to_round << "): " << std::ceil(num_to_round) << std::endl;
    std::cout << "floor(" << num_to_round << "): " << std::floor(num_to_round) << std::endl;
    std::cout << "round(" << num_to_round << "): " << std::round(num_to_round) << std::endl;
    std::cout << "trunc(" << num_to_round << "): " << std::trunc(num_to_round) << std::endl;

    // Absolute value
    double negative_num = -5.2;
    std::cout << "fabs(" << negative_num << "): " << std::fabs(negative_num) << std::endl;

    // Remainder
    std::cout << "fmod(10.5, 3.0): " << std::fmod(10.5, 3.0) << std::endl;
    std::cout << "remainder(10.5, 3.0): " << std::remainder(10.5, 3.0) << std::endl;

    // Classification functions
    double nan_val = std::nan("");
    std::cout << "isnan(" << nan_val << "): " << std::boolalpha << std::isnan(nan_val) << std::endl;

    return 0;
}
```

**Best Practices and Considerations:**

- **Include the "cmath" header:** Always include "cmath" to use the mathematical functions.
    
- **Use radians for trigonometric functions:** Be mindful that trigonometric functions in cmath typically work with angles in radians. Convert degrees to radians if necessary.
    
- **Handle potential errors:** Be aware of domain errors (e.g., taking the square root of a negative number or the logarithm of zero). These can lead to undefined behavior or return NaN.
    
- **Consider function overloads:** Many functions in cmath are overloaded to work with different floating-point types (float, double, long double). Choose the appropriate version for your needs.
    
- **Be aware of floating-point precision:** Remember that floating-point numbers have limited precision, which can lead to small inaccuracies in calculations.
    
- **Use constants from "numbers" (C++20):** If you are using C++20 or later, prefer using constants like std::numbers::pi for better type safety and precision compared to the macro M_PI.
    
- **Consult documentation:** For detailed information on each function's behavior, edge cases, and potential errors, refer to the C++ standard library documentation.
