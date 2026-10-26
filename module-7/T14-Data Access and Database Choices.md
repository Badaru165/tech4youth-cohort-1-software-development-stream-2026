# Data Access and Database Choices: Connecting React to Real, Persistent Data

**Session length:** 6 hours, project-based
**Prior lessons covered:** Components, `useState`, `useEffect` (plus Express backend setup, Tailwind, and TypeScript as stated in the brief)
**Stack:** Vite + React + TypeScript (`react-ts` template) on the client; Express + TypeScript + SQLite (`better-sqlite3`) on the server
**Project thread:** **BookShelf**, a reading-list app (add, list, search, update, and delete books) that learners build across the session

> **Assumptions.**
> 1. Learners have completed the three prior React lessons, so `useState` and `useEffect` are used without re-teaching. They can already scaffold an Express server in TypeScript.
> 2. "Data access" means the **full path** from a React component to stored data (React → HTTP API → database), and "database choices" means choosing *what kind of storage to use and why*. The build uses **SQLite** because it needs zero installation, while section 3.12 shows how the same repository would look on **PostgreSQL** and **MongoDB**.
> 3. Layout is two folders: `bookshelf/server` and `bookshelf/client`, run in two terminals.
> 4. `.map()` with `key`, conditional rendering (`if`/ternary/`&&`) and custom hooks appear lightly in examples; each gets a one-line placeholder and is taught in its own lesson.
>
> **Not covered here (later lessons):** ORMs (Prisma, Drizzle), authentication and sessions, custom hooks, TanStack Query and other data-fetching libraries, React Router, deployment, and schema migration tools. Each is named once, where it naturally comes up.

---

## 1. Learning Objectives

By the end of this session, learners will be able to:

1. **Explain** the path data takes from a React component to a database and back, and why the browser must never talk to the database directly.
2. **Compare** SQL, document, key-value, and hosted (BaaS) databases and **justify** a choice for a given project using concrete criteria.
3. **Build** a typed data layer on the server (schema, repository with parameterized SQL, REST routes with validation and error handling) using Express and SQLite.
4. **Implement** a typed client API module and React components that read and write data with `fetch`, `useEffect`, and `useState`, including loading, error, optimistic-update, filtering, and pagination behavior.
5. **Debug** the most common data-access failures (SQL injection, `fetch` not rejecting on 404, stale `useEffect` dependencies, CORS errors, route ordering) and **refactor** a database swap so only the repository changes.

---

## 2. Prerequisite Check

Learners must already be able to:

- Build typed function components and manage state with `useState<T>()`.
- Use `useEffect` with a dependency array and a cleanup function.
- Use `async/await` and `fetch`, and read JSON.
- Create a basic Express server with `app.get`/`app.post` and `express.json()` in TypeScript.
- Read basic TypeScript: interfaces, union types, optional properties, generics basics.

**Where this fits:** This is the lesson where a React app stops losing its data on refresh. It follows `useState`/`useEffect` and comes before data-fetching libraries, custom hooks, authentication, and deployment.

---

## 3. Section Breakdown

### 3.1 The Data Path: Browser → API → Database

**Explanation**
Data in a real app lives in a **database**: software whose job is to store information durably and answer questions about it. Your React code runs in the **user's browser**, which you do not control. So React never connects to the database. Instead it sends an HTTP request to an **API** (a server program that exposes **endpoints**, such as `GET /api/books`) and the API talks to the database.

On the server we split the work into small layers so each does one job:

```
React component  ->  client API module  ->  HTTP  ->  Express route  ->  repository  ->  database
 (renders UI)        (typed fetch calls)            (validate, status)   (SQL lives here)  (stores rows)
```

**Why it matters**
Putting database credentials or queries in the browser would let any visitor read or wipe your data. Layering also makes code replaceable: you can change the database without touching React, as long as the API contract stays the same.

**Mental model**
A restaurant. The **dining room** is React (what customers see). The **waiter** is the API (takes orders, returns food, enforces rules like "no order without a table number"). The **kitchen and pantry** is the database. Customers never walk into the pantry.

In vanilla JS you may already have done the first step of this path:

```ts
// Vanilla JS: fetch + DOM
const res = await fetch("/api/books");
const books = await res.json();
list.innerHTML = books.map((b) => `<li>${b.title}</li>`).join("");
```

The fetch is identical in React. What changes is that the result goes into state and the JSX *describes* the list, rather than you writing `innerHTML`.

**Common beginner mistakes**
- **Mistake:** Trying to `import` a database driver (such as `better-sqlite3`) into a React file.
  **Symptom:** Vite build errors like `Module "fs" has been externalized for browser compatibility` or a native-module error.
  **Fix:** Database drivers belong in the `server` project only; React calls the API with `fetch`.
- **Mistake:** Putting business logic and SQL directly inside Express route handlers.
  **Symptom:** Handlers grow to 80 lines and can't be reused or tested.
  **Fix:** Keep routes thin and move SQL into a repository (3.6).

**Try it (3 min)**
On paper, trace what happens when a user clicks "Add book" and names every hop (component → client module → HTTP → route → repository → database → back).

---

### 3.2 Where Can Data Live? State, Browser Storage, and Servers

**Explanation**
You already know one place data lives: `useState`. It's fast but **ephemeral**; a page refresh wipes it. Storage options form a spectrum:

| Where | Survives refresh? | Shared across devices/users? | Notes |
|---|---|---|---|
| `useState` (memory) | No | No | UI state, form drafts |
| `localStorage` | Yes | No (one browser only) | Strings only, about 5 MB, user can edit it |
| A JSON file on the server | Yes | Yes | Breaks under concurrent writes; no querying |
| A database on the server | Yes | Yes | Concurrency, querying, constraints |

```tsx
// src/App.tsx (experiment: localStorage persistence)
import { useEffect, useState } from "react";

export default function App() {
  const [notes, setNotes] = useState<string[]>(() => {
    const saved = localStorage.getItem("notes");
    return saved ? (JSON.parse(saved) as string[]) : [];
  });

  useEffect(() => {
    localStorage.setItem("notes", JSON.stringify(notes));
  }, [notes]);

  return (
    <button onClick={() => setNotes((n) => [...n, `Note ${n.length + 1}`])}>
      Add note ({notes.length})
    </button>
  );
}
```

Line by line:
- `useState(() => …)` passes an **initializer function**: it runs once on first render to compute the starting value, instead of re-reading storage on every render.
- The effect writes to `localStorage` after any change to `notes`.
- Refresh and the count survives. Open the same page in another browser and it doesn't.

**Why it matters**
Choosing the right *place* comes before choosing the right *database*. Many beginners put shared data in `localStorage`, then wonder why a teammate can't see it.

**Mental model**
`useState` is a sticky note on your monitor (gone when you leave), `localStorage` is a notebook in your own bag (yours only), and a server database is the shared company filing system.

**Common beginner mistakes**
- **Mistake:** Treating `localStorage` as a database for data other people need.
  **Symptom:** "It works on my machine" but other users see nothing.
  **Fix:** Anything shared or important goes on a server.
- **Mistake:** `JSON.parse(localStorage.getItem("x"))` without handling `null`.
  **Error message:** TypeScript complains about `string | null`, or `Unexpected token`/`null` at runtime.
  **Fix:** Check for `null` first, as above.
- **Mistake:** Storing secrets or tokens in `localStorage` casually.
  **Symptom:** Any script running on the page can read it.
  **Fix:** Don't store secrets there. Authentication storage is a later lesson.

**Try it (3 min)**
Add the localStorage experiment to a scratch component, add three notes, refresh, then open DevTools → Application → Local Storage and edit the value by hand to see how little protection it has.

---

### 3.3 Choosing a Database: SQL, Document, Key-Value, and Hosted Options

**Explanation**
Databases fall into a few families. The most important split:

- **Relational (SQL) databases** store data in **tables** (rows and columns) with a fixed **schema**, the declared structure. You query with **SQL** (Structured Query Language). Relationships between tables are expressed with keys and combined using **JOINs**. Examples: SQLite, PostgreSQL, MySQL.
- **Document databases** store flexible JSON-like **documents** in **collections**, with no fixed schema. Example: MongoDB.
- **Key-value stores** map a key to a value, fast and simple, often used for caching. Example: Redis.
- **Backend-as-a-Service (BaaS)** platforms host a database *and* an API for you (Firebase, Supabase). Faster to start, with trade-offs in control and portability.

Two terms you'll hear: a **transaction** groups several changes so they all succeed or all fail together, and **ACID** is the name for the safety guarantees (atomic, consistent, isolated, durable) relational databases are known for.

| Option | Type | Best for | Trade-offs | Setup effort |
|---|---|---|---|---|
| **SQLite** | Relational, file-based | Learning, prototypes, small apps, local tools | Single-file; limited for many concurrent writers or multi-server apps | None (a file) |
| **PostgreSQL** | Relational, server | Most production web apps, complex relationships | You run or rent a server | Medium |
| **MySQL / MariaDB** | Relational, server | Widely hosted web apps | Similar to Postgres; fewer advanced features | Medium |
| **MongoDB** | Document | Varied or evolving shapes, nested documents | Relationships and consistency are your job | Medium |
| **Redis** | Key-value | Caching, sessions, queues | Not a primary store for relational data | Low-medium |
| **Firebase / Supabase** | BaaS | Fast MVPs, real-time features | Vendor lock-in, less control | Low |

**A practical decision guide**
1. Is the data **structured with relationships** (users → orders → items), or must it stay consistent (money, inventory)? → **Relational**.
2. Are the shapes truly **varied** and mostly read as whole documents (CMS content, event logs)? → **Document** may fit.
3. Do you need it **very fast and short-lived** (cache, rate limits)? → **Key-value**.
4. Do you need to **ship fast with auth and real-time included** and accept a vendor? → **BaaS**.
5. When unsure, **start relational**. It is the safest default and Postgres can store JSON too.

**For BookShelf we choose SQLite**: books and reviews are structured and related, we want zero setup, and the repository layer keeps a later move to Postgres cheap.

**Why it matters**
Switching databases late is expensive. A short, reasoned decision up front beats choosing by hype.

**Mental model**
A **relational** DB is a set of spreadsheets with strict column types, where rows can reference other sheets. A **document** DB is a folder of JSON files where each file can have its own shape. You already know both: tables resemble arrays of same-shaped objects, documents resemble arbitrary nested objects.

```ts
// Same data, two shapes (illustrative)
// Relational: two tables linked by book_id
//   books(id, title)         reviews(id, book_id, rating)
// Document: one nested object
const bookDoc = { title: "Dune", reviews: [{ rating: 5 }, { rating: 4 }] };
```

**Common beginner mistakes**
- **Mistake:** Choosing MongoDB "because it's JSON and I know JavaScript."
  **Symptom:** Months later you hand-write joins and consistency checks in application code.
  **Fix:** Pick by data shape and requirements, not by language familiarity.
- **Mistake:** Over-engineering with a distributed database for a ten-user app.
  **Symptom:** More ops work than product work.
  **Fix:** Start simple (SQLite or one Postgres), and measure before scaling.
- **Mistake:** Using SQLite on a host with an ephemeral disk, or across several server instances.
  **Symptom:** Data disappears after a redeploy, or instances disagree.
  **Fix:** Use a hosted server database (Postgres) once you deploy beyond one machine with persistent storage.
- **Mistake:** Assuming "NoSQL means no schema, so no design needed."
  **Symptom:** Inconsistent documents (`name` in some, `fullName` in others).
  **Fix:** Design your data shape regardless of the database type.

**Try it (5 min)**
Pick one app (a chat app, a shop, a fitness log) and write: (a) its main entities, (b) which relate to which, and (c) the database family you'd pick and one sentence of reasoning.

---

### 3.4 Modeling Data and Typing It Across Layers

**Explanation**
Before writing code, decide the **entities** (things your app stores) and their fields. In a relational database, an entity becomes a **table**, each record is a **row**, each field is a **column**, and each row has a **primary key** (a unique ID). A column that stores another table's primary key is a **foreign key**.

BookShelf has two entities: `books` and `reviews` (one book has many reviews, a **one-to-many** relationship).

The same book appears in three shapes as it travels, so we type each one:

```ts
// server/src/types.ts
export type BookStatus = "to-read" | "reading" | "done";

// 1) The database row: snake_case, exactly what SQLite returns
export interface BookRow {
  id: number;
  title: string;
  author: string;
  status: BookStatus;
  created_at: string;
}

// 2) The API/domain shape: camelCase, what the client receives
export interface Book {
  id: number;
  title: string;
  author: string;
  status: BookStatus;
  createdAt: string;
}

// 3) The input to create a book (no id, no timestamp yet)
export interface NewBook {
  title: string;
  author: string;
  status?: BookStatus;
}

export type BookPatch = Partial<NewBook>;
```

Line by line:
- `BookRow` mirrors the table, so a column rename shows up as a compile error where rows are read.
- `Book` is what the outside world sees. Keeping it separate from `BookRow` means database naming (`created_at`) never leaks into your React code.
- `createdAt` is a string because JSON has no `Date` type; dates travel as ISO-like strings and are converted in the UI if needed.
- `NewBook` omits server-owned fields (`id`, `createdAt`). The client must not invent them.

**Why it matters**
Typed boundaries catch mismatches early, and separating the row from the API shape lets the database and UI evolve independently.

**Mental model**
Same as keeping a raw API response type separate from your UI's view-model in TypeScript, which you've likely done with `fetch` results already.

**Common beginner mistakes**
- **Mistake:** Using one type for everything.
  **Symptom:** `created_at` and `createdAt` both appear in React code; renaming a column breaks UI files.
  **Fix:** Separate row, domain, and input types, with one mapper function (3.6).
- **Mistake:** Trusting `res.json() as Book[]` as proof the data is correct.
  **Symptom:** TypeScript is happy, but a server bug ships `null` for `title` and the UI crashes.
  **Fix:** A cast is a promise, not a check. Validate at the server boundary now; runtime validation libraries are a later lesson.
- **Mistake:** Storing a list in one text column (`"fantasy,scifi"`).
  **Symptom:** You can't query or index individual items.
  **Fix:** Model it as a related table or, in a document DB, as an array field.

**Try it (4 min)**
Add a `Review` row type (`id`, `book_id`, `rating`, `body`) and a camelCase `Review` type. Which field is the foreign key?

---

### 3.5 Setting Up SQLite in the Express Server

**Explanation**
**SQLite** is a database stored in a single file, accessed through a library. We use `better-sqlite3`, a synchronous driver (calls return results directly, with no `await`), which keeps the first examples simple.

```bash
# bookshelf/server
npm init -y
npm install express better-sqlite3
npm install -D typescript tsx @types/express @types/node @types/better-sqlite3
```

> **Version note:** Express **5** is now the default on npm, and it forwards errors thrown inside route handlers (including rejected promises) to error middleware automatically. In Express 4 you needed wrappers for async errors. `better-sqlite3` is a native add-on, so it needs a Node version it has prebuilt binaries for. If install fails, check its docs for supported Node versions. Node also ships an experimental built-in `node:sqlite` module; its API and stability may have changed, so verify before substituting. Run the server with `npx tsx watch src/index.ts`, and make sure your `tsconfig.json` enables `esModuleInterop` so `import express from "express"` works.

```ts
// server/src/db.ts
import Database from "better-sqlite3";

const db = new Database("bookshelf.db");

db.pragma("journal_mode = WAL");
db.pragma("foreign_keys = ON");

db.exec(`
  CREATE TABLE IF NOT EXISTS books (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    title      TEXT NOT NULL,
    author     TEXT NOT NULL,
    status     TEXT NOT NULL DEFAULT 'to-read'
               CHECK (status IN ('to-read', 'reading', 'done')),
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
  );

  CREATE TABLE IF NOT EXISTS reviews (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    book_id INTEGER NOT NULL REFERENCES books(id) ON DELETE CASCADE,
    rating  INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    body    TEXT NOT NULL DEFAULT ''
  );
`);

export default db;
```

Line by line:
- `new Database("bookshelf.db")` opens (or creates) the file. The file is the whole database, so add it to `.gitignore`.
- `foreign_keys = ON` is required: SQLite does **not** enforce foreign keys unless you enable it per connection.
- `CREATE TABLE IF NOT EXISTS` makes the script safe to run on every startup.
- `NOT NULL`, `DEFAULT`, and `CHECK` are **constraints**: the database itself rejects bad data (a `status` outside the three values), even if buggy code tries to insert it.
- `ON DELETE CASCADE` deletes a book's reviews when the book is deleted.

A small seed script gives you data to look at:

```ts
// server/src/seed.ts
import { bookRepository } from "./bookRepository";

const samples = [
  { title: "Dune", author: "Frank Herbert", status: "done" as const },
  { title: "The Pragmatic Programmer", author: "Hunt & Thomas", status: "reading" as const },
  { title: "Project Hail Mary", author: "Andy Weir" },
];

for (const s of samples) bookRepository.create(s);
console.log("Seeded", samples.length, "books");
```

```bash
npx tsx src/seed.ts
```

**Why it matters**
Constraints are your last line of defense. Application code has bugs; a `CHECK` constraint does not forget.

**Mental model**
The schema is a **TypeScript interface for your storage**: it describes the allowed shape, and the database enforces it at runtime, which TypeScript cannot do.

**Common beginner mistakes**
- **Mistake:** Changing `CREATE TABLE` after the DB file exists and expecting new columns to appear.
  **Symptom:** `table books has no column named favorite`.
  **Fix:** `IF NOT EXISTS` skips existing tables. During development, delete `bookshelf.db` and re-seed, or run an `ALTER TABLE`. Real projects use **migrations** (versioned schema changes; tooling is a later lesson).
- **Mistake:** Forgetting `foreign_keys = ON`.
  **Symptom:** You can insert a review for a book that doesn't exist.
  **Fix:** Enable the pragma on every connection.
- **Mistake:** Committing `bookshelf.db` (and its `-wal` files) to Git.
  **Symptom:** Merge conflicts in a binary file; real data leaked.
  **Fix:** Add `*.db`, `*.db-wal`, and `*.db-shm` to `.gitignore`.

**Try it (3 min)**
Run the server once, then open the file with a SQLite viewer (VS Code extension or `sqlite3 bookshelf.db ".tables"`) and confirm `books` and `reviews` exist.

**Project step: Milestone 1 (about 15 min)**
Create `db.ts` and `types.ts`. Confirm the file is created and tables exist. (The seed script runs after the repository exists, in the next section.)

---

### 3.6 The Repository: CRUD with SQL

**Explanation**
**CRUD** stands for Create, Read, Update, Delete, the four basic data operations. A **repository** is a module that holds all database code for one entity and exposes simple functions. Routes call the repository and never write SQL.

The essential SQL statements:

```sql
SELECT * FROM books WHERE id = ?;                       -- Read
INSERT INTO books (title, author) VALUES (?, ?);        -- Create
UPDATE books SET status = ? WHERE id = ?;               -- Update
DELETE FROM books WHERE id = ?;                         -- Delete
```

The `?` marks are **parameters** (placeholders). This is how you safely pass user input into SQL, covered below.

```ts
// server/src/bookRepository.ts
import db from "./db";
import type { Book, BookPatch, BookRow, NewBook } from "./types";

const toBook = (r: BookRow): Book => ({
  id: r.id,
  title: r.title,
  author: r.author,
  status: r.status,
  createdAt: r.created_at,
});

function list(): Book[] {
  const rows = db.prepare("SELECT * FROM books ORDER BY id DESC").all() as BookRow[];
  return rows.map(toBook);
}

function get(id: number): Book | undefined {
  const row = db.prepare("SELECT * FROM books WHERE id = ?").get(id) as BookRow | undefined;
  return row ? toBook(row) : undefined;
}

function create(input: NewBook): Book {
  const info = db
    .prepare("INSERT INTO books (title, author, status) VALUES (?, ?, ?)")
    .run(input.title, input.author, input.status ?? "to-read");
  return get(Number(info.lastInsertRowid))!;
}

function update(id: number, patch: BookPatch): Book | undefined {
  db.prepare(
    `UPDATE books
     SET title  = COALESCE(?, title),
         author = COALESCE(?, author),
         status = COALESCE(?, status)
     WHERE id = ?`
  ).run(patch.title ?? null, patch.author ?? null, patch.status ?? null, id);
  return get(id);
}

function remove(id: number): boolean {
  return db.prepare("DELETE FROM books WHERE id = ?").run(id).changes > 0;
}

export const bookRepository = { list, get, create, update, remove };
```

Line by line:
- `db.prepare(sql)` compiles a statement; `.all()` returns every row, `.get()` returns one row (or `undefined`), and `.run()` executes a write and returns metadata (`changes`, `lastInsertRowid`).
- Values (`id`, `input.title`) are **never** concatenated into the SQL string. They're passed separately as parameters.
- `toBook` is the single place where `created_at` becomes `createdAt`.
- `COALESCE(?, title)` means "use the new value, but keep the old one if it's `NULL`", so a partial update only changes provided fields. `undefined` isn't accepted by the driver, so we convert with `?? null`.
- `lastInsertRowid` can be a `bigint`, so we convert with `Number(...)`.
- `remove` returns whether anything was deleted, which the route turns into 204 or 404.

**SQL injection, the most important security idea in this lesson**
**SQL injection** is when user input is pasted into a SQL string and changes what the query *means*.

```ts
// DANGEROUS: never do this
const title = "x' OR '1'='1";
db.prepare(`SELECT * FROM books WHERE title = '${title}'`).all();
// Becomes: SELECT * FROM books WHERE title = 'x' OR '1'='1'   -> returns EVERY book
```

```ts
// SAFE: the value is sent separately and is always treated as data
db.prepare("SELECT * FROM books WHERE title = ?").all(title);
```

**Why it matters**
The repository is where data integrity and security meet. Parameterized queries are a non-negotiable habit: injection attacks have caused some of the largest breaches in history.

**Mental model**
Placeholders are like TypeScript function parameters: you don't build a function by string-concatenating its arguments into the source code. A parameterized query is the same discipline for SQL.

**Common beginner mistakes**
- **Mistake:** Template-literal SQL with user values (`` `WHERE id = ${id}` ``).
  **Symptom:** Works in testing, exploitable in production (see above).
  **Fix:** Always use `?` placeholders.
- **Mistake:** Passing `undefined` to `.run()`.
  **Error message:** `TypeError: SQLite3 can only bind numbers, strings, bigints, buffers, and null`.
  **Fix:** Convert `undefined` to `null` (`?? null`).
- **Mistake:** `SELECT *` mapped straight into API responses.
  **Symptom:** A future sensitive column (like `password_hash`) leaks automatically.
  **Fix:** Map through `toBook`, which only copies the fields you intend to expose.
- **Mistake:** `UPDATE` or `DELETE` without `WHERE`.
  **Symptom:** Every row changed or deleted.
  **Fix:** Always include `WHERE`, and check `changes` in the result.

**Try it (4 min)**
In a scratch script, call `bookRepository.create({ title: "Test", author: "Me" })`, then `list()`, and print the results. Then try creating a book with `status: "invalid" as never` and read the `CHECK constraint failed` error.

**Project step: Milestone 2 (about 25 min)**
Write `bookRepository.ts`, then run `npx tsx src/seed.ts` and verify the rows with the SQLite viewer.

> **Placeholder (later lesson: ORMs):** Tools like Prisma and Drizzle generate much of this code and types for you. We write SQL by hand first so you understand what those tools do.

---

### 3.7 Exposing Data: REST Routes, Validation, and Errors

**Explanation**
**REST** is a convention for APIs: URLs name **resources** (`/api/books`, `/api/books/3`) and HTTP **methods** say what to do (`GET` read, `POST` create, `PATCH` partially update, `DELETE` remove). Responses use **status codes** so clients know what happened:

| Code | Meaning | Used when |
|---|---|---|
| 200 | OK | Successful read or update |
| 201 | Created | A new resource was created (POST) |
| 204 | No Content | Successful delete (no body) |
| 400 | Bad Request | The client sent invalid data |
| 404 | Not Found | The resource doesn't exist |
| 500 | Server Error | Something broke on our side |

**Never trust the request body.** Anyone can send anything to your endpoint, so TypeScript types (which vanish at runtime) don't protect you. Validate manually at the boundary:

```ts
// server/src/routes/books.ts
import { Router } from "express";
import { bookRepository } from "../bookRepository";
import type { BookPatch, BookStatus, NewBook } from "../types";

const STATUSES: BookStatus[] = ["to-read", "reading", "done"];
const isStatus = (v: unknown): v is BookStatus => STATUSES.includes(v as BookStatus);

function parseId(raw: string): number | null {
  const n = Number(raw);
  return Number.isInteger(n) && n > 0 ? n : null;
}

function parseNewBook(body: unknown): NewBook | string {
  if (typeof body !== "object" || body === null) return "Body must be a JSON object";
  const { title, author, status } = body as Record<string, unknown>;
  if (typeof title !== "string" || title.trim() === "") return "title is required";
  if (typeof author !== "string" || author.trim() === "") return "author is required";
  if (status !== undefined && !isStatus(status)) return "status is invalid";
  return { title: title.trim(), author: author.trim(), status };
}

function parsePatch(body: unknown): BookPatch | string {
  if (typeof body !== "object" || body === null) return "Body must be a JSON object";
  const { title, author, status } = body as Record<string, unknown>;
  const patch: BookPatch = {};
  if (title !== undefined) {
    if (typeof title !== "string" || title.trim() === "") return "title must be a non-empty string";
    patch.title = title.trim();
  }
  if (author !== undefined) {
    if (typeof author !== "string" || author.trim() === "") return "author must be a non-empty string";
    patch.author = author.trim();
  }
  if (status !== undefined) {
    if (!isStatus(status)) return "status is invalid";
    patch.status = status;
  }
  return patch;
}

export const booksRouter = Router();

booksRouter.get("/", (_req, res) => {
  res.json(bookRepository.list());
});

booksRouter.get("/:id", (req, res) => {
  const id = parseId(req.params.id);
  if (id === null) return res.status(400).json({ error: "Invalid id" });
  const book = bookRepository.get(id);
  if (!book) return res.status(404).json({ error: "Book not found" });
  res.json(book);
});

booksRouter.post("/", (req, res) => {
  const parsed = parseNewBook(req.body);
  if (typeof parsed === "string") return res.status(400).json({ error: parsed });
  res.status(201).json(bookRepository.create(parsed));
});

booksRouter.patch("/:id", (req, res) => {
  const id = parseId(req.params.id);
  if (id === null) return res.status(400).json({ error: "Invalid id" });
  const parsed = parsePatch(req.body);
  if (typeof parsed === "string") return res.status(400).json({ error: parsed });
  const book = bookRepository.update(id, parsed);
  if (!book) return res.status(404).json({ error: "Book not found" });
  res.json(book);
});

booksRouter.delete("/:id", (req, res) => {
  const id = parseId(req.params.id);
  if (id === null) return res.status(400).json({ error: "Invalid id" });
  if (!bookRepository.remove(id)) return res.status(404).json({ error: "Book not found" });
  res.status(204).end();
});
```

```ts
// server/src/index.ts
import express from "express";
import type { NextFunction, Request, Response } from "express";
import { booksRouter } from "./routes/books";

const app = express();
app.use(express.json());

app.use("/api/books", booksRouter);

app.use((err: unknown, _req: Request, res: Response, _next: NextFunction) => {
  console.error(err);
  res.status(500).json({ error: "Internal server error" });
});

const PORT = Number(process.env.PORT ?? 3001);
app.listen(PORT, () => console.log(`API listening on http://localhost:${PORT}`));
```

Line by line:
- `parseNewBook` returns either a valid `NewBook` or a **string error message**, and the route checks `typeof parsed === "string"`. It's a simple, dependency-free pattern.
- `isStatus` is a **type guard** (`v is BookStatus`), so after the check TypeScript knows the type.
- Every response is JSON, including errors (`{ error: "…" }`), so the client can show the message in a consistent way.
- The final four-argument middleware is Express's **error handler**: Express recognizes it by its four parameters. We log the real error but send the client a generic message (never leak stack traces).
- `req.params.id` is always a string, so `parseId` converts and validates it.

**Test it with curl** (before touching React):

```bash
curl http://localhost:3001/api/books
curl -X POST http://localhost:3001/api/books \
  -H "Content-Type: application/json" \
  -d '{"title":"Dune","author":"Frank Herbert"}'
curl -X PATCH http://localhost:3001/api/books/1 \
  -H "Content-Type: application/json" -d '{"status":"done"}'
curl -i -X DELETE http://localhost:3001/api/books/1
```

**Why it matters**
Clear status codes and consistent error bodies are the "contract" React relies on. A well-validated API turns confusing database errors into helpful 400 responses.

**Mental model**
Validation is a bouncer at the door, and the database `CHECK` constraints are the second bouncer inside. You want both.

**Common beginner mistakes**
- **Mistake:** Forgetting `app.use(express.json())`.
  **Symptom:** `req.body` is `undefined`, so every POST returns "Body must be a JSON object".
  **Fix:** Register `express.json()` before the routes.
- **Mistake:** Always returning 200, even for failures.
  **Symptom:** The client can't tell success from failure without parsing the body.
  **Fix:** Use proper status codes (see the table).
- **Mistake:** Declaring a more general route before a more specific one (e.g. `/:id` before `/summary`).
  **Symptom:** `GET /api/books/summary` returns `Invalid id` (the router treated "summary" as an id).
  **Fix:** Declare specific static routes before parameterized ones.
- **Mistake:** Returning the raw error to the client.
  **Symptom:** Stack traces and SQL fragments visible in the browser.
  **Fix:** Log on the server, respond with a generic message.

**Try it (4 min)**
Send a POST with an empty `title` and one with `status: "finished"`. Confirm both return 400 with a useful message and that a request for `/api/books/999` returns 404.

**Project step: Milestone 3 (about 30 min)**
Build the router and `index.ts`, then exercise every endpoint with curl, including the error cases.

---

### 3.8 The Client Data Layer: API Module, Dev Proxy, and Environment Variables

**Explanation**
Now to the React project. Instead of scattering `fetch` calls across components, create **one typed API module**. Components call `booksApi.list()` and never build URLs or parse errors themselves.

First, scaffold and configure the client:

```bash
# bookshelf/
npm create vite@latest client -- --template react-ts
cd client && npm install
```

Browsers enforce **CORS** (Cross-Origin Resource Sharing): a page served from `http://localhost:5173` is not allowed to read responses from `http://localhost:3001` unless the server opts in. In development, the simplest fix is a **proxy**: Vite forwards `/api` requests to Express, so the browser sees one origin.

```ts
// client/vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      "/api": "http://localhost:3001",
    },
  },
});
```

> **Version note:** If you use Tailwind v4, also add its Vite plugin per the current Tailwind docs. The Vite template's `tsconfig` currently enables strict options such as `verbatimModuleSyntax` (you must write `import type { … }` for type-only imports) and may enable `erasableSyntaxOnly` (which forbids TypeScript-only syntax like constructor parameter properties and `enum`). The code below is written to satisfy both. Verify against your generated `tsconfig`.

```ts
// client/src/types.ts
export type BookStatus = "to-read" | "reading" | "done";

export interface Book {
  id: number;
  title: string;
  author: string;
  status: BookStatus;
  createdAt: string;
}

export interface NewBook {
  title: string;
  author: string;
  status?: BookStatus;
}

export type BookPatch = Partial<NewBook>;
```

```ts
// client/src/api/books.ts
import type { Book, BookPatch, NewBook } from "../types";

const BASE_URL: string = import.meta.env.VITE_API_URL ?? "";

export class ApiError extends Error {
  status: number;
  constructor(status: number, message: string) {
    super(message);
    this.status = status;
  }
}

async function request<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${BASE_URL}${path}`, {
    ...init,
    headers: { "Content-Type": "application/json", ...init?.headers },
  });

  if (!res.ok) {
    const body = (await res.json().catch(() => null)) as { error?: string } | null;
    throw new ApiError(res.status, body?.error ?? `Request failed (${res.status})`);
  }

  if (res.status === 204) return undefined as T;
  return (await res.json()) as T;
}

export const booksApi = {
  list: (signal?: AbortSignal) => request<Book[]>("/api/books", { signal }),
  create: (input: NewBook) =>
    request<Book>("/api/books", { method: "POST", body: JSON.stringify(input) }),
  update: (id: number, patch: BookPatch) =>
    request<Book>(`/api/books/${id}`, { method: "PATCH", body: JSON.stringify(patch) }),
  remove: (id: number) => request<void>(`/api/books/${id}`, { method: "DELETE" }),
};
```

Line by line:
- `request<T>` is a **generic** wrapper: every call site declares what it expects back (`request<Book[]>`).
- `if (!res.ok)` is essential. `fetch` only **rejects on network failure**; a 404 or 500 *resolves* normally. We convert those into thrown `ApiError`s, using the server's `{ error }` message when there is one.
- `.catch(() => null)` protects against error responses that aren't JSON.
- 204 responses have no body, so we return early instead of calling `res.json()`.
- `signal` lets a component cancel a request (used in 3.9).
- `import.meta.env.VITE_API_URL` is Vite's way to read environment variables. It's empty in development (the proxy handles `/api`) and set to the real API URL in production builds.

**Why it matters**
One module means one place to add auth headers, change URLs, or handle errors later. Components stay focused on UI.

**Mental model**
`booksApi` is your own tiny SDK, like the client libraries of services you've used (Stripe, Firebase), but written for your API. Components call functions, not URLs.

**Common beginner mistakes**
- **Mistake:** Calling `fetch("http://localhost:3001/api/books")` directly in a component.
  **Error message:** `Access to fetch … has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header`.
  **Fix:** Use relative `/api/…` URLs with the Vite proxy (or enable CORS on the server).
- **Mistake:** Assuming `fetch` throws on a 404/500.
  **Symptom:** The error body is parsed as data, and later `books.map is not a function`.
  **Fix:** Check `res.ok` (done in `request`).
- **Mistake:** Forgetting to restart `npm run dev` after editing `vite.config.ts` or `.env`.
  **Symptom:** The proxy or env variable seems ignored.
  **Fix:** Restart the dev server.
- **Mistake:** Putting secrets in `VITE_` variables.
  **Symptom:** The "secret" appears in the browser bundle.
  **Fix:** Anything prefixed `VITE_` is public. Secrets live only on the server.

**Try it (3 min)**
Temporarily call `fetch("http://localhost:3001/api/books")` from the browser console on the Vite page and read the CORS error. Then call `fetch("/api/books")` and see it succeed through the proxy.

**Project step: Milestone 4 (about 20 min)**
Scaffold the client, configure the proxy, add `types.ts` and `api/books.ts`. Verify with `booksApi.list().then(console.log)` from a temporary `useEffect`.

---

### 3.9 Reading Data in React

**Explanation**
Reading is "fetch once on mount": a side effect, so `useEffect` is the right tool. Keep three pieces of state: the data, a loading flag, and an error message. Cancel the request in the cleanup so stale responses can't overwrite fresh ones.

```tsx
// client/src/App.tsx
import { useEffect, useState } from "react";
import { booksApi } from "./api/books";
import type { Book } from "./types";

export default function App() {
  const [books, setBooks] = useState<Book[]>([]);
  const [loading, setLoading] = useState(true);
  const [loadError, setLoadError] = useState<string | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    booksApi
      .list(controller.signal)
      .then((data) => {
        setBooks(data);
        setLoadError(null);
      })
      .catch((err: unknown) => {
        if (err instanceof DOMException && err.name === "AbortError") return;
        setLoadError(err instanceof Error ? err.message : "Could not load books");
      })
      .finally(() => {
        if (!controller.signal.aborted) setLoading(false);
      });

    return () => controller.abort();
  }, []);

  if (loading) return <p className="p-6 text-slate-500">Loading books…</p>;
  if (loadError) return <p className="p-6 text-red-600">{loadError}</p>;

  return (
    <main className="mx-auto max-w-xl p-6">
      <h1 className="text-2xl font-bold">BookShelf</h1>
      {books.length === 0 ? (
        <p className="mt-4 text-slate-500">No books yet.</p>
      ) : (
        <ul className="mt-4 space-y-2">
          {books.map((b) => (
            <li key={b.id} className="rounded border bg-white p-3">
              <span className="font-medium">{b.title}</span>
              <span className="text-slate-500"> by {b.author}</span>
              <span className="ml-2 text-xs uppercase text-indigo-600">{b.status}</span>
            </li>
          ))}
        </ul>
      )}
    </main>
  );
}
```

> **Placeholders:** `books.map(...)` with `key` is covered in **Lists and Keys**. The early returns and ternary are the subject of the **Conditional Rendering** lesson. For now they simply say "show this *instead*." Repeating this fetch pattern in many components is the cue to extract a **custom hook**, and libraries like TanStack Query manage caching and loading flags for you (both are later lessons).

Line by line:
- Initial `loading` is `true`, so the first render never shows "No books yet" before data arrives.
- `AbortController` + cleanup cancels the request when the component unmounts or the effect re-runs.
- The `AbortError` check means a deliberate cancel isn't reported as a failure.
- `.finally` runs on success *and* failure. We skip it for aborted runs so the loading flag isn't flipped by a cancelled request.
- The empty dependency array `[]` means "run once after the first render."

> **Version note:** In development, React `<StrictMode>` (in the Vite template) deliberately mounts, unmounts, and re-mounts each component once, so this effect runs twice and you'll see two requests in the Network tab. That's expected, and the cleanup is what makes it harmless. Production builds run it once. Verify the behavior against your installed React version.

**Why it matters**
Nearly every screen starts by loading something. Doing it correctly (cleanup, error handling, no flash of wrong content) is a daily skill.

**Mental model**
In vanilla JS you'd call `loadBooks()` inside `DOMContentLoaded`. `useEffect(..., [])` is the React version of "run once when this page appears," with the bonus of a built-in cleanup hook.

**Common beginner mistakes**
- **Mistake:** Making the effect callback `async`: `useEffect(async () => {…})`.
  **Error message:** `Effect callbacks are synchronous to prevent race conditions` (or a warning about the returned Promise).
  **Fix:** Use `.then()` chains, or define and call an inner async function.
- **Mistake:** Fetching in the component body (outside `useEffect`).
  **Symptom:** An infinite loop of requests: fetch → `setBooks` → re-render → fetch…
  **Fix:** Put the fetch in an effect with a dependency array.
- **Mistake:** Forgetting `[]` (no dependency array).
  **Symptom:** The same infinite loop.
  **Fix:** Add the dependency array.
- **Mistake:** Rendering `books.map` when `books` could be `undefined`/`null`.
  **Error message:** `Cannot read properties of undefined (reading 'map')`.
  **Fix:** Initialize with `[]`, and never set state to the raw error body.

**Try it (4 min)**
Stop the Express server and refresh the page to see the error state. Restart it, then throttle the network to "Slow 3G" in DevTools to watch the loading state.

**Project step: Milestone 5 (about 20 min)**
Render the seeded books from the real database and confirm they survive a page refresh (unlike `useState`-only data).

---

### 3.10 Writing Data: Create, Update, Delete

**Explanation**
Writes start from a user action (submit, click), not from an effect, so they live in **event handlers**. After the server confirms a write, the UI must reflect it. There are two strategies:

- **Pessimistic (wait, then update):** call the API, and update state with the server's response once it succeeds. Simple and always accurate. Use it for create and delete.
- **Optimistic (update now, repair on failure):** update the UI immediately, call the API, and **roll back** if it fails. Feels instant; use it for low-risk toggles such as changing status.

Keep **action errors** (a failed save) separate from **load errors** (the page couldn't load), so a failed delete doesn't replace the whole page with an error message.

```tsx
// client/src/components/AddBookForm.tsx
import { useState } from "react";
import type { FormEvent } from "react";
import type { NewBook } from "../types";

interface AddBookFormProps {
  onAdd: (input: NewBook) => Promise<void>;
}

export default function AddBookForm({ onAdd }: AddBookFormProps) {
  const [title, setTitle] = useState("");
  const [author, setAuthor] = useState("");
  const [submitting, setSubmitting] = useState(false);

  const handleSubmit = async (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    if (!title.trim() || !author.trim()) return;
    setSubmitting(true);
    try {
      await onAdd({ title: title.trim(), author: author.trim() });
      setTitle("");
      setAuthor("");
    } catch {
      // The parent shows the error; keep the typed text so nothing is lost
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="mt-4 flex gap-2">
      <input value={title} onChange={(e) => setTitle(e.target.value)} placeholder="Title"
        className="flex-1 rounded border p-2" />
      <input value={author} onChange={(e) => setAuthor(e.target.value)} placeholder="Author"
        className="flex-1 rounded border p-2" />
      <button type="submit" disabled={submitting}
        className="rounded bg-indigo-600 px-3 text-white disabled:opacity-50">
        {submitting ? "Adding…" : "Add"}
      </button>
    </form>
  );
}
```

```tsx
// client/src/App.tsx (additions to the component from 3.9)
const [actionError, setActionError] = useState<string | null>(null);

const message = (err: unknown) => (err instanceof Error ? err.message : "Something went wrong");

// CREATE: pessimistic
const handleAdd = async (input: NewBook) => {
  try {
    const created = await booksApi.create(input);
    setBooks((prev) => [created, ...prev]);
    setActionError(null);
  } catch (err) {
    setActionError(message(err));
    throw err; // lets the form keep its text
  }
};

// UPDATE: optimistic with rollback
const handleStatusChange = async (book: Book, next: BookStatus) => {
  const previous = book.status;
  setBooks((prev) => prev.map((b) => (b.id === book.id ? { ...b, status: next } : b)));
  try {
    await booksApi.update(book.id, { status: next });
    setActionError(null);
  } catch (err) {
    setBooks((prev) => prev.map((b) => (b.id === book.id ? { ...b, status: previous } : b)));
    setActionError(message(err));
  }
};

// DELETE: pessimistic
const handleDelete = async (id: number) => {
  if (!window.confirm("Delete this book?")) return;
  try {
    await booksApi.remove(id);
    setBooks((prev) => prev.filter((b) => b.id !== id));
    setActionError(null);
  } catch (err) {
    setActionError(message(err));
  }
};
```

Add `import type { Book, BookStatus, NewBook } from "./types";`, render `<AddBookForm onAdd={handleAdd} />`, a status `<select>` per book calling `handleStatusChange`, a Delete button calling `handleDelete`, and `{actionError && <p className="text-red-600">{actionError}</p>}` above the list.

Line by line:
- `setBooks((prev) => …)` uses the **functional update form**, so concurrent updates never read stale state, and arrays are replaced, not mutated.
- Create uses the **server's returned book** (with the real `id` and `createdAt`) rather than inventing one on the client.
- The optimistic update captures `previous` *before* changing state so rollback is exact.
- `throw err` in `handleAdd` lets the form's `catch` know the add failed, so it doesn't clear the inputs.
- `submitting` disables the button to prevent double-submits.

**Why it matters**
Writes are where data gets corrupted: duplicates, lost updates, and UI that disagrees with the database. Treating the **server as the source of truth** and keeping the UI honest about failures is what separates a demo from an app.

**Mental model**
Optimistic updates are like writing a check: you act as if it will clear, but you must handle the bounce. Pessimistic is cash on the counter: slower, no surprises.

**Common beginner mistakes**
- **Mistake:** Mutating state: `books.push(created); setBooks(books)`.
  **Symptom:** The UI doesn't update (same array reference, so React sees no change).
  **Fix:** Create a new array: `setBooks((prev) => [created, ...prev])`.
- **Mistake:** Generating IDs on the client (`Date.now()`) and sending them.
  **Symptom:** Collisions, or IDs that don't match the database.
  **Fix:** Let the database assign IDs and use the response.
- **Mistake:** Not handling failure on optimistic updates.
  **Symptom:** The UI shows "done" but a refresh reveals the change never saved.
  **Fix:** Always roll back and tell the user.
- **Mistake:** Forgetting `e.preventDefault()` in the submit handler.
  **Symptom:** The page reloads and state resets.
  **Fix:** Call it first.
- **Mistake:** Not awaiting the API call before clearing the form.
  **Symptom:** Inputs clear even though the request failed.
  **Fix:** `await`, and only reset on success.

**Try it (5 min)**
Stop the server, then try to add a book. Confirm the error shows and the typed text remains. Restart the server and retry.

**Project step: Milestone 6 (about 40 min)**
Add create, status change (optimistic), and delete. Test every failure path by stopping the server mid-session.

---

### 3.11 Querying: Filtering, Pagination, and Relationships

**Explanation**
Loading *every* row doesn't scale. Move the work to the database with **query parameters** (the `?key=value` part of a URL):

`GET /api/books?status=reading&q=dune&limit=10&offset=20`

- **Filtering:** `WHERE status = ?`
- **Search:** `WHERE title LIKE ?` (with `%` wildcards)
- **Pagination:** `LIMIT` (page size) and `OFFSET` (rows to skip)

Since the response now needs a `total` (so the UI can show page counts), the response shape changes from `Book[]` to `{ items, total }`. This is a **breaking API change**, so the server and client must change together.

```ts
// server/src/bookRepository.ts (replace list)
export interface ListOptions {
  status?: BookStatus;
  q?: string;
  limit: number;
  offset: number;
}

function list(opts: ListOptions): { items: Book[]; total: number } {
  const where: string[] = [];
  const params: (string | number)[] = [];

  if (opts.status) {
    where.push("status = ?");
    params.push(opts.status);
  }
  if (opts.q) {
    where.push("(title LIKE ? OR author LIKE ?)");
    params.push(`%${opts.q}%`, `%${opts.q}%`);
  }

  const clause = where.length ? `WHERE ${where.join(" AND ")}` : "";

  const total = (
    db.prepare(`SELECT COUNT(*) AS n FROM books ${clause}`).get(...params) as { n: number }
  ).n;

  const rows = db
    .prepare(`SELECT * FROM books ${clause} ORDER BY id DESC LIMIT ? OFFSET ?`)
    .all(...params, opts.limit, opts.offset) as BookRow[];

  return { items: rows.map(toBook), total };
}
```

```ts
// server/src/routes/books.ts (replace the GET "/" route)
function clampInt(raw: unknown, fallback: number, min: number, max: number): number {
  const n = typeof raw === "string" ? Number.parseInt(raw, 10) : Number.NaN;
  if (Number.isNaN(n)) return fallback;
  return Math.min(Math.max(n, min), max);
}

booksRouter.get("/", (req, res) => {
  const { status, q } = req.query;
  if (status !== undefined && !isStatus(status)) {
    return res.status(400).json({ error: "status is invalid" });
  }
  const limit = clampInt(req.query.limit, 10, 1, 50);
  const offset = clampInt(req.query.offset, 0, 0, Number.MAX_SAFE_INTEGER);
  const search = typeof q === "string" ? q.trim() : "";

  res.json(bookRepository.list({ status, q: search || undefined, limit, offset }));
});
```

Line by line:
- Only **fixed strings** (`"status = ?"`) are concatenated into the SQL. Every user-supplied value goes into `params`, so this is still injection-safe.
- The same `clause` is reused for `COUNT(*)` and the page query, so `total` always matches the filter.
- `clampInt` stops absurd values (`limit=1000000`) from turning one request into a database dump.
- We never allow the client to choose a column name to sort by. If you add sorting, **whitelist** the allowed columns (identifiers can't be parameterized).

Now the client:

```ts
// client/src/api/books.ts (replace list, add types)
import type { BookStatus } from "../types";

export interface ListParams {
  status?: BookStatus;
  q?: string;
  page: number;
  limit: number;
}

export interface ListResponse {
  items: Book[];
  total: number;
}

// inside booksApi:
list: (p: ListParams, signal?: AbortSignal) => {
  const qs = new URLSearchParams();
  if (p.status) qs.set("status", p.status);
  if (p.q) qs.set("q", p.q);
  qs.set("limit", String(p.limit));
  qs.set("offset", String((p.page - 1) * p.limit));
  return request<ListResponse>(`/api/books?${qs.toString()}`, { signal });
},
```

```tsx
// client/src/App.tsx (the reading effect, now driven by filters)
const [status, setStatus] = useState<BookStatus | "">("");
const [queryInput, setQueryInput] = useState("");
const [q, setQ] = useState("");
const [page, setPage] = useState(1);
const [total, setTotal] = useState(0);
const LIMIT = 5;

useEffect(() => {
  const controller = new AbortController();
  setLoading(true);

  booksApi
    .list({ status: status || undefined, q, page, limit: LIMIT }, controller.signal)
    .then((res) => {
      setBooks(res.items);
      setTotal(res.total);
      setLoadError(null);
    })
    .catch((err: unknown) => {
      if (err instanceof DOMException && err.name === "AbortError") return;
      setLoadError(err instanceof Error ? err.message : "Could not load books");
    })
    .finally(() => {
      if (!controller.signal.aborted) setLoading(false);
    });

  return () => controller.abort();
}, [status, q, page]);

const totalPages = Math.max(1, Math.ceil(total / LIMIT));
```

Controls (excerpt):

```tsx
<select value={status} onChange={(e) => { setStatus(e.target.value as BookStatus | ""); setPage(1); }}>
  <option value="">All</option>
  <option value="to-read">To read</option>
  <option value="reading">Reading</option>
  <option value="done">Done</option>
</select>

<form onSubmit={(e) => { e.preventDefault(); setQ(queryInput.trim()); setPage(1); }}>
  <input value={queryInput} onChange={(e) => setQueryInput(e.target.value)} placeholder="Search…" />
  <button type="submit">Search</button>
</form>

<button disabled={page <= 1} onClick={() => setPage((p) => p - 1)}>Prev</button>
<span>Page {page} of {totalPages}</span>
<button disabled={page >= totalPages} onClick={() => setPage((p) => p + 1)}>Next</button>
```

Line by line:
- The effect's **dependency array** `[status, q, page]` means "re-fetch whenever any filter changes." This is `useEffect` doing its job as a synchronizer between state and the server.
- The cleanup `controller.abort()` cancels the previous request, so a slow earlier response can't overwrite a newer one (a **race condition**).
- `setPage(1)` whenever a filter changes. Otherwise you could be on page 4 of a result set that now has only 1 page.
- Search is applied on submit rather than on every keystroke to avoid firing a request per letter. (Delaying requests until typing pauses is called **debouncing**, a later lesson.)
- Setting `setLoading(true)` at the top of the effect would replace the whole page with "Loading…" on every filter change; consider keeping the previous list visible while loading as a polish step.

**Relationships and JOINs**
Rows in different tables are combined with a **JOIN**. To show each book's average rating:

```sql
SELECT b.*, ROUND(AVG(r.rating), 1) AS avg_rating, COUNT(r.id) AS review_count
FROM books b
LEFT JOIN reviews r ON r.book_id = b.id
GROUP BY b.id;
```

`LEFT JOIN` keeps books that have zero reviews (an `INNER JOIN` would drop them). The classic mistake is the **N+1 problem**: fetch 20 books, then fire 20 *more* requests, one `GET /api/books/:id/reviews` per book. That's 21 round trips where one query would do. Let the database combine the data in a single query and return it in one response.

An **index** is a lookup structure that makes `WHERE`/`JOIN` on a column fast. Add one on foreign keys you filter or join by:

```sql
CREATE INDEX IF NOT EXISTS idx_reviews_book_id ON reviews(book_id);
```

**Why it matters**
Real apps have thousands of rows, searchable lists, and related data. Moving filtering, paging, and joining into the database keeps pages fast and payloads small.

**Mental model**
`Array.filter().slice()` in JavaScript versus `WHERE … LIMIT … OFFSET` in SQL: same idea, but the database does it *before* sending data, so you never download rows you'll throw away.

**Common beginner mistakes**
- **Mistake:** Fetching everything and filtering in React.
  **Symptom:** Slow load and huge responses once data grows.
  **Fix:** Filter and paginate on the server.
- **Mistake:** Forgetting to reset to page 1 when a filter changes.
  **Symptom:** "No results" while the filter actually has matches on page 1.
  **Fix:** `setPage(1)` in the filter handlers.
- **Mistake:** Putting a user-chosen column name directly into `ORDER BY`.
  **Symptom:** Injection risk (placeholders can't be used for identifiers).
  **Fix:** Map allowed sort keys to fixed column names.
- **Mistake:** Changing the response shape on the server without updating the client.
  **Symptom:** `res.items is undefined` or `books.map is not a function`.
  **Fix:** Change both sides together; treat the API as a contract.
- **Mistake:** Not including a filter variable in the effect's dependencies.
  **Symptom:** Changing the dropdown does nothing until you refresh.
  **Fix:** Every state value used inside the effect belongs in the dependency array.

**Try it (5 min)**
Request `/api/books?limit=2&offset=2` with curl and confirm you get the second page and the correct `total`. Then try `?limit=abc` and `?status=finished` and note the results.

**Project step: Milestone 7 (about 35 min)**
Add status filter, search, and pagination end to end. (Stretch: add the `LEFT JOIN` rating query as `GET /api/books/summary`, declared *before* `/:id`.)

---

### 3.12 Taking It to Production: Swapping Databases, Secrets, and Safety

**Explanation**
Because all SQL lives in `bookRepository.ts`, moving to a different database means rewriting *that file*, not your routes or React code, apart from one ripple effect: server databases are accessed over a network, so the calls become **asynchronous** and routes need `await`. (Some teams make repository functions `async` from day one to avoid this ripple.)

The same `listByStatus` on two other databases (**illustrative, not part of the project**):

```ts
// PostgreSQL (using the `pg` package): server/src/bookRepository.pg.ts
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

export async function listByStatus(status: string) {
  const { rows } = await pool.query(
    `SELECT id, title, author, status, created_at AS "createdAt"
     FROM books WHERE status = $1 ORDER BY id DESC`,
    [status]
  );
  return rows;
}
```

```ts
// MongoDB (using the official `mongodb` driver): server/src/bookRepository.mongo.ts
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI!);
const books = client.db("bookshelf").collection<{ title: string; author: string; status: string; createdAt: Date }>("books");

export async function listByStatus(status: string) {
  return books.find({ status }).sort({ _id: -1 }).toArray();
}
```

> **Version note:** Driver APIs and connection options change between major versions of `pg` and `mongodb`. Treat these as sketches and verify against current docs before using them. Postgres uses `$1, $2` placeholders instead of `?`. MongoDB identifies documents by an `_id` (an `ObjectId`, not a number), which would change your `Book.id` type and every route that parses `id`.

**Production checklist**
- **Secrets stay on the server.** Database URLs and passwords go in environment variables (`process.env.DATABASE_URL`) loaded from a `.env` file that is **never committed**. Anything prefixed `VITE_` in the client is visible to every visitor.
- **Hosted databases.** Managed Postgres (Neon, Supabase, AWS RDS, and others) or MongoDB Atlas give you backups, scaling, and persistent storage. Pick one when you deploy beyond a single machine.
- **Backups.** A database with no backup is a prototype. SQLite is a file, so copy it; hosted services typically automate this.
- **Least privilege and validation.** Never connect with an admin account from the app, and keep validating input at the API boundary.
- **Authentication.** Right now anyone can delete any book. Identifying users and protecting routes is a later lesson.

```ts
// server/src/index.ts (add a health endpoint useful for hosting platforms)
app.get("/api/health", (_req, res) => res.json({ ok: true }));
```

**Why it matters**
The most common "my app is live but broken" causes are leaked secrets, missing persistent storage, and databases chosen without a plan to move. Isolating data access is what makes those fixable.

**Mental model**
The repository is an **adapter**, like a travel plug converter. The rest of your app speaks one shape (`list()`, `create()`), and the adapter translates to whatever socket (database) is on the wall.

**Common beginner mistakes**
- **Mistake:** Committing `.env` or a database file.
  **Symptom:** Credentials in Git history (bots scan public repos within minutes).
  **Fix:** `.gitignore` them; if leaked, rotate the credentials immediately.
- **Mistake:** Using a `VITE_DATABASE_URL`.
  **Symptom:** The connection string is readable in the browser bundle.
  **Fix:** Database access happens only on the server.
- **Mistake:** Deploying SQLite to a platform with a temporary filesystem.
  **Symptom:** Data resets on every deploy or restart.
  **Fix:** Use persistent storage or a hosted database.
- **Mistake:** Forgetting that routes must now `await` after moving to Postgres.
  **Symptom:** Responses contain `{}` (a pending Promise serialized) instead of data.
  **Fix:** Make routes `async` and `await` repository calls (Express 5 forwards rejections to the error handler).

**Try it (4 min)**
Add `.env`, `*.db`, `*.db-wal`, and `*.db-shm` to `server/.gitignore`. Read `PORT` from `process.env` (already done) and start the server with `PORT=4000`.

**Project step: Milestone 8 (about 15 min)**
Add the health route and `.gitignore` entries, and write a three-sentence note in the README on what you would change to move BookShelf to Postgres.

---

## 4. How the Sub-Sections Connect

**Dependency order**

```
3.1 The data path (browser -> API -> DB)         [the big picture]
   │
3.2 Where data can live                          [state vs localStorage vs server]
   │
3.3 Choosing a database                          [decision: SQLite for this project]
   │
3.4 Modeling and typing data                     [row / domain / input types]
   │
3.5 SQLite setup + schema        ──┐
3.6 Repository + SQL CRUD          ├─ SERVER SIDE (needs 3.4's types)
3.7 REST routes + validation     ──┘
   │
3.8 Client API module + proxy    ──┐             [needs 3.7's endpoints]
3.9 Reading data in React          ├─ CLIENT SIDE
3.10 Writing data                  │
3.11 Filtering, pagination, joins ─┘             [changes both sides: API contract]
   │
3.12 Swapping databases, secrets, production     [applies 3.3 + 3.6: the repository boundary]
```

**Hierarchy:** 3.1-3.4 are *concepts and decisions* (low code). 3.5-3.7 are the *server data layer*. 3.8-3.11 are the *client data layer* plus an API change that touches both. 3.12 *generalizes* everything by showing the boundary that makes the stack replaceable.

**Key dependencies:** 3.7 needs 3.6; 3.8 needs a running 3.7; 3.10 needs 3.9; 3.11 modifies 3.6, 3.7, 3.8, and 3.9; 3.12 depends on the repository boundary from 3.6.

**Suggested live-teaching flow (6 hours = 360 min)**

| Clock | Min | Section | Mode |
|---|---|---|---|
| 0:00-0:15 | 15 | 3.1 The data path | Talk and whiteboard; learners trace the "Add book" path |
| 0:15-0:25 | 10 | 3.2 Where data can live | Short demo: the localStorage experiment; learners try it |
| 0:25-0:50 | 25 | 3.3 Choosing a database | Discussion of the comparison table; pair exercise (the "pick a DB" task) |
| 0:50-1:05 | 15 | 3.4 Modeling and typing | Live-code `types.ts`; learners add `Review` |
| 1:05-1:25 | 20 | 3.5 SQLite setup | Everyone installs and creates `db.ts` (Milestone 1) |
| 1:25-1:35 | 10 | Break | |
| 1:35-2:05 | 30 | 3.6 Repository and SQL | Demo injection live; learners code Milestone 2 |
| 2:05-2:35 | 30 | 3.7 Routes and validation | Live-code POST and error cases; learners test with curl (Milestone 3) |
| 2:35-3:00 | 25 | 3.8 Client data layer | Scaffold, proxy, show the CORS error on purpose; Milestone 4 |
| 3:00-3:15 | 15 | Lunch / longer break | |
| 3:15-3:45 | 30 | 3.9 Reading in React | Live-code the effect; kill the server to show errors; Milestone 5 |
| 3:45-4:25 | 40 | 3.10 Writing data | Demo pessimistic vs optimistic; learners code Milestone 6 (the biggest build block) |
| 4:25-5:00 | 35 | 3.11 Querying | Demo query params and the dependency array; Milestone 7 |
| 5:00-5:15 | 15 | 3.12 Production | Walk the Postgres and Mongo sketches; checklist; Milestone 8 |
| 5:15-5:50 | 35 | Knowledge check and applied exercise | Learners work solo on the **Favorites** exercise (Section 6, Part B) |
| 5:50-6:00 | 10 | Recap | Review the answer key, summary, and next lesson |

*Trim rule if running long:* shorten 3.2 and 3.12 first, and turn the 3.11 stretch (JOIN endpoint) and the Part A questions into homework.

---

## 5. Key Terms Glossary

*(in order of first appearance)*

- **Database**: software that stores data durably and answers queries about it.
- **API**: a server program that exposes endpoints other software (like React) can call over HTTP.
- **Endpoint**: a specific URL plus HTTP method an API responds to (for example `GET /api/books`).
- **Repository**: a module that holds all database code for one entity behind simple functions.
- **Ephemeral**: lost when the page reloads or process stops (like `useState`).
- **`localStorage`**: browser storage for small strings that persists across reloads on one browser.
- **Relational (SQL) database**: stores data in tables with a fixed schema and relationships between them.
- **Schema**: the declared structure of the data (tables, columns, types, constraints).
- **SQL**: the language used to query and change data in relational databases.
- **Document database**: stores flexible JSON-like documents without a fixed schema (for example MongoDB).
- **Key-value store**: maps keys to values, usually for fast caching (for example Redis).
- **BaaS (Backend-as-a-Service)**: a hosted platform providing database and API features out of the box.
- **Transaction**: a group of changes that all succeed or all fail together.
- **ACID**: the guarantees (atomicity, consistency, isolation, durability) of reliable transactions.
- **Table / row / column**: a collection of records / one record / one field of a record.
- **Primary key**: a column that uniquely identifies each row.
- **Foreign key**: a column holding another table's primary key, linking rows together.
- **One-to-many**: a relationship where one row relates to many others (a book has many reviews).
- **Constraint**: a rule enforced by the database (`NOT NULL`, `CHECK`, `REFERENCES`).
- **Migration**: a versioned, repeatable change to a database schema.
- **CRUD**: Create, Read, Update, Delete, the four basic data operations.
- **Parameterized query**: SQL where values are sent separately from the statement via placeholders.
- **SQL injection**: an attack where user input changes the meaning of a SQL query.
- **REST**: a convention for APIs using resource URLs and HTTP methods.
- **HTTP status code**: a number telling the client the result (200, 201, 204, 400, 404, 500).
- **Type guard**: a function whose return type (`v is T`) lets TypeScript narrow a value.
- **Middleware**: a function Express runs on requests before or after your route handlers.
- **CORS**: browser rule restricting cross-origin requests unless the server allows them.
- **Proxy (dev)**: the dev server forwarding certain requests (like `/api`) to another server.
- **Environment variable**: configuration read from the environment instead of hard-coded.
- **Pessimistic update**: update the UI only after the server confirms success.
- **Optimistic update**: update the UI immediately and roll back if the request fails.
- **Race condition**: a bug where the outcome depends on which of several async operations finishes first.
- **Query parameter**: the `?key=value` part of a URL used to pass options.
- **Pagination**: returning data in pages using `LIMIT` and `OFFSET`.
- **JOIN**: a SQL operation that combines rows from related tables.
- **N+1 problem**: running one query for a list, then one more per item, instead of one combined query.
- **Index**: a lookup structure that speeds up queries on a column.
- **Debouncing**: delaying an action until the user pauses (a later lesson).

---

## 6. Knowledge Check

### Part A: Questions

**Q1 (Recall).** Why should a React component never connect directly to a database? Give two concrete reasons.

**Q2 (Spot the bug).** What does this return when `title` is `x' OR '1'='1`, and how do you fix it?

```ts
function findByTitle(title: string) {
  return db.prepare(`SELECT * FROM books WHERE title = '${title}'`).all();
}
```

**Q3 (Predict the output).** The server has no book with id 999. What happens in this client code, and what does the user eventually see?

```tsx
useEffect(() => {
  fetch("/api/books/999")
    .then((res) => res.json())
    .then((book: Book) => setBook(book));
}, []);

// JSX: <h2>{book.title.toUpperCase()}</h2>
```

**Q4 (Spot the bug).** Changing the dropdown does nothing until the page is refreshed. Why?

```tsx
const [status, setStatus] = useState<BookStatus | "">("");

useEffect(() => {
  booksApi.list({ status: status || undefined, page: 1, limit: 5 }).then((r) => setBooks(r.items));
}, []);
```

**Q5 (Scenario / recall).** A teammate must choose storage for each of: (a) an online store's orders, items, and payments, (b) a cache of the last 5 minutes of API responses, (c) a prototype for a weekend hackathon that needs login and live updates in a day. Which database family fits each, and why?

### Part B: Applied mini-exercise: Favorites, end to end

**Goal:** Let users mark a book as a **favorite**, persisted in the database, using an optimistic update in React.

**Starter code:**

```ts
// server/src/db.ts: books table (TODO: add the column; remember the DB file already exists)
// TODO 1: add a `favorite` column (0 or 1, default 0)

// server/src/types.ts
// TODO 2: add `favorite` to BookRow (0 | 1), Book (boolean), and BookPatch

// server/src/bookRepository.ts
// TODO 3: map favorite -> boolean in toBook; accept it in update()

// server/src/routes/books.ts
// TODO 4: validate `favorite` in parsePatch (must be a boolean if present)

// client/src/types.ts + api
// TODO 5: add `favorite: boolean` to Book; allow `favorite` in BookPatch

// client/src/App.tsx
// TODO 6: add a ★ button per book that toggles favorite optimistically,
//         rolls back and shows an error if the request fails
```

**Requirements:**
1. Store favorite as an integer `0/1` in SQLite and expose it as a boolean in the API.
2. `PATCH /api/books/:id` with `{ "favorite": true }` updates only that field; non-boolean values return `400`.
3. Existing rows must work (default to not favorite) without losing data.
4. The ★ toggle updates instantly in the UI and rolls back on failure.
5. All SQL uses placeholders, and no new `any` types are introduced.

**Acceptance criteria:**
- `curl -X PATCH …/api/books/1 -H "Content-Type: application/json" -d '{"favorite":"yes"}'` returns `400` with a clear message.
- Toggling the star and refreshing keeps the state.
- Stopping the server and clicking the star shows an error and reverts the star.
- Existing books are still listed after the change (no data loss).
- TypeScript compiles in both projects.

---

## 7. Summary Recap

- **React never talks to the database.** Data travels component → typed client module → HTTP → Express route → repository → database, and each layer has one job.
- **Choose storage by the data, not the hype.** `useState` and `localStorage` are not databases; start relational (SQLite to learn, Postgres to deploy) unless your data is truly document-shaped or you need a hosted BaaS.
- **Never trust input and never concatenate SQL.** Validate at the API boundary, use parameterized queries, and let database constraints be the last line of defense.
- **`fetch` only rejects on network errors.** Check `res.ok`, throw typed errors, and keep load errors separate from action errors.
- **Reads belong in `useEffect`; writes belong in event handlers.** Include every value the effect uses in its dependency array, clean up with `AbortController`, and choose pessimistic or optimistic updates deliberately.
- **Keep the database behind a repository.** It makes querying, pagination, and joins a server concern, and it makes swapping databases (and protecting secrets) a one-file change.

---

## 8. Next Lesson Bridge

BookShelf now persists data, but every component repeats the same fetch, loading, and error plumbing, and anyone can delete anyone's books. The next lessons build on this directly: **Custom Hooks** (and data-fetching libraries such as TanStack Query) will extract and cache the read/write patterns you wrote by hand, and **Authentication** will identify users and protect the routes you just built. ORMs such as Prisma or Drizzle will then replace much of the hand-written repository SQL.

---

## Instructor Answer Key

**Q1.**
1. **Security:** browser code is public and user-controlled. Database credentials or direct query access in the browser would let anyone read, change, or delete all data.
2. **Architecture and capability:** databases speak their own protocols over server-side connections, not HTTP from a browser; an API layer is also where validation, authorization, and business rules live, and where you can change the database without changing the UI.
(Also acceptable: bundle exposure of secrets, no place to enforce rules, scaling and connection limits.)

**Q2.** The string becomes `SELECT * FROM books WHERE title = 'x' OR '1'='1'`. `'1'='1'` is always true, so it returns **every book**. This is SQL injection. Fix: use a placeholder: `db.prepare("SELECT * FROM books WHERE title = ?").all(title)`. The value is always treated as data, never as SQL.

**Q3.** `fetch` resolves (it does not reject) on a 404, so `res.json()` parses the server's error body, `{ "error": "Book not found" }`, and it's cast as a `Book`. `book.title` is `undefined`, so `book.title.toUpperCase()` throws `TypeError: Cannot read properties of undefined (reading 'toUpperCase')` and the component crashes (a blank screen in production). Fix: check `res.ok` (or use the `request` helper that throws `ApiError`) and render an error state. A secondary issue: `book` is probably `null` initially, so the render must handle that case as well.

**Q4.** The effect has an empty dependency array `[]`, so it runs only once on mount, even though it reads `status`. When `status` changes, nothing re-runs the fetch, so the list keeps showing the first result until a refresh remounts the component. Fix: add `status` to the dependency array: `[status]`, include a cleanup that aborts the previous request, and reset `page` to 1 when the filter changes.

**Q5.**
- **(a) Online store:** **Relational (PostgreSQL)**. Orders, items, and payments are related and need transactions and consistency (all-or-nothing writes).
- **(b) Cache of API responses:** **Key-value store (Redis)**. Fast reads and expiry (TTL) suit short-lived, disposable data.
- **(c) Hackathon prototype with login and live updates:** **BaaS (Firebase or Supabase)**. Auth and real-time are built in, which saves time under a one-day deadline, accepting vendor lock-in.

### Model solution: Favorites

```ts
// server/src/db.ts: change the table definition for fresh databases
// (add inside CREATE TABLE books)
//   favorite   INTEGER NOT NULL DEFAULT 0 CHECK (favorite IN (0, 1)),
```

For an **existing** database file, `CREATE TABLE IF NOT EXISTS` won't add the column, so run this once (or delete `bookshelf.db` and re-seed during development):

```bash
sqlite3 bookshelf.db "ALTER TABLE books ADD COLUMN favorite INTEGER NOT NULL DEFAULT 0 CHECK (favorite IN (0, 1));"
```

```ts
// server/src/types.ts
export interface BookRow {
  id: number;
  title: string;
  author: string;
  status: BookStatus;
  created_at: string;
  favorite: 0 | 1;
}

export interface Book {
  id: number;
  title: string;
  author: string;
  status: BookStatus;
  createdAt: string;
  favorite: boolean;
}

export type BookPatch = Partial<NewBook> & { favorite?: boolean };
```

```ts
// server/src/bookRepository.ts
const toBook = (r: BookRow): Book => ({
  id: r.id,
  title: r.title,
  author: r.author,
  status: r.status,
  createdAt: r.created_at,
  favorite: r.favorite === 1,
});

function update(id: number, patch: BookPatch): Book | undefined {
  db.prepare(
    `UPDATE books
     SET title    = COALESCE(?, title),
         author   = COALESCE(?, author),
         status   = COALESCE(?, status),
         favorite = COALESCE(?, favorite)
     WHERE id = ?`
  ).run(
    patch.title ?? null,
    patch.author ?? null,
    patch.status ?? null,
    patch.favorite === undefined ? null : patch.favorite ? 1 : 0,
    id
  );
  return get(id);
}
```

```ts
// server/src/routes/books.ts: inside parsePatch
const { title, author, status, favorite } = body as Record<string, unknown>;
// ...existing title/author/status checks...
if (favorite !== undefined) {
  if (typeof favorite !== "boolean") return "favorite must be a boolean";
  patch.favorite = favorite;
}
```

```ts
// client/src/types.ts
export interface Book {
  id: number;
  title: string;
  author: string;
  status: BookStatus;
  createdAt: string;
  favorite: boolean;
}

export type BookPatch = Partial<NewBook> & { favorite?: boolean };
```

```tsx
// client/src/App.tsx
const handleToggleFavorite = async (book: Book) => {
  const next = !book.favorite;
  setBooks((prev) => prev.map((b) => (b.id === book.id ? { ...b, favorite: next } : b)));
  try {
    await booksApi.update(book.id, { favorite: next });
    setActionError(null);
  } catch (err) {
    setBooks((prev) => prev.map((b) => (b.id === book.id ? { ...b, favorite: !next } : b)));
    setActionError(err instanceof Error ? err.message : "Could not update favorite");
  }
};

// in each list item:
<button
  onClick={() => handleToggleFavorite(b)}
  aria-pressed={b.favorite}
  className={b.favorite ? "text-amber-500" : "text-slate-300"}
>
  ★
</button>
```

**Talking points for review:**
- SQLite has no real boolean type, so we store `0/1`, constrain it with `CHECK`, and convert at the repository boundary (`toBook`), keeping that detail out of the API and React.
- `COALESCE` with `null` keeps the update **partial**: patching only `favorite` leaves the other fields alone.
- Optimistic rollback restores `!next`, which equals the original value, without needing to capture the old one separately.
- The `ALTER TABLE` step is a hand-run **migration**, and the reason real projects use migration tooling.
- Validation rejects `"yes"` with a `400` before the database is touched.

> **Version note for the instructor:** verify the installed versions of Express (5.x error-handling behavior), `better-sqlite3` (Node compatibility), Vite and its `react-ts` template `tsconfig` flags (`verbatimModuleSyntax`, `erasableSyntaxOnly`), and the Tailwind setup before class. The `pg` and `mongodb` snippets are illustrative sketches; confirm their current APIs in each driver's docs. All React code uses only stable hooks (`useState`, `useEffect`) and the standard `fetch`/`AbortController` APIs.
