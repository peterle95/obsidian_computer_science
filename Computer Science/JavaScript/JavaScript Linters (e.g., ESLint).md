---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date:
last-reviewed: 2025-09-01
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 3
last-struggle-date: 2025-07-23

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
--> check slack
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

# **Purpose/Why**:

---
JavaScript linters solve the fundamental problem of ==maintaining code quality, consistency, and catching potential bugs before runtime==. They perform ==static code analysis to identify syntax errors, style violations, potential runtime errors, and anti-patterns without executing the code. Linters are crucial in modern JavaScript development because they enforce coding standards across teams, catch common mistakes early, improve code readability, and can prevent many categories of bugs==. They're especially important in JavaScript's dynamic nature where many errors only surface at runtime, making static analysis invaluable for robust applications.

# **Core Explanation:**

---

Linters are tools that can automatically check the style of your code and make improving suggestions.

--> For correct formatting it's recommended to use Prettier ESLint

The great thing about them is that style-checking can also find some bugs, like typos in variable or function names. Because of this feature, using a linter is recommended even if you don't want to stick to one particular "code style".
Here are some well-known linting tools:
- JSLint – one of the first linters.
- JSHint – more settings than JSLint.
- ESLint – probably the newest one.
All of them can do the job. The author uses ESLint.
Most linters are integrated with many popular editors: just enable the plugin in the editor and configure the style.
For instance, for ESLint you should do the following:
1. Install Node.js.
2. Install ESLint with the command `npm install -g eslint` (npm is a JavaScript package installer).
3. Create a config file named `.eslintrc` in the root of your JavaScript project (in the folder that contains all your files).
4. Install/enable the plugin for your editor that integrates with ESLint. The majority of editors have one.
Here's an example of an `.eslintrc` file:
```javascript
{
 "extends": "eslint:recommended",
 "env": {
 "browser": true,
 "node": true,
 "es6": true
 },
 "rules": {
 "no-console": 0,
 "indent": 2
 }
}
````

Here the directive `"extends"` denotes that the configuration is based on the "eslint:recommended" set of settings. After that, we specify our own. It is also possible to download style rule sets from the web and extend them instead. See .org/docs/user-guide/getting-started for more details about installation. Also certain IDEs have built-in linting, which is convenient but not as customizable as ESLint.

#

# Framework support and Practical notes:

- Purpose: ESLint is a configurable linter that checks code for style issues, likely bugs, and anti-patterns across plain JavaScript and modern frameworks when the right parser/plugins are installed.
- Run it from your project root so it scans the whole codebase:
 - One-off CLI:
 npx eslint . --ext .js,.jsx,.ts,.tsx
 - As npm scripts (recommended):
 "scripts": { "lint": "eslint . --ext .js,.jsx,.ts,.tsx", "lint:fix": "eslint . --ext .js,.jsx,.ts,.tsx --fix" }
- Make CLI strict for CI:
 npx eslint . --ext .js,.jsx,.ts,.tsx --no-ignore --max-warnings 0

#

#

# ESLint has expanded beyond JavaScript to support:

1. **CSS** - Official support through the `@eslint/css` plugin
2. **JSON** - Official support via `@eslint/json` plugin
3. **Markdown** - Official support via `@eslint/markdown` plugin
4. **TypeScript** - Through `typescript-eslint` project

ESLint has been evolving to support more languages through its language plugins architecture. Starting with ESLint v9.0, the core became language-agnostic, allowing plugins to define support for virtually any programming language.

The ESLint team's goal is to ensure ESLint can lint any type of file you might use in a web project, either through officially supported language plugins or community-written plugins.

For your specific technologies:
- TypeScript: Use `@typescript-eslint/parser` and `@typescript-eslint/eslint-plugin`
- React: Use `eslint-plugin-react` and `eslint-plugin-react-hooks`
- Node.js/Express.js: Already supported in core ESLint
- Three.js: Being a JavaScript library, it's supported through standard JavaScript linting

#

#

#

# Framework support:
 - Plain JavaScript: works out of the box or via shared configs (airbnb, standard).
 - React: add eslint-plugin-react and enable plugin:react/recommended (handles .jsx/.tsx).
 - TypeScript: add @typescript-eslint/parser and @typescript-eslint/eslint-plugin; extend plugin:@typescript-eslint/recommended.
 - Next.js: extend next/core-web-vitals (create-next-app can auto-configure); combine with TypeScript/React plugins when needed.

#

#

#

# Practical notes:
 - ESLint ignores node_modules and .eslintignore by default; use --no-ignore to include them.
 - Type-aware TypeScript rules may require parserOptions.project pointing at tsconfig.json (slower).
 - Use editor extensions (e.g., VS Code ESLint) for live feedback and run npm run lint in CI with --max-warnings 0 to fail builds on issues.

# **Related Concepts:**

---
#

# _JavaScript linters connect to several important development and quality assurance concepts:_

- **Static Code Analysis**: Examining code without executing it to find issues
- **Code Quality Metrics**: Measuring maintainability, complexity, and adherence to standards
- **Continuous Integration**: Automated linting in build pipelines and version control
>- **Code Formatters**: Tools like Prettier ESLint that automatically format code style
- **TypeScript**: Static type checking that complements linting for additional safety
- **Testing**: Linting rules that enforce testing best practices and patterns
- **Security Analysis**: Rules that catch common security vulnerabilities
- **Performance Optimization**: Rules that identify performance anti-patterns
- **Team Collaboration**: Establishing consistent coding standards across developers

# **Examples:**

---
```javascript
// COMMON ESLINT CONFIGURATION FILES

// .eslintrc.json - Most common configuration format
{
 "env": {
 "browser": true, // Enable browser global variables
 "es2021": true, // Enable ES2021 syntax
 "node": true // Enable Node.js global variables
 },
 "extends": [
 "eslint:recommended" // Use ESLint's recommended rules
 ],
 "parserOptions": {
 "ecmaVersion": 12, // Support ECMAScript 2021
 "sourceType": "module" // Allow import/export
 },
 "rules": {
 "indent": ["error", 2], // Enforce 2-space indentation
 "quotes": ["error", "single"], // Enforce single quotes
 "semi": ["error", "always"], // Require semicolons
 "no-unused-vars": "warn", // Warn about unused variables
 "no-console": "off" // Allow console statements
 }
}

// .eslintrc.js - JavaScript configuration (allows dynamic config)
module.exports = {
 env: {
 browser: true,
 es2021: true,
 },
 extends: [
 'eslint:recommended',
 'airbnb-base', // Popular style guide
 ],
 parserOptions: {
 ecmaVersion: 12,
 sourceType: 'module',
 },
 rules: {
 // Custom rule overrides
 'no-console': process.env.NODE_ENV === 'production' ? 'error' : 'warn',
 'max-len': ['error', { code: 100 }],
 'prefer-const': 'error',
 'no-var': 'error',
 },
 overrides: [
 {
 // Different rules for test files
 files: ['**/*.test.js', '**/*.spec.js'],
 env: {
 jest: true,
 },
 rules: {
 'no-unused-expressions': 'off',
 },
 },
 ],
};

// EXAMPLES OF CODE THAT TRIGGERS ESLINT RULES

// Example 1: Variables and declarations
console.log('=== Variable Declaration Issues ===');

// ❌ BAD: var usage (eslint: no-var)
var oldStyleVariable = 'avoid var';

// ❌ BAD: Unused variable (eslint: no-unused-vars)
const unusedVariable = 'this is never used';

// ❌ BAD: Variable reassignment when const could be used (eslint: prefer-const)
let shouldBeConst = 'never reassigned';

// ✅ GOOD: Proper variable declarations
const properConstant = 'immutable value';
let mutableVariable = 'can be changed';
mutableVariable = 'new value';

// Example 2: Function and syntax issues
console.log('=== Function and Syntax Issues ===');

// ❌ BAD: Function with too many parameters (eslint: max-params)
function tooManyParams(a, b, c, d, e, f, g) {
 return a + b + c + d + e + f + g;
}

// ❌ BAD: Function too complex (eslint: complexity)
function tooComplex(x) {
 if (x > 10) {
 if (x > 20) {
 if (x > 30) {
 if (x > 40) {
 return 'very high';
 }
 return 'high';
 }
 return 'medium-high';
 }
 return 'medium';
 }
 return 'low';
}

// ✅ GOOD: Simplified function
function betterComplexity(x) {
 if (x > 40) return 'very high';
 if (x > 30) return 'high';
 if (x > 20) return 'medium-high';
 if (x > 10) return 'medium';
 return 'low';
}

// Example 3: Style and formatting issues
console.log('=== Style and Formatting Issues ===');

// ❌ BAD: Inconsistent quotes (eslint: quotes)
const mixedQuotes = "double quotes" + 'single quotes';

// ❌ BAD: Missing semicolons (eslint: semi)
const noSemicolon = 'missing semicolon'

// ❌ BAD: Inconsistent indentation (eslint: indent)
function badIndentation {
if (true) {
 return 'inconsistent indentation';
}
}

// ✅ GOOD: Consistent style
const goodQuotes = 'consistent single quotes';
const withSemicolon = 'proper semicolon';

function goodIndentation {
 if (true) {
 return 'consistent indentation';
 }
}

// Example 4: Potential bugs that ESLint catches
console.log('=== Potential Bug Detection ===');

// ❌ BAD: Duplicate keys in object (eslint: no-dupe-keys)
const duplicateKeys = {
 name: 'first',
 age: 25,
 name: 'duplicate' // This overwrites the first 'name'
};

// ❌ BAD: Unreachable code (eslint: no-unreachable)
function unreachableCode {
 return 'early return';
 console.log('This will never execute'); // Unreachable
}

// ❌ BAD: Assignment in condition (eslint: no-cond-assign)
let userInput = '';
if (userInput = getUserInput) { // Should be === or ==
 console.log('Processing input');
}

// ❌ BAD: Using undefined variables (eslint: no-undef)
// console.log(undefinedVariable); // Would cause ReferenceError

// ✅ GOOD: Proper implementations
const properObject = {
 name: 'John',
 age: 25,
 email: 'john@example.com'
};

function reachableCode {
 const result = 'proper return';
 console.log('This executes before return');
 return result;
}

// Proper condition checking
if (userInput === getUserInput) {
 console.log('Proper comparison');
}

// ADVANCED ESLINT CONFIGURATION

// package.json scripts for ESLint
{
 "scripts": {
 "lint": "eslint src/", // Check all files in src/
 "lint:fix": "eslint src/ --fix", // Auto-fix fixable issues
 "lint:watch": "nodemon --exec npm run lint", // Watch for changes
 "precommit": "lint-staged" // Run on git commit
 },
 "lint-staged": {
 "*.js": ["eslint --fix", "git add"] // Auto-fix on commit
 }
}

// .eslintignore file - Files to exclude from linting
node_modules/
dist/
build/
*.min.js
coverage/
docs/

// CUSTOM ESLINT RULES EXAMPLE

// Custom rule configuration
const customRules = {
 rules: {
 // Enforce specific naming conventions
 'camelcase': ['error', {
 properties: 'always',
 ignoreDestructuring: false
 }],

 // Limit line length
 'max-len': ['error', {
 code: 100,
 ignoreUrls: true,
 ignoreStrings: true
 }],

 // Enforce consistent return statements
 'consistent-return': 'error',

 // Prevent specific functions
 'no-restricted-syntax': [
 'error',
 {
 selector: 'CallExpression[callee.name="eval"]',
 message: 'eval is dangerous and should not be used.'
 }
 ],

 // Custom complexity rules
 'complexity': ['error', { max: 10 }],
 'max-depth': ['error', 4],
 'max-params': ['error', 5],

 // Accessibility rules (with eslint-plugin-jsx-a11y)
 'jsx-a11y/alt-text': 'error',
 'jsx-a11y/anchor-has-content': 'error'
 }
};

// INTEGRATING WITH POPULAR STYLE GUIDES

// Airbnb style guide configuration
{
 "extends": [
 "airbnb-base" // or "airbnb" for React projects
 ],
 "rules": {
 // Override specific Airbnb rules if needed
 "import/prefer-default-export": "off",
 "no-param-reassign": ["error", { "props": false }]
 }
}

// Google style guide configuration
{
 "extends": [
 "google"
 ],
 "rules": {
 // Override Google rules if needed
 "require-jsdoc": "off"
 }
}

// Standard style guide (no semicolons)
{
 "extends": [
 "standard"
 ]
}

// ESLINT WITH TYPESCRIPT

// .eslintrc.js for TypeScript projects
module.exports = {
 parser: '@typescript-eslint/parser',
 plugins: ['@typescript-eslint'],
 extends: [
 'eslint:recommended',
 '@typescript-eslint/recommended',
 ],
 rules: {
 '@typescript-eslint/no-unused-vars': 'error',
 '@typescript-eslint/explicit-function-return-type': 'warn',
 '@typescript-eslint/no-explicit-any': 'error',
 },
};

// ESLINT WITH REACT

// .eslintrc.js for React projects
module.exports = {
 extends: [
 'eslint:recommended',
 'plugin:react/recommended',
 'plugin:react-hooks/recommended',
 ],
 plugins: ['react', 'react-hooks'],
 settings: {
 react: {
 version: 'detect',
 },
 },
 rules: {
 'react/prop-types': 'error',
 'react-hooks/rules-of-hooks': 'error',
 'react-hooks/exhaustive-deps': 'warn',
 },
};

// CONTINUOUS INTEGRATION EXAMPLE

// GitHub Actions workflow for ESLint
// .github/workflows/lint.yml
/*
name: Lint Code Base

on:
 push:
 branches: [ main, develop ]
 pull_request:
 branches: [ main ]

jobs:
 lint:
 runs-on: ubuntu-latest

 steps:
 - uses: actions/checkout@v2

 - name: Setup Node.js
 uses: actions/setup-node@v2
 with:
 node-version: '16'

 - name: Install dependencies
 run: npm ci

 - name: Run ESLint
 run: npm run lint
*/

// PERFORMANCE AND OPTIMIZATION

console.log('=== ESLint Performance Tips ===');

// 1. Use .eslintignore to exclude unnecessary files
// 2. Configure parser options appropriately
// 3. Use eslint-disable comments sparingly
// 4. Consider using eslint-plugin-import for better module resolution

// Example of inline ESLint control
/* eslint-disable no-console */
console.log('This console.log is allowed');
/* eslint-enable no-console */

// Single line disable
console.log('One-time console allowed'); // eslint-disable-line no-console

// Disable specific rule for entire file
/* eslint-disable no-var */

// Function to demonstrate ESLint integration benefits
function demonstrateLinterBenefits {
 // ESLint helps catch these issues:

 // 1. Syntax errors before runtime
 // 2. Unused variables and functions
 // 3. Potential null/undefined access
 // 4. Complex code that needs refactoring
 // 5. Inconsistent code style
 // 6. Security vulnerabilities
 // 7. Performance anti-patterns

 return 'Linting improves code quality significantly';
}

// EDITOR INTEGRATION EXAMPLES

// VS Code settings.json for ESLint
{
 "eslint.autoFixOnSave": true,
 "eslint.validate": [
 "javascript",
 "typescript",
 "javascriptreact",
 "typescriptreact"
 ],
 "editor.codeActionsOnSave": {
 "source.fixAll.eslint": true
 }
}

// Vim/Neovim with ALE plugin
// let g:ale_linters = {'javascript': ['eslint']}
// let g:ale_fixers = {'javascript': ['eslint']}

console.log('ESLint examples completed');

// Helper function for getUserInput (to avoid eslint errors in examples)
function getUserInput {
 return 'sample input';
}
```

# **Flashcards:**

---
What is the primary purpose of JavaScript linters like ESLint?;; To perform static code analysis that identifies syntax errors, style violations, potential bugs, and anti-patterns before code execution, improving code quality and consistency

How do you install and configure ESLint for a project?;; Install Node.js, run `npm install -g eslint`, create `.eslintrc` config file in project root, and install editor plugin. Configuration extends rule sets and customizes specific rules

What does the "extends" property in ESLint configuration do?;; It inherits rules from predefined rule sets like "eslint:recommended", popular style guides (Airbnb, Google), or custom configurations, providing a base set of linting rules

What types of issues can ESLint catch beyond style violations?;; Potential bugs (undefined variables, unreachable code), security vulnerabilities, performance anti-patterns, unused variables/functions, and complex code that needs refactoring

How do you disable ESLint rules for specific lines or files?;; Use `// eslint-disable-line rule-name` for single lines, `/* eslint-disable rule-name */` for blocks, or add files to `.eslintignore` to exclude entirely