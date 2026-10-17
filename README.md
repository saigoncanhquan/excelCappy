# USER GUIDE: FIFO FUNCTION IN EXCELCAPPY
> **Calculate Inventory Issue & Balance Values using First-In, First-Out (FIFO) Method**  
> Compliant with International & Vietnamese Accounting Standards (VAS 02 / Circular 200) | Powered by Ultra-Fast Rust Native Binary Core

---

## 1. OVERVIEW & KEY HIGHLIGHTS

The `=FIFO()` function in **excelCappy** is powered by a high-performance **Rust Native Core**, solving the most demanding inventory valuation challenges in Excel:

- 🚀 **Instantaneous Performance:** Linear $O(N)$ streaming queue processing handles tens of thousands of rows smoothly without freezing or slowing down Excel.
- ⚡ **Dynamic Array 2D Spill by Default:** Write a single formula in the top-left cell; results automatically spill down and across the table. No manual dragging or copying required.
- 🛡️ **Zero `#VALUE!` Errors in Arithmetic:** Inbound/purchase rows return numeric `0.0` (instead of empty string `""`), allowing seamless addition, subtraction, multiplication, and division (`+`, `-`, `*`, `/`) across numeric columns.
- 📊 **Detailed Batch Allocation Breakdown (Mode 10):** Transparently traces cost of goods sold (COGS) back to original purchase batches and receipts, fulfilling strict auditor and chief accountant requirements.
- 💡 **Consolidated Unit Price Expressions:** Explanations automatically group lots sharing identical purchase prices into compact `(Qty x Price)` expressions, standardized with the `'x'` multiplication symbol for full compatibility with `=DIEN_GIAI_KHOI_LUONG()`.
- 🔄 **Standard Negative Quantity Handling (VAS 02):**
  - Purchase returns to vendors (`In Qty < 0`): Automatically deducts backwards from previously received lots.
  - Sales returns from customers (`Out Qty < 0`): Automatically re-enters stock at the most recent issue cost.
- ⚠️ **Flexible Over-issue (Negative Stock) Handling:** Gracefully prices negative inventory using the latest purchase price (`allow_negative = 1`) or strictly interrupts with `#NUM!` (`allow_negative = 0`).

---

## 2. SYNTAX OF `=FIFO()`

```excel
=FIFO(DATA_RANGE, [ITEM_COL], [IN_QTY_COL], [IN_PRICE_COL], [OUT_QTY_COL], [MODE], [LOT_DATE_COL], [ALLOW_NEGATIVE])
```

### Detailed Argument Specifications:

| # | Parameter | Required? | Default | Meaning & Configuration Guide |
| :---: | :--- | :---: | :---: | :--- |
| **1** | `DATA_RANGE` | **Required** | — | Inventory ledger data range (e.g., `$B$8:$R$301`). Absolute references with `$` are recommended. |
| **2** | `ITEM_COL` | Optional | `1` | 1-based column index for **Item Code / Item Name** within the data range. |
| **3** | `IN_QTY_COL` | Optional | `2` | 1-based column index for **Inbound / Purchase Quantity**. |
| **4** | `IN_PRICE_COL`| Optional | `3` | 1-based column index for **Inbound / Purchase Unit Price**. |
| **5** | `OUT_QTY_COL` | Optional | `4` | 1-based column index for **Outbound / Issue Quantity**. |
| **6** | `MODE` | Optional | `0` | Output mode selector (see detailed mode table below). |
| **7** | `LOT_DATE_COL`| Optional | `0` | 1-based column index for **Lot Number / Receipt Date / Voucher Number** (used in Mode 10). If `0`, defaults to `Row X`. |
| **8** | `ALLOW_NEGATIVE`| Optional | `1` | **Policy for negative inventory (over-issue):**<br>• `1` *(Recommended)*: Temporarily prices at latest purchase price without breaking formulas.<br>• `0`: Strictly returns `#NUM!` on negative stock. |

---

## 3. AVAILABLE OUTPUT MODES (`MODE`)

| `MODE` | Spill Dimension | Result Columns | Accounting Purpose |
| :---: | :---: | :--- | :--- |
| **0** *(Default)* | **$N \times 3$** | **[Issue Unit Price \| Issue Amount \| Ending Balance Amount]** | **Complete Ledger View:** Fills 3 primary columns. Inbound rows return `0.0` for price and issue amount to guarantee arithmetic stability. |
| **1** | $N \times 1$ | **[Issue Amount]** | Used when only a single cost of goods sold (COGS) amount column is needed. |
| **2** | $N \times 1$ | **[Issue Explanation]** | Mathematical breakdown of depleted lots, e.g., `(100 x 50,000) + (20 x 60,000)`. Inbound rows return `""`. |
| **3** | $N \times 1$ | **[Ending Quantity Balance]** | Remaining physical stock quantity after each transaction line. |
| **4** | $N \times 1$ | **[Weighted Average Ending Price]** | Average unit cost of remaining inventory lots after each line. |
| **5** | $N \times 1$ | **[Ending Balance Amount]** | Monetary value of ending inventory balance after each line. |
| **6** | $N \times 1$ | **[Ending Stock Explanation]** | Mathematical breakdown of remaining stock by price tier, e.g., `(28 x 2,585,000) + (8 x 2,531,925)`. |
| **10** | **$M \times 5$** | **[Item Code \| Lot/Date \| Deducted Qty \| Lot Unit Price \| Amount]** | **Detailed Allocation Breakdown Table:** Traces every issue back to original purchase receipts (equivalent to `=FIFO_PHAN_BO()`). |

---

## 4. PRACTICAL EXAMPLES

Suppose your inventory transaction ledger is in `$B$11:$O$304`, structured with columns relative to column B (column 1):
- Column 1 (Col B): Item Code / Item Name
- Column 2 (Col C): Voucher Date / Receipt Date
- Column 8 (Col I): Inbound Quantity
- Column 9 (Col J): Inbound Unit Price
- Column 11 (Col L): Outbound Quantity

### Example 1: Standard Complete Inventory Ledger (Default Mode 0)
In the top cell of your Issue Unit Price column (e.g., cell `M11`), enter:
```excel
=FIFO($B$11:$O$304, 1, 8, 9, 11)
```
- **Result:** Automatically spills into 3 columns:
  - Column 1 (M): Issue Unit Price
  - Column 2 (N): Issue Amount (COGS)
  - Column 3 (O): Ending Balance Amount
- **Feature:** Inbound-only rows display `0.0` for issue price and amount, while ending balance reflects cumulative inventory value. Cells can be safely referenced in math formulas without `#VALUE!`.

---

### Example 2: Detailed Audit Allocation Breakdown Table (Mode 10 or `=FIFO_PHAN_BO`)
In an empty section next to the ledger (e.g., cell `S11`), enter:
```excel
=FIFO($B$11:$O$304, 1, 8, 9, 11, 10, 2)
```
*Or use the dedicated alias:*
```excel
=FIFO_PHAN_BO($B$11:$O$304, 1, 8, 9, 11, 2)
```
*(Where the final parameter `2` specifies Column C - Voucher Date as the lot tracking identifier).*

- **Result:** Spills a 5-column independent audit table:
  - Column 1: Item Code
  - Column 2: Original Receipt Date / Lot Number
  - Column 3: Deducted Quantity
  - Column 4: Original Purchase Unit Price
  - Column 5: Allocated COGS Amount

---

### Example 3: Issue Calculation Explanation String (Mode 2)
In the top cell of the Issue Explanation column (e.g., `P11`), enter:
```excel
=FIFO($B$11:$O$304, 1, 8, 9, 11, 2)
```
- **Result:** Displays structured expressions like:
  ```text
  (100 x 50,000) + (20 x 60,000)
  ```
- **Evaluate directly with formula:**
  ```excel
  =DIEN_GIAI_KHOI_LUONG(P11)  --> Returns 6,200,000
  ```

---

### Example 4: Ending Stock Valuation by Price Tier (Mode 6)
In the top cell of the Ending Stock Explanation column (e.g., `Q11`), enter:
```excel
=FIFO($B$11:$O$304, 1, 8, 9, 11, 6)
```
- **Result:** Lots with identical purchase prices are automatically consolidated:
  ```text
  (28 x 2,585,000) + (8 x 2,531,925) + (300 x 2,587,500)
  ```

---

## 5. IMPORTANT NOTES & BEST PRACTICES

1. **Automatic Data Normalization:**
   - The function automatically trims extra whitespace, matches item codes case-insensitively (`SP01` matches `  sp01  `), and strips non-breaking Unicode spaces (`\u00A0`).
2. **Preventing `#SPILL!` Errors in Excel:**
   - Ensure the cells below and to the right of the formula cell are completely blank, without text, numbers, or merged cells obstructing the dynamic array spill range.
3. **Chronological Row Order:**
   - Transactions in the table should be sorted chronologically from top to bottom to ensure the First-In, First-Out principle accurately reflects physical inventory flow.
