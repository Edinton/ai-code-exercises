# Understanding What to Change with AI

**Exercise:** Understanding What to Change with AI

**Languages covered:** Java (readability), Python (refactoring), JavaScript (duplication)

---

## Exercise 1: Code Readability Improvement (Java)

### Original Code

```java
class UserMgr {
    private List<U> u_list;
    private DBConn db;

    public UserMgr(DBConn d) {
        db = d;
        u_list = new ArrayList<>();
    }

    public boolean a(String un, String pw, String em) {
        if (un.length() < 3 || pw.length() < 8 || !em.contains("@")) {
            return false;
        }
        for (U user : u_list) {
            if (user.getUn().equals(un)) {
                return false;
            }
        }
        U nu = new U(un, pw, em);
        u_list.add(nu);
        boolean res = db.execute("INSERT INTO users VALUES ('" + un + "', '" + pw + "', '" + em + "')");
        return res;
    }

    public U f(String un) {
        for (U user : u_list) {
            if (user.getUn().equals(un)) {
                return user;
            }
        }
        return null;
    }
}

class U {
    private String un;
    private String pw;
    private String em;

    public U(String un, String pw, String em) {
        this.un = un;
        this.pw = pw;
        this.em = em;
    }

    public String getUn() { return un; }
    public String getPw() { return pw; }
    public String getEm() { return em; }
}
```

### AI Analysis (Prompt 1)

**Difficult-to-understand parts:**
- Nearly every identifier (`U`, `u_list`, `un`, `pw`, `em`, `db`, `a()`, `f()`) is a cryptic abbreviation — `a(un, pw, em)` gives no indication it registers a new user.
- `a()` does three unrelated things at once: validation, duplicate-checking, and persistence, with no separation.
- The SQL string is built via direct concatenation, burying intent (and introducing a SQL-injection risk) inside string-building syntax.

**Better names suggested:**

| Current | Suggested |
|---|---|
| `UserMgr` | `UserManager` |
| `U` | `User` |
| `u_list` | `users` |
| `db` | `database` |
| `un` / `pw` / `em` | `username` / `password` / `email` |
| `a(un, pw, em)` | `registerUser(username, password, email)` |
| `f(un)` | `findUserByUsername(username)` |
| `nu` | `newUser` |
| `res` | `insertSucceeded` |

**Breaking down complex sections:**
- Extract validation into `isValidRegistration(username, password, email)`.
- Extract the duplicate-username check into a reusable `isUsernameTaken(username)`.
- Separate persistence from business logic (e.g. a `UserRepository.insert(user)`), and use a parameterized query instead of string concatenation to remove the SQL-injection risk.

**Style/formatting issues:**
- Two classes in one file, against standard Java convention (one public class per file).
- Getter names (`getUn()`, etc.) don't match the suggested renamed fields.
- Cramped field declarations with no spacing, inconsistent with typical Java formatting.

**What I might have missed on my own:** the SQL-injection risk from string concatenation is easy to overlook when focused purely on "readability" — the fix for that doubled as a security improvement, showing naming/structure and security concerns aren't always cleanly separable in review.

---

## Exercise 2: Function Refactoring (Python)

### Original Code

```python
def process_orders(orders, inventory, customer_data):
    results = []
    total_revenue = 0
    error_orders = []

    for order in orders:
        item_id = order['item_id']
        quantity = order['quantity']
        customer_id = order['customer_id']

        if item_id not in inventory:
            error_orders.append({'order_id': order['order_id'], 'error': 'Item not in inventory'})
            continue
        if inventory[item_id]['quantity'] < quantity:
            error_orders.append({'order_id': order['order_id'], 'error': 'Insufficient quantity'})
            continue
        if customer_id not in customer_data:
            error_orders.append({'order_id': order['order_id'], 'error': 'Customer not found'})
            continue

        price = inventory[item_id]['price'] * quantity
        if customer_data[customer_id]['premium']:
            price = price * 0.9
        inventory[item_id]['quantity'] -= quantity

        shipping = 0
        if customer_data[customer_id]['location'] == 'domestic':
            if price < 50:
                shipping = 5.99
        else:
            shipping = 15.99

        tax = price * 0.08
        final_price = price + shipping + tax
        total_revenue += final_price

        result = {
            'order_id': order['order_id'], 'item_id': item_id, 'quantity': quantity,
            'customer_id': customer_id, 'price': price, 'shipping': shipping,
            'tax': tax, 'final_price': final_price
        }
        results.append(result)

    return {'processed_orders': results, 'error_orders': error_orders, 'total_revenue': total_revenue}
```

### AI Analysis (Prompt 2)

**Distinct responsibilities identified, all mixed into one function:**
1. Validation (item exists, quantity available, customer exists)
2. Pricing calculation (base price, premium discount, shipping, tax)
3. State mutation (decrementing inventory)
4. Aggregation (revenue total, results/errors lists)
5. Orchestration (looping and deciding outcome per order)

### Refactored Code

```python
def validate_order(order, inventory, customer_data):
    """Returns an error string if invalid, or None if the order is valid."""
    item_id = order['item_id']
    quantity = order['quantity']
    customer_id = order['customer_id']

    if item_id not in inventory:
        return 'Item not in inventory'
    if inventory[item_id]['quantity'] < quantity:
        return 'Insufficient quantity'
    if customer_id not in customer_data:
        return 'Customer not found'
    return None


def calculate_order_pricing(item_id, quantity, customer_id, inventory, customer_data):
    """Returns (price, shipping, tax, final_price) for a valid order."""
    price = inventory[item_id]['price'] * quantity

    if customer_data[customer_id]['premium']:
        price *= 0.9

    if customer_data[customer_id]['location'] == 'domestic':
        shipping = 5.99 if price < 50 else 0
    else:
        shipping = 15.99

    tax = price * 0.08
    final_price = price + shipping + tax

    return price, shipping, tax, final_price


def process_single_order(order, inventory, customer_data):
    """Processes one order: validates, prices, and updates inventory.
    Returns (result_dict, error_dict) -- exactly one will be None."""
    error = validate_order(order, inventory, customer_data)
    if error:
        return None, {'order_id': order['order_id'], 'error': error}

    item_id = order['item_id']
    quantity = order['quantity']
    customer_id = order['customer_id']

    price, shipping, tax, final_price = calculate_order_pricing(
        item_id, quantity, customer_id, inventory, customer_data
    )
    inventory[item_id]['quantity'] -= quantity

    result = {
        'order_id': order['order_id'], 'item_id': item_id, 'quantity': quantity,
        'customer_id': customer_id, 'price': price, 'shipping': shipping,
        'tax': tax, 'final_price': final_price
    }
    return result, None


def process_orders(orders, inventory, customer_data):
    """Orchestrates processing of a batch of orders."""
    results = []
    error_orders = []
    total_revenue = 0

    for order in orders:
        result, error = process_single_order(order, inventory, customer_data)
        if error:
            error_orders.append(error)
        else:
            results.append(result)
            total_revenue += result['final_price']

    return {
        'processed_orders': results,
        'error_orders': error_orders,
        'total_revenue': total_revenue
    }
```

**Improved organization and flow:** `process_orders` now reads as a clean high-level summary — loop, delegate, aggregate — with implementation detail pushed into named functions. Each helper (`validate_order`, `calculate_order_pricing`) is independently unit-testable with plain dictionaries, no need to construct a full batch.

**Other issues noticed:**
- Inventory is only mutated after all validations pass in the original — not a current bug, but a fragile invariant that could silently break if the function is ever reordered. The refactor's clear validate-then-mutate separation makes this much easier to preserve.
- No rollback if a later step were to fail after inventory is already decremented — out of scope for a pure structural refactor, but far more visible as a risk once the steps are separated.
- Magic numbers (`0.9`, `5.99`, `15.99`, `0.08`, `50`) should become named constants (e.g. `PREMIUM_DISCOUNT_RATE = 0.9`) so future rule changes are one-line edits instead of a search through the function body.

---

## Exercise 3: Code Duplication Detection (JavaScript)

### Original Code

```javascript
function calculateUserStatistics(userData) {
  let totalAge = 0;
  for (let i = 0; i < userData.length; i++) { totalAge += userData[i].age; }
  const averageAge = totalAge / userData.length;

  let totalIncome = 0;
  for (let i = 0; i < userData.length; i++) { totalIncome += userData[i].income; }
  const averageIncome = totalIncome / userData.length;

  let totalScore = 0;
  for (let i = 0; i < userData.length; i++) { totalScore += userData[i].score; }
  const averageScore = totalScore / userData.length;

  let highestAge = userData[0].age;
  for (let i = 1; i < userData.length; i++) {
    if (userData[i].age > highestAge) { highestAge = userData[i].age; }
  }

  let highestIncome = userData[0].income;
  for (let i = 1; i < userData.length; i++) {
    if (userData[i].income > highestIncome) { highestIncome = userData[i].income; }
  }

  let highestScore = userData[0].score;
  for (let i = 1; i < userData.length; i++) {
    if (userData[i].score > highestScore) { highestScore = userData[i].score; }
  }

  return {
    age: { average: averageAge, highest: highestAge },
    income: { average: averageIncome, highest: highestIncome },
    score: { average: averageScore, highest: highestScore }
  };
}
```

### AI Analysis (Prompt 3)

**Duplicated patterns identified:**
- **Sum-then-average** pattern, repeated 3 times (age, income, score), identical except for the property read.
- **Find-maximum** pattern, also repeated 3 times, same structure each time.

### Refactored Code

```javascript
function average(items, field) {
  const total = items.reduce((sum, item) => sum + item[field], 0);
  return total / items.length;
}

function highest(items, field) {
  return items.reduce((max, item) => item[field] > max ? item[field] : max, items[0][field]);
}

function calculateUserStatistics(userData) {
  const fields = ['age', 'income', 'score'];
  const stats = {};
  for (const field of fields) {
    stats[field] = {
      average: average(userData, field),
      highest: highest(userData, field)
    };
  }
  return stats;
}
```

**Benefits:**
- ~50 lines of repeated loops reduced to 2 small generic helpers plus a short driving loop (~4x reduction).
- Adding a 4th statistic becomes a one-line change (`fields.push('yearsEmployed')`) instead of copy-pasting two more loops.
- `average`/`highest` are now independently testable and reusable elsewhere.
- Bugs (e.g. handling an empty array) get fixed once, not three times.

**Readability trade-off for junior developers:** `reduce()`-based helpers are concise and DRY, but a plain `for` loop inside the same helper may be more approachable for developers less comfortable with functional array methods:

```javascript
function average(items, field) {
  let total = 0;
  for (const item of items) {
    total += item[field];
  }
  return total / items.length;
}
```
Functionally identical either way — a genuine style trade-off (concision vs. approachability) worth discussing with the team rather than assuming one is objectively correct.

---

## Reflection

Across all three exercises, the AI consistently surfaced issues beyond what the stated goal of each exercise implied: the "readability" prompt (Exercise 1) also caught a SQL-injection risk; the "refactoring" prompt (Exercise 2) also caught a fragile ordering invariant and magic numbers; the "duplication" prompt (Exercise 3) also raised a genuine team-communication question (concision vs. approachability) rather than declaring one style universally correct. The pattern worth taking away: these three concerns (readability, structure, duplication) overlap far more than the exercise's clean separation suggests — improving one often surfaces or fixes something in another category almost as a side effect.
