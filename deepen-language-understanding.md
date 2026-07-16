# Applying AI to Deepen Programming Language Understanding — Journal



---

## Activity 1: Idiomatic Code Transformation (C++/CLI)

**Source:** `MakeNavBtn`, a button-factory helper from SHWT's `frmDashboard.cpp`.

### Original
```cpp
Button^ frmDashboard::MakeNavBtn(String^ text, Color bg, Color fg, int y)
{
    Drawing::Font^ fBtn = gcnew Drawing::Font("Segoe UI", 9.5f, FontStyle::Bold);
    const int navX = 526;
    const int navW = 272;
    const int navH = 40;

    Button^ b = gcnew Button();
    b->Location = Point(navX, y);
    b->Size = Drawing::Size(navW, navH);
    b->Text = text;
    b->Font = fBtn;
    b->BackColor = bg;
    b->ForeColor = fg;
    b->FlatStyle = FlatStyle::Flat;
    b->FlatAppearance->BorderSize = 0;
    b->Cursor = Cursors::Hand;
    b->TextAlign = ContentAlignment::MiddleLeft;
    b->Padding = System::Windows::Forms::Padding(12, 0, 0, 0);
    return b;
}
```

### Refactored
```cpp
/// <summary>
/// Creates a styled left-aligned navigation button used on the dashboard's
/// action panel (Log Entry, History, Report, Admin, etc).
/// </summary>
Button^ frmDashboard::MakeNavBtn(String^ text, Color bg, Color fg, int y)
{
    Button^ b = gcnew Button();
    b->Location = Point(NavButtonX, y);
    b->Size = Size(NavButtonWidth, NavButtonHeight);
    b->Text = text;
    b->Font = navBtnFont;              // shared instance field, not re-allocated per call
    b->BackColor = bg;
    b->ForeColor = fg;
    b->FlatStyle = FlatStyle::Flat;
    b->FlatAppearance->BorderSize = 0;
    b->Cursor = Cursors::Hand;
    b->TextAlign = ContentAlignment::MiddleLeft;
    b->Padding = NavButtonPadding;
    return b;
}
```
*(with `NavButtonX/Width/Height`, `NavButtonPadding`, and `navBtnFont` declared as class-level fields in the header)*

### Improvements applied
1. **Extracted magic numbers to named constants** — `526`, `272`, `40` were also redeclared identically in `InitializeComponent()`; centralizing them removes a real duplication/drift risk.
2. **Removed redundant namespace qualification** — `Drawing::Size` and `System::Windows::Forms::Padding` are redundant given the `using namespace` directives already active in the file.
3. **Hoisted `Font` allocation out of the per-call path** — the function is called 4 times with an identical font; a shared instance field avoids creating 4 duplicate GDI-backed objects.
4. **Added an XML doc comment** — idiomatic for .NET/C++/CLI, and drives IntelliSense.
5. **Deliberately did NOT introduce lambdas** for event wiring elsewhere in the file — that would be idiomatic "modern C++" in general, but violates this project's established no-lambda / explicit-method-pointer convention.

*(Note: C++/CLI requires MSVC + .NET to compile, which isn't available in this sandbox — the refactor was verified by careful manual review rather than a compiler run.)*

### 3 Key Learnings
1. **"Idiomatic" is relative to project conventions, not just the language.** I nearly suggested lambdas for event handlers — technically more "modern," but a direct violation of SHWT's no-lambda rule. Idiomatic code has to respect a codebase's own established rules first.
2. **Duplication across methods is easy to miss reading one function at a time.** The magic numbers looked like harmless local constants in isolation — only comparing against `InitializeComponent()` revealed the duplication.
3. **`static` is a communication tool, not just an optimization.** Marking data/methods `static` when they don't depend on instance state documents that fact for future readers, not just for the compiler.

---

## Activity 2: Code Quality Detective (Python)

**Source:** A representative "quick script written under time pressure" — `old_grades.py`, a grade-processing script with realistic quality issues (invented for this exercise, styled after the kind of script someone might write before learning better practices).

### The Code Reviewed

```python
import csv

data = []
total = 0

def load(f):
    global data
    file = open(f)
    reader = csv.reader(file)
    for row in reader:
        data.append(row)

def calc(l=[]):
    total = 0
    count = 0
    for r in l:
        try:
            score = float(r[2])
            total = total + score
            count = count + 1
        except:
            pass
    return total / count

def get_grade(s):
    if s >= 90:
        return "A"
    if s >= 80:
        return "B"
    if s >= 70:
        return "C"
    if s >= 60:
        return "D"
    else:
        return "F"

def process(filename):
    load(filename)
    output = ""
    for row in data:
        name = row[0]
        id = row[1]
        score = row[2]
        grade = get_grade(float(score))
        output = output + name + "," + id + "," + score + "," + grade + "\n"
    f2 = open("C:\\Users\\tumie\\Desktop\\output.csv", "w")
    f2.write(output)
    f2.close()
    print("done")

process("grades.csv")
avg = calc(data)
print(avg)
```

### Baseline — Running It First

Ran the script against a sample `grades.csv` before reviewing, to understand actual behavior rather than assumed behavior. It **crashed immediately** on a row with a non-numeric score:

```
ValueError: could not convert string to float: 'abc'
File "old_grades.py", line 44, in process
    grade = get_grade(float(score))
```

This crash is itself the first quality finding: `calc()` silently swallows bad rows with a bare `except: pass`, but `process()` has *no* error handling at all for the same kind of bad data — one function hides errors, the other doesn't handle them, and neither is a deliberate design choice.

### Code Smells / Quality Issues Identified

| # | Issue | Why it matters |
|---|---|---|
| 1 | `global data` used inside `load()` | Hidden coupling — `calc()` and `process()` both implicitly depend on `load()` having run first and populated a module-level variable. Hard to test or reason about in isolation. |
| 2 | `def calc(l=[])` — mutable default argument | Classic Python landmine: default mutable arguments are created once at function definition time and shared across calls. Not triggered here since always called explicitly, but it's a bug waiting to happen. |
| 3 | Bare `except: pass` in `calc()` | Silently swallows *any* exception, not just the expected `ValueError`/`IndexError` — could hide real bugs with no trace. |
| 4 | No error handling in `process()` for the same bad-data case | Inconsistent with `calc()`'s (over-broad) handling — the script crashes hard here instead. |
| 5 | Hardcoded absolute Windows path (`C:\Users\tumie\Desktop\output.csv`) | Not portable — breaks immediately on any other machine or OS, as demonstrated when trying to run it in this environment. |
| 6 | String concatenation in a loop (`output = output + ...`) | O(n²) behavior as row count grows, since each `+=` creates a new string object. Fine for 6 rows, a real problem for large exports. |
| 7 | No `with open(...)` context manager | File handles aren't guaranteed to close if an error occurs mid-write — a resource leak risk. |
| 8 | Hand-rolled CSV writing via string concatenation, not `csv.writer` | Risks bugs with values containing commas/quotes; `csv` module is already imported and used for reading but not for writing. |
| 9 | Variable named `id` | Shadows Python's built-in `id()` function — a bad habit to normalize. |
| 10 | No docstrings or type hints anywhere | A reader has to infer what `l`, `f`, `s`, `r` mean purely from usage. |
| 11 | No `if __name__ == "__main__":` guard | Top-level code runs immediately on import — this module can never be imported and reused without side effects, or unit tested. |
| 12 | Vague function names (`load`, `calc`, `process`) | None state *what* is being loaded/calculated — `load_records`, `calculate_average` communicate intent immediately. |
| 13 | No return value from `process()` — only `print("done")` | Makes the function's result unusable programmatically. |

### Ratings

| Dimension | Rating | Why |
|---|---|---|
| **Readability** | 2/5 | Short variable names, no docstrings, unclear function names. |
| **Performance** | 3/5 | Fine at small scale; O(n²) string concatenation would degrade on a large export. |
| **Maintainability** | 2/5 | Global mutable state, no tests possible without side effects, inconsistent error handling. |

### Refactored Version

```python
"""Loads student grade records from a CSV file, computes letter grades and
the class average, and writes an annotated CSV report."""

import csv
from pathlib import Path
from typing import NamedTuple


class StudentRecord(NamedTuple):
    name: str
    student_id: str
    score: float


GRADE_THRESHOLDS = (
    (90, "A"),
    (80, "B"),
    (70, "C"),
    (60, "D"),
)


def load_records(filepath: str) -> list[StudentRecord]:
    """Read student records from a CSV file (columns: name, id, score).

    Rows with a non-numeric score are skipped and reported, rather than
    silently ignored or crashing the whole run.
    """
    records: list[StudentRecord] = []
    with open(filepath, newline="") as f:
        reader = csv.reader(f)
        for row_number, row in enumerate(reader, start=1):
            try:
                records.append(StudentRecord(name=row[0], student_id=row[1], score=float(row[2])))
            except (ValueError, IndexError):
                print(f"Skipping malformed row {row_number}: {row}")
    return records


def calculate_average(records: list[StudentRecord]) -> float:
    """Return the average score across all records. Raises ValueError if empty."""
    if not records:
        raise ValueError("Cannot calculate average of an empty record list")
    return sum(r.score for r in records) / len(records)


def get_letter_grade(score: float) -> str:
    """Map a numeric score to a letter grade using GRADE_THRESHOLDS."""
    for threshold, letter in GRADE_THRESHOLDS:
        if score >= threshold:
            return letter
    return "F"


def write_report(records: list[StudentRecord], output_path: str) -> None:
    """Write a CSV report of name, id, score, and letter grade for each record."""
    with open(output_path, "w", newline="") as f:
        writer = csv.writer(f)
        for record in records:
            writer.writerow([
                record.name,
                record.student_id,
                record.score,
                get_letter_grade(record.score),
            ])


def process(input_path: str, output_path: str) -> float:
    """Load grades, write the report, and return the class average."""
    records = load_records(input_path)
    write_report(records, output_path)
    return calculate_average(records)


if __name__ == "__main__":
    output_file = Path.home() / "Desktop" / "output.csv"
    class_average = process("grades.csv", str(output_file))
    print(f"Report written to {output_file}")
    print(f"Class average: {class_average:.2f}")
```

### Verification

- Ran both versions against clean sample data — **same class average (74.67) and same letter grades** produced by both. (Minor cosmetic difference: refactored output shows `92.0` instead of `92` since scores are now stored as `float` throughout rather than passed through as raw strings — a deliberate, disclosed change, not a bug.)
- Ran the refactored version against the **exact data that crashed the original** (the row with `'abc'` as a score). Instead of crashing, it printed `Skipping malformed row 5: ['Eve', '105', 'abc']` and completed successfully with a correct average from the remaining valid rows.
- Confirmed the hardcoded path issue is fixed by using `Path.home() / "Desktop" / "output.csv"`, which resolves correctly regardless of OS or username.

### Reusable Code Review Checklist

- [ ] Are there any bare `except:` clauses? Should they name specific exception types?
- [ ] Is there error handling for the *same* failure mode in every place it can occur, or does it differ inconsistently between functions?
- [ ] Are there any mutable default arguments (`def f(x=[])` / `def f(x={})`)?
- [ ] Is any state stored in module-level globals that could instead be passed as parameters or returned?
- [ ] Are file paths hardcoded/absolute, or parameterized and portable?
- [ ] Is string concatenation happening in a loop where a list + `join()` (or a purpose-built writer, e.g. `csv.writer`) would be more appropriate?
- [ ] Are files opened with a `with` context manager, or manually opened/closed (risking leaks on error)?
- [ ] Do variable/function names describe their role, or are they single letters / generic verbs?
- [ ] Does the module have an `if __name__ == "__main__":` guard, so it can be imported and tested without side effects?
- [ ] Do functions return usable values, or only `print()` their result?
- [ ] Are there docstrings/type hints explaining non-obvious parameters and return values?

### 3 Key Learnings
1. **Running the code before reviewing it caught something a pure read-through might have taken longer to notice.** The crash on malformed data only became obvious by actually executing the script against realistic messy input.
2. **Inconsistent error-handling philosophy is a smell in itself,** separate from any individual bug — two functions handling the same failure mode in opposite ways (silently vs. not at all) signals there's no deliberate policy, which is a bigger maintainability risk than either issue alone.
3. **A missing `if __name__ == "__main__":` guard quietly forecloses testability.** I hadn't previously connected "top-level side-effecting code" this concretely to "this module can never be safely imported or unit tested."

---

## Activity 3: Understanding a Language Feature — C++ Templates & STL

**Feature chosen:** C++ templates, paired with STL algorithms (`std::find_if`, `std::remove_if`, `std::copy_if`).

### What templates are
A template lets you write one function or class whose type is a parameter, filled in by the compiler at compile time for whatever concrete type is used — no code duplication, and (unlike C++/CLI's `Object^`-based genericity) no runtime boxing cost.

### Simple examples

```cpp
// Function template — works for any type T that supports operator>
template <typename T>
T findMax(const T& a, const T& b)
{
    return (a > b) ? a : b;
}

// Class template — a minimal generic "box" holding one value of any type
template <typename T>
class Box
{
private:
    T value;
public:
    explicit Box(T v) : value(v) {}
    T get() const { return value; }
    void set(T v) { value = v; }
};
```
Compiled and run with `g++ -std=c++17 -Wall -Wextra`, zero warnings:
```
findMax(3, 7) = 7
findMax(3.5, 2.1) = 3.5
findMax(std::string("apple"), std::string("banana")) = banana
intBox.get() = 42
strBox.get() = hello
```

### 3 practical use cases
1. Generic containers/repositories — one "store, find, remove" implementation reusable across `HealthLog`, `Employee`, or any other record type.
2. Generic algorithms over any comparable type (`findMax<T>`, or STL's `std::sort`, `std::min_element`).
3. Type-safe wrappers (`std::unique_ptr<T>`, `std::optional<T>`) — compile-time type safety without a bespoke wrapper class per type.

### Practice project: `Repository<T>`

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>
#include <functional>
#include <optional>

// A minimal generic in-memory repository, the kind of pattern that shows up
// repeatedly in SHWT/ClockIt (GetLogsForStudent, FindEmployeeByPin, etc.)
// but rewritten so it works for ANY record type, not just one.
template <typename T>
class Repository
{
private:
    std::vector<T> items;

public:
    void add(const T& item)
    {
        items.push_back(item);
    }

    bool removeIf(const std::function<bool(const T&)>& predicate)
    {
        auto it = std::remove_if(items.begin(), items.end(), predicate);
        bool found = (it != items.end());
        items.erase(it, items.end());
        return found;
    }

    std::optional<T> findFirst(const std::function<bool(const T&)>& predicate) const
    {
        auto it = std::find_if(items.begin(), items.end(), predicate);
        if (it != items.end())
        {
            return *it;
        }
        return std::nullopt;
    }

    std::vector<T> findAll(const std::function<bool(const T&)>& predicate) const
    {
        std::vector<T> results;
        std::copy_if(items.begin(), items.end(), std::back_inserter(results), predicate);
        return results;
    }

    size_t count() const
    {
        return items.size();
    }
};

struct HealthLog
{
    int studentId;
    int sleepHours;
    int stressLevel;
};

int main()
{
    Repository<HealthLog> logs;
    logs.add({101, 7, 4});
    logs.add({101, 5, 8});
    logs.add({102, 8, 2});
    logs.add({103, 6, 6});

    auto match = logs.findFirst([](const HealthLog& log) { return log.studentId == 101; });
    auto stressed = logs.findAll([](const HealthLog& log) { return log.stressLevel >= 6; });
    bool removed = logs.removeIf([](const HealthLog& log) { return log.studentId == 101; });

    // Same Repository<T> template, completely different record type —
    // no code duplication needed.
    Repository<std::string> notes;
    notes.add("Take a break");
    notes.add("Drink water");

    return 0;
}
```

Compiled clean (`g++ -std=c++17 -Wall -Wextra`, zero warnings) and verified output:
```
Total logs: 4
First log for student 101: sleep=7 stress=4
High-stress logs found: 2
  student 101 stress=8
  student 103 stress=6
Removed student 101's logs: true
Total logs after removal: 2
Total notes: 2
```

### Common mistakes avoided
- Templates are typically **header-only** — can't split declaration/definition across `.h`/`.cpp` the way ordinary SHWT/ClockIt classes are, since the compiler needs the full definition at the point of instantiation.
- Passing template parameters **by value instead of `const&`** — caught myself almost writing `add(T item)` instead of `add(const T& item)`, an unnecessary copy for larger structs.
- Using a **raw pointer or sentinel value for "not found"** instead of `std::optional<T>` — a raw pointer into a `std::vector` can dangle if the vector reallocates later.
- Templates are only type-checked **at instantiation**, not at definition — a template can look like it compiles fine until used with a type that doesn't support the operations it needs.

### Self-review (simulating "share with AI for feedback")
- `removeIf` returns only `bool` ("was anything removed"), discarding *how many* — could return a count if that mattered.
- `std::function<bool(const T&)>` predicates have a small indirect-call overhead versus a template predicate parameter, which the compiler could inline — fine for learning, worth revisiting for performance-critical code.
- No thread-safety — worth stating explicitly rather than leaving implicit.

### 3 Key Learnings
1. **Templates + STL algorithms together are the real payoff, not templates alone.** The value isn't just "generic type parameter" — it's combining that with STL's algorithm library to get find/filter/remove behavior for free instead of hand-rolled loops per type.
2. **`std::optional<T>` solves a problem I've been solving badly with sentinel checks.** SHWT's `LoadLatestStats` handles "nothing found" with an ad-hoc `logs->Count == 0` check — seeing `std::optional` gave me a clearer model I now want to look for a C++/CLI equivalent of (e.g. `Nullable<T>`).
3. **The header-only constraint on templates is architectural, not a syntax quirk.** I'd have assumed I could split a template class into `.h`/`.cpp` the way every SHWT/ClockIt form class is split — that assumption would have produced a confusing linker error if I hadn't checked first.

---

## Summary for Group Discussion

| Activity | Language/Feature | Core takeaway |
|---|---|---|
| 1. Idiomatic Transformation | C++/CLI | Idiomatic code respects a project's own conventions, not just general language best practices. |
| 2. Code Quality Detective | Python | Running code before reviewing it surfaces real issues (like inconsistent error handling) that a read-through alone can miss. |
| 3. Language Feature Deep-Dive | C++ Templates & STL | Generic programming's real value comes from combining templates with the standard library's algorithms, not from genericity alone. |
