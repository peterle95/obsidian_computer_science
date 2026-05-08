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
---

# Purpose/Why:
---

Prisma solves the problem of safely and efficiently interacting with databases in JavaScript/TypeScript applications. Traditionally, developers either write raw SQL queries (which can be verbose, error-prone, and vulnerable to SQL injection) or use older ORMs with less type safety. Prisma provides a type-safe, auto-generated query API that matches your database schema, ensuring fewer runtime errors and faster development. It’s important in modern backend development because it bridges the gap between relational databases and typed programming languages, improving reliability and developer productivity.

# Core Explanation:
---

Prisma is a modern Object-Relational Mapper (ORM) for Node.js and TypeScript/JavaScript.

Key characteristics:

• Schema-first approach — You define models in a schema.prisma file, and Prisma generates the database queries for you.

• Type safety — The Prisma Client is auto-generated based on your schema, giving you type-checked queries in TypeScript.

• Database migrations — Prisma keeps track of changes to your schema and applies them to the database in a controlled way.

• Database agnostic — Works with PostgreSQL, MySQL, SQLite, SQL Server, and MongoDB.

How it works:

1. You define your models in schema.prisma.
2. Prisma generates a client library tailored to your schema.
3. You import and use the client in your app to query the database.
4. Prisma’s migration system updates your database schema to match your code.
## Prisma Migrations & Supabase:

How migrations work and how they connect with Supabase:

When using Prisma with [[Supabase]], you’re essentially connecting Prisma to a hosted PostgreSQL database. The migrations you run in Prisma will directly alter the database schema in your Supabase instance.

### Why migrations are important:

• They keep your Prisma schema and Supabase database in sync.
• Without migrations, your app’s generated Prisma Client won’t match the database’s actual structure.
• This can cause runtime errors, broken queries, or incorrect data handling.

### Common workflow with Supabase:

1. Update your schema.prisma file (add models, change fields, etc.).
2. Create a new migration and apply it to Supabase.
3. Prisma updates its migration history table in the Supabase database.
4. Supabase immediately reflects the new structure, so you can use its SQL editor, APIs, or Prisma Client seamlessly.

### Key Commands:

- Create and apply a migration (in development)
```bash
npx prisma migrate dev --name <migration-name>
```

- Create a migration without applying (good for production pipelines)
```bash
npx prisma migrate deploy
```
  
- Regenerate the Prisma Client after schema changes
```bash
npx prisma generate
```
  
- View the current database structure as seen by Prisma
```bash
npx prisma studio
```
  
- Example connection string for Supabase (in .env):
```env
DATABASE_URL="postgresql://postgres:<password>@db.<your-project-ref>.supabase.co:5432/postgres"
```


# Related Concepts:
--- 
  
• ORM (Object-Relational Mapping): A programming technique to map database tables to programming objects. Prisma is a next-gen ORM that emphasizes type safety.

• SQL: Structured Query Language; Prisma queries ultimately compile down to SQL commands.

• Supabase: A backend-as-a-service with PostgreSQL — often paired with Prisma to provide hosted databases.

• Migrations: Controlled changes to a database schema; Prisma handles these automatically through CLI commands.

• TypeScript: Prisma leverages TypeScript for strong type checking.

• GraphQL: While not required, Prisma can integrate into GraphQL APIs as the data access layer.  

# Examples:
---
```js
// 1. Import the Prisma Client generated from your schema
import { PrismaClient } from '@prisma/client';

// 2. Create a new Prisma Client instance
const prisma = new PrismaClient();

async function main() {

  // 3. Create a new user in the database
  const newUser = await prisma.user.create({
    data: {
      name: 'Alice',
      email: 'alice@example.com',
    },
  });

  console.log('User created:', newUser);

  // 4. Fetch all users

  const allUsers = await prisma.user.findMany();
  console.log('All users:', allUsers);

  // 5. Update a user’s name

  const updatedUser = await prisma.user.update({
    where: { email: 'alice@example.com' },
    data: { name: 'Alice Wonderland' },
  });

  console.log('Updated user:', updatedUser);

  // 6. Delete a user

  await prisma.user.delete({
    where: { email: 'alice@example.com' },
  });

  console.log('User deleted.');
}

// 7. Run and handle disconnection/errors

main()
  .catch((e) => console.error(e))
  .finally(async () => await prisma.$disconnect());
  ```
  

# Flashcards:
---

1. What is Prisma?;; A modern type-safe ORM for Node.js and TypeScript that provides an auto-generated query API based on a schema definition.

2. What problem does Prisma solve?;; It simplifies database access, ensures type safety, and automates schema migrations in JavaScript/TypeScript applications.

3. What is the schema.prisma file used for?;; To define data models, relationships, and database connection settings for Prisma to generate its client API.

4. How does Prisma ensure type safety?;; It generates TypeScript types and query methods based on the schema, catching errors at compile time instead of runtime.

5. What command applies database schema changes in Prisma?;; npx prisma migrate dev --name <migration-name>
6. How does Prisma interact with Supabase?;; Prisma connects to Supabase’s PostgreSQL database via a connection string, managing queries and migrations while Supabase hosts the database.

  