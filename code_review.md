# Code Review: `feature/lead-search` Branch

## Context & Overview
- **Branch**: `feature/lead-search` compared to `main`
- **Target File**: [`server.ts`](file:///home/abdannaufal7/crm/server.ts#L230-L255)
- **Change Description**: Adds `GET /api/leads/search` endpoint to query leads by name, company, or email matching a search parameter `q`.
- **Review Axis Coverage**: Correctness, Readability & Simplicity, Architecture, Security, and Performance (per `code-review-and-quality` skill).

---

## Five-Axis Review Findings

### 1. Security 🛡️
- **Critical SQL Injection Vulnerability** *(RESOLVED)*:
  - **Location**: [`server.ts:238`](file:///home/abdannaufal7/crm/server.ts#L238)
  - **Issue**: Direct string interpolation of `req.query.q` into the SQL query string (`LIKE '%${query}%'`). This enables unauthenticated SQL injection attacks.
  - **Resolution**: Updated to use parameterized queries with `better-sqlite3` placeholders (`?`).
- **Information Disclosure (Error Leaks)** *(RESOLVED)*:
  - **Location**: [`server.ts:248-251`](file:///home/abdannaufal7/crm/server.ts#L248-L251)
  - **Issue**: Returning `details: error.message` in HTTP 500 error responses exposed internal database driver error details to callers.
  - **Resolution**: Removed `details: error.message` from public error response payload.

### 2. Correctness & Edge Cases 🎯
- **Input Type Invalidation & Edge Cases** *(RESOLVED)*:
  - **Location**: [`server.ts:231-235`](file:///home/abdannaufal7/crm/server.ts#L231-L235)
  - **Issue**: `req.query.q as string` assumed `q` is always a single string. If multiple `q` parameters were passed (`?q=a&q=b`), Express parsed `req.query.q` as an array `string[]`. Empty/whitespace queries were also not validated.
  - **Resolution**: Added strict `typeof rawQuery === "string"` check and whitespace trimming (`rawQuery.trim()`).

### 3. Architecture & Consistency 🏗️
- **Inconsistent Database Access Pattern** *(RESOLVED)*:
  - **Location**: [`server.ts:238-242`](file:///home/abdannaufal7/crm/server.ts#L238-L242)
  - **Issue**: Standard routes in [`server.ts`](file:///home/abdannaufal7/crm/server.ts) use prepared statements (`db.prepare(...).all(...)`). The search route broke consistency with dynamic SQL string assembly.
  - **Resolution**: Realigned search route with standard prepared statement bindings used across the project.

### 4. Readability & Simplicity 📖
- **Verbose Console Logging** *(RESOLVED)*:
  - **Location**: [`server.ts:240,244,247`](file:///home/abdannaufal7/crm/server.ts#L240-L247)
  - **Issue**: Logging raw SQL statements and result counts to `console.log` polluted production stdout logs.
  - **Resolution**: Removed debug log statements.

### 5. Performance ⚡
- **Unbounded Database Query** *(RESOLVED)*:
  - **Location**: [`server.ts:238`](file:///home/abdannaufal7/crm/server.ts#L238)
  - **Issue**: `SELECT * FROM leads WHERE ...` lacked a `LIMIT` clause, risking memory spikes on broad queries.
  - **Resolution**: Added `LIMIT 100` and `ORDER BY updated_at DESC`.

---

## Categorized Summary of Issues

| Severity | Issue | Location | Status |
| --- | --- | --- | --- |
| **Critical** | SQL Injection via unparameterized string concatenation | [`server.ts:238`](file:///home/abdannaufal7/crm/server.ts#L238) | ✅ Fixed |
| **Required** | Exposing internal DB error details (`error.message`) in HTTP 500 response | [`server.ts:248-251`](file:///home/abdannaufal7/crm/server.ts#L248-L251) | ✅ Fixed |
| **Required** | Untrimmed/Unvalidated query parameter handling (`req.query.q`) | [`server.ts:231-235`](file:///home/abdannaufal7/crm/server.ts#L231-L235) | ✅ Fixed |
| **Nit** | Remove debug `console.log` noise from production endpoint | [`server.ts:240,244,247`](file:///home/abdannaufal7/crm/server.ts#L240-L247) | ✅ Fixed |
| **Consider** | Add `LIMIT 100` to prevent unbounded query execution | [`server.ts:238`](file:///home/abdannaufal7/crm/server.ts#L238) | ✅ Fixed |

---

## Final Implementation

```typescript
  app.get("/api/leads/search", (req, res) => {
    const rawQuery = req.query.q;

    if (typeof rawQuery !== "string" || !rawQuery.trim()) {
      return res.status(400).json({ error: "Search query is required" });
    }

    const searchTerm = rawQuery.trim();

    try {
      const pattern = `%${searchTerm}%`;
      const results = db
        .prepare(
          `SELECT * FROM leads 
           WHERE name LIKE ? OR company LIKE ? OR email LIKE ?
           ORDER BY updated_at DESC
           LIMIT 100`
        )
        .all(pattern, pattern, pattern);

      res.json(results);
    } catch (error) {
      res.status(500).json({ error: "Search failed" });
    }
  });
```

---

## Review Checklist & Verdict

- [x] **Correctness**: Spec matching verified, edge case & input validation resolved.
- [x] **Readability**: Excessive logging removed.
- [x] **Architecture**: Inconsistent SQL execution pattern aligned.
- [x] **Security**: Critical SQL injection & info disclosure resolved.
- [x] **Performance**: Query bounded with `LIMIT 100`.

**Verdict**: ✅ **Approve** — All critical, required, and recommended fixes have been applied and verified. Ready to merge into `main`.
