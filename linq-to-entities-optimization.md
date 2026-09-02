# C# LINQ Optimization: LINQ to Entities (EF Core)

This guide covers performance optimization when translating LINQ queries into database operations via **Entity Framework Core (EF Core)**. The primary objective is to reduce database server load, optimize network payload sizes, and minimize application memory overhead.

---

## 1. Database Query Optimizations

### Restrict Payloads via Selective Projections
Never extract full database entities or database records if you only require specific properties. Projecting directly into anonymous objects or custom Records forces the query compiler to request only the designated columns in the underlying SQL statement.

```csharp
// ❌ Slow: SELECT * FROM Users (Transfers unneeded columns/blobs)
var names = db.Users.ToList().Select(u => u.Name); 

// ✔️ Fast: SELECT Name FROM Users (Transfers ONLY the name column)
var names = db.Users.Select(u => new { u.Name }).ToList();
```

### Apply Read-Only Non-Tracking Clauses
By default, EF Core tracks instantiated entities in memory to intercept updates. If you only intend to read data without executing modifications, turn tracking off to save memory and CPU cycles.

```csharp
// ✔️ Optimized for reading: Bypasses entity state tracking structures
var readOnlyData = db.Products.AsNoTracking()
                              .Where(p => p.IsAvailable)
                              .ToList();
```

### Prevent the $N+1$ Query Problem
Referencing un-loaded child relational collections inside loops causes EF Core to query the database independently for each iteration. Resolve this by explicitly eager-loading dependencies at the database level.

```csharp
// ❌ Bad: Triggers 1 database query for orders + N subsequent database queries for customers
foreach(var order in db.Orders.ToList()) {
    Console.WriteLine(order.Customer.Name); 
}

// ✔️ Good: Combines relations into a single SQL JOIN operation upfront
var orders = db.Orders.Include(o => o.Customer).ToList();
```

### Maintain Database-Side Evaluation (`IQueryable`)
Ensure filtering expressions act entirely on `IQueryable` before executing terminal operators. Converting to `IEnumerable` or calling `.ToList()` early drops data context translation, forcing subsequent operators to evaluate in application memory instead of on the database engine.

```csharp
// ❌ Slow: Pulls down entire table to memory, then filters locally
var query = db.Logs.ToList().Where(l => l.Level == "Error");

// ✔️ Fast: Translates the filter directly into a database "WHERE" clause
var query = db.Logs.Where(l => l.Level == "Error").ToList();
```

---

## 2. LINQ Optimization Cheat Sheet

| Objective | Avoid | Use Instead | Data Source |
| :--- | :--- | :--- | :--- |
| **Check Existence** | `.Count() > 0` | `.Any()` | Global |
| **Extract Min/Max** | `.OrderBy().First()` | `.MinBy()` / `.MaxBy()` | Global |
| **Sub-set Iteration**| Custom Loop Indexing | `.Chunk()` | In-Memory |
| **Read-Only SQL** | Default Tracking | `.AsNoTracking()` | Database |
| **Column Filtering** | Fetching Full Entities | `.Select(x => new { ... })` | Database |
| **Pagination** | Client-side Loops | `.Skip().Take()` | Database |