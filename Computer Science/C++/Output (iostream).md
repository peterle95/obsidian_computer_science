---
memory: to_finish
tags:
  - mastered
language:
  - C++
review-date: ""
last-reviewed: 2025-10-21
scheda: done
visit-count: 3
confidence-level: 2.5
consecutive-correct: 3
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

The `<iostream>` header provides the foundational tools for interacting with the user and external devices, primarily through streams. It defines objects and manipulators that facilitate the flow of data into and out of your program.

**Extensive Explanation:**

The `<iostream>` library is built around the concept of **streams**. A stream is an abstraction representing a source of input or a destination for output. Think of it as a conduit through which data flows. This abstraction allows you to interact with various input/output mechanisms (like the keyboard, console, files, network sockets) in a uniform way.

**Key Components and Concepts:**

* **Standard Streams:**  `<iostream>` predefines several standard stream objects:
    * **`std::cin` (Standard Input Stream):**  Typically connected to the keyboard. Used to read data entered by the user.
    * **`std::cout` (Standard Output Stream):** Typically connected to the console (terminal window). Used to display output to the user.
    * **`std::cerr` (Standard Error Stream):** Also typically connected to the console, but intended for error messages and diagnostics. It is usually unbuffered, meaning output appears immediately.
    * **`std::clog` (Standard Log Stream):** Similar to `std::cerr`, but it is buffered. This can be more efficient for logging large amounts of data.

* **Stream Operators:** These operators are used to insert data into an output stream or extract data from an input stream.
    * **Insertion Operator (`<<`):**  Often called the "put to" operator. It sends data to an output stream. You can chain insertions: `std::cout << "Hello, " << "world!" << std::endl;`
    * **Extraction Operator (`>>`):** Often called the "get from" operator. It reads data from an input stream and stores it in a variable. Like insertion, you can chain extractions: `int age; std::string name; std::cin >> name >> age;`

* **Manipulators:** These are special objects that modify the behavior or formatting of streams. They are inserted into or extracted from streams just like data. Some common manipulators include:
    * **`std::endl`:** Inserts a newline character and flushes the output buffer. Flushing ensures that the output is immediately displayed.
    * **`std::flush`:**  Only flushes the output buffer without inserting a newline.
    * **`std::setw(int n)` (from `<iomanip>`):** Sets the field width for the next output operation to `n` characters. This is useful for aligning output.
    * **`std::setprecision(int n)` (from `<iomanip>`):** Sets the precision (number of digits after the decimal point) for floating-point output.
    * **`std::fixed` (from `<iomanip>`):** Forces floating-point numbers to be displayed in fixed-point notation (e.g., 3.14 instead of 3.14e+00).
    * **`std::scientific` (from `<iomanip>`):** Forces floating-point numbers to be displayed in scientific notation.
    * **`std::hex`, `std::oct`, `std::dec`:** Set the base for integer output (hexadecimal, octal, decimal, respectively).
    * **`std::uppercase`, `std::nouppercase`:** Control the case of hexadecimal digits and the 'E' in scientific notation.
    * **`std::left`, `std::right`, `std::internal`:** Control the alignment of output within the specified field width.

* **Stream States:** Each stream object maintains a state that reflects its current condition. You can check these states to handle potential errors during input/output operations. Common state flags include:
    * **`good()`:** Returns `true` if no error flags are set.
    * **`eof()`:** Returns `true` if the end-of-file has been reached on an input stream.
    * **`fail()`:** Returns `true` if a logical error has occurred during the last input/output operation (e.g., trying to read an integer when the input is a string). `fail()` also includes the `bad()` state.
    * **`bad()`:** Returns `true` if a severe error has occurred, making the stream unusable (e.g., a hardware failure).

* **File Input/Output (fstream):**  The `<fstream>` header (which implicitly includes `<iostream>`) provides classes for working with files:
    * **`std::ifstream`:**  Used for reading data from files.
    * **`std::ofstream`:** Used for writing data to files.
    * **`std::fstream`:** Used for both reading and writing to files.
    You can open files using their constructors or the `open()` method, perform input/output operations using `<<` and `>>`, and close files using the `close()` method. It's crucial to check if a file was opened successfully using the `is_open()` method or by directly checking the stream object's truthiness.

* **String Streams (sstream):** The `<sstream>` header (also implicitly includes `<iostream>`) provides classes for working with strings as if they were streams:
    * **`std::stringstream`:** Allows both input and output operations on a string. Useful for formatting data into a string or parsing data from a string.
    * **`std::istringstream`:**  For reading data from a string.
    * **`std::ostringstream`:** For writing data to a string.

**Complete Explanation with Examples:**

```cpp
#include <iostream>
#include <fstream>
#include <sstream>
#include <iomanip>
#include <string>

int main() {
    // Standard Output (cout)
    std::cout << "Hello, world!" << std::endl;
    int age = 30;
    std::cout << "My age is: " << age << std::endl;
    double pi = 3.1415926535;
    std::cout << "Pi with default precision: " << pi << std::endl;
    std::cout << "Pi with set precision (3): " << std::setprecision(3) << pi << std::endl;
    std::cout << "Pi in fixed notation (3): " << std::fixed << std::setprecision(3) << pi << std::endl;
    std::cout << "Integer in hexadecimal: " << std::hex << age << std::endl;
    std::cout << "Integer back to decimal: " << std::dec << age << std::endl;

    // Standard Input (cin)
    std::string name;
    std::cout << "Enter your name: ";
    std::cin >> name;
    std::cout << "Hello, " << name << "!" << std::endl;

    int number;
    std::cout << "Enter an integer: ";
    if (std::cin >> number) { // Check if input was successful
        std::cout << "You entered: " << number << std::endl;
    } else {
        std::cout << "Invalid input. Please enter an integer." << std::endl;
        std::cin.clear(); // Clear error flags
        std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n'); // Discard bad input
    }

    // File Output (ofstream)
    std::ofstream outputFile("output.txt");
    if (outputFile.is_open()) {
        outputFile << "This is some text written to the file." << std::endl;
        outputFile << "Another line of text." << std::endl;
        outputFile.close();
        std::cout << "Data written to output.txt" << std::endl;
    } else {
        std::cerr << "Unable to open output file." << std::endl;
    }

    // File Input (ifstream)
    std::ifstream inputFile("output.txt");
    std::string line;
    if (inputFile.is_open()) {
        std::cout << "\nReading from output.txt:" << std::endl;
        while (std::getline(inputFile, line)) { // Read line by line
            std::cout << line << std::endl;
        }
        inputFile.close();
    } else {
        std::cerr << "Unable to open input file." << std::endl;
    }

    // String Stream (stringstream)
    std::stringstream ss;
    ss << "The answer is " << 42;
    std::string result = ss.str();
    std::cout << "\nString stream result: " << result << std::endl;

    std::stringstream parser("123 4.56 Hello");
    int val1;
    double val2;
    std::string val3;
    parser >> val1 >> val2 >> val3;
    std::cout << "Parsed values: " << val1 << ", " << val2 << ", " << val3 << std::endl;

    return 0;
}
```

**Best Practices and Considerations:**

- **Include necessary headers:** Always include <iostream> for basic input/output, <fstream> for file operations, <sstream> for string streams, and <iomanip> for manipulators.
    
- Check stream states: After input operations, especially with std::cin, check the stream's state (e.g., using if (std::cin >> number)) to handle potential errors and invalid input gracefully.
    
- Clear error flags and ignore bad input: If an input operation fails, use std::cin.clear() to clear the error flags and std::cin.ignore() to discard the problematic input from the buffer to prevent infinite loops.
    
- Flush output buffers when necessary: Use std::endl or std::flush when you need to ensure that output is immediately displayed, especially for important messages or interactive prompts.
    
- Close files explicitly While destructors will often close files, it's good practice to explicitly call close() on ifstream and ofstream objects when you are finished with them to release resources promptly.
    
- Use string streams for formatting and parsing Leverage std::stringstream for converting between strings and other data types, and for complex string formatting tasks.
    
- Be mindful of locale settings: The behavior of some input/output operations can be affected by the current locale (e.g., how decimal points are represented).
