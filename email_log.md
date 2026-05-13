# Attendance Manager — Two Updates In One Patch

This patch contains TWO independent updates. Apply BOTH.

---

# PART A: Fix Night Shift Check-out Pairing Bug

## The Bug

In `server/routes/attendance.ts`, both the employee endpoint and the kiosk endpoint use the same wrong logic to find an open check-in from yesterday when today has no check-in.

The current logic (around lines 122-126 for kiosk, and 203-212 for employee):

```javascript
if (yesterdayCheckIn && !yesterdayCheckOut) {
  allRelevantLogs = [...yesterdayLogs, ...userTodayLogs];
}
```

This is WRONG because: when an employee works consecutive night shifts, yesterday's records contain BOTH a check-out (closing the day-before-yesterday's check-in) AND a still-open check-in. The condition `!yesterdayCheckOut` fails, so the system can't find the open check-in.

### Real production example:
- Day 10, 12:57 PM: check-in
- Day 11, 08:53 AM: check-out (closes day 10's check-in) ✅
- Day 11, 12:06 PM: check-in (open, awaiting checkout)
- Day 12: employee tries to check out → "You must check in first" ❌

## The Fix

Replace the wrong logic with proper timestamp-based pairing.

## Changes to `server/routes/attendance.ts`

### Change A1: Add helper function

Add this NEW function in `server/routes/attendance.ts`, right after the existing `getUserShift` function (around line 29), BEFORE the `isShiftExpired` function:

```typescript
// Find the last open check-in (one without a check-out paired after it by timestamp).
// Returns the open check-in log, or null if none.
// This handles consecutive night shifts correctly: it pairs check-ins with check-outs
// by timestamp order, not by calendar day.
function findOpenCheckIn(combinedLogs: any[]): any | null {
  const sorted = [...combinedLogs].sort(
    (a, b) => new Date(a.timestamp).getTime() - new Date(b.timestamp).getTime()
  );
  const openCheckIns: any[] = [];
  for (const log of sorted) {
    if (log.type === 'check-in') {
      openCheckIns.push(log);
    } else if (log.type === 'check-out' && openCheckIns.length > 0) {
      openCheckIns.shift();
    }
  }
  return openCheckIns.length > 0 ? openCheckIns[openCheckIns.length - 1] : null;
}
```

### Change A2: Fix kiosk endpoint (around line 120)

REPLACE this block:
```typescript
      // للشيفت الليلي: لو check-out ومفيش check-in اليوم، يدور في اليوم اللي قبله
      let allKioskLogs = [...todayLogs];
      if (type === 'check-out' && !todayLogs.some(l => l.type === 'check-in')) {
        if (yesterdayLogs.find(l => l.type === 'check-in') && !yesterdayLogs.find(l => l.type === 'check-out')) {
          allKioskLogs = [...yesterdayLogs, ...todayLogs];
        }
      }
```

WITH:
```typescript
      // للشيفت الليلي: نجمع logs أمبارح واليوم في array واحد ونعتمد على timestamp-based pairing
      const allKioskLogs = [...yesterdayLogs, ...todayLogs];
```

### Change A3: Fix duplicate-type check in kiosk endpoint (around line 134)

REPLACE:
```typescript
      if (todayLogs.find(log => log.type === type))
        return res.status(400).json({ message: type === "check-in" ? "Already checked in today | تم تسجيل الدخول مسبقاً اليوم" : "Already checked out today | تم تسجيل الخروج مسبقاً اليوم" });
      if (type === 'check-out' && !allKioskLogs.some(l => l.type === 'check-in'))
        return res.status(400).json({ message: "You must check in first | لا يمكن تسجيل الخروج بدون تسجيل دخول أولاً" });
```

WITH:
```typescript
      // اسمح بـ check-in واحد كل يوم
      if (type === 'check-in' && todayLogs.find(log => log.type === 'check-in'))
        return res.status(400).json({ message: "Already checked in today | تم تسجيل الدخول مسبقاً اليوم" });

      // الـ check-out مسموح لو فيه check-in مفتوح (حتى لو فيه check-out تاني في اليوم لشيفت ليلي سابق)
      if (type === 'check-out') {
        const openCheckIn = findOpenCheckIn(allKioskLogs);
        if (!openCheckIn) {
          return res.status(400).json({ message: "You must check in first | لا يمكن تسجيل الخروج بدون تسجيل دخول أولاً" });
        }
      }
```

### Change A4: Fix kiosk check-out section (around line 140)

REPLACE:
```typescript
      if (type === 'check-out') {
        const checkInLog = allKioskLogs.find(l => l.type === 'check-in');
```

WITH:
```typescript
      if (type === 'check-out') {
        const checkInLog = findOpenCheckIn(allKioskLogs);
```

(Keep the rest of that block unchanged.)

### Change A5: Fix employee endpoint (around line 203)

REPLACE:
```typescript
      // للشيفت الليلي: لو check-out ومفيش check-in اليوم، يدور في اليوم اللي قبله
      let allRelevantLogs = [...userTodayLogs];
      if (input.type === 'check-out' && !userTodayLogs.some(l => l.type === 'check-in')) {
        const yesterdayCheckIn = yesterdayLogs.find(l => l.type === 'check-in');
        const yesterdayCheckOut = yesterdayLogs.find(l => l.type === 'check-out');
        // لو فيه check-in أمبارح ومفيش check-out أمبارح — يبقى ده شيفت ليلي
        if (yesterdayCheckIn && !yesterdayCheckOut) {
          allRelevantLogs = [...yesterdayLogs, ...userTodayLogs];
        }
      }
```

WITH:
```typescript
      // للشيفت الليلي: نجمع logs أمبارح واليوم في array واحد ونعتمد على timestamp-based pairing
      const allRelevantLogs = [...yesterdayLogs, ...userTodayLogs];
```

### Change A6: Fix duplicate-type check in employee endpoint (around line 220)

REPLACE:
```typescript
      if (userTodayLogs.find(log => log.type === input.type))
        return res.status(400).json({ message: "Already recorded today | تم تسجيل الحضور مسبقاً اليوم" });
      if (!input.selfie || !input.selfie.startsWith('data:image/'))
        return res.status(400).json({ message: "Selfie photo is required | صورة السيلفي مطلوبة" });
      if (input.type === 'check-out' && !allRelevantLogs.some(l => l.type === 'check-in'))
        return res.status(400).json({ message: "You must check in first | لا يمكن تسجيل الخروج بدون تسجيل دخول أولاً" });
```

WITH:
```typescript
      // اسمح بـ check-in واحد كل يوم
      if (input.type === 'check-in' && userTodayLogs.find(log => log.type === 'check-in'))
        return res.status(400).json({ message: "Already recorded today | تم تسجيل الحضور مسبقاً اليوم" });

      if (!input.selfie || !input.selfie.startsWith('data:image/'))
        return res.status(400).json({ message: "Selfie photo is required | صورة السيلفي مطلوبة" });

      // الـ check-out مسموح لو فيه check-in مفتوح (حتى لو فيه check-out تاني في اليوم لشيفت ليلي سابق)
      if (input.type === 'check-out') {
        const openCheckIn = findOpenCheckIn(allRelevantLogs);
        if (!openCheckIn)
          return res.status(400).json({ message: "You must check in first | لا يمكن تسجيل الخروج بدون تسجيل دخول أولاً" });
      }
```

### Change A7: Fix employee check-out section (around line 228)

REPLACE:
```typescript
      if (input.type === 'check-out') {
        const checkInLog = allRelevantLogs.find(l => l.type === 'check-in');
```

WITH:
```typescript
      if (input.type === 'check-out') {
        const checkInLog = findOpenCheckIn(allRelevantLogs);
```

(Keep the rest of that block unchanged.)

---

# PART B: New Feature — Attendance Correction Requests

A complete new feature: the employee can request to add a missing check-in or check-out for a past date. The supervisor or admin reviews and approves/rejects. On approval, an attendance record is created automatically.

Mirror the existing `shiftChangeRequests` feature structure exactly (same patterns for DB table, storage, routes, employee page, admin page, sidebar entries, App.tsx route).

## Changes to `shared/schema.ts`

### Change B1: Add the new table

Add this NEW table right after the `shiftChangeRequests` table (after line 215):

```typescript
// جدول طلبات تصحيح البصمة (إضافة بصمة دخول أو خروج ناقصة)
export const attendanceCorrectionRequests = pgTable("attendance_correction_requests", {
  id: serial("id").primaryKey(),
  userId: integer("user_id").notNull().references(() => users.id, { onDelete: 'cascade' }),
  requestedDate: timestamp("requested_date").notNull(), // التاريخ والوقت المطلوب
  requestedType: text("requested_type").notNull(), // 'check-in' أو 'check-out'
  reason: text("reason").notNull(),
  status: text("status").default("pending").notNull(),
  reviewedBy: integer("reviewed_by").references(() => users.id),
  reviewedAt: timestamp("reviewed_at"),
  rejectionReason: text("rejection_reason"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  createdAttendanceId: integer("created_attendance_id").references(() => attendance.id),
});
```

### Change B2: Add the type exports

After the existing `ShiftChangeRequest` type export (around line 306), add:

```typescript
export type AttendanceCorrectionRequest = typeof attendanceCorrectionRequests.$inferSelect;
```

And after the existing `ShiftChangeRequestWithDetails` type (around line 327), add:

```typescript
export type AttendanceCorrectionRequestWithDetails = AttendanceCorrectionRequest & { user: SafeUser; reviewerName?: string | null };
```

## New file: `server/storage/attendance-correction-requests.ts`

Create this NEW file:

```typescript
import { db } from "../db";
import { eq, desc } from "drizzle-orm";
import { users, attendanceCorrectionRequests, attendance,
  type AttendanceCorrectionRequest, type AttendanceCorrectionRequestWithDetails
} from "@shared/schema";

export const attendanceCorrectionMethods = {
  async createAttendanceCorrectionRequest(data: {
    userId: number;
    requestedDate: Date;
    requestedType: string;
    reason: string;
  }): Promise<AttendanceCorrectionRequest> {
    const [req] = await db.insert(attendanceCorrectionRequests).values({
      userId: data.userId,
      requestedDate: data.requestedDate,
      requestedType: data.requestedType,
      reason: data.reason,
      status: "pending",
    }).returning();
    return req;
  },

  async getAttendanceCorrectionRequests(departmentId?: number): Promise<AttendanceCorrectionRequestWithDetails[]> {
    const rows = await db.select().from(attendanceCorrectionRequests)
      .innerJoin(users, eq(attendanceCorrectionRequests.userId, users.id))
      .orderBy(desc(attendanceCorrectionRequests.createdAt));
    let results = rows.map(r => {
      const { password, ...safeUser } = r.users;
      return {
        ...r.attendance_correction_requests,
        user: safeUser,
        reviewerName: null as any,
      };
    });
    if (departmentId) results = results.filter(r => r.user.departmentId === departmentId);
    // hydrate reviewer name
    const allUsers = await db.select().from(users);
    const userMap = new Map(allUsers.map(u => [u.id, u.name]));
    results = results.map(r => ({
      ...r,
      reviewerName: r.reviewedBy ? (userMap.get(r.reviewedBy) || null) : null,
    }));
    return results;
  },

  async getAttendanceCorrectionRequestsByUser(userId: number): Promise<AttendanceCorrectionRequest[]> {
    return await db.select().from(attendanceCorrectionRequests)
      .where(eq(attendanceCorrectionRequests.userId, userId))
      .orderBy(desc(attendanceCorrectionRequests.createdAt));
  },

  async approveAttendanceCorrectionRequest(requestId: number, reviewerId: number): Promise<void> {
    const [req] = await db.select().from(attendanceCorrectionRequests).where(eq(attendanceCorrectionRequests.id, requestId));
    if (!req) throw new Error("Request not found | الطلب غير موجود");
    if (req.status !== "pending") throw new Error("Request already processed | الطلب تمت معالجته مسبقاً");

    // إنشاء سجل الحضور بنفس التاريخ والنوع المطلوب
    const [att] = await db.insert(attendance).values({
      userId: req.userId,
      timestamp: req.requestedDate,
      type: req.requestedType,
      status: "manual-correction",
      latitude: 0,
      longitude: 0,
      selfieUrl: null,
    } as any).returning();

    await db.update(attendanceCorrectionRequests).set({
      status: "approved",
      reviewedBy: reviewerId,
      reviewedAt: new Date(),
      createdAttendanceId: att.id,
    }).where(eq(attendanceCorrectionRequests.id, requestId));
  },

  async rejectAttendanceCorrectionRequest(requestId: number, reviewerId: number, reason?: string): Promise<void> {
    await db.update(attendanceCorrectionRequests).set({
      status: "rejected",
      reviewedBy: reviewerId,
      rejectionReason: reason || null,
      reviewedAt: new Date(),
    }).where(eq(attendanceCorrectionRequests.id, requestId));
  },
};
```

## Changes to `server/storage/index.ts`

### Change B3: Register the new storage methods

Add the import after the existing `shiftChangeRequestMethods` import:

```typescript
import { attendanceCorrectionMethods } from "./attendance-correction-requests";
```

And add the assignment after the existing `Object.assign(DatabaseStorage.prototype, shiftChangeRequestMethods);` line:

```typescript
Object.assign(DatabaseStorage.prototype, attendanceCorrectionMethods);
```

## New file: `server/routes/attendance-correction-requests.ts`

Create this NEW file:

```typescript
import type { Express } from "express";
import { storage } from "../storage";
import { requireAuth, requireSupervisor, logAudit } from "./middleware";

export function registerAttendanceCorrectionRoutes(app: Express) {
  // الموظف يقدم طلب
  app.post("/api/attendance-correction-requests", requireAuth, async (req: any, res) => {
    try {
      const user = await storage.getUser(req.session.userId!);
      if (!user) return res.status(401).json({ message: "Unauthorized" });
      const { requestedDate, requestedType, reason } = req.body;
      if (!requestedDate || !requestedType || !reason)
        return res.status(400).json({ message: "All fields required | كل الحقول مطلوبة" });
      if (requestedType !== "check-in" && requestedType !== "check-out")
        return res.status(400).json({ message: "Invalid type | نوع غير صحيح" });

      const date = new Date(requestedDate);
      const now = new Date();
      if (date > now)
        return res.status(400).json({ message: "Cannot request for future date | لا يمكن الطلب لتاريخ مستقبلي" });

      // ما يقدمش طلب لتاريخ أقدم من 30 يوم
      const thirtyDaysAgo = new Date(now.getTime() - 30 * 24 * 60 * 60 * 1000);
      if (date < thirtyDaysAgo)
        return res.status(400).json({ message: "Cannot request for dates older than 30 days | لا يمكن الطلب لتاريخ أقدم من 30 يوم" });

      const request = await storage.createAttendanceCorrectionRequest({
        userId: user.id,
        requestedDate: date,
        requestedType,
        reason,
      });

      res.status(201).json(request);
    } catch (err) { res.status(500).json({ message: "Failed to submit request | فشل إرسال الطلب" }); }
  });

  // الموظف يشوف طلباته
  app.get("/api/attendance-correction-requests/my", requireAuth, async (req: any, res) => {
    try { res.json(await storage.getAttendanceCorrectionRequestsByUser(req.session.userId!)); }
    catch (err) { res.status(500).json({ message: "Failed to fetch requests | فشل جلب الطلبات" }); }
  });

  // المشرف/الأدمن يشوف الطلبات
  app.get("/api/attendance-correction-requests", requireSupervisor, async (req: any, res) => {
    try {
      const currentUser = await storage.getUser(req.session.userId!);
      let departmentId: number | undefined;
      if (currentUser?.role === "supervisor" && currentUser.departmentId) departmentId = currentUser.departmentId;
      res.json(await storage.getAttendanceCorrectionRequests(departmentId));
    } catch (err) { res.status(500).json({ message: "Failed to fetch requests | فشل جلب الطلبات" }); }
  });

  // موافقة
  app.post("/api/attendance-correction-requests/:id/approve", requireSupervisor, async (req: any, res) => {
    try {
      const requestId = Number(req.params.id);
      const allRequests = await storage.getAttendanceCorrectionRequests();
      const request = allRequests.find((r: any) => r.id === requestId);
      if (!request) return res.status(404).json({ message: "Request not found | الطلب غير موجود" });

      const currentUser = await storage.getUser(req.session.userId!);
      // المشرف يوافق فقط على موظفين قسمه
      if (currentUser?.role === "supervisor" && currentUser.departmentId) {
        if (request.user.departmentId !== currentUser.departmentId)
          return res.status(403).json({ message: "Cannot approve request from another department | لا يمكنك الموافقة على طلب من قسم آخر" });
      }

      await storage.approveAttendanceCorrectionRequest(requestId, req.session.userId!);

      logAudit(req.session.userId!, currentUser?.role || 'unknown', 'approve_attendance_correction',
        'attendance_correction', requestId, request.user?.name,
        { requestedDate: request.requestedDate, requestedType: request.requestedType }, request.user?.departmentId);

      res.json({ success: true, message: "Correction approved and attendance recorded | تمت الموافقة وتسجيل الحضور" });
    } catch (err: any) { res.status(500).json({ message: err.message || "Approval failed | فشل الموافقة" }); }
  });

  // رفض
  app.post("/api/attendance-correction-requests/:id/reject", requireSupervisor, async (req: any, res) => {
    try {
      const { reason } = req.body;
      const requestId = Number(req.params.id);
      const allRequests = await storage.getAttendanceCorrectionRequests();
      const request = allRequests.find((r: any) => r.id === requestId);
      if (!request) return res.status(404).json({ message: "Request not found | الطلب غير موجود" });

      const currentUser = await storage.getUser(req.session.userId!);
      if (currentUser?.role === "supervisor" && currentUser.departmentId) {
        if (request.user.departmentId !== currentUser.departmentId)
          return res.status(403).json({ message: "Cannot reject request from another department | لا يمكنك رفض طلب من قسم آخر" });
      }

      await storage.rejectAttendanceCorrectionRequest(requestId, req.session.userId!, reason);
      logAudit(req.session.userId!, currentUser?.role || 'unknown', 'reject_attendance_correction',
        'attendance_correction', requestId, request.user?.name, { reason }, request.user?.departmentId);
      res.json({ success: true, message: "Request rejected | تم رفض الطلب" });
    } catch (err) { res.status(500).json({ message: "Rejection failed | فشل رفض الطلب" }); }
  });
}
```

## Changes to `server/routes/index.ts`

### Change B4: Register the new route module

Add the import after the existing `registerShiftChangeRequestRoutes` import:

```typescript
import { registerAttendanceCorrectionRoutes } from "./attendance-correction-requests";
```

And in the `registerRoutes` function, add this call after the existing `registerShiftChangeRequestRoutes(app);` line:

```typescript
registerAttendanceCorrectionRoutes(app);
```

## New file: `client/src/pages/employee/attendance-correction.tsx`

Create this NEW page. Mirror the structure of `client/src/pages/employee/shift-request.tsx` — same imports, same Card/Layout/Button/Toast pattern, bilingual labels with `isAr` based on `useLanguage()`. The page should:

- Show a form with: date+time picker (HTML `<input type="datetime-local">`), select for type (check-in / check-out), and a required reason textarea
- Submit button posts to `POST /api/attendance-correction-requests`
- Below the form, list the employee's previous requests from `GET /api/attendance-correction-requests/my`
- Each row shows: requested date, type (with Arabic label), reason, status badge (pending / approved / rejected), reviewer name if available, rejection reason if rejected
- Use the same `EmployeeLayout` wrapper as `shift-request.tsx`
- Page heading: "Attendance Correction Request" / "طلب تصحيح بصمة"
- Form labels in both languages
- All toasts in both languages

The exact field requirements:
- `requestedDate`: HTML datetime-local input, sent as ISO string
- `requestedType`: select with two options: "check-in" (دخول) and "check-out" (خروج)
- `reason`: textarea, required, min 5 chars (validate on submit)

## New file: `client/src/pages/admin/attendance-correction-requests.tsx`

Create this NEW page. Mirror `client/src/pages/admin/shift-change-requests.tsx` structure. The page should:

- Query `GET /api/attendance-correction-requests`
- Display a table with columns: Employee, Department, Requested Date, Type, Reason, Status, Actions
- For each pending row, show two buttons: Approve and Reject
- Approve button POSTs to `/api/attendance-correction-requests/:id/approve`
- Reject button opens a dialog asking for a rejection reason, then POSTs to `/api/attendance-correction-requests/:id/reject` with `{ reason }`
- Use the same Admin/Supervisor layout pattern as the existing `shift-change-requests.tsx`
- Show the reviewer name and reviewedAt for processed requests
- Bilingual throughout

## Changes to `client/src/App.tsx`

### Change B5: Register routes for both pages

Add imports near the existing `EmployeeShiftRequest` import:

```typescript
import EmployeeAttendanceCorrection from "@/pages/employee/attendance-correction";
import AdminAttendanceCorrectionRequests from "@/pages/admin/attendance-correction-requests";
```

Add this route after the existing employee `shift-request` route (around line 69):

```typescript
<ProtectedRoute path="/employee/attendance-correction" component={EmployeeAttendanceCorrection} allowedRoles={["employee"]} />
```

Add this route in the supervisor section (after the existing supervisor shift-change route):

```typescript
<ProtectedRoute path="/supervisor/attendance-correction-requests" component={AdminAttendanceCorrectionRequests} allowedRoles={["supervisor"]} />
```

And in the admin section (after the existing admin shift-change route):

```typescript
<ProtectedRoute path="/admin/attendance-correction-requests" component={AdminAttendanceCorrectionRequests} adminOnly />
```

## Changes to sidebars

### Change B6: Add sidebar entry for employee

In `client/src/components/layout/employee-layout.tsx`, after the existing shift-request `Link` block (around line 83), add a new `Link` block with the same styling pattern, pointing to `/employee/attendance-correction`. Use an appropriate Lucide icon (e.g. `ClipboardEdit` or `FileEdit`). Label: "Attendance Correction" / "تصحيح بصمة".

### Change B7: Add sidebar entry for admin and supervisor

In the admin sidebar (find it the same way as the employee sidebar — look in `client/src/components/layout/admin-layout.tsx` or wherever admin nav is defined), add an entry pointing to `/admin/attendance-correction-requests`. Label: "Attendance Corrections" / "تصحيح البصمات".

In the supervisor sidebar/page (`client/src/pages/supervisor.tsx` or similar), add an entry pointing to `/supervisor/attendance-correction-requests`. Same label.

## Database migration

After all code changes, run:

```bash
npm run db:push
```

This will create the new `attendance_correction_requests` table via Drizzle.

---

# DO NOT TOUCH

- `processAttendance` in `server/routes/middleware.ts`
- `handleOpenShiftOnCheckIn` in `server/routes/attendance.ts`
- Geofencing logic
- Exit time limit logic
- 15-minute minimum rule
- Manual attendance feature (separate from this)
- Business trips, announcements, evaluation, public holidays, payroll, leaves, location requests, shift change requests
- Reports logic
- Any unrelated storage method

# After all changes

```bash
npm run db:push
npm run dev
```

Then test:
1. PART A — login as employee with night shift, perform consecutive night shifts, verify checkout on day 3 succeeds
2. PART B — login as employee → go to "Attendance Correction" page → submit a request for yesterday → login as supervisor → approve the request → verify attendance record is created with correct date/type
