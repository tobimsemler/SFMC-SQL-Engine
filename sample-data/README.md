# Sample training datasets

Three small, clearly synthetic CSV files for practicing SQL in [`cloudpage.html`](../cloudpage.html). No real data, just enough variety to exercise joins, grouping, dates, and NULLs.

| File | Rows | Notable quirks |
|---|---|---|
| `customers.csv` | 15 | Mixed countries, one missing `SignupDate` (row 12), Unicode name `Freja Ærø` (row 13). |
| `products.csv` | 10 | Flat product catalog, no quirks, used to look up names/categories/prices. |
| `orders.csv` | 30 | One order with a blank `ProductID` (row 5015), customer 15 (`Astrid Vik`) has zero orders. |

## Loading them

1. Open `cloudpage.html`.
2. Add three datasets named `Customers`, `Products`, and `Orders`.
3. Upload the matching CSV to each.

## Queries to try

```sql
-- Basic join and filter
SELECT c.CustomerName, c.Country, o.OrderID, o.OrderDate
FROM Customers c
INNER JOIN Orders o ON o.CustomerID = c.CustomerID
WHERE c.Country = 'Denmark'
ORDER BY o.OrderDate;

-- LEFT JOIN to find customers with no orders (Astrid Vik should appear with NULLs)
SELECT c.CustomerName, o.OrderID
FROM Customers c
LEFT JOIN Orders o ON o.CustomerID = c.CustomerID
WHERE o.OrderID IS NULL;

-- Aggregation: total spend per customer
SELECT c.CustomerName, SUM(o.Quantity * o.UnitPrice) AS TotalSpend, COUNT(o.OrderID) AS Orders
FROM Customers c
INNER JOIN Orders o ON o.CustomerID = c.CustomerID
GROUP BY c.CustomerName
HAVING SUM(o.Quantity * o.UnitPrice) > 300
ORDER BY TotalSpend DESC;

-- Three-way join with a product lookup
SELECT o.OrderID, c.CustomerName, p.ProductName, p.Category, o.Quantity
FROM Orders o
INNER JOIN Customers c ON c.CustomerID = o.CustomerID
INNER JOIN Products p ON p.ProductID = o.ProductID
ORDER BY o.OrderID;

-- Window function: rank customers by spend
SELECT c.CustomerName, SUM(o.Quantity * o.UnitPrice) AS TotalSpend,
       RANK() OVER (ORDER BY SUM(o.Quantity * o.UnitPrice) DESC) AS SpendRank
FROM Customers c
INNER JOIN Orders o ON o.CustomerID = c.CustomerID
GROUP BY c.CustomerName;

-- Date math: months since signup
SELECT CustomerName, SignupDate, DATEDIFF(month, SignupDate, GETDATE()) AS MonthsSinceSignup
FROM Customers
WHERE SignupDate IS NOT NULL
ORDER BY MonthsSinceSignup DESC;
```
