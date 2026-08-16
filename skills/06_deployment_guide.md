# SmartBill Approve — Deployment & DevOps Guide

> Reverse-Engineered from source code
> Generated: 2026-08-16

---

## 1. Project Identifiers

| Item | Value |
|------|-------|
| **GAS Script ID** | `1y0SFUCYYa_njyHGH7jWr2hdB5T6wPHnAIOt4FwLPBprNq3XNy-Kx7D7w` |
| **Google Sheet ID** | `1amztKC_QEVv9H7u6ubGCJYEHCHo0NWnJhT6ksNQCpnA` |
| **LIFF ID** | `2009018471-gUZQupT4` |
| **GAS Web App URL** | `https://script.google.com/macros/s/AKfycbyCnwzQVEQect0BhrdnzF62beYGfF9I-Sy4sS7G4Khbwgrv3BcM_fz9nzR5LjKHwh1_gQ/exec` |
| **GitHub Repo** | `https://github.com/pingly69/smartbill_approve` |
| **GitHub Pages URL** | `https://pingly69.github.io/smartbill_approve/` |
| **LIFF Endpoint** | `https://pingly69.github.io/smartbill_approve/` (ชี้ไป GitHub Pages ไม่ใช่ GAS) |

---

## 2. Development Workflow

### Prerequisites
- Python 3.x installed
- `~/.clasprc.json` with valid Google OAuth credentials (from `clasp login`)
- อยู่ใน directory ที่มี `.clasp.json`

### Pull (ดึง source ลง local)
```bash
cd c:\Antigravity_Data\SmartBill_Approve
python gas_sync.py pull
```
**Result**: ดึง 3 ไฟล์ลงมา:
- `appsscript.json` — Project manifest
- `code.js` — Server-side code
- `index.html` — Frontend

### Push (ส่ง local → GAS + Auto-deploy)
```bash
cd c:\Antigravity_Data\SmartBill_Approve
python gas_sync.py push
```
**Result**: 
1. อัพโหลดไฟล์ทั้งหมดไปยัง GAS project
2. สร้าง Version ใหม่อัตโนมัติ
3. อัพเดต Deployment ทุกตัวให้ชี้ไป Version ล่าสุด

### Files Included in Push (Backend Only)
| File Pattern | GAS Type | Included |
|-------------|----------|----------|
| `*.js`, `*.gs` | SERVER_JS | ✅ |
| `*.html` | HTML | ✅ (ยกเว้นที่ exclude) |
| `appsscript.json` | JSON | ✅ (always) |
| `gas_sync.py` | - | ❌ (excluded) |
| `app.js`, `index.html`, `styles.css` | - | ❌ (excluded) |
| `.*` (dotfiles) | - | ❌ (excluded) |

### Excluded Files (hardcoded in `gas_sync.py:102`)
```python
if fname in ('appsscript.json', 'gas_sync.py', 'app.js', 'index.html', 'styles.css') or fname.startswith('.'):
    continue
```
> ✅ **Intentional**: `index.html` ถูก exclude จากการ push โดยตั้งใจ เพราะ frontend ถูก host แยกบน GitHub Pages 
> ไม่ต้อง push ขึ้นไป GAS เพราะ GAS ทำหน้าที่เป็นแค่ API backend เท่านั้น

---

## 3. Split Deployment Architecture

ระบบนี้แยก deployment เป็น 2 ส่วนเพื่อแก้ปัญหา LINE LIFF iframe:

```
┌────────────────────────────────────────────────┐
│              LOCAL DEVELOPMENT                    │
│                                                    │
│  index.html  ─────  git push  ───▶ GitHub Pages    │
│  (Frontend)       (main branch)   (auto-deploy)    │
│                                                    │
│  code.js     ─── gas_sync.py ──▶ GAS Web App     │
│  (Backend)       push             (auto-deploy)    │
│                                                    │
└────────────────────────────────────────────────┘
```

### ทำไมต้องแยก? (LINE LIFF iframe restriction)

**ปัญหา**: GAS Web App เมื่อ deploy จะ serve HTML ผ่าน `script.google.com` ซึ่งทำงานผ่าน redirect + iframe ซ้อนกันหลายชั้น
เมื่อ LINE LIFF โหลด Web App เข้ามาใน LINE Browser จะมี iframe ซ้อนกันอีกชั้น ทำให้:
- `liff.init()` ไม่สามารถทำงานได้ถูกต้อง (ต้องอยู่ top-level window)
- `liff.login()` redirect ไม่ทำงานใน nested iframe

**วิธีแก้**: Host `index.html` บน GitHub Pages ซึ่งเป็น static hosting ที่ serve HTML โดยตรง (ไม่ผ่าน iframe)
แล้วให้ frontend เรียก GAS Web App URL ผ่าน `fetch()` เป็น API เท่านั้น

### Frontend Deployment (แก้ UI)
```bash
# 1. แก้ไข index.html
# 2. Push ขึ้น GitHub
cd c:\Antigravity_Data\SmartBill_Approve
git add index.html
git commit -m "update UI"
git push origin main
# 3. GitHub Pages auto-deploy (ไม่กี่นาที)
```

### Backend Deployment (แก้ API logic)
```bash
# 1. แก้ไข code.js
# 2. Push ขึ้น GAS
cd c:\Antigravity_Data\SmartBill_Approve
python gas_sync.py push
# 3. Auto-version + auto-deploy
```

---

## 3. gas_sync.py Architecture

### Token Management
```
~/.clasprc.json → read refresh_token
      │
      ▼
Google OAuth2 Token Endpoint → get access_token
      │
      ▼
Write back access_token to ~/.clasprc.json
```

### API Endpoints Used
| Operation | Method | URL |
|-----------|--------|-----|
| Pull Content | GET | `https://script.googleapis.com/v1/projects/{id}/content` |
| Push Content | PUT | `https://script.googleapis.com/v1/projects/{id}/content` |
| Create Version | POST | `https://script.googleapis.com/v1/projects/{id}/versions` |
| List Deployments | GET | `https://script.googleapis.com/v1/projects/{id}/deployments` |
| Update Deployment | PUT | `https://script.googleapis.com/v1/projects/{id}/deployments/{depId}` |

---

## 4. LINE LIFF Configuration

### Required Setup in LINE Developers Console

| Setting | Value |
|---------|-------|
| LIFF App ID | `2009018471-gUZQupT4` |
| Endpoint URL | Web App URL (deployed GAS URL) |
| Scope | `profile` (minimum required) |
| BotLink | Optional |

### LIFF SDK
```html
<script src="https://static.line-scdn.net/liff/edge/2/sdk.js"></script>
```

---

## 5. Google Sheets Access

### Required Permissions (for deployer account)
- `SpreadsheetApp` — read/write access to Sheet ID
- `DriveApp` — manage file permissions
- `ContentService` — serve JSON responses
- `Utilities` — date formatting

### Sheet URL
```
https://docs.google.com/spreadsheets/d/1amztKC_QEVv9H7u6ubGCJYEHCHo0NWnJhT6ksNQCpnA/edit
```

---

## 6. Environment Dependencies

```
┌─────────────────────────────────────────────────────┐
│                  Dependencies                        │
├─────────────────────────────────────────────────────┤
│ Runtime:                                             │
│   ├─ Google Apps Script V8 Engine                    │
│   ├─ LINE LIFF SDK v2 (CDN)                         │
│   └─ TailwindCSS (CDN)                              │
│                                                      │
│ Services:                                            │
│   ├─ Google Sheets (SpreadsheetApp)                  │
│   ├─ Google Drive (DriveApp)                         │
│   ├─ LINE Login (LIFF)                               │
│   └─ Google OAuth2 (for deployment)                  │
│                                                      │
│ Dev Tools:                                           │
│   ├─ Python 3.x (gas_sync.py)                       │
│   └─ clasp credentials (~/.clasprc.json)             │
└─────────────────────────────────────────────────────┘
```
