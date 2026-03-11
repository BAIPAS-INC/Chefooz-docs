# Report Module — Feature Overview

**Last Updated:** 11 March 2026  
**Status:** Live (V2 MongoDB implementation)

---

## What It Does

Users can report problematic content on the Chefooz platform. Admins can view, review, and dismiss reports via the admin portal.

## Supported Report Targets

| Target Type | V2 Value | Description |
|---|---|---|
| Reel | `reel` | A cooking video |
| User / Profile | `user` | A user account |
| Comment | `comment` | A comment on a reel |

## Key Business Rules

1. A user cannot report the same content twice (MongoDB unique index enforced)
2. All reports enter the system as `pending`
3. Rate limit: max 5 report submissions per user per hour
4. Reports are stored in MongoDB (V2) — PostgreSQL V1 is legacy and not used for new reports
5. Admin-only endpoint requires `admin` JWT role

## Report Status Lifecycle

```
pending → reviewed → action_taken
         [or]
pending → dismissed
```

## Screens

### Mobile App
- **ReportSheet** (`src/components/report/ReportSheet.tsx`) — Bottom sheet modal, used inline from reel/comment/profile menus. Primary report path.
- **ReportScreen** (`src/app/report/`) — Standalone report form screen (currently not linked from main navigation; kept as fallback for deep links).

### Admin Portal
- **Reports Dashboard** (`/dashboard/reports`) — Full-page admin report management table with status/type filters, action buttons (Take Action / Dismiss).

## Admin Portal Behaviour

- Default filter: **All statuses** (changed from 'pending' in March 2026 bug fix)
- Admin can filter by: status, targetType
- Pagination via infinite scroll / "Load More"
- Status actions: `reviewed → action_taken` or `pending → dismissed`
