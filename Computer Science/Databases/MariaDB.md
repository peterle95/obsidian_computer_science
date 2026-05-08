---
memory: to_finish
tags:
  - will_learn
language:
  - Databases
review-date:
last-reviewed: ""
scheda: done
visit-count: 0
confidence-level: 1
consecutive-correct: 0
last-struggle-date: ""
cssclasses:
  - important
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

MariaDB is an open-source relational database management system (RDBMS) that provides a robust, scalable, and reliable way to store and manage structured data. The fundamental problem it solves is the need for a high-performance, enterprise-grade database that remains free and open-source. It was created out of concern that MySQL, its predecessor, would become a closed-source, commercial product after being acquired by Oracle Corporation in 2009.

Its primary application is to serve as the backend database for a wide variety of applications, from websites and e-commerce platforms to large-scale enterprise solutions and data warehousing. Its importance lies in its "drop-in" compatibility with MySQL, which allows for easy migration, and its commitment to community-driven development. This ensures it remains a powerful, feature-rich, and cost-effective alternative to proprietary databases, making it a cornerstone of the LAMP (Linux, Apache, MariaDB, PHP) stack and a popular choice for developers and businesses worldwide.

# **Core Explanation:**
---

MariaDB is a community-developed, commercially supported fork of the MySQL relational database management system. It is designed to be a "drop-in" replacement for MySQL, meaning it maintains high compatibility with MySQL's APIs and commands, allowing for a seamless transition for many applications. The project is led by some of the original developers of MySQL, including Michael "Monty" Widenius, who named both databases after his daughters, My and Maria.

**Key Characteristics:**

*   **Open Source:** MariaDB is guaranteed to remain free and open-source under the GNU General Public License (GPLv2).
*   **High Compatibility with MySQL:** It is designed to be a direct replacement for MySQL, allowing users to uninstall MySQL and install MariaDB without significant changes to their applications.
*   **Enhanced Performance:** It includes various performance optimizations and enhancements over MySQL, including a broader range of storage engines like Aria, ColumnStore, and MyRocks.
*   **Modern SQL Features:** MariaDB supports advanced SQL features, including common table expressions (CTEs), window functions, and GIS data types.
*   **Pluggable Storage Engines:** A key feature is its architecture that allows different storage engines to be used for different tables, enabling users to choose the best engine for a specific use case (e.g., InnoDB for transactions, ColumnStore for analytics).
*   **Security Focused:** It includes enhanced security features and is actively maintained and developed by a global community and the MariaDB Corporation.

MariaDB works as a client-server model. The MariaDB Server is responsible for managing the databases and tables, and handling all the data manipulation and retrieval requests. Clients, which can be command-line tools or applications using a specific connector, send SQL (Structured Query Language) statements to the server. The server processes these queries to read, write, update, or delete data stored in tables, which are organized into rows and columns within a specific database.

# **Related Concepts:**
---

*   **MySQL:** MariaDB is a fork of MySQL and its closest relative. It was created by the original MySQL developers to ensure a free and open-source future for the codebase. While they share a common ancestry and high compatibility, MariaDB has since introduced its own unique features and performance improvements.

*   **RDBMS (Relational Database Management System):** This is the category of database systems to which MariaDB belongs. An RDBMS stores data in a structured format using tables with rows and columns and uses SQL for data manipulation and querying. Other popular RDBMSs include PostgreSQL, SQL Server, and Oracle Database.

*   **SQL (Structured Query Language):** This is the standard language used to communicate with and manage data in a relational database like MariaDB. All operations, from creating tables to querying for specific data, are performed using SQL commands.

*   **LAMP Stack:** This is a popular open-source web development stack. The "M" in LAMP, which originally stood for MySQL, is now frequently represented by MariaDB, as it has become the default in many Linux distributions. The other components are Linux (operating system), Apache (web server), and PHP/Python/Perl (programming language).

*   **PostgreSQL:** Another major open-source RDBMS. While MariaDB's development was focused on performance and ease of use, PostgreSQL originated from a research project focusing on a rich feature set and extensibility. They are both powerful alternatives to proprietary databases but have different design philosophies and feature sets.

# **Examples:**
---
### **Example 1: Basic SQL Interaction**

```sql
-- This is a comment in SQL.
-- The following command creates a new database named 'company_db'.
CREATE DATABASE company_db;

-- This command selects the 'company_db' to use for subsequent commands.
USE company_db;

-- This command creates a new table named 'employees'.
-- It defines columns for employee_id, first_name, last_name, and hire_date with specific data types.
-- `employee_id` is the primary key, which uniquely identifies each record.
CREATE TABLE employees (
    employee_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    hire_date DATE
);

-- This command inserts a new record (a new row) into the 'employees' table.
INSERT INTO employees (first_name, last_name, hire_date) VALUES ('John', 'Doe', '2025-11-07');

-- This command retrieves all columns (*) for all records from the 'employees' table.
SELECT * FROM employees;
```

### **Example 2: Docker Compose for MariaDB**

This example shows how to run a MariaDB server using Docker Compose, a common practice in modern development.

```yaml
# docker-compose.yml

# Specifies the Docker Compose file version.
version: '3.8'

# Defines the services (containers) for the application.
services:
  # Defines a service named 'db'.
  db:
    # Specifies the Docker image to use. This pulls the official MariaDB image.
    image: mariadb:10.9
    # Restarts the container automatically if it stops.
    restart: always
    # Sets the environment variables required for the MariaDB container.
    # MARIADB_ROOT_PASSWORD is mandatory to set the root user's password.
    environment:
      MARIADB_ROOT_PASSWORD: your_strong_password
      MARIADB_DATABASE: my_app_db
      MARIADB_USER: my_app_user
      MARIADB_PASSWORD: user_password
    # Maps port 3306 on the host machine to port 3306 in the container,
    # allowing external applications to connect to the database.
    ports:
      - "3306:3306"
    # Defines a named volume to persist the database data.
    # This ensures data is not lost when the container is stopped or removed.
    volumes:
      - mariadb_data:/var/lib/mysql

# Top-level key to declare the named volume used by the 'db' service.
volumes:
  mariadb_data:
```

# **Flashcards:**
---

What is MariaDB?;;A community-developed, open-source relational database management system (RDBMS) that is a fork of MySQL.
Why was MariaDB created?;;It was created by the original developers of MySQL due to concerns that Oracle Corporation would commercialize or neglect MySQL after acquiring it.
What does it mean that MariaDB is a "drop-in replacement" for MySQL?;;It means that in many cases, you can switch from MySQL to MariaDB without making any changes to your applications due to their high compatibility.
Who is the main creator of both MySQL and MariaDB?;;Michael "Monty" Widenius.
What is the standard language used to interact with a MariaDB database?;;SQL (Structured Query Language).
What is a key architectural feature of MariaDB that allows it to handle diverse workloads?;;Its support for multiple, pluggable storage engines (like InnoDB for transactions and ColumnStore for analytics).