# Contents
- [General Formatting](#general-formatting)
    - [Case](#case)
    - [Indentation](#indentation)
    - [White Space](#white-space)
- [Query Structure](#query-structure)
    - [Root Keywords Left-Aligned](#root-keywords-left-aligned)
    - [Listing Fields](#listing-fields)
- [Naming and Aliasing](#naming-and-aliasing)
    - [Explicit Aliasing](#explicit-aliasing)
    - [Meaningful Aliases](#meaningful-aliases)
    - [Naming for Data Type](#naming-for-data-type)
- [Joins and Logic](#joins-and-logic)
    - [Explicit Joins](#explicit-joins)
    - [Join Conditions](#join-conditions)
    - [CTEs > Subqueries](#ctes--subqueries)
    - [Things to Avoid](#things-to-avoid)
    - [Examples](#examples)

# General Formatting

## Case

- Capitalise all keywords e.g: `SELECT`, `FROM`, `ORDER BY`.
- Capitalise all literals e.g. `NULL`, `TRUE`, `FALSE`.
- **snake_case** for all identifiers (tables, columns, aliases).

**Why?** This creates an immediate visual distinction between the "grammar" of SQL and your specific "data".

## Indentation

- Use **4 spaces** for indentation. Do not use tabs - these can render differently in different IDEs and result in inconsistent alignment.
- Indent code inside logical blocks (CTEs, subqueries, `CASE` statements).

Example:

```sql
SELECT
    CASE
        WHEN is_live 
         AND live_service_count > 4
         AND tenure_years > 7
        THEN 'loyal'

        WHEN is_live 
         AND live_service_count > 4
         AND tenure_years <= 1
        THEN 'newby'
        
        ELSE 'unknown'
    END AS cohort
FROM
...
```

## White Space

- Use spaces around operators (`=`, `+`, `>`, etc.).
- One empty line between distinct CTEs or major logical blocks.

# Query Structure

## Root Keywords Left-Aligned

Align root keywords (`SELECT`, `FROM`, `JOIN`, `WHERE`, `GROUP BY`, `HAVING`, `LIMIT`) to the left margin.

Example:

```sql
SELECT
    sales_month,
    COUNT(1) AS total_sales,
FROM `uw.customers.orders` AS ord
INNER JOIN `uw.sales.ledger` AS led
        ON ord.order_id = led.order_id
WHERE
    line_item = 'business_cards'
GROUP BY
    sales_month
HAVING
    COUNT(1) >= 5
ORDER BY
    sales_month
LIMIT 100
...
```

## Listing Fields

- List field names on new lines (never on the same line as the keyword) with a trailing comma.

```sql
SELECT
    full_name,
    age,
    dob,
    country,
FROM
...
```

**Why?** Trailing commas allow for alignment of fields across all lines and make it easy to comment out any field without causing a syntax violation. They also provide a cleaner `git diff` as a new field can be inserted anywhere and is represented as a single line change (opposed to a two line change). It also allows all fields to use the same indentation, improving readability.

# Naming and Aliasing

## Explicit Aliasing

**Always** use `AS` for aliasing. It separates the source from the new name visually.

**Always** include aliases in field listings when a `JOIN` is present - don't leave users guessing which table the field originates from.

Example:

```sql
SELECT
    usr.user_id,
    usr.full_name,
    usr.email,
    ord.total_orders,
    ord.total_revenue,
FROM `uw.customer.users` AS usr
LEFT JOIN `uw.customer.orders` AS ord
       ON usr.user_id = ord.user_id
```

## Meaningful Aliases

Avoid single-letter aliases (`u`, `o`, `p`). Use abbreviations that are recognizable (`users` becomes `usr`, `orders` becomes `ord`). Avoid anything cryptic or needless shortened to the point of obscurity.

## Naming for Data Type

- Boolean columns should answer a question: `is_active`, `has_subscription`.
- Date and time columns should be specific: `created_at` (timestamp), `signup_date` (date only).

# Joins and Logic

## Explicit Joins

Always use `INNER JOIN` instead of just `JOIN`. Be explicit about intent.

## Join Conditions

Place the `ON` indented on the next line aligned to the end of the JOIN keyword.

Example:

```sql
SELECT
    ...
FROM `uw.customer.users` AS usr
LEFT JOIN `uw.customer.orders` AS ord
       ON usr.user_id = ord.user_id
      AND usr.signup_date = ord.order_date
```

## CTEs > Subqueries

- **Always** prefer Common Table Expressions (CTEs) over nested subqueries. CTEs read top-to-bottom like a story; subqueries read inside-out.
- Use CTEs to "import", clean and transform your data before the final `SELECT`.
- Separate the cleaning and transformation from any business logic being codified in SQL. [Further guidance](https://wiki.uw.systems/posts/data-model-best-practice-l7piqs0t).

# Things to Avoid

1. **Avoid** `SELECT *`: Always explicitly list columns. It protects downstream models and consumers from schema changes. This also works to keep costs down in BigQuery.
1. **Avoid Positional References**: Never use `GROUP BY 1, 2`. If you change column order, your logic breaks silently. Repeat the column names or aliases.
1. **No "Smart" Formatting**: Avoid "river" formatting (aligning all `=` signs vertically). It looks nice until you change one field name and have to re-indent 20 lines.
1. **Avoid** leaving any unnecessary blank lines at the start or end of files. A single trailing blank line is good.

# Examples

Bad:

```sql
select u.id, u.name, count(o.id) as orders, sum(o.total) as revenue from users u left join orders o on u.id = o.user_id where u.created_at > '2023-01-01' and o.status != 'cancelled' group by 1, 2 order by 4 desc;
```

Good:

```sql
WITH user_orders AS (
    -- Aggregate order metrics first to avoid fan-out issues in main join
    SELECT
        user_id,
        COUNT(order_id) AS total_orders,
        SUM(order_total) AS total_revenue
    FROM orders
    WHERE
        status != 'cancelled'
    GROUP BY
        user_id
)

SELECT
    usr.user_id,
    usr.full_name,
    usr.email,
    COALESCE(ord.total_orders, 0) AS total_orders,
    COALESCE(ord.total_revenue, 0.00) AS total_revenue
FROM users AS usr
LEFT JOIN user_orders AS ord
       ON usr.user_id = ord.user_id
WHERE
    usr.created_at >= '2023-01-01'
ORDER BY
    total_revenue DESC;
```
