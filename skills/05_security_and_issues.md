# SmartBill Approve — Security Analysis & Known Issues

> Reverse-Engineered from source code
> Generated: 2026-08-16

---

## 1. Security Concerns

### 🔴 Critical

| # | Issue | Location | Description | Recommendation |
|---|-------|----------|-------------|----------------|
| S1 | **No API Authentication** | `code.js:10` | `doPost` ไม่มีการตรวจสอบ origin หรือ token ใดๆ ทุกคนที่รู้ Web App URL สามารถเรียก API ได้ | เพิ่ม LIFF token verification หรือ shared secret |
| S2 | **Password Stored in Plain Text** | Sheet `Approve_users` | รหัสผ่านเก็บเป็น plain text ใน Google Sheets ไม่มีการ hash | ใช้ hashing หรือเปลี่ยนไปใช้ invite-link แทน |
| S3 | **Anyone Anonymous Access** | `appsscript.json:8` | Web App ตั้งค่าเป็น `ANYONE_ANONYMOUS` = ไม่ต้อง Google Login | เปลี่ยนเป็น `ANYONE` (ต้อง Google Login) ถ้าเป็นไปได้ |
| S4 | **Auto-Publish Private Files** | `code.js:113-120` | ไฟล์ภาพบิลถูกเปลี่ยนจาก private → public อัตโนมัติ ไม่เคย revert กลับ | เพิ่ม time-limited access หรือ signed URL |

### 🟡 Medium

| # | Issue | Location | Description | Recommendation |
|---|-------|----------|-------------|----------------|
| S5 | **No Rate Limiting** | `code.js:10` | ไม่มีการจำกัดจำนวนครั้งการเรียก API | เพิ่ม rate limiting logic |
| S6 | **No Input Validation** | `code.js:11` | ไม่มีการตรวจสอบ input format (line_uid, password, recordIds) | เพิ่ม validation schema |
| S7 | **Password Brute Force** | `code.js:52-64` | ไม่มี lockout mechanism, สามารถลองรหัสผ่านได้ไม่จำกัด | เพิ่ม attempt counter + lockout |
| S8 | **No CORS Protection** | `code.js:32-36` | Response เป็น JSON แต่ไม่มี CORS headers, GAS จัดการ CORS เอง | ตรวจสอบ GAS CORS behavior |

### 🟢 Low

| # | Issue | Location | Description | Recommendation |
|---|-------|----------|-------------|----------------|
| S9 | **LINE UID Spoofing** | `index.html:121` | LINE UID ส่งมาจาก client-side LIFF profile, ไม่ได้ verify server-side | ใช้ LIFF ID Token + server-side verification |
| S10 | **Error Message Leaks** | `code.js:34-36` | Error message ส่งกลับ client แบบ raw, อาจ leak internal info | ใช้ generic error messages |

---

## 2. Performance Issues

| # | Issue | Location | Description | Recommendation |
|---|-------|----------|-------------|----------------|
| P1 | **Full Table Scan** | `code.js:44,56,77,128` | ทุก API call อ่านข้อมูลทั้ง sheet (`getDataRange().getValues()`) ไม่มี indexing | จำกัด range ที่อ่าน หรือใช้ cache |
| P2 | **N+1 Drive API Calls** | `code.js:84-94` | ทุกรายการ pending เรียก `setFilePublic` = N DriveApp calls ต่อ request | batch permission update หรือ pre-set permissions |
| P3 | **No Pagination** | `code.js:106` | `LIMIT_PER_PAGE = 5` แต่ยังต้อง scan ทั้ง sheet เพื่อหา 5 รายการ | เพิ่ม index column หรือ offset tracking |
| P4 | **No Client Caching** | `index.html:6-8` | Cache headers ปิดหมด (`no-cache, no-store, must-revalidate`) | พิจารณา cache static assets |

---

## 3. Functional Bugs / Risks

| # | Issue | Location | Description | Impact |
|---|-------|----------|-------------|--------|
| B1 | **Password Overwritten by DisplayName** | `code.js:59` | หลัง register, password column ถูกแทนที่ด้วย LINE display name ทำให้ไม่สามารถ re-register ได้ | Admin ต้องแก้ Sheet ด้วยมือหากต้องการ reset |
| B2 | **No Status Re-check Before Update** | `code.js:122-137` | `updateBatchStatus` ไม่ได้ตรวจว่า status ปัจจุบันยังเป็น `pending` ก่อน update → race condition | อาจ approve รายการที่ถูก reject ไปแล้วโดยคนอื่น |
| B3 | **RecordId Comparison Type Mismatch** | `code.js:129-130` | ใช้ `String()` แปลง sheet data แต่ `recordIds` จาก frontend ไม่ได้แปลง → อาจ match ไม่ถูก | ควรแปลง type ทั้งสองฝั่ง |
| B4 | **First Password Match Wins** | `code.js:56-57` | ถ้ามี password ซ้ำกันในหลาย row จะ match row แรกเสมอ | ทำให้ user ผิดคนอาจได้สิทธิ์ |
| B5 | **Limit Without Offset** | `code.js:106` | แสดง 5 รายการแรกที่เจอ ไม่มี pagination offset | ถ้ามี > 5 pending items, user ต้อง approve ชุดแรกก่อนจึงเห็นชุดถัดไป |
| B6 | **Silent Error on File Permission** | `code.js:92-93, 117-119` | error จาก Drive permission ถูก swallow silently | รูปอาจไม่แสดงโดยไม่มี warning |

---

## 4. Code Quality Issues

| # | Issue | Location | Description |
|---|-------|----------|-------------|
| Q1 | **Magic Numbers** | `code.js:73` | Column indices เป็น hard-coded numbers (19, 16, 9, 13, ...) ไม่มี enum หรือ constant |
| Q2 | **Mixed Column Indexing** | `code.js:73 vs 127-133` | `getPendingData` ใช้ 0-based index, `updateBatchStatus` ใช้ mix ของ 0-based (data read) และ 1-based (sheet write) |
| Q3 | **No Error Handling in Frontend** | `index.html:166-214` | `loadData` rendering ไม่มี try-catch, ถ้า data format ผิดจะพังทั้ง page |
| Q4 | **DOM Injection via innerHTML** | `index.html:213` | ใช้ string concatenation + innerHTML = potential XSS ถ้า data มี HTML |
| Q5 | **No Logging/Audit** | `code.js` | ไม่มี structured logging นอกจาก `console.error` สำหรับ file permission |
| Q6 | **Hardcoded URLs** | `index.html:84` | Web App URL และ LIFF ID hardcoded ในหน้า HTML |

---

## 5. Improvement Priorities

### Phase 1 (Quick Wins)
- [ ] เพิ่ม input validation ให้ทุก API action
- [ ] เพิ่ม status re-check ก่อน update (ป้องกัน race condition)
- [ ] แก้ XSS risk โดยใช้ DOM API แทน innerHTML
- [ ] เพิ่ม structured error logging

### Phase 2 (Security Hardening)
- [ ] Implement LIFF ID Token verification server-side
- [ ] เปลี่ยน password system เป็น invite-link based
- [ ] เพิ่ม rate limiting
- [ ] ใช้ signed URL สำหรับ image access

### Phase 3 (Performance & UX)
- [ ] เพิ่ม pagination ที่แท้จริง (offset-based)
- [ ] Optimize sheet reads (read specific range)
- [ ] เพิ่ม real-time notification ผ่าน LINE Messaging API
- [ ] เพิ่ม approval history view
