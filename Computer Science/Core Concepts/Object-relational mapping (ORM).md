---
memory: to_finish
tags:
 - learned
language:
 - Core Concepts
review-date: ""
last-reviewed: 2025-08-11
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-07-07

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

# **Core Explanation:**

---
Object-Relational Mapping (ORM, O/RM, O/R mapping tool) is a programming technique in computer science that ==facilitates the conversion of data between a [[Relational database]] and the memory (typically the heap) of an object-oriented programming language==. Essentially, ORM <mark style="background:

# FFB86CA6;">creates a virtual object database that can be manipulated directly from within the programming language.</mark>

The ==primary problem ORM addresses is the "object-relational impedance mismatch."== This mismatch arises because object-oriented programming languages model data as objects (which can combine scalar values, lists, and references to other objects, supporting concepts like inheritance and lifecycle management through garbage collection), while relational databases store data in tables as [[Tuple]]s (grouping scalars, using foreign keys for relationships, and lacking direct support for inheritance or object lifecycle management).

ORM provides automated support for <mark style="background:

# BBFABBA6;">translating the logical representation of objects into an atomized form suitable for storage in a database, and vice versa</mark>. This process ensures that object properties and relationships are preserved, allowing objects to be reloaded as needed, making them "persistent." By doing so, ORM often significantly reduces the amount of boilerplate code required for data access compared to traditional techniques involving direct [[SQL queries]].

# **Related Concepts:**

---
- **Relational Database:** A database that organizes data into one or more tables (or "relations") of rows and columns, with a unique key for each row.

Examples include SQL databases.
- **Object-Oriented Programming (OOP):** A programming paradigm based on the concept of "objects," which can contain data (attributes) and code (methods) that operate on the data.
- **Object-Relational Impedance Mismatch:** The fundamental difficulties and differences encountered when trying to map an object-oriented paradigm to a relational database model. These include differences in lifecycle management, references, and inheritance.
- **Persistence:** The characteristic of data that outlives the execution of the program that created it. In the context of ORM, objects are persistent if they can be stored in and retrieved from a database.
- **Object-Oriented Database Management System (OODBMS):** A database designed specifically to work with object-oriented values, storing data in its original object representation, thus eliminating the need for ORM. They are often used in complex, niche applications but may have limitations with ad-hoc queries compared to relational databases.
- **Document-Oriented Databases:** A type of NoSQL database that stores data in a document-like structure, often JSON or XML. Object-document mappers (ODMs) are the equivalent of ORMs for these databases.
- **SQL (Structured Query Language):** A domain-specific language used in programming and designed for managing data held in a relational database management system.
- **Data Access Object (DAO) Pattern:** A design pattern used to abstract and encapsulate all access to the data source. It provides a lightweight object-oriented interface to the rest of the application, often by wrapping native procedural database languages.
- **Active Record Pattern:** A design pattern found in software that stores data in a relational database. It maps rows in database tables to objects, where object attributes map to columns in the table, and methods implement database operations like insert, update, and delete.
- **Data Mapper Pattern:** A data access layer that performs bidirectional transfer of data between a persistent data store (often a relational database) and an in-memory data representation (the domain model). Unlike Active Record, the Data Mapper separates the in-memory objects from the database itself.

# **Examples:**

---

ORM can be applied to C++. While ORM is more commonly associated with languages like Java, C

# , or Python, there are several ORM frameworks available for C++.

Here's an example using ODB, a popular C++ ORM:

```cpp
// Person.h - Class definition

# pragma once

# include <string>

# include <odb/core.hxx>

# pragma db object
class Person {
private:
 friend class odb::access;

# pragma db id auto
 unsigned long id_;

 std::string firstName_;
 std::string lastName_;
 int age_;

public:
 Person {}

 Person(const std::string& firstName,
 const std::string& lastName,
 int age)
 : firstName_(firstName),
 lastName_(lastName),
 age_(age) {}

 // Getters and setters
 unsigned long getId const { return id_; }

 void setFirstName(const std::string& firstName) { firstName_ = firstName; }
 const std::string& getFirstName const { return firstName_; }

 void setLastName(const std::string& lastName) { lastName_ = lastName; }
 const std::string& getLastName const { return lastName_; }

 void setAge(int age) { age_ = age; }
 int getAge const { return age_; }

 std::string getFullName const {
 return firstName_ + " " + lastName_;
 }
};

// Using the ORM in main.cpp

# include "Person.h"

# include "Person-odb.hxx" // Generated by ODB compiler

# include <odb/database.hxx>

# include <odb/transaction.hxx>

# include <odb/mysql/database.hxx>

int main {
 // Create database connection
 auto db = odb::mysql::database("test", "username", "password", "localhost");

 // Create a new person
 Person john("John", "Doe", 30);

 // Persist to database
 {
 odb::transaction t(db.begin);
 db.persist(john);
 t.commit;
 }

 // Query from database
 {
 odb::transaction t(db.begin);
 std::shared_ptr<Person> result = db.load<Person>(john.getId);

 if (result) {
 std::cout << "Found: " << result->getFullName << std::endl;
 }
 t.commit;
 }

 return 0;
}
```

Other C++ ORM libraries include:
- SQLite ORM
- LiteSQL
- SOCI
- QxOrm
- hiberlite

```C

# //
---
Traditional Data Access (using raw SQL)
---
// This approach requires writing SQL queries directly and managing the mapping of results to application variables manually.
// It can be more verbose and less aligned with object-oriented principles.

using System;
using System.Data;
using System.Data.SqlClient; // Example for SQL Server, specific to the database

public class TraditionalDataAccess
{
 public static void GetPersonData
 {
 string connectionString = "Data Source=server;Initial Catalog=database;Integrated Security=True";
 string sqlQuery = "SELECT id, first_name, last_name, phone, birth_date, sex, age FROM persons WHERE id = 10";

 try
 {
 using (SqlConnection connection = new SqlConnection(connectionString))
 {
 connection.Open;
 using (SqlCommand command = new SqlCommand(sqlQuery, connection))
 {
 using (SqlDataReader reader = command.ExecuteReader)
 {
 if (reader.Read)
 {
 // Manually extract data from the reader and assign to variables
 int id = reader.GetInt32(reader.GetOrdinal("id"));
 string firstName = reader.GetString(reader.GetOrdinal("first_name"));
 string lastName = reader.GetString(reader.GetOrdinal("last_name"));
 string phone = reader.GetString(reader.GetOrdinal("phone"));
 DateTime birthDate = reader.GetDateTime(reader.GetOrdinal("birth_date"));
 string sex = reader.GetString(reader.GetOrdinal("sex"));
 int age = reader.GetInt32(reader.GetOrdinal("age"));

 Console.WriteLine($"Traditional Access: Person Name: {firstName} {lastName}, Phone: {phone}");
 // In a real application, you would typically create a Person object here
 // Person person = new Person(id, firstName, lastName, ...);
 }
 }
 }
 }
 }
 catch (Exception ex)
 {
 Console.WriteLine($"Error: {ex.Message}");
 }
 }
}

//
---

ORM-based Data Access (conceptual example, similar to Entity Framework or similar ORMs)
---
// This approach uses an ORM framework to abstract away the SQL, allowing developers to work with objects directly.
// The ORM handles the translation between objects and database rows.

// Define a simple Person class representing the database table
public class Person
{
 public int Id { get; set; }
 public string FirstName { get; set; }
 public string LastName { get; set; }
 public string Phone { get; set; }
 public DateTime BirthDate { get; set; }
 public string Sex { get; set; }
 public int Age { get; set; }

 // Methods related to the Person object can be defined here
 public string GetFullName
 {
 return $"{FirstName} {LastName}";
 }
}

// Conceptual Repository or DbContext class provided by the ORM
public class ApplicationDbContext // Represents the database context
{
 // A 'set' of Persons, conceptually mapping to the 'persons' table
 public PersonRepository Persons { get; set; }

 public ApplicationDbContext
 {
 // In a real ORM, this would be configured to connect to the database
 Persons = new PersonRepository; // Simplified for example
 }
}

// Conceptual Repository for Person objects
public class PersonRepository
{
 // Method to retrieve a person by ID, provided by the ORM
 // The ORM translates this into the appropriate SQL query behind the scenes
 public Person GetById(int id)
 {
 // Simulate fetching from a database via ORM
 // In a real ORM, this would involve LINQ queries or specific methods
 // For example, context.Persons.FirstOrDefault(p => p.Id == id);

 // Simulating the result of an ORM query for id = 10
 if (id == 10)
 {
 return new Person
 {
 Id = 10,
 FirstName = "John",
 LastName = "Doe",
 Phone = "123-456-7890",
 BirthDate = new DateTime(1990, 5, 15),
 Sex = "M",
 Age = 35
 };
 }
 return null;
 }
}

public class ORMDataAccess
{
 public static void GetPersonData
 {
 var context = new ApplicationDbContext; // Initialize the ORM context
 var person = context.Persons.GetById(10); // Use ORM to get the person object directly

 if (person != null)
 {
 var firstName = person.FirstName; // Access object properties directly
 var fullName = person.GetFullName; // Call object methods

 Console.WriteLine($"ORM Access: Person Name: {fullName}, Phone: {person.Phone}");
 }
 else
 {
 Console.WriteLine("Person not found.");
 }
 }
}

//
---

Example of raw SQL execution within an ORM (e.g., Entity Framework Core's FromSqlRaw)
---
// Sometimes, ORMs provide an escape hatch to execute raw SQL when complex or performance-critical queries are needed.
public class ORMRawSqlExample
{
 public static void GetPersonDataWithRawSql
 {
 // This is a simplified representation. In a real EF Core application,
 // 'context.Persons' would be a DbSet<Person> and 'FromSqlRaw' would be called on it.
 // Assume 'context.Persons' can execute raw SQL and map results to Person objects.

 // Simulating an ORM context where you can execute raw SQL
 var context = new ApplicationDbContext;

 // The raw SQL query
 var sql = "SELECT id, first_name, last_name, phone, birth_date, sex, age FROM persons WHERE id = 10";

 // In a real ORM (like Entity Framework), you would call something like:
 // var result = context.Persons.FromSqlRaw(sql).ToList;
 // Here, we simulate the result as a list of Person objects.

 var result = new List<Person>;
 // Simulate the result of executing the raw SQL and mapping it to objects
 if (sql.Contains("id = 10")) // Simple check for simulation
 {
 result.Add(new Person
 {
 Id = 10,
 FirstName = "Jane",
 LastName = "Smith",
 Phone = "987-654-3210",
 BirthDate = new DateTime(1988, 1, 20),
 Sex = "F",
 Age = 37
 });
 }

 if (result.Count > 0)
 {
 var person = result;
 var firstName = person.FirstName;
 Console.WriteLine($"ORM with Raw SQL Access: Person Name: {firstName} {person.LastName}, Phone: {person.Phone}");
 }
 }
}
```

# **Flashcards:**

---
What is Object-Relational Mapping (ORM)?;; A programming technique for converting data between a relational database and the memory (heap) of an object-oriented programming language, creating a virtual object database.

What problem does ORM solve?;; The "object-relational impedance mismatch," which are the difficulties in mapping object-oriented models to relational database structures due to their fundamental differences.

Name one advantage and one disadvantage of ORM.;;Advantage: Reduces the amount of code needed for data access. Disadvantage: High level of abstraction can obscure underlying database operations (SQL queries).