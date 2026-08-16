# SmartBill Approve — User Flow & Business Logic

> Reverse-Engineered from source code
> Generated: 2026-08-16

---

## 1. Complete User Flow Diagram

```
┌──────────────────┐
│  เปิด LIFF App   │
│  (LINE Browser)  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐     ┌──────────────────┐
│  LIFF Init       │────▶│  Not Logged In?  │
│  liffId:         │     │  → liff.login()  │
│  2009018471-...  │     │  → redirect back │
└────────┬─────────┘     └──────────────────┘
         │ logged in
         ▼
┌──────────────────┐
│  getProfile()    │
│  → userId        │
│  → displayName   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  checkUser(uid)  │
│  API call        │
└────────┬─────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
 authorized  not_found
    │         │
    │         ▼
    │    ┌──────────────────┐
    │    │  Registration    │
    │    │  Screen          │
    │    │  → input password│
    │    └────────┬─────────┘
    │             │
    │             ▼
    │    ┌──────────────────┐
    │    │  register()      │
    │    │  → match password│
    │    │  → save UID      │
    │    │  → save name     │
    │    └────────┬─────────┘
    │             │ success
    │             │
    ▼             ▼
┌──────────────────────────┐
│  showListScreen()        │
│  → show #list-screen     │
│  → show LINE UID         │
│  → call loadData()       │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  loadData()              │
│  → getPending API        │
│  → filter: pending +     │
│    matching approver     │
│  → auto-set file public  │
│  → max 5 items           │
└────────────┬─────────────┘
             │
    ┌────────┴────────┐
    │                 │
    ▼                 ▼
 items > 0        items = 0
    │                 │
    ▼                 ▼
┌──────────┐   ┌────────────┐
│ Show     │   │ Show       │
│ Cards +  │   │ "จัดการ    │
│ Action   │   │ เรียบร้อย" │
│ Bar      │   │ message    │
└────┬─────┘   └────────────┘
     │
     ▼
┌──────────────────────────┐
│  User selects checkbox   │
│  (one or more cards)     │
└────────────┬─────────────┘
             │
    ┌────────┴────────┐
    │                 │
    ▼                 ▼
 Approve          Reject
    │                 │
    ▼                 ▼
┌──────────────────────────┐
│  confirm dialog          │
│  "Confirm Approve/Reject │
│   สำหรับ N รายการ?"      │
└────────────┬─────────────┘
             │ OK
             ▼
┌──────────────────────────┐
│  updateStatus API        │
│  → items: [recordIds]    │
│  → status: Approved/     │
│    Rejected              │
│  → line_uid: approver    │
│  → writes:               │
│    - status column       │
│    - approver UID        │
│    - timestamp           │
└────────────┬─────────────┘
             │ success
             ▼
┌──────────────────────────┐
│  loadData() again        │
│  → refresh list          │
│  (approved items no      │
│   longer appear)         │
└──────────────────────────┘
```

---

## 2. Business Rules

### 2.1 Authentication & Authorization

| Rule | Description |
|------|-------------|
| LINE Login Required | ต้อง login ผ่าน LINE ก่อนเข้าใช้งาน (LIFF enforces) |
| One-Time Registration | ใช้ password matching เพื่อจับคู่ LINE UID กับ approver slot |
| Password Overwrite | หลัง register สำเร็จ, password ถูกแทนที่ด้วย displayName |
| Re-registration | ไม่สามารถ re-register ได้ (password ถูก overwrite แล้ว) |
| Device Change | ถ้า approver เปลี่ยน LINE account/device จะต้องแก้ Sheet ด้วยมือ |

### 2.2 Data Visibility

| Rule | Description |
|------|-------------|
| Filtered by Approver | แต่ละ approver เห็นเฉพาะบิลที่ assign ให้ตัวเอง (`reqBy === approve_request`) |
| Status Filter | แสดงเฉพาะ status = `pending` |
| Page Limit | แสดงสูงสุด 5 รายการต่อครั้ง |
| No Pagination | ไม่มีปุ่ม next page, ต้อง approve ชุดปัจจุบันก่อนจึงจะเห็นชุดถัดไป |

### 2.3 Approval Process

| Rule | Description |
|------|-------------|
| Batch Operation | เลือกหลายรายการแล้ว approve/reject พร้อมกัน |
| Confirmation Required | แสดง confirm dialog ก่อนดำเนินการทุกครั้ง |
| Status Values | `pending` → `Approved` หรือ `Rejected` |
| Audit Trail | บันทึก: สถานะ, LINE UID ผู้อนุมัติ, timestamp |
| Irreversible | ไม่มีฟีเจอร์ undo หรือกลับสถานะ |

### 2.4 Image Access

| Rule | Description |
|------|-------------|
| Auto-Public | ไฟล์ภาพถูกเปิดเป็น public อัตโนมัติตอนดึงข้อมูล |
| Thumbnail | ใช้ Google Drive thumbnail API ขนาด 400px |
| Full View | กดรูปเปิดดู original URL ใน browser ใหม่ |
| Fallback | หากโหลดรูปไม่ได้ แสดง placeholder "VIEW BILL" |

---

## 3. State Machine

```
                         ┌─────────┐
                         │ pending │
                         └────┬────┘
                              │
               ┌──────────────┼──────────────┐
               │                             │
               ▼                             ▼
        ┌────────────┐                ┌────────────┐
        │  Approved  │                │  Rejected  │
        └────────────┘                └────────────┘
```

**สถานะเป็น one-way** — ไม่สามารถกลับไป pending ได้จากระบบนี้

---

## 4. Data Flow Summary

```
[External System] ── เขียนข้อมูลบิลลง Sheet ──▶ [Google Sheets: TaxData]
                                                   │ status = "pending"
                                                   │ reqBy = "approver_name"
                                                   │
                                                   ▼
                                            [SmartBill Approve]
                                                   │
                                              Approve/Reject
                                                   │
                                                   ▼
                                            [Google Sheets: TaxData]
                                                   │ status = "Approved"/"Rejected"
                                                   │ line_uid = approver UID
                                                   │ timestamp = now
                                                   │
                                                   ▼
                                            [External System reads result]
```

> **สิ่งที่ระบบนี้ไม่ทำ**: ไม่ได้สร้างข้อมูลบิล, ไม่ได้ส่ง notification, ไม่ได้ทำ post-approval processing
> ระบบนี้เป็นเพียง **approval interface** เท่านั้น
