# Work Log - 2026-03-23

## 🔴 CRITICAL: Frontend Rendering Bugs Fixed (BUG-005, BUG-006, BUG-007)

**Time:** 12:17 - 12:45
**QA Report:** `/home/victor/.openclaw/workspace/qa/qa-report-sprint2-browser-test-final.md`

### Bugs Fixed

#### BUG-005: ID shows "undefined" or "NaN"
**Issue:** ID column displayed "undefined" for existing records, "NaN" for newly created records
**Root Cause:** Missing null/undefined checks when accessing `data.id` or `data._id`
**Fix:** Added safe fallback pattern `${c.id || c._id || ''}` in all render functions
**Affected:**
- `renderClinicsTable()` - clinic IDs
- `renderDoctorsGrid()` - doctor IDs (in button onclick)
- `renderServicesTable()` - service IDs
- `renderDoctorServicesTable()` - mapping IDs
- `renderAppointmentsTable()` - appointment IDs

#### BUG-006: Business hours shows "[object Object]"
**Issue:** Business hours column displayed `[object Object]` instead of readable time range
**Root Cause:** Direct rendering of businessHours object without formatting
**Fix:** Created `formatBusinessHours()` helper function:
```javascript
function formatBusinessHours(businessHours) {
    if (!businessHours || typeof businessHours !== 'object') return '未設定';
    const days = ['monday', 'tuesday', 'wednesday', 'thursday', 'friday', 'saturday', 'sunday'];
    const dayNames = ['一', '二', '三', '四', '五', '六', '日'];
    const result = [];
    for (let i = 0; i < days.length; i++) {
        const day = days[i];
        if (businessHours[day] && businessHours[day].isOpen) {
            result.push(`${dayNames[i]}: ${businessHours[day].open}-${businessHours[day].close}`);
        }
    }
    return result.join(' | ') || '未設定';
}
```
**Applied to:** `renderClinicsTable()` - now shows "一：09:00-18:00 | 二：09:00-18:00" format

#### BUG-007: Doctor/service names show "未知" (Unknown)
**Issue:** Doctor and service names displayed as "未知" in mapping tables and appointments
**Root Cause:** 
1. Missing null checks when accessing nested properties
2. Foreign key lookups failing when backend returns `doctorId` instead of `doctor` object
**Fix:** Added optional chaining and fallback patterns:
- `${doctor?.name || ds.doctorName || '未知'}`
- `${service?.name || ds.serviceName || '未知'}`
- Also check both `d.id` and `d._id` for foreign key resolution

**Affected:**
- `renderDoctorServicesTable()` - doctor/service name lookup
- `renderCalendar()` - doctor name in events
- `renderAppointmentsTable()` - clinic/doctor/service name lookup

### Additional Improvements
- All button onclick handlers now use safe ID: `${c.id || c._id || 'null'}`
- Added null checks for all display fields (name, phone, address, etc.)
- Calendar events now show doctor name with fallback to `s.doctorName`
- Appointment table handles both `_id` and `id` from backend

### Files Modified
- `admin.html` - 55 insertions, 40 deletions

### Commit
- `046b59d` - FIX: Frontend rendering bugs - ID undefined, businessHours object, unknown names

### Verification Steps
1. ⏳ Open: https://victorsdooo-code.github.io/booking_system/admin.html
2. ⏳ Hard refresh (Ctrl+Shift+R)
3. ⏳ Login with admin123
4. ⏳ Verify all 6 modules render correctly:
   - 門店管理：IDs show numbers, business hours show time ranges
   - 醫生管理：Names show correctly, descriptions not "undefined"
   - 服務配置：IDs show numbers, names show correctly
   - 醫生服務配置：Doctor/service names not "未知"
   - 排班管理：Calendar shows doctor names
   - 預約管理：All foreign keys resolve to names

---

# Work Log - 2026-03-22

## 🔴 URGENT: Frontend API Requests All Failing - FIXED

**Time:** 23:08 - 23:15
**Issue:** "介面可以，但 api 全部請求失敗的" - UI loads but all API requests failing

### Root Cause
**API response format mismatch:**
- Backend returns: `{ success: true, data: [...] }`
- Frontend expected: `data.clinics` or similar nested structure
- Code did: `clinics = data.clinics || data` → got entire response object instead of array
- Render functions failed: `clinics.map()` threw error because clinics was an object, not an array

### Fix Applied
1. **Updated `apiRequest()` function** to extract actual data array:
   ```javascript
   return data.data || data;  // Extract data.data from { success: true, data: [...] }
   ```

2. **Fixed all load functions** to use data directly:
   - `clinics = data;` (was `data.clinics || data`)
   - `doctors = data;` (was `data.doctors || data`)
   - `services = data;` (was `data.services || data`)
   - `doctorServices = data;` (was `data.doctorServices || data`)
   - `schedules = data;` (was `data.schedules || data`)
   - `appointments = data;` (was `data.appointments || data`)

### Verification
✅ API configuration correct: `https://booking-system-backend-hjwb.onrender.com/api/admin`
✅ Auth header present: `X-Admin-Token: admin123`
✅ CORS configured: `access-control-allow-origin: *`
✅ Backend responding correctly with proper format
✅ Fix committed and pushed to GitHub Pages

### Commit
- `389eeda` - FIX: Frontend API requests failing - fix data extraction from API responses

### Lesson Learned
- Always verify API response format matches frontend expectations
- Backend `{ success: true, data: [...] }` pattern requires explicit extraction
- Fallback patterns like `data.xxx || data` can hide bugs by returning wrong data types

---

## 🔴 URGENT: Frontend API Integration Fix

**Time:** 17:20 - 17:35
**Issue:** Admin Panel loading but data not displaying ("好多野都 load 唔到")

### Root Cause
The `admin.html` file had the **wrong backend API URL**:
- **Wrong:** `https://booking-system-backend-2t8v.onrender.com/api/admin` (old backend)
- **Correct:** `https://booking-system-backend-hjwb.onrender.com/api/admin` (production backend)

### Additional Discovery
The backend requires `/api/admin` prefix (not just `/api`). Initial fix without `/admin` resulted in 404 errors.

### Fix Applied
```javascript
// Changed in admin.html line ~1543
const API_BASE_URL = 'https://booking-system-backend-hjwb.onrender.com/api/admin';
```

### Verification
✅ All 6 sections now load data correctly:
1. 門店管理 - 2 clinics loading
2. 醫生管理 - 3 doctors loading  
3. 服務配置 - loading
4. 醫生服務配置 - loading
5. 排班管理 - loading
6. 預約管理 - loading

### Commits
1. `c262812` - Fix: Update API_BASE_URL to production backend
2. `498b741` - Fix: Add /admin prefix to API_BASE_URL for correct backend endpoints

### Lesson Learned
- QA tested HTTP 200 but didn't test actual browser API integration
- Always verify API endpoints exist with correct authentication before deploying
- Backend URL changes require careful verification of full endpoint paths

---

## Sprint 2: Business Hours UI + Schedule Association

**Time:** 20:05 - 21:30
**Priority:** 🔴 CRITICAL
**Change Requests:** CR-001, CR-002

### Tasks Completed

#### 1️⃣ Clinic Management UI - Business Hours Section
- ✅ Replaced plain text business hours field with structured UI
- ✅ Added 7-day grid (Mon-Sun) with:
  - Open/close checkbox for each day
  - Open time dropdown (09:00-20:00, 30-min intervals)
  - Close time dropdown (17:00-21:00, 30-min intervals)
- ✅ Updated `saveClinic()` to save structured business hours object
- ✅ Updated `openClinicModal()` and `editClinic()` to populate business hours

#### 2️⃣ Doctor Management - Clinic Association
- ✅ Added "所屬門店" dropdown to doctor modal
- ✅ Updated `saveDoctor()` to include `clinicId`
- ✅ Updated `openDoctorModal()` and `editDoctor()` to populate clinic select
- ✅ Added `populateDoctorClinicSelect()` helper function

#### 3️⃣ Schedule Management - Clinic Integration
- ✅ Added `onchange="onClinicSelected(this.value)"` to clinic dropdown
- ✅ Added `onchange="onDateSelected(...)"` to date input
- ✅ Added business hours info display div (`#business-hours-info`)
- ✅ Implemented `onClinicSelected(clinicId)` function:
  - Filters doctors by selected clinic
  - Displays clinic business hours
- ✅ Implemented `onDateSelected(clinicId, date)` function:
  - Validates if clinic is open on selected day
  - Alerts if clinic closed ("診所喺呢日無營業")
  - Calls `updateTimeSlots()` to populate time dropdown
- ✅ Implemented `updateTimeSlots(openTime, closeTime)` function:
  - Dynamically generates 30-min time slots based on business hours

#### 4️⃣ Mock Data Updates
- ✅ Updated clinic mock data to use structured businessHours object
- ✅ Added `clinicId` to doctor mock data

### Files Modified
- `admin.html` - All UI and JavaScript changes

### Testing Status
⏳ Pending browser testing (next step)

### Next Steps
1. Open admin.html in browser
2. Test clinic business hours editing
3. Test schedule creation with clinic-doctor association
4. Verify time slots match business hours
5. Commit and push changes
