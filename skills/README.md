# SmartBill Approve — Skills Index (สารบัญเอกสาร)

> **Reverse-Engineered Documentation Suite**
> Generated: 2026-08-16
> GAS Project ID: `1y0SFUCYYa_njyHGH7jWr2hdB5T6wPHnAIOt4FwLPBprNq3XNy-Kx7D7w`

---

## เอกสารทั้งหมด

| # | File | Description | Content Summary |
|---|------|-------------|-----------------|
| 1 | [`01_system_overview.md`](file:///c:/Antigravity_Data/SmartBill_Approve/skills/01_system_overview.md) | **ภาพรวมระบบ** | Technology stack, architecture diagram, Google Sheets structure, deployment config, key constants |
| 2 | [`02_backend_api_spec.md`](file:///c:/Antigravity_Data/SmartBill_Approve/skills/02_backend_api_spec.md) | **Backend API Spec** | ทุก API action (checkUser, register, getPending, updateStatus), request/response format, column mapping |
| 3 | [`03_frontend_ui_spec.md`](file:///c:/Antigravity_Data/SmartBill_Approve/skills/03_frontend_ui_spec.md) | **Frontend UI Spec** | Screen layout, component detail, image handling, CSS/styling, design system |
| 4 | [`04_user_flow_and_business_logic.md`](file:///c:/Antigravity_Data/SmartBill_Approve/skills/04_user_flow_and_business_logic.md) | **User Flow & Business Logic** | Complete flow diagram, business rules, state machine, data flow |
| 5 | [`05_security_and_issues.md`](file:///c:/Antigravity_Data/SmartBill_Approve/skills/05_security_and_issues.md) | **Security & Issues** | Security concerns, performance issues, functional bugs, improvement priorities |
| 6 | [`06_deployment_guide.md`](file:///c:/Antigravity_Data/SmartBill_Approve/skills/06_deployment_guide.md) | **Deployment Guide** | gas_sync.py workflow, project IDs, LINE LIFF setup, environment dependencies |

---

## สรุปย่อระบบ (Quick Reference)

**ระบบนี้คืออะไร**: LINE LIFF Web App สำหรับอนุมัติบิลเบิกจ่าย

**มีกี่ไฟล์ source**: 3 ไฟล์ (code.js, index.html, appsscript.json)

**ใช้อะไรเก็บข้อมูล**: Google Sheets (2 sheets: `Approve_users`, `TaxData`)

**มี API กี่ตัว**: 4 actions (checkUser, register, getPending, updateStatus)

**มีหน้าจอกี่หน้า**: 2 screens (Registration, Approval List) + Loading overlay

**Frontend อยู่ที่ไหน**: GitHub Pages — `https://pingly69.github.io/smartbill_approve/`

**Backend อยู่ที่ไหน**: Google Apps Script Web App (API only, ไม่ serve HTML)

**ทำไมแยก Frontend?**: เพราะ LINE LIFF iframe restriction — GAS Web App ทำงานใน nested iframe ซึ่ง LIFF SDK init ไม่ได้

**สิ่งที่ต้องระวัง**: ดู `05_security_and_issues.md` — มี security concerns ระดับ Critical 4 รายการ


---

## ข้อเสนอแนะสำหรับเฟสถัดไป

> ตามที่วิเคราะห์ใน `05_security_and_issues.md` แนะนำลำดับการปรับปรุง:

1. **Phase 1 — Quick Wins**: Input validation, race condition fix, XSS prevention
2. **Phase 2 — Security**: LIFF token verification, invite-based auth, rate limiting
3. **Phase 3 — Performance & UX**: Pagination, optimized reads, LINE notifications
