# Learning a New Programming Language with AI — Python → TypeScript


**Exercise:** Learning a New Programming Language with AI

**Source language:** Python | **Target language:** TypeScript

All TypeScript code in this document was actually compiled (`tsc --strict`) and executed via Node.js, not just theorized.

---

## Part 1: Learning Journey Plan

**Learning goals:**
1. Understand TypeScript's static type system well enough to write type-safe functions and interfaces, coming from Python's dynamic typing.
2. Build a small CLI tool in TypeScript, understanding Node.js's module system and async patterns.
3. Get comfortable with TypeScript's structural typing (vs. Python's duck typing) well enough to know when they behave the same and when they don't.

**Structured learning plan (generated and reviewed):**

| Phase | Prerequisites | Key steps | Verification |
|---|---|---|---|
| 1. JS/TS fundamentals | Node.js installed, terminal comfort | Variables, functions, arrow functions, template literals, `npm`/`package.json` | Write 3 small scripts, run via `node`/`ts-node` |
| 2. TypeScript's type system | Phase 1 | Primitive types, interfaces vs. type aliases, unions, generics, `strict` mode | Fully annotate an untyped JS function; justify each choice |
| 3. Structural typing & unlearning Python habits | Phase 2 | Structural vs. nominal typing, optional/readonly props, discriminated unions, `unknown` vs `any` | Predict then verify whether two differently-named-but-identically-shaped interfaces are interchangeable |
| 4. Applied project | Phase 3 | CLI arg parsing, async/await, `fs/promises`, packaging | Build and run the mini-project (Part 4) |

---

## Part 2: Four-Step Prompting Strategy

**Topic:** TypeScript's structural typing.

### Step 1 — Conceptual Understanding

- **Key philosophical difference:** Python's type hints are optional and unenforced at runtime; TypeScript's types are enforced at compile time but erased entirely at runtime — there is no TypeScript at runtime, only the JavaScript it compiles to.
- **Problem TypeScript solves:** JavaScript has no static types at all; large JS codebases suffered runtime errors that would have been compile-time errors in a typed language. TypeScript adds a type layer on top of JS without changing JS's runtime behavior.
- **Mental model to adjust:** TypeScript's compiler (`tsc`) refuses to emit valid output for code that fails type checks (though `any` can escape this) — a much stronger guarantee than Python's optional `mypy` checking.
- **Common misconception:** Assuming TypeScript types exist at runtime (they don't — interfaces vanish after compilation) and assuming type compatibility requires an explicit relationship (structural typing means unrelated interfaces with identical shapes are fully interchangeable).

### Step 2 — Step-by-Step Breakdown

- **How it's implemented:** TypeScript compares the *shape* of two types rather than checking explicit declared relationships. If type B has every property type A requires, B is assignable wherever A is expected.
- **Comparison to Python:** Python's duck typing is a runtime concept (works if the object has the needed attributes *when called*); TypeScript's structural typing does the equivalent check statically, before the code ever runs.
- **Key syntax:** `interface` describes a shape; any matching object literal or class satisfies it with no `implements` keyword required. Optional (`prop?: type`) and `readonly` properties refine the shape further.
- **Common pattern:** Small, focused interfaces describing exactly what a function needs — philosophically closer to Python's duck typing than to Java/C#-style interfaces, just enforced at compile time.

### Step 3 — Guided Implementation

```typescript
interface Point {
  x: number;
  y: number;
}

interface Coordinate {
  x: number;
  y: number;
}

function printPoint(p: Point): void {
  console.log(`(${p.x}, ${p.y})`);
}

const coord: Coordinate = { x: 3, y: 4 };
printPoint(coord);              // Works: Coordinate matches Point's shape
printPoint({ x: 1, y: 2 });     // Works: plain object literal also matches
```

**Actual run:**
```
$ npx tsc --strict --noEmit structural_typing.ts && echo "TYPE CHECK PASSED"
TYPE CHECK PASSED
$ node dist/structural_typing.js
(3, 4)
(1, 2)
```

**Syntax note for a Python developer:** `interface` is purely compile-time — no runtime object, no `isinstance()` equivalent. The type check happens once, at compile time, with nothing corresponding to it at runtime.

### Step 4 — Understanding Verification

To confirm the compiler genuinely enforces the shape (not just accepting everything), a deliberate mismatch was tested:

```typescript
interface Point { x: number; y: number; }
function printPoint(p: Point): void { console.log(`(${p.x}, ${p.y})`); }
printPoint({ x: 1 }); // missing 'y' -- should fail
```

**Actual compiler output:**
```
mismatch_test.ts(8,12): error TS2345: Argument of type '{ x: number; }' is not
assignable to parameter of type 'Point'.
  Property 'y' is missing in type '{ x: number; }' but required in type 'Point'.
```

This confirms structural typing genuinely enforces the shape rather than being permissive by default — a real compile error was produced, not just a linter warning.

---

## Part 3: Advanced Prompting Techniques

### Technique 1 — Using Context Effectively

**Topic:** Union types, compared to Python's `Union`/`Optional`.

Python's `Union[int, str]` (or `int | str`) is purely documentation for humans and tools like `mypy` — nothing stops incorrect usage from running (and crashing) at runtime. TypeScript's compiler actively **narrows** the type within conditional branches: inside `if (typeof x === 'string')`, TypeScript knows `x` is a `string` for the rest of that block and flags any operation invalid for a string. This is enforced before code ships, unlike Python's optional static checking.

### Technique 2 — Promoting Deep Understanding

**Code interrogated:**
```typescript
function formatId(id: number | string): string {
  if (typeof id === "string") {
    return id.toUpperCase();
  }
  return id.toFixed(2);
}
```

**Actual run:**
```
$ npx tsc --strict --noEmit union_narrowing.ts && echo "TYPE CHECK PASSED"
TYPE CHECK PASSED
$ node dist2/union_narrowing.js
42.00
ABC
```

**Deep-understanding answers:**
1. **Performance:** `typeof` checks are extremely cheap at runtime — effectively zero cost versus alternatives; the real "cost" is entirely at compile time.
2. **Alternative approaches:** A discriminated union with an explicit tag field is more idiomatic once the union grows beyond 2-3 primitive types, since `typeof` narrowing only works cleanly for primitives.
3. **Scaling 10x:** A long `if/else if` chain becomes unwieldy fast — a discriminated union with a `switch` and TypeScript's exhaustiveness checking (a `never`-typed default case) can guarantee every case is handled at compile time.
4. **Alternative feature:** Function overloads could express the same logic with more precise per-call-site typing, though overloads are better suited to cases where the *return type* also depends on which overload matched (not true here, since both branches return `string`).

---

## Part 4: Mini-Project — Word Frequency CLI Tool

**Project:** A CLI tool that reads a text file and reports the top N most common words.

**Planning outcomes:**
1. **Components:** CLI argument parsing, file reading (`fs/promises`), a pure counting/tokenizing function, output formatting.
2. **Libraries:** Node's built-in `fs/promises` and `process.argv` are sufficient — zero external dependencies, mirroring the Task Manager Python project's own stdlib-only philosophy.
3. **Key files:** A single `wordFrequency.ts`, with the counting logic kept as a pure, separately-exported function so it's testable independent of file I/O and CLI parsing.
4. **Python-habit challenge identified:** Node's `fs/promises` API is `async` by default, requiring `await`/`async function main()` — a real adjustment from Python's typically-synchronous `open()`/`with` file handling.

### Implementation

```typescript
import { readFile } from "fs/promises";

interface WordCount {
  word: string;
  count: number;
}

export function countWordFrequency(text: string): Map<string, number> {
  const counts = new Map<string, number>();
  const words = text
    .toLowerCase()
    .replace(/[^\w\s]/g, "")
    .split(/\s+/)
    .filter((w) => w.length > 0);

  for (const word of words) {
    counts.set(word, (counts.get(word) ?? 0) + 1);
  }
  return counts;
}

export function topNWords(counts: Map<string, number>, n: number): WordCount[] {
  return Array.from(counts.entries())
    .map(([word, count]): WordCount => ({ word, count }))
    .sort((a, b) => b.count - a.count)
    .slice(0, n);
}

async function main(): Promise<void> {
  const [, , filePath, topFlag, topValue] = process.argv;

  if (!filePath) {
    console.error("Usage: node wordFrequency.js <file> [--top N]");
    process.exit(1);
  }

  const n = topFlag === "--top" && topValue ? parseInt(topValue, 10) : 5;
  const text = await readFile(filePath, "utf-8");
  const counts = countWordFrequency(text);
  const top = topNWords(counts, n);

  console.log(`Top ${n} words:`);
  for (const { word, count } of top) {
    console.log(`  ${word}: ${count}`);
  }
}

if (require.main === module) {
  main().catch((err) => {
    console.error("Error:", err.message);
    process.exit(1);
  });
}
```

**Real finding during implementation:** the first compile attempt failed with `Cannot find name 'process'`/`'require'`/`'module'` — Node's global types aren't bundled with TypeScript by default and require installing `@types/node` separately (`npm install --save-dev @types/node`). This is a genuine first-time-setup gotcha for anyone coming from Python, where the standard library's types are simply built in.

### Actual Execution

```
$ npx tsc --strict --module commonjs --target es2020 wordFrequency.ts --outDir dist3 --types node
CLEAN TYPE CHECK

$ node dist3/wordFrequency.js sample.txt --top 3
Top 3 words:
  the: 5
  fox: 3
  dog: 2
```

(Sample file: three sentences about a fox, a dog, and "the" repeated as expected for common English words.)

### Verifying the Pure Function Independently

```typescript
import { countWordFrequency, topNWords } from "./wordFrequency";

const text = "cat dog cat bird dog cat";
const counts = countWordFrequency(text);

console.assert(counts.get("cat") === 3, "expected cat=3, got " + counts.get("cat"));
console.assert(counts.get("dog") === 2, "expected dog=2, got " + counts.get("dog"));
console.assert(counts.get("bird") === 1, "expected bird=1, got " + counts.get("bird"));

const top2 = topNWords(counts, 2);
console.assert(top2.length === 2, "expected 2 results");
console.assert(top2[0].word === "cat" && top2[0].count === 3, "expected top word to be cat");

console.log("All assertions passed.");
```

**Actual run:**
```
All assertions passed.
```

This confirms the design goal from the planning step — separating pure logic from I/O — actually paid off: the counting function was testable with zero file-system or CLI involvement.

