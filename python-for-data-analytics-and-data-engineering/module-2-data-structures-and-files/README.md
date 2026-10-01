# Module 2: Core Data Structures & File Handling

With the fundamentals in place, Module 2 introduces how Python represents and stores *collections* of data — the structures every dataset ultimately reduces to before it reaches pandas — and how to read and write that data from files.

Every lesson continues using the `ecommerce` schema (`categories`, `products`, `customers`, `orders`, `order_items`, `events`), now represented as lists, dictionaries, and the raw CSV/JSON files those tables get exported to in practice.

## Lessons

| # | Lesson | File |
|---|--------|------|
| 2.1 | Lists, Tuples & Sets | `lesson-2.1-lists-tuples-sets.md` |
| 2.2 | Dictionaries & Nested Structures | `lesson-2.2-dictionaries.md` |
| 2.3 | Comprehensions | `lesson-2.3-comprehensions.md` |
| 2.4 | Dates & Times | `lesson-2.4-datetime.md` |
| 2.5 | Reading & Writing CSV | `lesson-2.5-csv.md` |
| 2.6 | Reading & Writing JSON | `lesson-2.6-json.md` |
| 2.7 | File Paths with `pathlib` | `lesson-2.7-pathlib.md` |
| 2.8 | Regular Expressions for Text Cleaning | `lesson-2.8-regex.md` |

## Lesson Format

Same six-part structure as Module 1: CONCEPT → WHY IT MATTERS → SYNTAX REFERENCE → WORKED EXAMPLE → COMMON MISTAKES → PRACTICE EXERCISES, with hidden hints/solutions and a Data Engineer Tip per lesson.

**Data Engineer Tip (module-level):** This module is the last stop before pandas. Everything here — lists of dicts, nested JSON, CSV rows, date parsing, path handling, regex cleanup — is exactly what a DataFrame abstracts away for you. Understanding it manually first means you'll actually know what pandas is doing under the hood, instead of treating it as a black box.

---

**Previous module:** [Module 1 — Python Fundamentals](../module-1-python-fundamentals/README.md)
**Next module:** [Module 3 — NumPy](../module-3-numpy/README.md)
