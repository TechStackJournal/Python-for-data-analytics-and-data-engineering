# Module 1: Python Fundamentals

Welcome to Module 1 of the Python for Data Analytics & Data Engineering course. This module assumes **zero prior programming experience**. By the end of it, you'll be comfortable enough with core Python to start working with real data in Module 2.

> **Haven't set up Python yet?** Complete [Module 0 — Prerequisites & Setup](../module-0-setup/README.md) first. It walks through installing Python, creating a virtual environment, and configuring VS Code — everything below assumes that's already done.

Every lesson uses the same running example: a small e-commerce business, tracked through the `categories`, `products`, `customers`, `orders`, `order_items`, and `events` tables — the same `ecommerce` schema used throughout the companion SQL course. You don't need a database for this module; we simulate the data with plain Python variables and lists so you can focus on syntax first.

## Lessons

| # | Lesson | File |
|---|--------|------|
| 1.1 | Variables, Data Types & Type Conversion | `lesson-1.1-variables-and-types.md` |
| 1.2 | Operators & Expressions | `lesson-1.2-operators-and-expressions.md` |
| 1.3 | Strings & String Methods | `lesson-1.3-strings.md` |
| 1.4 | Conditional Logic | `lesson-1.4-conditionals.md` |
| 1.5 | Loops & Loop Control | `lesson-1.5-loops.md` |
| 1.6 | Functions | `lesson-1.6-functions.md` |
| 1.7 | Scope & Namespaces | `lesson-1.7-scope.md` |
| 1.8 | Error Handling Basics | `lesson-1.8-error-handling.md` |

## Lesson Format

Every lesson follows the same six-part structure:

1. **CONCEPT** — plain-language explanation of the idea
2. **WHY IT MATTERS** — why this matters specifically for data analytics/engineering
3. **SYNTAX REFERENCE** — a quick-lookup box for the exact syntax
4. **WORKED EXAMPLE** — a full example using the e-commerce dataset
5. **COMMON MISTAKES** — beginner pitfalls, and how to fix them
6. **PRACTICE EXERCISES** — 3–5 exercises, ordered easy → hard, each with a hidden hint and solution

## How to Work Through This Module

1. Read the CONCEPT and WHY IT MATTERS sections first — don't skip to code.
2. Type out the WORKED EXAMPLE yourself rather than copy-pasting. Typing builds muscle memory that copy-paste doesn't.
3. Attempt every practice exercise before opening the hint. Struggle for a few minutes — that's where the learning happens.
4. Only open the solution after you've either solved it or genuinely gotten stuck.

## Setup

You only need a plain Python file or an interactive shell for this module:

```bash
python3 --version   # confirm Python 3.12+
python3             # opens the interactive REPL, good for experimenting
```

Or create a file per lesson, e.g. `lesson_1_1.py`, and run it with:

```bash
python3 lesson_1_1.py
```

---

**Data Engineer Tip (module-level):** Everything in this module — variables, types, loops, functions — is the vocabulary you'll use to describe *transformations*. When you later write "clean this column" or "flag this row as invalid" in a real pipeline, it will compile down to exactly the fundamentals covered here. There is no shortcut around this module; even advanced pandas code is built entirely out of these pieces.
