# Financial Overview Module

**Frontend Route:** `/finances`
**API Base Path:** `/finances`
**Auth Required:** Yes (Bearer token on all endpoints)

---

## TypeScript Interfaces

```typescript
interface MonthlyFinance {
  month: string;          // Short month name: "Jan", "Feb", "Mar", ..., "Dec"
  income: number;
  expenses: number;
}

interface Transaction {
  id: string;
  description: string;
  type: "income" | "expense";
  amount: number;         // Always positive; `type` indicates direction
  date: string;           // ISO date string — "2024-03-15"
  category: string;       // Free-form category, e.g. "Crop Sales", "Feed Purchase"
}

interface ExpenseCategory {
  label: string;          // Category name, e.g. "Feed", "Labor", "Equipment"
  value: number;          // Percentage of total expenses: 0–100
}

interface FinanceSummary {
  totalIncome: number;
  totalExpenses: number;
  netProfit: number;      // totalIncome - totalExpenses (can be negative)
  period: string;         // Label for the period, e.g. "January 2024", "Q1 2024", "2024"
}
```

---

## Endpoints

---

### GET `/finances/summary`

Returns high-level financial summary for a given period.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description                                      |
|-----------|--------|----------|--------------------------------------------------|
| `farm_id` | string | Yes      | Active farm UUID                                 |
| `period`  | string | No       | One of: `monthly \| quarterly \| yearly`. Default: `monthly` |
| `year`    | string | No       | Four-digit year string, e.g. `"2024"`. Default: current year |
| `month`   | string | No       | Two-digit month `"01"`–`"12"`. Required if `period=monthly` |
| `quarter` | string | No       | `"1"`, `"2"`, `"3"`, or `"4"`. Required if `period=quarterly` |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "totalIncome": 185000,
    "totalExpenses": 112000,
    "netProfit": 73000,
    "period": "March 2024"
  }
}
```

**Frontend Impact:** The summary cards at the top of the Finances page are populated from this endpoint. `netProfit` is displayed in green if positive, red if negative.

---

### GET `/finances/monthly`

Returns monthly income and expense totals for a full year. Used to populate the bar/line chart.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description                                       |
|-----------|--------|----------|---------------------------------------------------|
| `farm_id` | string | Yes      | Active farm UUID                                  |
| `year`    | string | No       | Four-digit year, e.g. `"2024"`. Default: current year |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": [
    { "month": "Jan", "income": 14200,  "expenses": 8500  },
    { "month": "Feb", "income": 16800,  "expenses": 9200  },
    { "month": "Mar", "income": 21000,  "expenses": 11500 },
    { "month": "Apr", "income": 18400,  "expenses": 10800 },
    { "month": "May", "income": 22500,  "expenses": 12300 },
    { "month": "Jun", "income": 19800,  "expenses": 11000 },
    { "month": "Jul", "income": 25000,  "expenses": 14200 },
    { "month": "Aug", "income": 23600,  "expenses": 13500 },
    { "month": "Sep", "income": 20100,  "expenses": 11800 },
    { "month": "Oct", "income": 17400,  "expenses": 9900  },
    { "month": "Nov", "income": 15200,  "expenses": 8800  },
    { "month": "Dec", "income": 12000,  "expenses": 7500  }
  ]
}
```

**Important:** Always return **all 12 months**, even if there are no transactions for some months (return `0` for those months). This ensures the chart always shows all months on the X-axis.

**Frontend Impact:** The chart library maps over this array using `month` as the X-axis key. If months are missing, the chart will have gaps.

---

### GET `/finances/transactions`

Paginated list of all income and expense transactions.

**Auth Required:** Yes

**Query Parameters:**

| Param       | Type    | Required | Description                                     |
|-------------|---------|----------|-------------------------------------------------|
| `farm_id`   | string  | Yes      | Active farm UUID                                |
| `page`      | integer | No       | Page number, default `1`                        |
| `limit`     | integer | No       | Page size, default `20`                         |
| `type`      | string  | No       | Filter by: `income \| expense`                   |
| `startDate` | string  | No       | ISO date — return transactions on or after this |
| `endDate`   | string  | No       | ISO date — return transactions on or before this |
| `category`  | string  | No       | Filter by category string                       |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "total": 128,
    "page": 1,
    "limit": 20,
    "data": [
      {
        "id": "txn-uuid-001",
        "description": "Corn harvest sale — 4,800 kg",
        "type": "income",
        "amount": 21600,
        "date": "2024-03-18",
        "category": "Crop Sales"
      },
      {
        "id": "txn-uuid-002",
        "description": "Monthly feed purchase — Warehouse B",
        "type": "expense",
        "amount": 3400,
        "date": "2024-03-05",
        "category": "Feed"
      }
    ]
  }
}
```

**Note:** Results should be ordered by `date DESC` by default (most recent first).

---

### POST `/finances/transactions`

Create a new transaction record.

**Auth Required:** Yes

**Request Body:**

```json
{
  "description": "Corn harvest sale — 4,800 kg",
  "type": "income",
  "amount": 21600,
  "date": "2024-03-18",
  "category": "Crop Sales",
  "farm_id": "farm-uuid-here"
}
```

**Success Response — `201 Created`:**

```json
{
  "message": "Transaction recorded successfully",
  "data": { /* full Transaction object with id */ }
}
```

**Validation Rules:**
- `description`: required, 1–300 characters
- `type`: required, one of: `income | expense`
- `amount`: required, number > 0 (always positive; use `type` for direction)
- `date`: required, valid ISO date
- `category`: optional string, max 100 characters
- `farm_id`: required

---

### PUT `/finances/transactions/:id`

Update an existing transaction.

**Auth Required:** Yes

**Request Body:** Same shape as `POST /finances/transactions` (all fields, without `farm_id`).

**Success Response — `200 OK`:**

```json
{
  "message": "Transaction updated successfully",
  "data": { /* updated Transaction object */ }
}
```

---

### DELETE `/finances/transactions/:id`

Delete a transaction record.

**Auth Required:** Yes

**Success Response — `204 No Content`**

---

### GET `/finances/expense-breakdown`

Returns expense totals broken down by category as percentages. Used for the pie/donut chart.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description                                         |
|-----------|--------|----------|-----------------------------------------------------|
| `farm_id` | string | Yes      | Active farm UUID                                    |
| `year`    | string | No       | Four-digit year, default: current year              |
| `month`   | string | No       | Two-digit month; if omitted, aggregates full year   |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": [
    { "label": "Feed",          "value": 32 },
    { "label": "Labor",         "value": 25 },
    { "label": "Equipment",     "value": 18 },
    { "label": "Seeds",         "value": 12 },
    { "label": "Medicine",      "value": 8  },
    { "label": "Other",         "value": 5  }
  ]
}
```

**Calculation:** Group expense transactions by `category`, sum the `amount` for each category, then compute each as a percentage of total expenses. Percentages should sum to approximately `100`.

**Frontend Impact:** The pie chart maps each entry using `label` as the segment name and `value` as the size. If percentages do not sum to 100 (due to rounding), the chart may render incorrectly. Return values rounded to whole integers; if rounding causes the sum to differ from 100, adjust the largest category to compensate.

---

## Notes

- `amount` is always stored as a positive number. The `type` field (`"income"` or `"expense"`) indicates the direction of the transaction. Do not store negative amounts.
- `netProfit` in the summary endpoint can be negative (when expenses exceed income). The frontend displays it with a negative sign and red color.
- The `category` field in transactions is a free-form string (not a controlled enum). This gives flexibility but means the backend must aggregate categories by exact string match in the expense breakdown. Consider normalizing or trimming whitespace on write.
- All monetary values are stored and returned as plain numbers (not currency-formatted strings). The frontend applies local currency formatting for display.
- The `/finances/monthly` endpoint data feeds both the Finance page chart and the Reports module's `/reports/revenue` endpoint. Consider a shared data source or caching layer to avoid duplicate computation.
- Future consideration: Add a budget tracking feature with a `GET /finances/budget` endpoint that compares planned vs. actual spending per category.
