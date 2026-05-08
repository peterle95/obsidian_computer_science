---
memory: to_finish
tags:
 - to_learn
language:
 - JavaScript
review-date:
last-reviewed: ""
scheda: done
visit-count: 0
confidence-level: 1
consecutive-correct: 0
last-struggle-date: ""
cssclasses:

---
```dataviewjs
const currentPage = dv.current;
let visitCount = currentPage.file.frontmatter["visit-count"] || 0;
let confidence = currentPage.file.frontmatter["confidence-level"] || 1;
let streak = currentPage.file.frontmatter["consecutive-correct"] || 0;

const container = this.container.createEl('div');
container.style.cssText = `
 background:

# 2a2a2a; border: 1px solid

# 404040; border-radius: 6px;
 padding: 12px; margin: 10px 0; display: inline-block;
`;

// Status display
const status = container.createEl('div');
status.innerHTML = `
 <strong>Learning Progress</strong><br>
 Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
`;
status.style.cssText = 'margin-bottom: 10px; font-size: 13px; color:

# cccccc;';

// Quick feedback buttons
const buttonContainer = container.createEl('div');
['Got it! ✅', 'Struggled ⚠️', 'Failed ❌'].forEach((label, index) => {
 const btn = buttonContainer.createEl('button');
 btn.textContent = label;
 btn.style.cssText = `
 margin-right: 8px; padding: 4px 8px; border: none; border-radius: 3px;
 cursor: pointer; font-size: 11px;
 background: ${['

# 28a745', '

# ffc107', '

# dc3545'][index]};
 color: ${index === 1 ? '

# 000' : '

# fff'};
 `;

 btn.addEventListener('click', async => {
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
 fm["last-reviewed"] = new Date.toISOString.split('T');
 if (index > 0) fm["last-struggle-date"] = new Date.toISOString.split('T');
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
const currentPage = dv.current;
const content = await app.vault.read(app.vault.getAbstractFileByPath(currentPage.file.path));

// Split content into lines
const lines = content.split('\n');
let flashcardLines = ;
let inCodeBlock = false;

// Collect all potential flashcard lines - simplified approach
for (let i = 0; i < lines.length; i++) {
 const line = lines[i];

 // Track code blocks
 if (line.trim.startsWith('```')) {
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
 line.trim.startsWith('const ') ||
 line.trim.startsWith('let ') ||
 line.trim.startsWith('function ') ||
 line.trim.startsWith('return ') ||
 line.trim.startsWith('if (') ||
 line.trim.startsWith('for (') ||
 line.trim.startsWith('while (') ||
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
const flashcards = ;
for (let i = 0; i < filteredLines.length; i++) {
 const line = filteredLines[i];
 try {
 const separatorIndex = line.indexOf(';;');
 if (separatorIndex === -1) continue;

 const front = line.substring(0, separatorIndex).trim;
 const back = line.substring(separatorIndex + 2).trim;

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
 errorMsg.style.cssText = 'background:

# 2a2a2a; padding: 15px; border-radius: 6px; color:

# cccccc;';
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
 background:

# 2a2a2a;
 border: 1px solid

# 404040;
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
title.style.cssText = 'margin: 0; color:

# ffffff;';

const progress = header.createEl('div');
progress.style.cssText = 'color:

# cccccc; font-size: 14px; text-align: right;';

// Card container
const cardContainer = container.createEl('div');
cardContainer.style.cssText = `
 background:

# 1a1a1a;
 border: 2px solid

# 404040;
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
 color:

# ffffff;
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
 background:

# 4a9eff; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; font-weight: 500;
`;

const easyButton = buttonContainer.createEl('button');
easyButton.textContent = 'Easy ✅';
easyButton.style.cssText = `
 background:

# 28a745; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const hardButton = buttonContainer.createEl('button');
hardButton.textContent = 'Hard ❌';
hardButton.style.cssText = `
 background:

# dc3545; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const nextButton = buttonContainer.createEl('button');
nextButton.textContent = 'Next →';
nextButton.style.cssText = `
 background:

# 6c757d; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const prevButton = buttonContainer.createEl('button');
prevButton.textContent = '← Prev';
prevButton.style.cssText = `
 background:

# 6c757d; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const shuffleButton = buttonContainer.createEl('button');
shuffleButton.textContent = '🔀 Shuffle';
shuffleButton.style.cssText = `
 background:

# 17a2b8; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

// Functions
function updateDisplay {
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
 cardContainer.style.borderColor = '

# ffc107';
 cardContainer.style.backgroundColor = '

# 252525';
 } else {
 easyButton.style.display = 'none';
 hardButton.style.display = 'none';
 flipButton.textContent = 'Flip Card';
 cardContainer.style.borderColor = '

# 404040';
 cardContainer.style.backgroundColor = '

# 1a1a1a';
 }

 // Update navigation buttons
 prevButton.style.display = currentCardIndex > 0 ? 'inline-block' : 'none';
 nextButton.textContent = currentCardIndex < flashcards.length - 1 ? 'Next →' : 'Restart';
}

function flipCard {
 showingBack = !showingBack;
 updateDisplay;
}

function nextCard {
 if (currentCardIndex < flashcards.length - 1) {
 currentCardIndex++;
 } else {
 currentCardIndex = 0;
 }
 showingBack = false;
 updateDisplay;
}

function prevCard {
 if (currentCardIndex > 0) {
 currentCardIndex--;
 showingBack = false;
 updateDisplay;
 }
}

function markCorrect {
 if (showingBack) {
 correctCount++;
 totalReviewed++;
 nextCard;
 }
}

function markIncorrect {
 if (showingBack) {
 totalReviewed++;
 nextCard;
 }
}

function shuffleCards {
 for (let i = flashcards.length - 1; i > 0; i--) {
 const j = Math.floor(Math.random * (i + 1));
 [flashcards[i], flashcards[j]] = [flashcards[j], flashcards[i]];
 }
 currentCardIndex = 0;
 showingBack = false;
 correctCount = 0;
 totalReviewed = 0;
 updateDisplay;
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
instructions.style.cssText = 'font-size: 12px; color:

# 888; text-align: center; line-height: 1.4;';
instructions.innerHTML = `
 <strong>Controls:</strong> Click card to flip | Navigation buttons | Easy/Hard to mark
`;

// Initialize
updateDisplay;
```

# Purpose/Why:

---
Regular expressions (regex) provide a compact, declarative language for describing patterns in text. They solve problems related to searching, validating, extracting, and transforming text — common tasks in parsing, input validation, data cleaning, log analysis, and many automation workflows. In programming languages like JavaScript, Python, and others, regex is integrated into standard libraries and enables concise solutions that would otherwise require verbose string-manipulation code.

# Core Explanation:

---
- Definition: A regular expression is a sequence of characters that defines a search pattern. Patterns match substrings in target text according to rules: literal characters, character classes, quantifiers, groups, anchors, and alternation.
- Key characteristics:
 - Pattern-driven: you express "what" to match, not "how" to find it.
 - Compositional: small constructs (like \d or [a-z]) combine into larger expressions.
 - Greedy vs lazy matching: quantifiers default to greedy (match as much as possible); add ? to make them lazy (match as little as possible).
 - Flavors: syntax and features can vary slightly (ECMAScript/JS, PCRE, Python's re, etc.). Check flavor-specific docs when using advanced features.
- How it works:
 - Engine parses pattern into an internal structure and attempts to match input text, usually left-to-right.
 - Anchors (^, $) restrict positions; character classes match sets; quantifiers repeat subpatterns; groups capture or form sub-expressions; alternation (|) offers alternatives.
- Common operations:
 - test/match: check if pattern exists
 - search/find: locate occurrences
 - replace/substitute: transform text (e.g., str.replace in JS, re.sub in Python)
 - split: break text by pattern

#

# Cheatsheet Table: Patterns & Meaning

| Pattern | Meaning / Use |
|
---
|
---
|
| . | Any character except newline (depends on flag) |
| ^ | Start of string (or line with multiline flag) |
| $ | End of string (or line with multiline flag) |
| \d | Digit (equivalent to [0-9]) |
| \D | Non-digit |
| \w | Word character (letters, digits, underscore) |
| \W | Non-word character |
| \s | Whitespace (space, tab, newline) |
| \S | Non-whitespace |
| [abc] | Character class: match a, b, or c |
| [^abc] | Negated class: match any character except a, b, or c |
| [a-zA-Z0-9] | Range: letters and digits (case sensitive) |
| \b | Word boundary |
| \B | Not a word boundary |
| ? | Quantifier: 0 or 1 (make preceding token optional) |
| * | Quantifier: 0 or more |
| + | Quantifier: 1 or more |
| {n} | Exactly n repetitions |
| {n,} | n or more repetitions |
| {n,m} | Between n and m repetitions |
| | Capturing group |
| (?: ) | Non-capturing group |
| (?=...) | Positive lookahead (assertion) |
| (?!...) | Negative lookahead |
| (?<=...) | Positive lookbehind (if supported) |
| (?<!...) | Negative lookbehind (if supported) |
| \ | Escape special character (e.g., \., \*, \\) |
| | | Alternation: OR between patterns |
| x+? / *? / {n,m}? | Lazy (non-greedy) versions of quantifiers |
| /pattern/g | Global flag (JS): find all matches |
| /pattern/i | Case-insensitive flag |
| /pattern/m | Multiline flag: ^ and $ match line boundaries |
| /pattern/s | Dotall/Single-line flag: dot matches newline (flavor-specific) |
| ^...$ | Full-string match (useful for validation) |

# Common Practical Patterns (Examples)

---
- Digits-only: ^\d+$
- Optional sign with digits: ^[+-]?\d+$
- Remove +, -, and spaces: /[+\-\s]/g
- Basic email: ^[^\s@]+@[^\s@]+\.[^\s@]+$
- Simple URL: ^https?:\/\/\S+$
- ISO Date (YYYY-MM-DD): ^\d{4}-\d{2}-\d{2}$
- Time HH:MM (24h): ^(\d|2[0-3]):[0-5]\d$
- HTML tag (simple): /<([a-z]+)([^>]*?)>(.*?)<\/\1>/i (use with caution)
- Split camelCase into words: s.replace(/([a-z0-9])([A-Z])/g, '$1 $2')

#

# Tools & Testing

- regex101.com — interactive testing, explains groups and matches; choose flavor (PCRE, JS, Python).
- regexr.com — interactive tester and documentation.
- Language docs: MDN RegExp (JS), Python re module docs, PCRE docs.

#

# Best Practices & Pitfalls

- Prefer anchored patterns ( ^...$ ) for validation to avoid partial matches.
- Escape user input before embedding into a regex.
- Avoid overly complex single regexes when readability suffers; break into steps.
- Watch performance on large inputs: catastrophic backtracking can cause exponential runtime for some patterns (especially nested quantifiers with ambiguous groups).
- Use non-capturing groups (?: ) if you don't need capture to reduce overhead.
- Test patterns with representative edge cases (empty strings, extreme lengths, Unicode).

# Related Concepts:

---
- Finite Automata: Theoretical model underlying regex (regular languages). Regex corresponds to patterns matched by finite automata (some regex engines add features beyond pure regular languages).
- Parsing / Grammars: Regex is for regular languages (flat patterns). For nested or hierarchical structures (e.g., full programming languages), context-free grammars and parsers are used instead.
- Tokenization: Regex is often used to split input into tokens for parsing or processing.
- String methods/APIs: Language-specific APIs (e.g., JavaScript RegExp, Python re) that execute regex operations.
- Unicode handling: Character classes and boundaries behave differently with Unicode; many engines offer flags or escapes for Unicode-aware matching.
- Lookahead / Lookbehind: Advanced zero-width assertions used to require context without consuming characters.

# Examples:

---

JavaScript examples:

```js
// Example 1: Simple test for digits-only string
// This returns true for "12345", false for "12a45"
const digitsOnly = /^\d+$/; // ^ anchor = start, \d+ = one or more digits, $ anchor = end
// Usage:
"12345".match(digitsOnly); // matches
"12a45".match(digitsOnly); // null

// Example 2: Remove +, -, and spaces from an input string
// Use a character class with global flag to replace all instances
function cleanInputString(str) {
 // /[+\-\s]/g matches plus sign, minus sign, or any whitespace; 'g' = global
 return str.replace(/[+\-\s]/g, "");
}
// " +12 - 34 " => "1234"
cleanInputString(" +12 - 34 "); // "1234"

// Example 3: Capture named groups (modern JS) to parse YYYY-MM-DD
// This extracts year, month, day into groups for easy access
const dateRegex = /^(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})$/;
const m = "2025-08-12".match(dateRegex);
if (m) {
 // m.groups.year, m.groups.month, m.groups.day are available
 // Example: validate month range quickly (01-12)
}

// Example 4: Email (simple, practical validation - not RFC-perfect)
// Use a pragmatic pattern: non-space local part, '@', domain with dot
const simpleEmail = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
// This checks typical email formatting without full RFC complexity
simpleEmail.test("alice@example.com"); // true
simpleEmail.test("bad@@example"); // false

// Example 5: Replace multiple whitespace runs with single space (normalization)
const normalizeSpace = (s) => s.replace(/\s+/g, " ").trim;
// " hello world \n" => "hello world"
normalizeSpace(" hello world \n"); // "hello world"
```

Python examples:

```python

# Example 6: re.sub to remove non-digits
import re

# Remove all non-digit characters from a phone number
def digits_only(s):

# \D matches any non-digit; replace with empty string
 return re.sub(r"\D+", "", s)

digits_only("(555) 123-4567")

# "5551234567"
```

# Flashcards:

---
Remove special characters using regex;;Use /[+\-\s]/g with replace to remove plus, minus, and whitespace from a string
What does \d represent in regex?;;A digit character (equivalent to [0-9])
How do you match the start and end of a string for validation?;;Use ^ at the start and $ at the end, e.g., ^\d+$ to match only digits for the whole string
Difference between greedy and lazy quantifiers;;Greedy quantifiers try to match as much as possible (e.g., .*), lazy quantifiers match as little as possible (e.g., .*?)
What is a non-capturing group and when to use it?;;(?:...) is a non-capturing group; use it when grouping for quantification or alternation without creating a capture for backreferences
Simple pragmatic email validation pattern;;^[^\s@]+@[^\s@]+\.[^\s@]+$ — checks for a non-space local part, an @, and a domain with a dot (not RFC-perfect but practical)