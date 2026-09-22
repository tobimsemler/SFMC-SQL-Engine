# SFMC SQL Engine

**Run T-SQL against Salesforce Marketing Cloud data — without ever writing to SFMC.**

Salesforce Marketing Cloud's Query Studio has no API. There's no endpoint you can hit with a `SELECT` and get rows back. The only "official" way to run SQL against Marketing Cloud is to create a Query Activity, start it, wait, read the temporary target Data Extension it writes to, and then clean up after yourself — three write operations and a round trip through automation just to answer a read-only question.

SFMC SQL Engine skips all of that. It's a **self-contained, dependency-free SQL engine written in plain JavaScript** that parses SFMC-flavoured T-SQL and evaluates it directly against row data already sitting in the browser — no Query Activities, no temporary Data Extensions, no writes, no waiting.

## Why it exists

If you've ever used SFMC's Query Studio, you know the loop: write SQL → save as Query Activity → run it → poll until it finishes → open the target DE → read the rows → remember to clean up. Every step is a write against a shared, governed environment, and every step costs time.

This engine reproduces the *read* half of that experience entirely client-side. Point it at row sets you've already pulled from SFMC (via REST, SOAP, or however you get data out), and it gives you the same query results Query Studio would — instantly, locally, and without touching Marketing Cloud at all.

## Highlights

- 🔒 **No `eval()`, no `new Function()`, no WASM.** The parser and evaluator are hand-written from scratch, so there is no code-execution sink — and it runs safely under a strict MV3 content-script CSP.
- 🌐 **Zero network access.** This module never calls out to SFMC's API. It only ever touches data it's handed.
- 🚫 **Read-only by construction.** There is no AST node for `INSERT`, `UPDATE`, or `DELETE` — unsupported statements can't be parsed, let alone executed.
- 🇩🇰 **Unicode-aware identifiers.** Fields and Data Extensions with non-ASCII names (`Købsdato`, `Kunder Ærø`, etc.) work without bracket-escaping every reference.
- 🛡️ **Built-in guard rails.** Configurable row/join limits (`maxIntermediateRows`, `maxOutputRows`, `maxJoins`) so a runaway query fails loudly instead of freezing the tab.
- 🧪 **Runs anywhere.** Loadable as a browser content script or as a CommonJS module, so the exact same parser/evaluator is unit-testable under `node --test`.

## What SQL is supported

A broad, practical subset of T-SQL as SFMC actually uses it:

- `SELECT`, `DISTINCT`, `TOP ... PERCENT`
- `INNER` / `LEFT` / `RIGHT` / `FULL OUTER` / `CROSS JOIN`
- `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY` (including ordinal positions)
- `UNION` / `UNION ALL`
- Common Table Expressions (`WITH ... AS (...)`)
- Window functions: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE` over `PARTITION BY` / `ORDER BY`
- Aggregates: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `STDEV`, `STDEVP`, `VAR`, `VARP`
- `CASE WHEN`, `CAST`/`CONVERT` (with SQL Server style codes), `IN`, `BETWEEN`, `LIKE`, `IS NULL`
- A large scalar function library: string (`SUBSTRING`, `CHARINDEX`, `STUFF`, `REPLACE`, `LEFT`/`RIGHT`, ...), math (`ROUND`, `POWER`, `ABS`, ...), null-handling (`ISNULL`, `NULLIF`), and date functions (`GETDATE`, `DATEADD`, `DATEDIFF`, `DATEPART`, `CONVERT` date styles) — with T-SQL-accurate edge cases, like `DATEADD` clamping at month boundaries the same way SQL Server does.

## Quick start

```js
const sqlEngine = require('./sqlengine.js');

const sql = `
  SELECT c.CustomerName, COUNT(o.OrderID) AS Orders
  FROM Customers c
  LEFT JOIN Orders o ON o.CustomerID = c.CustomerID
  WHERE c.Country = 'Denmark'
  GROUP BY c.CustomerName
  ORDER BY Orders DESC
`;

const statement = sqlEngine.parse(sql);

const result = sqlEngine.execute(statement, {
  customers: { columns: ['CustomerID', 'CustomerName', 'Country'], rows: [...] },
  orders:    { columns: ['OrderID', 'CustomerID'], rows: [...] }
});

console.log(result.columns); // ['CustomerName', 'Orders']
console.log(result.rows);    // [{ CustomerName: '...', Orders: 3 }, ...]
console.log(result.truncated); // true if a row limit was hit
```

Need to know which tables/columns a query touches before running it (e.g. to fetch the right Data Extensions first)? Use `analyse()`:

```js
const { tables } = sqlEngine.analyse(sql);
// [{ name: 'Customers', qualifier: null, key: 'customers' }, { name: 'Orders', ... }]
```

## API surface

| Export | Purpose |
|---|---|
| `parse(sql)` | Tokenizes and parses SQL into an AST. Throws `SqlError` with a `.position` on invalid syntax. |
| `execute(statement, sources, options?)` | Evaluates a parsed statement against `{ tableName: { columns, rows } }` and returns `{ columns, rows, truncated }`. |
| `analyse(sql)` | Parses and returns the tables (and CTEs) a query references, so you know what to load before executing. |
| `tableReferences(statement)` | Lower-level accessor behind `analyse()`. |
| `tokenize(sql)` | Exposes the raw tokenizer for tooling/debugging. |
| `SqlError` | Thrown on parse/runtime errors; carries the character `position` of the failure. |
| `LIMITS` | Default guard-rail values (`maxIntermediateRows`, `maxOutputRows`, `maxJoins`). |

## Safety model

This engine is designed to be safe to embed in a browser extension running alongside a production Marketing Cloud tenant:

1. **No dynamic code execution.** The entire pipeline is tokenizer → recursive-descent parser → tree-walking evaluator. There's no path from user-supplied SQL to `eval`-like execution.
2. **No I/O.** The module has no knowledge of `fetch`, XHR, or any SFMC API client. It cannot write to, or even read from, Marketing Cloud on its own — data always comes in as plain arrays supplied by the caller.
3. **Statement-level allowlisting.** Only `SELECT` (and `WITH ... SELECT`, `UNION`) statements exist in the grammar. Anything else is a parse error, not a runtime permission check.

## Testing

The module is a plain CommonJS export, so it drops straight into Node's built-in test runner:

```bash
node --test
```
