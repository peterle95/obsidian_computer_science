---
memory: to_finish
tags:
  - learned
language:
  - C++
review-date: ""
last-reviewed: 2025-10-16
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

# **Core Explanation:**

## File Manipulation in C++

C++ provides powerful file I/O capabilities through the standard `<fstream>` library. This [[fstream library]] includes two main classes for working with files: `std::ifstream` for reading from files and `std::ofstream` for writing to files.

### When to Employ File I/O

- **Persistent Data Storage:** When you need to store data that persists beyond the execution of your program, files come into play. This encompasses scenarios like saving user preferences, storing game progress, or managing large datasets that exceed available memory.
- **Large Datasets:** For handling datasets larger than your computer's main memory, files provide a mechanism to process data in manageable chunks.
- **Data Sharing:** Files serve as a common format for sharing data between programs or systems.
- **Offline Processing:** When data needs to be available even when the system is offline, storing it in a file makes it accessible without requiring a constant network connection.
- **Testing and Debugging:** Files provide a stable and repeatable input source for testing and debugging your programs. This repeatability can be invaluable during development.

### Reading from a File

To read from a file, you can use the `std::ifstream` class. In the provided example, we see the following line:

```cpp
std::ifstream inFile(filename.c_str());
```

Let's break this down:

1. `std::ifstream inFile` - This declares a variable `inFile` of type `std::ifstream`, which represents the input file stream.
2. `(filename.c_str())` - The constructor for `std::ifstream` takes a C-style string (null-terminated character array) as its argument. The `c_str()` method is called on the `std::string` object `filename` to obtain this C-style string representation.

This line opens the file specified by the `filename` variable for reading.

### Writing to a File

To write to a file, you can use the `std::ofstream` class. In the example, we see the following lines:

```cpp
std::string outFilename = filename + ".replace";
std::ofstream outFile(outFilename.c_str());
```

1. `std::string outFilename = filename + ".replace";` - This creates a new `std::string` object `outFilename` by concatenating the original `filename` with the ".replace" suffix. This will be the name of the output file.
2. `std::ofstream outFile(outFilename.c_str());` - This declares a variable `outFile` of type `std::ofstream`, which represents the output file stream. The constructor takes the C-style string representation of the `outFilename` variable, obtained using the `c_str()` method.

This line opens the file specified by the `outFilename` variable for writing. The new file will be created if it doesn't already exist, or truncated if it does.

By using `std::ifstream` for reading and `std::ofstream` for writing, you can perform various file manipulation tasks, such as reading the contents of a file, modifying the data, and writing the updated content to a new file.

Remember to check the success of file opening operations and handle any errors that may occur during file I/O. Additionally, it's important to close the file streams when you're done with them to ensure proper resource management.

## Important methods:

* [[getline]]
* [[close]]
* [[C++ - empty]]
