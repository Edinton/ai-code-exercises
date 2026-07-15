# Code Readability Challenge 

**Example 4:** Poor Formatting and Structure


**Language:** Python — Discount Calculator

---

## 1. The Original Code

```python
def discount(cart,promos,user):
    d=0;tot=0
    for i in cart:tot+=i['price']*i['quantity']
    for p in promos:
        if p['type']=='percent' and (p['min_purchase'] is None or tot>=p['min_purchase']):val=tot*p['value']/100;d=max(d,val)
        elif p['type']=='fixed' and (p['min_purchase'] is None or tot>=p['min_purchase']):val=min(p['value'],tot);d=max(d,val)
        elif p['type']=='shipping' and tot>=p['min_purchase']:user['free_shipping']=True
    if user['status']=='vip':vd=tot*0.05;d=max(d,vd)
    elif user['status']=='member' and user['months']>6:vd=tot*0.02;d=max(d,vd)
    return {'original':tot,'discount':d,'final':tot-d,'free_shipping':user.get('free_shipping',False)}
```

Semicolon-chained statements, single-letter variables (`d`, `tot`, `p`, `i`, `val`, `vd`), and an 8-line function doing four unrelated jobs — cart totaling, promo evaluation, loyalty discounting, and result assembly — all crammed into deeply nested one-liners.

---

## 2. Baseline — Running the Original Tests First

Before touching anything, I ran the provided test suite against the original code to confirm my understanding of its behavior:

```
Ran 6 tests in 0.001s
OK
```

All 6 tests passed: percentage discount, fixed discount, free shipping, VIP discount, member discount, and best-discount-selection (where the largest of several eligible discounts wins).

---

## 3. Restructuring Applied

Used an AI prompt targeted at formatting/structure to break the single function into 4 single-responsibility functions, plus one small private helper to remove duplicated logic:

- **`calculate_cart_total(cart)`** — sums price × quantity across the cart
- **`_meets_minimum_purchase(promo, cart_total)`** — private helper that removes the duplicated `min_purchase is None or ...` condition that appeared twice in the original
- **`apply_promotions(promos, cart_total, user)`** — evaluates percent/fixed/shipping promos, returns the best promo discount found
- **`calculate_loyalty_discount(user, cart_total)`** — VIP/member loyalty discount logic
- **`discount(cart, promos, user)`** — orchestrates the above and assembles the final result dict

---

## 4. Refactored Code

```python
def calculate_cart_total(cart):
    """Sum price * quantity across every item in the cart."""
    total = 0
    for item in cart:
        total += item['price'] * item['quantity']
    return total


def _meets_minimum_purchase(promo, cart_total):
    """A promo with no min_purchase set applies unconditionally."""
    return promo['min_purchase'] is None or cart_total >= promo['min_purchase']


def apply_promotions(promos, cart_total, user):
    """
    Evaluate all promotions against the cart total and return the single
    best (largest) discount amount found among 'percent' and 'fixed' promos.

    'shipping' promos don't produce a discount amount; instead they set
    user['free_shipping'] = True as a side effect when the cart total meets
    the promo's minimum purchase requirement.

    NOTE: unlike the 'percent' and 'fixed' branches, the original 'shipping'
    branch does not treat a missing min_purchase as "always applies" — it
    directly compares tot >= p['min_purchase']. This is preserved as-is
    (behavior-only refactor), but is worth fixing separately: a shipping
    promo with min_purchase=None would raise a TypeError in Python 3.
    """
    best_discount = 0

    for promo in promos:
        if promo['type'] == 'percent' and _meets_minimum_purchase(promo, cart_total):
            discount_value = cart_total * promo['value'] / 100
            best_discount = max(best_discount, discount_value)

        elif promo['type'] == 'fixed' and _meets_minimum_purchase(promo, cart_total):
            discount_value = min(promo['value'], cart_total)
            best_discount = max(best_discount, discount_value)

        elif promo['type'] == 'shipping' and cart_total >= promo['min_purchase']:
            user['free_shipping'] = True

    return best_discount


def calculate_loyalty_discount(user, cart_total):
    """
    Calculate a discount based on user loyalty status:
      - VIP users get 5% off.
      - Members with more than 6 months tenure get 2% off.
      - Everyone else gets no loyalty discount.
    """
    if user['status'] == 'vip':
        return cart_total * 0.05
    elif user['status'] == 'member' and user['months'] > 6:
        return cart_total * 0.02
    return 0


def discount(cart, promos, user):
    """
    Calculate the final price for a shopping cart after applying the best
    available discount (from either promotions or user loyalty status) and
    determining free-shipping eligibility.

    Parameters:
        cart (list[dict]): items with 'price' and 'quantity' keys.
        promos (list[dict]): promotions, each with 'type' ('percent' | 'fixed'
            | 'shipping'), 'value', and 'min_purchase' (or None for no minimum).
        user (dict): must include 'status' ('regular' | 'member' | 'vip') and
            'months' (tenure in months, used for member discounts). May be
            mutated in place: 'free_shipping' is set to True if a shipping
            promo applies.

    Returns:
        dict with keys:
            'original' - cart total before any discount
            'discount' - the single best discount amount applied (promo or loyalty, whichever is larger)
            'final' - original minus discount
            'free_shipping' - whether free shipping was earned
    """
    cart_total = calculate_cart_total(cart)

    best_promo_discount = apply_promotions(promos, cart_total, user)
    loyalty_discount = calculate_loyalty_discount(user, cart_total)
    best_discount = max(best_promo_discount, loyalty_discount)

    return {
        'original': cart_total,
        'discount': best_discount,
        'final': cart_total - best_discount,
        'free_shipping': user.get('free_shipping', False)
    }
```

---

## 5. Verification

Re-ran the **exact same, unmodified** test file against the refactored code:

```
Ran 6 tests in 0.001s
OK
```

All 6 original tests pass unchanged — percentage discount, fixed discount, free shipping, VIP discount, member discount, and best-discount-selection all produce identical results to the original implementation.

---

## 6. A Latent Inconsistency the Readability Pass Surfaced

While extracting `apply_promotions`, it became visible that the `shipping` promo branch handles `min_purchase` **differently** from the `percent`/`fixed` branches: those two treat `min_purchase: None` as "always applies" via an explicit `is None or ...` check, but the `shipping` branch does a bare `tot >= p['min_purchase']` comparison — which would raise a `TypeError` in Python 3 if a shipping promo had `min_purchase: None`.

No existing test exercises this path, and it was easy to miss in the original single-line `elif` chain. I preserved the original behavior exactly (per the exercise's scope — behavior-preserving readability refactor, not a bug fix) but documented the inconsistency clearly in a comment, since it's the kind of thing a teammate should know about before writing a new test that trips over it.

---

## 7. Reflection

**How much easier is the code to understand now?**
Substantially — the original required tracing through semicolon-chained, deeply nested conditionals on single lines to understand what was happening. The refactored version reads top-to-bottom as: total the cart → check promos → check loyalty status → take the best discount → assemble result.

**What readability issues did AI catch that I might have missed?**
The duplicated `min_purchase is None or ...` condition appearing in both the `percent` and `fixed` branches — pulling that into `_meets_minimum_purchase` wasn't just a formatting fix, it removed real duplication.

**What did I notice that a purely mechanical reformat might miss?**
The `shipping` branch's inconsistent `min_purchase` handling. A simple "add whitespace and line breaks" pass wouldn't surface this — it only became obvious once the promo-evaluation logic was isolated into its own function and each `elif` branch could be compared side-by-side.

**Which improvement had the biggest impact?**
Function decomposition. Splitting one 8-line tangle into 4 named functions immediately made the code's structure (total → promos → loyalty → assemble) legible without reading a single line of implementation detail.

**How did the improved structure change understanding of the code's purpose?**
Before: an undifferentiated block computing "a discount." After: a clear pipeline showing that the final discount is actually the *best of two independent discount sources* (promotional and loyalty-based) — a distinction that was previously implicit in a shared `d = max(d, ...)` variable reused across unrelated conditions.

**Would breaking this into smaller functions help?**
Yes, clearly — beyond readability, `calculate_cart_total` and `calculate_loyalty_discount` are now independently unit-testable without constructing a full cart/promos/user scenario, which the original monolithic function didn't allow.

**Readability patterns to carry forward:**
1. Extract functions along responsibility boundaries, not arbitrary line counts — each of the 4 extracted functions here maps to a distinct business concept (totaling, promos, loyalty, assembly).
2. Pull out duplicated conditions into a named helper (`_meets_minimum_purchase`) — this also documents *why* the condition exists, not just *what* it checks.
3. When a readability pass surfaces an inconsistency (the shipping `min_purchase` bug), document it rather than silently fixing it, unless fixing it is explicitly in scope — this keeps the refactor's blast radius predictable and reviewable.
4. Always re-run the *original, unmodified* test file against refactored code — a passing test suite is the actual proof that a "readability-only" refactor didn't change behavior.

---

## Files Included

- `discount.py` — original function (unchanged, kept for comparison)
- `discount_refactored.py` — refactored version with extracted helpers
- `test_discount.py` — original test suite (unchanged)
- `test_discount_refactored.py` — original test suite pointed at the refactored module
