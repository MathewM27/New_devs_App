# Bug Fix Log

---

## Fix 1: Cross-Tenant Revenue Data Leak (Privacy Breach)

### Complaint
Ocean Rentals reported: *"Sometimes when we refresh the page, we see revenue numbers that look like they belong to another company."*

### How We Found It

1. Followed the request path for `GET /api/v1/dashboard/summary?property_id=prop-001`
2. `dashboard.py` correctly extracts `tenant_id` from the authenticated user and passes both `property_id` and `tenant_id` to `get_revenue_summary()`
3. Inside `cache.py`, the cache key was built as:
   ```python
   cache_key = f"revenue:{property_id}"
   ```
   The `tenant_id` was received but completely ignored when building the key.
4. Checked `database/seed.sql` — both tenants share the same property ID:
   ```sql
   ('prop-001', 'tenant-a', 'Beach House Alpha', ...)   -- Sunset Properties
   ('prop-001', 'tenant-b', 'Mountain Lodge Beta', ...) -- Ocean Rentals
   ```
5. Conclusion: whichever tenant requested `prop-001` first got their data cached under the key `revenue:prop-001`. The next tenant to request the same property ID received the first tenant's cached revenue — a full data privacy breach.

### Root Cause

**File:** `backend/app/services/cache.py` — line 13

The Redis cache key did not include `tenant_id`, making it shared across all tenants for properties with the same ID.

### Fix

```python
# Before (broken)
cache_key = f"revenue:{property_id}"

# After (fixed)
cache_key = f"revenue:{tenant_id}:{property_id}"
```

Each tenant now gets their own isolated cache namespace. `revenue:tenant-a:prop-001` and `revenue:tenant-b:prop-001` are completely separate keys.

---

## Fix 2: Floating Point Precision Loss (Revenue Off by Cents)

### Complaint
Finance team noticed: *"Revenue totals seem slightly off by a few cents here and there."*

### How We Found It

1. The database schema stores revenue as `NUMERIC(10,3)` — exact decimal precision, intentional for financial data
2. The seed data contains a deliberate precision test case:
   ```sql
   333.333 + 333.333 + 333.334 = 1000.000  -- exact
   ```
3. Traced the value from DB through the code:
   - `calculate_total_revenue()` correctly returns the value as `str(Decimal(...))` — still precise ✓
   - `get_revenue_summary()` stores and retrieves it as a string via Redis — still precise ✓
   - `dashboard.py` line 18: `float(revenue_data['total'])` — **precision lost here**
4. Python's `float` uses IEEE 754 binary representation which cannot exactly represent most decimal fractions:
   ```python
   float("333.333") == 333.3330000000000151...  # binary rounding error
   ```
   The error is tiny but compounds across multiple properties and shows up in financial reports.

### Root Cause

**File:** `backend/app/api/v1/dashboard.py` — line 18

Converting a precise `Decimal` string to `float` at the API response layer introduced binary rounding errors.

### Fix

```python
# Before (broken)
total_revenue_float = float(revenue_data['total'])

# After (fixed)
total_revenue = str(Decimal(revenue_data['total']))
```

`Decimal` preserves exact decimal arithmetic. The value stays as a precise string through the full pipeline from DB → cache → API response, matching what clients see in their own records.

---

## Fix 3: (Pending)
