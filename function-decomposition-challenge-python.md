# Function Decomposition Challenge (Python)

**Selected function:** `generate_sales_report` — Report Generation Function with Multiple Data Transformations (Python)
---

## 1. The Original Function

`generate_sales_report(sales_data, report_type, date_range, filters, grouping, include_charts, output_format)` is a 266-line function that builds a sales report in one of three shapes (`summary`, `detailed`, `forecast`), optionally filtered by date range and arbitrary field filters, optionally grouped by a field, optionally including chart data, and rendered in one of four output formats.

All of the following live inside the single function body:

- Input parameter validation
- Date-range validation and filtering
- Arbitrary field-based filtering
- Empty-result handling
- Basic metric calculation (total / average / max / min)
- Grouping and per-group aggregation
- Assembly of the base report structure
- Detailed-report transaction enrichment (pre-tax, profit, margin)
- Forecast calculation (monthly totals, growth rates, 3-month projection)
- Chart data assembly (time series + pie chart)
- Output-format dispatch

---

## 2. Step 1 — Identifying Distinct Responsibilities

Prompted an AI tool to identify the distinct responsibilities in the function. It confirmed **11 separable concerns**, each varying independently of the others:

| # | Responsibility | Original location |
|---|---|---|
| 1 | Validate `sales_data`, `report_type`, `output_format` | top of function |
| 2 | Validate + parse `date_range`, filter data by it | `if date_range:` block |
| 3 | Apply arbitrary `filters` dict | `if filters:` block |
| 4 | Handle "no data after filtering" case | `if not sales_data:` block |
| 5 | Calculate total/average/max/min | metric calculation lines |
| 6 | Group data by field + per-group averages | `if grouping:` block (first one) |
| 7 | Build the `summary` section of the report | `report_data = {...}` construction |
| 8 | Build the `grouping` section (with percentages) | second `if grouping:` block |
| 9 | Build `detailed` transactions (pre_tax/profit/margin) | `if report_type == 'detailed':` block |
| 10 | Build `forecast` section (monthly totals, growth, projection) | `if report_type == 'forecast':` block |
| 11 | Build `charts` section (time series + pie chart) | `if include_charts:` block |
| 12 | Dispatch to output-format renderer | final `if output_format == ...` chain |

**Misconception I initially had:** I assumed "grouping" was a single responsibility, but it's actually two distinct things done in two separate places in the original code — grouping the raw data (#6) and building the report-facing grouping section with percentages (#8). These have different reasons to change (e.g. adding a new aggregate like "median" only touches #6; changing how percentages are displayed only touches #8), so I kept them as two separate helpers rather than merging them.

**A pre-existing bug worth flagging (not something I introduced):** in the forecast section, `growth_rates` only gets an entry appended when the *previous* month's total is greater than 0, but the dict comprehension that builds `'growth_rates'` in the final report assumes `growth_rates[i-1]` always lines up positionally with `sorted_months[i]`. If any month's total sale is exactly `0`, this mapping silently misaligns. I preserved this behavior exactly during refactoring (see Section 6) since the exercise goal was behavior-preserving decomposition, not bug-fixing — but I documented it clearly with a comment in the refactored code and flag it here for visibility.

---

## 3. Step 2 — Decomposition Plan

1. Extract one function per responsibility identified above (11 top-level helpers).
2. Each helper takes only the data it actually needs, not the full parameter list, so its purpose is clear from its signature (e.g. `_apply_filters(sales_data, filters)` rather than passing the whole options bag).
3. Each helper returns a clearly-named data structure (a filtered list, a metrics dict, a report section dict) so the main function's job becomes assembling those pieces.
4. The main function is reduced to: validate → filter → check empty → compute metrics → group → assemble sections conditionally → render output.
5. Preserve all original behavior exactly, including the pre-existing forecast-alignment quirk noted above — the exercise goal is decomposition, not correctness fixes.

---

## 4. Step 3 — Extracted Helper Functions

| Helper | Purpose |
|---|---|
| `_validate_inputs(sales_data, report_type, output_format)` | Validates the three required top-level parameters |
| `_filter_by_date_range(sales_data, date_range)` | Validates and applies date-range filtering |
| `_apply_filters(sales_data, filters)` | Applies arbitrary field filters (single value or list) |
| `_calculate_basic_metrics(sales_data)` | Computes total, average, max sale, min sale |
| `_group_sales_data(sales_data, grouping)` | Groups sales by a field, with per-group count/total/average |
| `_build_summary_section(sales_data, metrics)` | Builds the `summary` report block |
| `_build_grouping_section(grouped_data, grouping, total_sales)` | Builds the `grouping` report block with percentages |
| `_build_detailed_transactions(sales_data)` | Builds enriched transaction list (pre_tax/profit/margin) |
| `_build_forecast_section(sales_data)` | Builds monthly sales, growth rates, and 3-month projection |
| `_build_charts_section(sales_data, grouping, grouped_data)` | Builds time-series and pie-chart data |
| `_render_output(report_data, output_format, include_charts)` | Dispatches to the correct format renderer |

The four `_generate_*_report` stub functions (html/excel/pdf/empty) were left untouched — they were already separate, single-purpose functions in the original code.

---

## 5. Step 4 — Refactored Main Function

```python
def generate_sales_report(sales_data, report_type='summary', date_range=None,
                         filters=None, grouping=None, include_charts=False,
                         output_format='pdf'):
    _validate_inputs(sales_data, report_type, output_format)

    sales_data = _filter_by_date_range(sales_data, date_range)
    sales_data = _apply_filters(sales_data, filters)

    if not sales_data:
        print("Warning: No data matches the specified criteria")
        if output_format == 'json':
            return {"message": "No data matches the specified criteria", "data": []}
        else:
            return _generate_empty_report(report_type, output_format)

    metrics = _calculate_basic_metrics(sales_data)
    grouped_data = _group_sales_data(sales_data, grouping)

    report_data = {
        'report_type': report_type,
        'date_generated': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        'date_range': date_range,
        'filters': filters,
        'summary': _build_summary_section(sales_data, metrics)
    }

    if grouping:
        report_data['grouping'] = _build_grouping_section(
            grouped_data, grouping, metrics['total_sales']
        )

    if report_type == 'detailed':
        report_data['transactions'] = _build_detailed_transactions(sales_data)

    if report_type == 'forecast':
        report_data['forecast'] = _build_forecast_section(sales_data)

    if include_charts:
        report_data['charts'] = _build_charts_section(sales_data, grouping, grouped_data)

    return _render_output(report_data, output_format, include_charts)
```

The main function dropped from 266 lines to roughly 30, and now reads as a straight-line pipeline: **validate → filter → guard-empty → compute → assemble → render**.

*(Full helper implementations are in `sales_report_refactored.py`, included with this submission.)*

---

## 6. Step 5 — Verifying Behavior Is Preserved

**a) Original test suite, unchanged, run against the refactored module**
The project's existing 8 unittest cases (`test_sales_report.py`) were re-pointed at `sales_report_refactored.py` with no other changes.

```
Ran 8 tests in 0.007s
OK
```

**b) Differential testing — original vs. refactored, full output comparison**
Wrote `test_differential.py`, which runs **both** implementations side-by-side across 20 scenarios and asserts the entire returned dict is identical — not just spot-checked fields. Scenarios covered: all three report types, all grouping fields (category/region/customer), date-range filtering (with and without matches), single-value and list-value field filters, combined date-range + filters + grouping + detailed + charts, and all four input-validation error paths (bad sales_data, bad report_type, bad output_format, malformed/reversed date_range).

```
Ran 20 tests in 0.006s
OK
```

**c) Full combined run**

```
Ran 36 tests in 0.005s
OK
```

No behavior changed, including the pre-existing forecast growth-rate alignment quirk — the refactor is a pure decomposition, not a bug fix.

---

## 7. Benefits Gained

- **Readability:** the main function now reads like an outline of the report-generation pipeline instead of 266 lines of interleaved validation, calculation, and assembly logic.
- **Testability:** helpers like `_calculate_basic_metrics` or `_build_forecast_section` can now be unit-tested directly with a small sales list, instead of needing to invoke the entire report pipeline (including format rendering) to check one calculation.
- **Isolated risk:** a change to how forecasts are calculated now only touches `_build_forecast_section` — previously, any edit inside the giant function risked interacting with unrelated report-type branches sharing the same scope.
- **Reusability:** `_apply_filters` and `_group_sales_data` are now standalone and could be reused by a future function (e.g. a CSV export) without depending on the report-assembly logic.
- **Surfaced a hidden bug during decomposition:** isolating the forecast logic into its own function made the growth-rate index-alignment issue visible and documentable, where before it was buried in a 60-line block mixed with unrelated grouping and summary logic. This is a concrete example of decomposition improving code review, even without fixing the bug outright.

---

## 8. Files Included

- `sales_report.py` — original function (unchanged, kept for comparison)
- `sales_report_refactored.py` — refactored version with extracted helpers
- `test_sales_report.py` — original test suite (unchanged)
- `test_sales_report_refactored.py` — original test suite pointed at the refactored module
- `test_differential.py` — new differential test comparing full output of both implementations across 20 scenarios
