# Solutions — Property Revenue Dashboard

This document is the plain-language companion to [ASSIGNMENT.md](ASSIGNMENT.md). It explains **what was going wrong**, **what we changed**, and **where to look in the code**—without assuming you already know the stack.

---

## How this maps to the assignment

| What people reported | In simple terms |
| -------------------- | ---------------- |
| **Client B (Ocean)** — numbers that look like another company after refresh | Data or cache was not strictly tied to **who is logged in**. |
| **Client A (Sunset)** — March totals do not match our books | Mix of **wrong expectations** (dashboard vs “March only”) and **making sure** totals are computed for the **right tenant and property**. |
| **Finance** — totals a few cents off | **Floating-point rounding** in computers, not “magic” money disappearing. |

---

## Fix 1: Stopping cross-tenant mix-ups (privacy)

**Idea:** Anything that remembers or returns revenue must include **both** the customer (tenant) and the property—not the property alone.

### Redis cache (`backend/app/services/cache.py`)

- **Before (the problem):** Cache keys looked like “revenue for property X” only. If two companies used the same property id pattern, or shared cache entries in a confusing way, one refresh could show **cached revenue that belonged to another tenant’s context**.
- **After (the fix):** Keys are `revenue:{tenant_id}:{property_id}` so each company’s cached summary is **separate**.

### When the database is unavailable (`backend/app/services/reservations.py`)

- **Before:** Fallback “mock” revenue was keyed **only by property id**, so the wrong tenant could still get numbers meant for another story in the assignment.
- **After:** Mock data is keyed by **`(tenant_id, property_id)`** so a fallback cannot serve another company’s figures.

### Real database totals (`backend/app/services/reservations.py`)

- Revenue is loaded with SQL that filters on **`property_id` and `tenant_id`**. So even if someone asks for a property id, the row set is still **scoped to that tenant**.

### API layer (`backend/app/api/v1/dashboard.py`)

- The dashboard summary endpoint **requires** a real tenant on the signed-in user. If tenant context is missing, it responds with **403** instead of inventing a default tenant. That avoids “everyone shares one fake tenant” bugs.

### Frontend request (`frontend/src/components/RevenueSummary.tsx`)

- The app only sends the optional **`X-Simulated-Tenant`** header when you explicitly pass a `debugTenant` prop (for local testing). Normal usage relies on the **real session**, so we do not accidentally override the tenant on every request.

**Takeaway for a Loom or demo:** “We made tenant part of the cache key, the fallback data, the SQL filters, and the API rules—so revenue is always anchored to the logged-in customer.”

---

## Fix 2: Client A, March, and “numbers don’t match”

**Idea:** Align **what the screen measures** with **what finance compares it to**.

### What the dashboard actually shows

- The backend totals **all reservations** for that **property + tenant** (lifetime-style sum), not “March only.”
- The dashboard copy in **`frontend/src/components/Dashboard.tsx`** says clearly that this is **all-time** revenue, **not** a single calendar month. That reduces apples-to-oranges comparisons (March books vs all-time dashboard).

### Monthly reporting (future-facing note)

- **`calculate_monthly_revenue`** in `backend/app/services/reservations.py` documents that real “March in Paris vs March in New York” reporting needs **property time zones** in the boundaries. The assignment’s “March” story is partly about **clarifying product scope** today and **knowing** what a proper monthly report would require later.

**Takeaway:** “If it still doesn’t match March, first check whether you’re comparing March P&L to an all-time total—and use the same tenant and property the API uses.”

---

## Fix 3: Pennies that look “slightly off” (finance)

**Idea:** Money should be rounded in a predictable way, and the UI should not re-introduce floating-point noise.

### Backend (`backend/app/api/v1/dashboard.py`)

- Totals are handled with **`Decimal`**, rounded **half-up** to two decimal places for display.
- The JSON includes:
  - **`total_revenue`** as a **string** (stable decimal text), and  
  - **`total_revenue_minor_units`** as an **integer** (exact cents).

That way the wire format is not a lossy float like `2250.0099999998`.

### Frontend (`frontend/src/components/RevenueSummary.tsx`)

- **`amountForDisplay`** prefers **`total_revenue_minor_units / 100`** when present, so the number on screen matches **integer cents** from the server.
- A small **precision warning** (when using raw debug view) only appears if minor units are missing and the string/number disagree—useful when debugging, not for normal users.

**Takeaway:** “We stopped treating money as a float end-to-end; we round once clearly on the server and display from exact cents when we can.”

---

## Quick file checklist

| Area | File |
| ---- | ---- |
| Tenant required, decimal output, minor units | `backend/app/api/v1/dashboard.py` |
| Cache key includes tenant | `backend/app/services/cache.py` |
| SQL + tenant-scoped mock fallback | `backend/app/services/reservations.py` |
| Display from cents, optional debug tenant header only | `frontend/src/components/RevenueSummary.tsx` |
| User-facing “all-time” explanation | `frontend/src/components/Dashboard.tsx` |

---

## One thing to know about the property dropdown

The dashboard still uses a **fixed list** of properties in the UI for the selector. The **server** enforces tenant on data, which is what fixes the serious privacy and cache issues. For a production polish, you would typically **load the property list from an API** filtered by tenant so users only see **their** properties in the dropdown. That keeps the UX aligned with the same multi-tenant rules as the backend.

---

*This repo’s assignment is a debugging exercise: the fixes above focus on **correct tenant boundaries**, **honest reporting scope**, and **stable monetary precision**.*
