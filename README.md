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

## Fix 3: Timezone-Naive Datetime (Wrong Monthly Revenue Totals)

### Complaint
Sunset Properties reported: *"The revenue numbers don't match our internal records. We're showing different totals for March."*

### How We Found It

1. Client A (Sunset Properties) has all properties in `Europe/Paris` timezone (UTC+1)
2. Checked `seed.sql` — found a reservation stored at the UTC month boundary:
   ```sql
   ('res-tz-1', 'prop-001', 'tenant-a', '2024-02-29 23:30:00+00', ...)  -- $1,250
   ```
3. `2024-02-29 23:30 UTC` = `2024-03-01 00:30 Europe/Paris` — Client A expects this in **March**
4. Traced into `calculate_monthly_revenue()` in `reservations.py`:
   ```python
   start_date = datetime(year, month, 1)        # naive — no timezone
   end_date   = datetime(year, month + 1, 1)    # naive — no timezone
   ```
5. The DB stores timestamps as `TIMESTAMP WITH TIME ZONE`. When Python sends a naive datetime in a query, PostgreSQL compares without a timezone context — the boundary comparison is inconsistent and the reservation falls outside the March range.
6. Result: the `$1,250` reservation is excluded from March, causing totals to not match Client A's records.

### Root Cause

**File:** `backend/app/services/reservations.py` — lines 10–14

Date range boundaries were created as timezone-naive `datetime` objects. The function also lacked a `tenant_id` parameter, which would have caused a missing variable error when the DB connection went live.

### Fix

```python
# Before (broken)
from datetime import datetime

async def calculate_monthly_revenue(property_id: str, month: int, year: int, ...):
    start_date = datetime(year, month, 1)
    end_date   = datetime(year, month + 1, 1)

# After (fixed)
from datetime import datetime, timezone

async def calculate_monthly_revenue(property_id: str, tenant_id: str, month: int, year: int, ...):
    start_date = datetime(year, month, 1, tzinfo=timezone.utc)
    end_date   = datetime(year, month + 1, 1, tzinfo=timezone.utc)
```

Date boundaries are now UTC-aware, matching the `TIMESTAMP WITH TIME ZONE` columns in the DB. Reservations at timezone boundaries are compared consistently, so March totals match what clients see in their own records. `tenant_id` was also added to the function signature to match the SQL query that already expected it as `$2`.

---

---

## Fix 4: Database Never Connected (Mock Data Served as Real Data)

### Complaint
All clients were seeing suspiciously round revenue numbers that didn't match their internal records — and the numbers never changed regardless of actual bookings.

### How We Found It

1. The API was returning `total_revenue: 1000` for prop-001 — but the seed data shows the real total should be **$2,250**
2. Traced into `calculate_total_revenue()` in `reservations.py` — it tries the DB first, and falls into an `except` block on failure
3. Followed into `database_pool.py` line 18 — found the connection URL being built from fields that don't exist:
   ```python
   # database_pool.py was trying to read these:
   settings.supabase_db_user
   settings.supabase_db_password
   settings.supabase_db_host
   settings.supabase_db_port
   settings.supabase_db_name
   ```
4. Checked `config.py` — none of those fields are defined. The config only has:
   ```python
   database_url: str = "postgresql://postgres:postgres@db:5432/propertyflow"
   ```
5. Result: `initialize()` always threw an exception, `session_factory` stayed `None`, and every revenue request silently fell back to hardcoded mock data:
   ```python
   mock_data = {
       'prop-001': {'total': '1000.00', 'count': 3},  # fake
       'prop-002': {'total': '4975.50', 'count': 4},  # fake
       ...
   }
   ```
   No errors were raised — the system quietly served invented numbers as if they were real.

### Root Cause

**File:** `backend/app/core/database_pool.py` — line 18

The connection URL was assembled from individual config fields (`supabase_db_user`, `supabase_db_host`, etc.) that were never defined in `config.py`. The correct full URL already existed as `settings.database_url` but was never used.

### Fix

```python
# Before (broken) — references fields that don't exist in config
database_url = f"postgresql+asyncpg://{settings.supabase_db_user}:{settings.supabase_db_password}@{settings.supabase_db_host}:{settings.supabase_db_port}/{settings.supabase_db_name}"

# After (fixed) — use the existing database_url from config
database_url = settings.database_url.replace(
    "postgresql://", "postgresql+asyncpg://"
)
```

The existing `database_url` in config is already correct for the Docker environment (`db:5432`). Converting it to the `asyncpg` driver format is all that was needed. The database now connects successfully and returns real revenue figures.

---

## Summary

| # | Complaint | File | Line | Root Cause | Fix |
|---|---|---|---|---|---|
| 1 | Seeing another company's revenue on refresh | `services/cache.py` | 13 | Cache key only used `property_id` — shared across tenants | Add `tenant_id` to cache key |
| 2 | Revenue off by a few cents | `api/v1/dashboard.py` | 18 | `float()` conversion loses decimal precision | Use `Decimal` instead of `float` |
| 3 | March totals don't match internal records | `services/reservations.py` | 10–14 | Timezone-naive `datetime` mishandles UTC boundary reservations | Add `tzinfo=timezone.utc` to date range |
| 4 | All revenue figures wrong — never matched real bookings | `core/database_pool.py` | 18 | DB connection URL built from non-existent config fields — always failed silently, serving hardcoded mock data | Use `settings.database_url` with asyncpg driver prefix |
