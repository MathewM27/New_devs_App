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

## Fix 2: (Pending)
## Fix 3: (Pending)
