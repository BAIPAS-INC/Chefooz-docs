# Report Module — Technical Guide

**Last Updated:** 11 March 2026

---

## Architecture

| Layer | Path |
|---|---|
| Backend Controller (V2) | `apps/chefooz-apis/src/modules/report/report-v2.controller.ts` |
| Backend Service (V2) | `apps/chefooz-apis/src/modules/report/report-v2.service.ts` |
| MongoDB Schema | `apps/chefooz-apis/src/modules/report/schemas/report.schema.ts` |
| Module | `apps/chefooz-apis/src/modules/report/report.module.ts` |
| DTOs | `apps/chefooz-apis/src/modules/report/dto/` |
| API Client | `libs/api-client/src/lib/clients/report.client.ts` |
| React Query Hooks | `libs/api-client/src/lib/hooks/reports/useReports.ts` |
| Shared Types | `libs/types/src/lib/report.types.ts` |
| Mobile Component | `apps/chefooz-app/src/components/report/ReportSheet.tsx` |
| Admin Page | `apps/chefooz-admin/src/app/dashboard/reports/page.tsx` |

---

## V2 Data Model (MongoDB)

Collection: `reports`

| Field | Type | Notes |
|---|---|---|
| `reporterId` | string | User who submitted the report |
| `targetType` | `'reel' \| 'comment' \| 'user'` | What is being reported |
| `targetId` | string | MongoDB ObjectId or userId |
| `reason` | string | min 5 chars |
| `status` | `'pending' \| 'reviewed' \| 'action_taken' \| 'dismissed'` | Default: `pending` |
| `metadata` | object | Snapshot of target content at time of report |
| `moderatorNotes` | string? | Admin review notes |
| `reviewedBy` | string? | Admin user ID |
| `createdAt` / `updatedAt` | Date | Mongoose timestamps |

Indexes:
- Unique compound: `reporterId + targetType + targetId` (prevents duplicate reports)
- Index on `status` (efficient filter queries)
- Index on `createdAt` (chronological sort)

---

## V1 API (Legacy — PostgreSQL)

The V1 controller (`report.controller.ts`) with `@Controller({ version: '1' })` and V1 service use PostgreSQL. **These are NOT used for new reports.** They exist for backward compatibility only. All new reports go through V2 MongoDB endpoints.

---

## Key Constraints & Edge Cases

### Duplicate Report Handling
If a user tries to report the same content twice, MongoDB's unique index prevents double-insertion. The service catches the duplicate key error (code 11000) and silently returns the existing report. The user sees "success" — no error shown.

### Rate Limiting
Enforced in-memory in `ReportControllerV2.checkRateLimit()`:
- Max 5 submissions per user per hour
- Rate limit state is server-process-scoped (resets on restart, no Redis persistence)
- See: `rateLimitMap: Map<string, number[]>` tracked per userId

### Report Screen (Legacy — V1 format bug, fixed March 2026)
`apps/chefooz-app/src/app/report/index.tsx` was submitting V1-format DTOs (`{category, message, reelId/profileId/...}`) to the V2 endpoint. This caused ValidationPipe 400 errors and reports went silently unrecorded. **Fixed March 2026:** submission now uses V2 format `{targetType, targetId, reason}`. Target type options updated from `['reel', 'profile', 'message', 'review']` to V2 types `['reel', 'user', 'comment']`.

### Admin Default Filter (Fixed March 2026)
The admin reports page previously defaulted to `status = 'pending'`. In environments with no pending reports (all reviewed/dismissed), this incorrectly showed "No reports found". **Fixed:** default changed to `'all'` to show all reports regardless of status.

### "No Reports Found" vs "No Matching Reports"
- If `status === 'all'` AND `targetType === 'all'`: message is "No reports have been submitted yet."
- Otherwise: message indicates the filter is active and suggests changing it to "All".

---

## API Endpoints (V2)

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/v1/reports` | JWT | Submit a report |
| GET | `/api/v1/reports/my` | JWT | Get current user's reports |
| GET | `/api/v1/reports/admin` | JWT + Admin | Get all reports with optional filters |
| PATCH | `/api/v1/reports/admin/:id` | JWT + Admin | Update report status |

### Admin List Query Params
- `status` — `pending \| reviewed \| action_taken \| dismissed` (optional, no default = all)
- `targetType` — `reel \| comment \| user` (optional, no default = all)
- `limit` — number, default 50
- `cursor` — MongoDB ObjectId for pagination

---

## Frontend Hook API

```ts
useAdminReports(filters?: { status?: string; targetType?: string })
// Returns: useInfiniteQuery  → data.pages[n].reports, data.pages[n].nextCursor

useUpdateReportStatus()
// Returns: useMutation → mutateAsync({ reportId, dto: { status, moderatorNotes? } })

useCreateReport()
// Returns: useMutation → mutateAsync({ targetType, targetId, reason })
```
