# Performance Optimization Challenge — Slow Code Analysis (Python)


**Exercise:** Performance Optimization Challenge

**Scenario chosen:** #1 — Slow Code Analysis (Python product combination finder)

---

## 1. Original Code and Context

```python
def find_product_combinations(products, target_price, price_margin=10):
    results = []
    for i in range(len(products)):
        for j in range(len(products)):
            if i != j:
                product1 = products[i]
                product2 = products[j]
                combined_price = product1['price'] + product2['price']
                if (target_price - price_margin) <= combined_price <= (target_price + price_margin):
                    if not any(r['product1']['id'] == product2['id'] and
                               r['product2']['id'] == product1['id'] for r in results):
                        pair = {
                            'product1': product1,
                            'product2': product2,
                            'combined_price': combined_price,
                            'price_difference': abs(target_price - combined_price)
                        }
                        results.append(pair)
    results.sort(key=lambda x: x['price_difference'])
    return results
```

**Context:** Used in an e-commerce app to suggest product pairs matching a target price. Processes 5,000+ products, runs on every visit to the "Product Recommendations" page, reportedly taking 20–30 seconds. Python 3.9, 4GB RAM web server.

---

## 2. Applying the "Slow Code Analysis" Prompt

**Why this is slow, in simple terms:** The function checks every possible *ordered* pair of products (comparing A-then-B and B-then-A separately), then for every candidate pair that matches the price range, scans the *entire growing results list so far* to check whether it's a duplicate of an already-found pair.

**Specific patterns causing the slowdown:**
- **O(n²) double loop** comparing every product to every other product in both orders — 2x the necessary comparisons.
- **The duplicate-check `any(...)` runs inside the innermost loop**, scanning the entire `results` list so far for every candidate pair. Since `results` itself grows as the loop progresses, this compounds the cost far beyond plain O(n²) whenever many pairs match — effectively closer to O(matches²) on top of the base O(n²) scan.
- **No pre-sorting** — since matching is based on a simple price sum, sorting first would enable a much smarter two-pointer search instead of brute force.

**Suggested improvements:**
1. Loop `j` only from `i+1` onward — eliminates self-comparison *and* duplicate pairs by construction, removing the need for the expensive `any()` check entirely.
2. Sort products by price first, then use a two-pointer technique to find qualifying pairs in O(n log n).
3. (A natural consequence of #1) — never generate a duplicate in the first place, rather than generating and filtering afterward.

**Underlying performance concepts:** Big-O complexity of nested loops; recognizing that anything *inside* a loop which itself scans a growing collection compounds the cost further; "check as you go" vs. "check after the fact" loop restructuring; when sorting first unlocks a fundamentally cheaper algorithm (two-pointer).

**Tools to measure actual bottlenecks:** `cProfile` (used below), `time.time()` before/after sections, `line_profiler` for line-level granularity.

---

## 3. Implementation

Three versions were implemented and benchmarked:

1. **Original** — as provided, unchanged.
2. **Optimized (`j = i+1`)** — same O(n²) shape, but eliminates the duplicate-check entirely by construction.
3. **Two-pointer** — sorts products by price first, then uses a two-pointer scan for O(n log n) overall complexity.

```python
def find_product_combinations_optimized(products, target_price, price_margin=10):
    results = []
    n = len(products)
    for i in range(n):
        product1 = products[i]
        for j in range(i + 1, n):
            product2 = products[j]
            combined_price = product1['price'] + product2['price']
            if (target_price - price_margin) <= combined_price <= (target_price + price_margin):
                results.append({
                    'product1': product1,
                    'product2': product2,
                    'combined_price': combined_price,
                    'price_difference': abs(target_price - combined_price)
                })
    results.sort(key=lambda x: x['price_difference'])
    return results


def find_product_combinations_two_pointer(products, target_price, price_margin=10):
    sorted_products = sorted(products, key=lambda p: p['price'])
    n = len(sorted_products)
    results = []

    low, high = 0, n - 1
    while low < high:
        combined_price = sorted_products[low]['price'] + sorted_products[high]['price']

        if combined_price < target_price - price_margin:
            low += 1
        elif combined_price > target_price + price_margin:
            high -= 1
        else:
            left = low
            while left < high:
                cp = sorted_products[left]['price'] + sorted_products[high]['price']
                if (target_price - price_margin) <= cp <= (target_price + price_margin):
                    results.append({
                        'product1': sorted_products[left],
                        'product2': sorted_products[high],
                        'combined_price': cp,
                        'price_difference': abs(target_price - cp)
                    })
                    left += 1
                else:
                    break
            high -= 1

    results.sort(key=lambda x: x['price_difference'])
    return results
```

---

## 4. Performance Measurements

**Note on methodology:** the original algorithm's duplicate-check bug makes it far more expensive than a plain O(n²) scan when many pairs match. With this dataset (random prices 5–500, target 500 ± 50, near the peak of the combined-price distribution), a very large fraction of pairs qualify, so the original could not complete at the full 5,000-product scale within a reasonable time limit — it was benchmarked at n=500 instead, with results extrapolated for context.

### Direct comparison at n=500 (all three, same random seed, identical results verified)

| Version | Time | Pairs found |
|---|---|---|
| Original | 31.762s | 24,041 |
| Optimized (`j = i+1`) | 0.053s | 24,041 |
| Two-pointer | 0.037s | 24,041 |

**Speedup: 604.5x (optimized) / 865.0x (two-pointer) over the original, at n=500.**

### Full-scale test at n=5,000 (optimized versions only — original could not complete in reasonable time)

| Version | Time | Pairs found |
|---|---|---|
| Optimized (`j = i+1`) | 6.93s | 2,432,833 |
| Two-pointer | 5.55s | 2,432,833 |

Both optimized implementations agree exactly on pair counts at every tested size, confirming correctness of the rewrite.

### Profiling the original (n=400, to keep runtime manageable)

```
243610059 function calls in 46.116 seconds

ncalls   tottime  cumtime  filename:lineno(function)
1        0.132    46.116   find_product_combinations_original
31210    19.394    45.966   {built-in method builtins.any}
243531630 26.572   26.572   <genexpr> (the duplicate-check generator)
15605    0.008     0.008    {built-in method builtins.abs}
```

This confirms the root cause directly: for only 400 products, the duplicate-check `any()` call and its inner generator expression account for essentially **all 46 seconds** of runtime — 243.5 million generator evaluations, dwarfing every other operation in the function combined.

---

## 5. Optimization Process Summary

1. Applied the "Slow Code Analysis" prompt to the original code and context, receiving a hypothesis that the O(n²) double loop plus an even more expensive duplicate-check were the primary bottlenecks.
2. Implemented the suggested `j = i+1` fix, which eliminates the duplicate-check requirement entirely by construction.
3. Implemented a further two-pointer optimization (sort-first) as a stretch improvement beyond the prompt's minimum suggestion.
4. Benchmarked all three versions at a common size (n=500) to get a direct, fair comparison, since the original could not complete at the full 5,000-product scale in reasonable time.
5. Verified correctness by confirming all three versions produce identical pair counts at every tested size.
6. Used `cProfile` to confirm, with real data, that the duplicate-check `any()` call — not the double loop itself — was responsible for nearly all the original's runtime.

---

## 6. Key Learnings

- **A double loop isn't always the real bottleneck** — here, the O(n²) loop structure was actually the *smaller* problem; the O(n)-scan-inside-an-O(n²)-loop duplicate check was the dominant cost, and only profiling revealed this precisely (46 of 46 seconds, 243M calls, all in one line).
- **"Check as you go" beats "check after the fact"** — restructuring the loop bounds (`j = i+1`) to make duplicates structurally impossible was far more effective than any attempt to make the duplicate-check itself faster.
- **A performance bug can be far worse than its Big-O class suggests** — the actual behavior here was closer to O(matches²) rather than a clean O(n²), because the growing `results` list is scanned repeatedly. High match density (from the specific price distribution and margin used) made this dramatically worse than a naive O(n²) estimate would predict.
- **Correctness must be verified alongside speed** — confirming all three versions returned identical pair counts at every tested size was essential to trust the speedup numbers rather than just accepting a faster-but-wrong result.
- **Profiling beats intuition for locating bottlenecks** — the prompt's hypothesis (duplicate-check is expensive) was correct, but the *scale* of the effect (99.9%+ of runtime in one line) was only clear from actually running `cProfile`, not from reading the code alone.
