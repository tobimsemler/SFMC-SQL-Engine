# SFMC SQL Engine

A backwards-engineered, read-only implementation of the SQL engine behind Salesforce Marketing Cloud's Query Studio.

SFMC has no API that takes a SQL statement and returns rows. The only official way to run a `SELECT` is through Query Studio, which creates a Query Activity, runs it, writes the result to a temporary Data Extension, and deletes it afterwards. This project reproduces that engine's read behaviour directly in JavaScript: given row data and a T-SQL statement, it parses and evaluates the query itself, with no Query Activity, no temporary Data Extension, and no writes of any kind.

It is a single, dependency-free file (`sqlengine.js`) containing a tokenizer, a recursive-descent parser, and a tree-walking evaluator.

## Design properties

- **No `eval()`, no `new Function()`, no WASM.** The parser is hand-written, so there is no dynamic code-execution path.
- **No network access.** The module never performs I/O. It only operates on row data (`{ columns, rows }`) that the caller supplies.
- **Read-only by construction.** There is no AST node for `INSERT`, `UPDATE`, or `DELETE`: the grammar only has statements for `SELECT`, `WITH ... SELECT`, and `UNION`, so nothing else can be parsed.
- **Unicode-aware identifiers.** Unquoted identifiers accept any Unicode letter, matching SQL Server's behaviour and SFMC field/Data Extension names that aren't restricted to ASCII.
- **Guard rails.** Configurable limits (`maxIntermediateRows`, `maxOutputRows`, `maxJoins`) cause a query to fail with a clear error instead of hanging.
- **Portable.** Works as a browser script (attaches `SqlEngine` on `window`) or as a CommonJS module, so the same code is unit-testable under `node --test`.

## Supported SQL

A broad, practical subset of T-SQL as SFMC actually uses it:

- `SELECT`, `DISTINCT`, `TOP ... PERCENT`
- `INNER` / `LEFT` / `RIGHT` / `FULL OUTER` / `CROSS JOIN`
- `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY` (including ordinal positions)
- `UNION` / `UNION ALL`
- Common Table Expressions (`WITH ... AS (...)`)
- Window functions: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE` over `PARTITION BY` / `ORDER BY`
- Aggregates: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `STDEV`, `STDEVP`, `VAR`, `VARP`
- `CASE WHEN`, `CAST`/`CONVERT` (with SQL Server style codes), `IN`, `BETWEEN`, `LIKE`, `IS NULL`
- Scalar functions:
  - String: `LEN`, `DATALENGTH`, `LOWER`, `UPPER`, `LTRIM`, `RTRIM`, `TRIM`, `REVERSE`, `SPACE`, `LEFT`, `RIGHT`, `SUBSTRING`, `REPLACE`, `CHARINDEX`, `STUFF`
  - Null handling: `ISNULL`, `NULLIF`
  - Math: `ABS`, `CEILING`, `FLOOR`, `SIGN`, `SQRT`, `POWER`, `ROUND`
  - Date: `GETDATE`, `GETUTCDATE`, `SYSDATETIME`, `YEAR`/`MONTH`/`DAY`, `DATEPART`, `DATEADD`, `DATEDIFF`, `DATENAME`, `CONVERT` date styles

`DATEADD`/`DATEDIFF` follow T-SQL's exact semantics, including clamping at month-end boundaries and counting boundary crossings rather than elapsed time.

## API

```js
const sqlEngine = require('./sqlengine.js');
```

| Export | Purpose |
|---|---|
| `parse(sql)` | Tokenizes and parses SQL into an AST. Throws `SqlError` (with a `.position`) on invalid syntax. |
| `execute(statement, sources, options?)` | Evaluates a parsed statement against `{ tableName: { columns, rows } }` and returns `{ columns, rows, truncated }`. |
| `analyse(sql)` | Parses and returns the tables (and CTEs) a query references. |
| `tableReferences(statement)` | Lower-level accessor behind `analyse()`. |
| `tokenize(sql)` | Exposes the raw tokenizer. |
| `SqlError` | Error type for parse/runtime failures; carries the character `position` of the failure. |
| `LIMITS` | Default guard-rail values (`maxIntermediateRows`, `maxOutputRows`, `maxJoins`). |
| `compareValues`, `orderValues`, `likeMatch`, `toNumber`, `toDate`, `toText` | Internal comparison/coercion helpers, exported for testing. |

### Example

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

console.log(result.columns);   // ['CustomerName', 'Orders']
console.log(result.rows);      // [{ CustomerName: '...', Orders: 3 }, ...]
console.log(result.truncated); // true if a row limit was hit
```

Use `analyse()` to find out which tables a query needs before executing it:

```js
const { tables } = sqlEngine.analyse(sql);
// [{ name: 'Customers', qualifier: null, key: 'customers' }, { name: 'Orders', ... }]
```

## Testing

```bash
node --test
```

## License

MIT: see [LICENSE](LICENSE). Use it, fork it, modify it, ship it.

## Disclaimer

This is an independent, backwards-engineered reimplementation of SQL parsing and evaluation behaviour, built by observing how Salesforce Marketing Cloud's Query Studio behaves. It is not affiliated with, endorsed by, or sponsored by Salesforce, Inc. "Salesforce Marketing Cloud" and "SFMC" are trademarks of Salesforce, Inc.; they are used here only to describe compatibility, not to imply an official relationship. No Salesforce source code, confidential material, or proprietary documentation was used to build this project. The engine is provided "as is" with no guarantee that its output matches Query Studio in every case; always validate results against your own environment before relying on them.

