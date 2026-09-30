# Part 17: Account Security - ความปลอดภัยของ LINE OA

## บทนำ

ความปลอดภัยของ LINE Official Account เป็นเรื่องที่มักถูกมองข้ามโดยเจ้าของธุรกิจ แต่การถูก Hack หรือใช้ Account ในทางที่ผิดสามารถสร้างความเสียหายอย่างมากทั้งต่อชื่อเสียงและรายได้ของธุรกิจ

ในบทนี้ เราจะครอบคลุมการรักษาความปลอดภัยทุกด้านของ LINE OA ตั้งแต่การป้องกันการเข้าถึงโดยไม่ได้รับอนุญาต ไปจนถึงการปฏิบัติตามกฎหมาย PDPA ของไทย

---

## 17.1 Security Best Practices สำหรับ LINE OA

### ภาพรวมของ Attack Vectors

ช่องทางที่ Account LINE OA อาจถูกโจมตี:

```
Attack Vectors:
├── Account Credential Theft (ขโมยรหัสผ่าน)
│   ├── Phishing Emails
│   ├── Password Reuse
│   └── Weak Passwords
├── Unauthorized Operator Access (Operator ที่ไม่ได้รับอนุญาต)
│   ├── Former Employee ที่ยังมี Access
│   ├── Misconfigured Permissions
│   └── Shared Credentials
├── API Token Compromise (API Token หลุด)
│   ├── Hardcoded in Source Code
│   ├── Leaked in Public Repositories
│   └── Insecure Storage
└── Social Engineering
    ├── Fake LINE Support
    └── Impersonation Attacks
```

### Security Checklist พื้นฐาน

```
✅ ใช้ Strong Password (อย่างน้อย 12 ตัวอักษร)
✅ เปิด Two-Factor Authentication (2FA)
✅ ตรวจสอบ Operator List ทุกเดือน
✅ ใช้ Role-Based Access Control (RBAC)
✅ เก็บ API Token อย่างปลอดภัย
✅ ตรวจสอบ Audit Log เป็นประจำ
✅ มีแผน Incident Response
✅ Training พนักงานเรื่อง Security Awareness
✅ ตั้ง Recovery Options สำรอง
✅ ปฏิบัติตาม PDPA
```

### Password Policy

**หลักการสร้าง Strong Password:**

```
ต้องมี:
- ความยาวอย่างน้อย 12 ตัวอักษร (แนะนำ 16+)
- ตัวพิมพ์ใหญ่ (A-Z) อย่างน้อย 1 ตัว
- ตัวพิมพ์เล็ก (a-z) อย่างน้อย 1 ตัว
- ตัวเลข (0-9) อย่างน้อย 1 ตัว
- อักขระพิเศษ (!@#$%^&*) อย่างน้อย 1 ตัว
- ไม่ใช้ข้อมูลส่วนตัว (ชื่อ, วันเกิด, เบอร์โทร)

ห้ามใช้:
- รหัสผ่านเดิมซ้ำกัน
- รหัสผ่านเดียวกับ Service อื่น
- คำที่อยู่ใน Dictionary
- Pattern ง่ายๆ (123456, qwerty, aaaaaa)
```

**ตัวอย่าง Password ที่ดี:**

```python
# ตัวอย่างการสร้าง Strong Password ด้วย Python

import secrets
import string

def generate_strong_password(length=16, include_symbols=True):
    """
    สร้าง Strong Password อัตโนมัติ
    
    Args:
        length: ความยาว Password (default 16)
        include_symbols: ใส่อักขระพิเศษหรือไม่
    
    Returns:
        password: รหัสผ่านที่สร้าง
    """
    # กำหนด Character Set
    lowercase = string.ascii_lowercase
    uppercase = string.ascii_uppercase
    digits = string.digits
    symbols = '!@#$%^&*()_+-=[]{}|;:,.<>?' if include_symbols else ''
    
    all_chars = lowercase + uppercase + digits + symbols
    
    # รับประกันว่ามีทุกประเภท
    password = [
        secrets.choice(lowercase),
        secrets.choice(uppercase),
        secrets.choice(digits),
    ]
    
    if include_symbols:
        password.append(secrets.choice(symbols))
    
    # เติมที่เหลือ
    password.extend(secrets.choice(all_chars) for _ in range(length - len(password)))
    
    # สับเปลี่ยนตำแหน่ง
    secrets.SystemRandom().shuffle(password)
    
    return ''.join(password)

# สร้าง Password
for i in range(3):
    pwd = generate_strong_password(16)
    print(f"Password {i+1}: {pwd}")

def check_password_strength(password):
    """
    ตรวจสอบความแข็งแกร่งของ Password
    """
    score = 0
    issues = []
    
    if len(password) >= 12:
        score += 2
    elif len(password) >= 8:
        score += 1
    else:
        issues.append("รหัสผ่านสั้นเกินไป (ต้องการอย่างน้อย 12 ตัว)")
    
    if any(c.isupper() for c in password):
        score += 1
    else:
        issues.append("ไม่มีตัวพิมพ์ใหญ่")
    
    if any(c.islower() for c in password):
        score += 1
    else:
        issues.append("ไม่มีตัวพิมพ์เล็ก")
    
    if any(c.isdigit() for c in password):
        score += 1
    else:
        issues.append("ไม่มีตัวเลข")
    
    if any(c in '!@#$%^&*()_+-=[]{}|;:,.<>?' for c in password):
        score += 2
    else:
        issues.append("ไม่มีอักขระพิเศษ")
    
    strength_map = {
        (0, 2): 'อ่อนมาก',
        (3, 4): 'อ่อน',
        (5, 5): 'ปานกลาง',
        (6, 6): 'แข็งแกร่ง',
        (7, 7): 'แข็งแกร่งมาก'
    }
    
    strength = 'อ่อนมาก'
    for (min_s, max_s), label in strength_map.items():
        if min_s <= score <= max_s:
            strength = label
            break
    
    return {
        'score': score,
        'strength': strength,
        'issues': issues
    }

# ทดสอบ
result = check_password_strength("MyL1ne0A$ecure!")
print(f"ความแข็งแกร่ง: {result['strength']} (คะแนน: {result['score']}/7)")
```

### Password Manager สำหรับ Business

แนะนำให้ใช้ Password Manager เพื่อจัดการรหัสผ่านทีม:

| Password Manager | ราคา | ฟีเจอร์หลัก | เหมาะกับ |
|-----------------|------|------------|---------|
| LastPass Teams | $4/user/เดือน | Sharing, Audit | ทีมขนาดกลาง |
| 1Password Business | $8/user/เดือน | Advanced Security, Reports | Enterprise |
| Bitwarden Business | $5/user/เดือน | Open Source, Affordable | SME |
| Dashlane Business | $8/user/เดือน | Dark Web Monitor | ธุรกิจที่ต้องการ Monitor |
| KeePass | ฟรี | Self-hosted, Open Source | ผู้ที่ต้องการ Control สูง |

---

## 17.2 Two-Factor Authentication (2FA)

### ทำไมต้องใช้ 2FA

2FA (Two-Factor Authentication) หรือการยืนยันตัวตนสองขั้นตอน เพิ่มชั้นความปลอดภัยเพิ่มเติมนอกเหนือจาก Password

```
โดยปกติ:
Username + Password → เข้าระบบได้

เมื่อมี 2FA:
Username + Password + OTP (6 หลัก) → เข้าระบบได้

ถ้าคนร้ายได้ Password ไป:
Username + Password + [ไม่มี OTP] → เข้าระบบไม่ได้ ✅
```

### การตั้งค่า 2FA สำหรับ LINE Account

**ขั้นตอนการเปิด 2FA บน LINE:**

1. เปิดแอป LINE บนโทรศัพท์
2. ไปที่ **Settings (การตั้งค่า)**
3. เลือก **Account (บัญชี)**
4. เลือก **Two-step verification (การยืนยันสองขั้นตอน)**
5. กด **Set up (ตั้งค่า)**
6. ใส่หมายเลขโทรศัพท์สำรอง
7. ยืนยัน PIN 4 หลัก

**สำหรับ LINE OA Manager:**
1. ใช้ LINE Account ที่มี 2FA แล้ว Login
2. ระบบจะใช้ 2FA จาก LINE Account โดยอัตโนมัติ

### ประเภทของ 2FA

| ประเภท | ตัวอย่าง | ความปลอดภัย | ความสะดวก |
|--------|---------|------------|---------|
| SMS OTP | รับ SMS รหัส 6 หลัก | ปานกลาง (ถูก SIM Swap ได้) | สูง |
| Authenticator App | Google Authenticator, Authy | สูง | ปานกลาง |
| Hardware Key | YubiKey | สูงมาก | ต่ำ |
| Biometric | ลายนิ้วมือ, Face ID | สูง | สูงมาก |
| Email OTP | รับ Email รหัส 6 หลัก | ต่ำ-ปานกลาง | สูง |

**แนะนำ:** ใช้ Authenticator App เช่น Google Authenticator หรือ Authy เนื่องจากมีความปลอดภัยสูงและสะดวกพอสมควร

### Backup Codes

เมื่อตั้ง 2FA ควรเก็บ Backup Codes ไว้ด้วย:

```
Backup Codes คืออะไร:
- รหัสสำรองที่ใช้แทน OTP เมื่อโทรศัพท์หาย
- มักให้มา 8-10 รหัส ใช้ได้ครั้งเดียว
- ควรพิมพ์และเก็บในที่ปลอดภัย (ห้ามเก็บใน Email หรือ Cloud ที่ไม่ Encrypt)

ตัวอย่าง Backup Code:
8294-1057
3916-4728
5047-2891
...
```

---

## 17.3 การจัดการ Operator Permissions

### ทำความเข้าใจ Operator Roles ใน LINE OA

LINE OA มีระดับการเข้าถึงสำหรับ Operator ดังนี้:

| Role | ระดับ | สิ่งที่ทำได้ |
|------|------|------------|
| Admin | สูงสุด | ทุกอย่าง รวมถึงจัดการ Operator อื่น |
| Operator | กลาง | ส่ง Broadcast, จัดการ Chat, ดู Analytics |
| Viewer | ต่ำสุด | ดูข้อมูลได้อย่างเดียว |

### Principle of Least Privilege

หลักการสำคัญ: **ให้ Permission เท่าที่จำเป็นเท่านั้น**

```
ตัวอย่างการจัดสรร Permission ที่เหมาะสม:

บริษัทขนาดกลาง (ทีม 10 คน):

CEO/Owner → Admin (1 คน)
Marketing Manager → Admin (1 คน)  
Content Writer → Operator (2 คน)
Customer Service → Operator (3 คน)
Freelancer/Agency → Operator (จำกัด, มีอายุ)
Intern → Viewer (หรือ Operator ที่ Supervised)
```

### การเพิ่ม Operator

**ขั้นตอน:**
1. เข้า LINE OA Manager
2. ไปที่ **Settings → Account → Operator Management**
3. คลิก **Add Operators**
4. ใส่ LINE ID หรือ QR Code ของคนที่จะเพิ่ม
5. เลือก Role ที่เหมาะสม
6. ยืนยัน

**ข้อควรระวัง:**
- ตรวจสอบ Identity ของคนที่เพิ่มก่อนเสมอ
- ไม่ให้ Access กับ Personal Account ของ Admin/เจ้าของ เพราะจะทำให้ Operator เห็น Personal Chat ได้

### การลบ Operator

**ต้องลบ Operator เมื่อ:**
1. พนักงานลาออกหรือย้ายแผนก
2. Freelancer หรือ Agency สิ้นสุดสัญญา
3. ทบทวนรายปีพบว่าไม่ได้ใช้งานนาน
4. เกิด Incident ด้านความปลอดภัย

```
ขั้นตอนการลบ Operator:
1. เข้า Settings → Account → Operator Management
2. ค้นหา Operator ที่ต้องการลบ
3. คลิก Edit/Remove
4. ยืนยันการลบ
5. แจ้งให้ Operator ทราบ (ถ้าเหมาะสม)
6. บันทึกในระบบ HR/Security Log
```

### Operator Offboarding Checklist

```
เมื่อพนักงานลาออก:
☐ ลบ Operator Permission จาก LINE OA ทุก Account
☐ เปลี่ยน Admin Password (ถ้าเคยแชร์)
☐ Revoke API Token ที่เคยให้ไว้
☐ ตรวจสอบว่าไม่มี Scheduled Broadcast ค้างอยู่
☐ Export Chat History ที่จำเป็น
☐ แจ้ง IT Department เพื่อ Revoke Access อื่นๆ
☐ บันทึกวันที่ Remove ใน Security Log
```

---

## 17.4 Role-Based Access Control (RBAC)

### การออกแบบ RBAC สำหรับ LINE OA

**ตัวอย่าง RBAC Framework สำหรับ Retail Business:**

```
Roles และ Permissions:

Role: Super Admin (เจ้าของ/ผู้บริหารสูงสุด)
├── ✅ Broadcast Messages
├── ✅ Manage Operators
├── ✅ View Analytics  
├── ✅ Manage Rich Menu
├── ✅ Access API Settings
├── ✅ Manage Coupons
├── ✅ Chat with Customers
└── ✅ Export Data

Role: Marketing Manager
├── ✅ Broadcast Messages
├── ❌ Manage Operators (ไม่ให้)
├── ✅ View Analytics
├── ✅ Manage Rich Menu
├── ❌ Access API Settings (ไม่ให้)
├── ✅ Manage Coupons
├── ✅ Chat with Customers (Limited)
└── ✅ Export Data (Limited)

Role: Content Creator
├── ✅ Draft Broadcast (ต้องรออนุมัติ)
├── ❌ Send Broadcast (ไม่ให้ส่งตรง)
├── ✅ View Analytics (Read Only)
├── ❌ Manage Operators
├── ❌ Access API Settings
├── ✅ Manage Content Templates
├── ❌ Chat with Customers
└── ❌ Export Data

Role: Customer Service Agent
├── ❌ Broadcast Messages
├── ❌ Manage Operators
├── ✅ View Basic Analytics
├── ❌ Manage Rich Menu
├── ❌ Access API Settings
├── ❌ Manage Coupons
├── ✅ Chat with Customers
└── ❌ Export Data
```

### การ Implement RBAC ผ่าน Webhook

สำหรับ Account ที่มี Custom Bot เชื่อมต่อ สามารถ Implement RBAC ที่ละเอียดกว่า LINE Manager ได้ผ่าน Code:

```python
# RBAC Implementation สำหรับ LINE Bot

from enum import Enum
from typing import List, Dict, Set
import json

class Permission(Enum):
    """
    Permission Types สำหรับ LINE OA
    """
    VIEW_ANALYTICS = "view_analytics"
    SEND_BROADCAST = "send_broadcast"
    MANAGE_OPERATORS = "manage_operators"
    CONFIGURE_RICH_MENU = "configure_rich_menu"
    CHAT_WITH_CUSTOMERS = "chat_with_customers"
    EXPORT_DATA = "export_data"
    MANAGE_COUPONS = "manage_coupons"
    ACCESS_API_SETTINGS = "access_api_settings"
    VIEW_REPORTS = "view_reports"

class Role:
    """
    Role พร้อม Permissions
    """
    
    ROLES = {
        'super_admin': {
            'name': 'Super Admin',
            'permissions': set([p.value for p in Permission])  # ทุก Permission
        },
        'marketing_manager': {
            'name': 'Marketing Manager',
            'permissions': {
                Permission.VIEW_ANALYTICS.value,
                Permission.SEND_BROADCAST.value,
                Permission.CONFIGURE_RICH_MENU.value,
                Permission.CHAT_WITH_CUSTOMERS.value,
                Permission.MANAGE_COUPONS.value,
                Permission.VIEW_REPORTS.value,
                Permission.EXPORT_DATA.value
            }
        },
        'content_creator': {
            'name': 'Content Creator',
            'permissions': {
                Permission.VIEW_ANALYTICS.value,
                Permission.VIEW_REPORTS.value
            }
        },
        'customer_service': {
            'name': 'Customer Service',
            'permissions': {
                Permission.CHAT_WITH_CUSTOMERS.value,
                Permission.VIEW_ANALYTICS.value
            }
        },
        'viewer': {
            'name': 'Viewer',
            'permissions': {
                Permission.VIEW_ANALYTICS.value,
                Permission.VIEW_REPORTS.value
            }
        }
    }
    
    @classmethod
    def get_permissions(cls, role_name: str) -> Set[str]:
        """ดึง Permissions ของ Role"""
        role = cls.ROLES.get(role_name)
        if role:
            return role['permissions']
        return set()

class AccessControl:
    """
    ระบบ Access Control
    """
    
    def __init__(self):
        self.user_roles: Dict[str, str] = {}  # user_id -> role
        self.audit_log: List[dict] = []
    
    def assign_role(self, user_id: str, role: str, assigned_by: str):
        """
        กำหนด Role ให้ User
        """
        if role not in Role.ROLES:
            raise ValueError(f"Role '{role}' ไม่ถูกต้อง")
        
        old_role = self.user_roles.get(user_id)
        self.user_roles[user_id] = role
        
        # บันทึก Audit Log
        self.log_action(
            action='assign_role',
            user_id=user_id,
            performed_by=assigned_by,
            details={
                'old_role': old_role,
                'new_role': role
            }
        )
        
        return True
    
    def check_permission(self, user_id: str, permission: str, action_context: str = '') -> bool:
        """
        ตรวจสอบว่า User มี Permission หรือไม่
        """
        role = self.user_roles.get(user_id)
        
        if not role:
            self.log_action(
                action='permission_denied',
                user_id=user_id,
                details={'permission': permission, 'reason': 'no_role_assigned', 'context': action_context}
            )
            return False
        
        permissions = Role.get_permissions(role)
        has_permission = permission in permissions
        
        # Log การตรวจสอบ (เฉพาะ Denied)
        if not has_permission:
            self.log_action(
                action='permission_denied',
                user_id=user_id,
                details={'permission': permission, 'role': role, 'context': action_context}
            )
        
        return has_permission
    
    def log_action(self, action: str, user_id: str, performed_by: str = None, details: dict = None):
        """
        บันทึก Audit Log
        """
        from datetime import datetime
        
        entry = {
            'timestamp': datetime.now().isoformat(),
            'action': action,
            'user_id': user_id,
            'performed_by': performed_by,
            'details': details or {}
        }
        
        self.audit_log.append(entry)
    
    def get_user_role(self, user_id: str) -> dict:
        """
        ดึงข้อมูล Role ของ User
        """
        role_name = self.user_roles.get(user_id)
        if not role_name:
            return {'role': None, 'permissions': []}
        
        role_data = Role.ROLES.get(role_name, {})
        return {
            'role': role_name,
            'role_name': role_data.get('name'),
            'permissions': list(role_data.get('permissions', []))
        }
    
    def remove_user(self, user_id: str, removed_by: str):
        """
        ลบ User และ Permissions ทั้งหมด
        """
        if user_id in self.user_roles:
            old_role = self.user_roles.pop(user_id)
            self.log_action(
                action='remove_user',
                user_id=user_id,
                performed_by=removed_by,
                details={'removed_role': old_role}
            )
            return True
        return False

# ตัวอย่างการใช้งาน
ac = AccessControl()

# กำหนด Roles
ac.assign_role('admin_user_001', 'super_admin', 'system')
ac.assign_role('marketing_001', 'marketing_manager', 'admin_user_001')
ac.assign_role('cs_agent_001', 'customer_service', 'admin_user_001')

# ตรวจสอบ Permissions
print("Marketing Manager ส่ง Broadcast ได้หรือไม่:", 
      ac.check_permission('marketing_001', Permission.SEND_BROADCAST.value))
# Output: True

print("CS Agent ส่ง Broadcast ได้หรือไม่:", 
      ac.check_permission('cs_agent_001', Permission.SEND_BROADCAST.value))
# Output: False

# ดู Role ของ User
user_role = ac.get_user_role('cs_agent_001')
print(f"Role ของ CS Agent: {user_role['role_name']}")
print(f"Permissions: {user_role['permissions']}")
```

---

## 17.5 Audit Logs

### ความสำคัญของ Audit Logs

Audit Log คือบันทึกทุกการกระทำที่เกิดขึ้นใน Account ซึ่งสำคัญสำหรับ:
- ตรวจสอบว่าใครทำอะไร เมื่อไหร่
- สอบสวนเหตุการณ์ผิดปกติ
- ปฏิบัติตามข้อกำหนดทางกฎหมาย
- ตรวจจับ Insider Threats

### ข้อมูลที่ควรบันทึกใน Audit Log

```python
# Audit Log Entry Format

AUDIT_LOG_SCHEMA = {
    "timestamp": "ISO 8601 format (2024-01-15T10:30:00+07:00)",
    "event_type": "หมวดหมู่ของเหตุการณ์",
    "action": "การกระทำที่เกิดขึ้น",
    "actor": {
        "user_id": "LINE User ID",
        "display_name": "ชื่อที่แสดง",
        "role": "Role ขณะนั้น",
        "ip_address": "IP Address"
    },
    "target": {
        "type": "ประเภทของ Object ที่ถูกกระทำ",
        "id": "ID ของ Object",
        "description": "คำอธิบาย"
    },
    "result": "success/failure/partial",
    "details": "ข้อมูลเพิ่มเติม"
}

# ประเภท Event Types ที่ควรบันทึก
EVENT_TYPES = {
    "auth": [
        "login",
        "logout", 
        "login_failed",
        "2fa_success",
        "2fa_failed",
        "password_changed"
    ],
    "operator": [
        "operator_added",
        "operator_removed",
        "operator_role_changed",
        "operator_permission_denied"
    ],
    "broadcast": [
        "broadcast_created",
        "broadcast_sent",
        "broadcast_cancelled",
        "broadcast_scheduled"
    ],
    "settings": [
        "rich_menu_changed",
        "account_info_updated",
        "api_token_generated",
        "api_token_deleted",
        "webhook_url_changed"
    ],
    "data": [
        "data_exported",
        "customer_data_viewed",
        "report_generated"
    ]
}
```

### การ Implement Audit Log ใน Custom Bot

```python
# Audit Log System สำหรับ LINE OA Bot

import json
import hashlib
from datetime import datetime
from typing import Optional

class AuditLogger:
    """
    ระบบ Audit Logging สำหรับ LINE OA
    """
    
    def __init__(self, storage_backend='file', log_file='audit_log.jsonl'):
        self.storage_backend = storage_backend
        self.log_file = log_file
        self.entries = []
    
    def log(self,
            event_type: str,
            action: str,
            actor_id: str,
            actor_name: str = '',
            actor_role: str = '',
            target_type: str = '',
            target_id: str = '',
            result: str = 'success',
            details: Optional[dict] = None,
            ip_address: str = ''):
        """
        บันทึก Audit Log Entry
        """
        entry = {
            'log_id': self._generate_log_id(),
            'timestamp': datetime.now().isoformat(),
            'event_type': event_type,
            'action': action,
            'actor': {
                'user_id': actor_id,
                'display_name': actor_name,
                'role': actor_role,
                'ip_address': ip_address
            },
            'target': {
                'type': target_type,
                'id': target_id
            },
            'result': result,
            'details': details or {}
        }
        
        self.entries.append(entry)
        
        if self.storage_backend == 'file':
            self._write_to_file(entry)
        
        return entry['log_id']
    
    def _generate_log_id(self) -> str:
        """
        สร้าง Unique Log ID
        """
        timestamp = datetime.now().isoformat()
        unique_str = f"{timestamp}-{len(self.entries)}"
        return hashlib.sha256(unique_str.encode()).hexdigest()[:16]
    
    def _write_to_file(self, entry: dict):
        """
        เขียน Log ลงไฟล์ (JSONL format)
        """
        with open(self.log_file, 'a', encoding='utf-8') as f:
            f.write(json.dumps(entry, ensure_ascii=False) + '\n')
    
    def query_logs(self,
                   event_type: str = None,
                   actor_id: str = None,
                   action: str = None,
                   start_time: str = None,
                   end_time: str = None,
                   limit: int = 100) -> list:
        """
        ค้นหา Log Entries
        """
        filtered = self.entries.copy()
        
        if event_type:
            filtered = [e for e in filtered if e['event_type'] == event_type]
        
        if actor_id:
            filtered = [e for e in filtered if e['actor']['user_id'] == actor_id]
        
        if action:
            filtered = [e for e in filtered if e['action'] == action]
        
        if start_time:
            filtered = [e for e in filtered if e['timestamp'] >= start_time]
        
        if end_time:
            filtered = [e for e in filtered if e['timestamp'] <= end_time]
        
        return filtered[-limit:]  # Return latest entries
    
    def generate_security_report(self) -> dict:
        """
        สร้าง Security Report จาก Audit Log
        """
        from collections import Counter
        
        report = {
            'total_events': len(self.entries),
            'failed_logins': 0,
            'permission_denials': 0,
            'operator_changes': 0,
            'data_exports': 0,
            'suspicious_activities': [],
            'top_actors': []
        }
        
        actor_counter = Counter()
        
        for entry in self.entries:
            actor_counter[entry['actor']['user_id']] += 1
            
            if entry['action'] == 'login_failed':
                report['failed_logins'] += 1
            elif entry['action'] == 'operator_permission_denied':
                report['permission_denials'] += 1
            elif entry['event_type'] == 'operator' and 'change' in entry['action']:
                report['operator_changes'] += 1
            elif entry['action'] == 'data_exported':
                report['data_exports'] += 1
        
        # ตรวจหา Suspicious Activities
        # ถ้า Login Failed มากกว่า 5 ครั้งใน 10 นาที
        login_fails = [
            e for e in self.entries 
            if e['action'] == 'login_failed'
        ]
        
        if len(login_fails) >= 5:
            report['suspicious_activities'].append({
                'type': 'brute_force_attempt',
                'description': f'พบการ Login ล้มเหลว {len(login_fails)} ครั้ง',
                'severity': 'high'
            })
        
        # Top Actors
        report['top_actors'] = [
            {'actor_id': actor, 'action_count': count}
            for actor, count in actor_counter.most_common(5)
        ]
        
        return report

# ตัวอย่างการใช้งาน
logger = AuditLogger()

# บันทึก Log Events
logger.log(
    event_type='auth',
    action='login',
    actor_id='user_001',
    actor_name='สมชาย ใจดี',
    actor_role='admin',
    result='success',
    ip_address='203.xxx.xxx.xxx'
)

logger.log(
    event_type='broadcast',
    action='broadcast_sent',
    actor_id='user_001',
    actor_name='สมชาย ใจดี',
    actor_role='admin',
    target_type='broadcast',
    target_id='BC-2024-001',
    details={'recipients': 5000, 'message': 'Summer Sale Announcement'}
)

logger.log(
    event_type='operator',
    action='operator_added',
    actor_id='user_001',
    actor_name='สมชาย ใจดี',
    actor_role='admin',
    target_type='operator',
    target_id='user_002',
    details={'new_role': 'marketing_manager'}
)

# สร้าง Security Report
report = logger.generate_security_report()
print("Security Report:")
print(json.dumps(report, indent=2, ensure_ascii=False))
```

---

## 17.6 การป้องกัน Account Hijacking

### Tactics ที่ใช้ในการ Hijack Account

**1. Phishing Attack**

```
ตัวอย่าง Phishing Email:

จาก: support@line-official-verify.com (อีเมลปลอม!)
ถึง: admin@yourbusiness.com
หัวข้อ: [สำคัญ] Account LINE OA ของคุณถูกระงับ

เนื้อหา:
"เราพบกิจกรรมที่น่าสงสัยใน LINE Official Account ของคุณ
คุณต้องยืนยันตัวตนใหม่ภายใน 24 ชั่วโมง
มิเช่นนั้น Account จะถูกปิดถาวร

คลิกที่นี่เพื่อยืนยัน: http://line-official-fake.com/verify

LINE Support Team"

วิธีสังเกต Phishing:
- Domain อีเมลผิดปกติ (ไม่ใช่ @line.me หรือ @linecorp.com)
- สร้างความเร่งด่วน (urgent)
- Link ที่แนบไปไม่ใช่ Domain ของ LINE จริง
- ขอ Credential ผ่าน Email
```

**2. SIM Swap Attack**

ป้องกันโดย:
- ใช้ Authenticator App แทน SMS OTP
- แจ้งผู้ให้บริการมือถือเพิ่ม Security PIN

**3. Account Takeover via Social Engineering**

```
กรณีศึกษา:
- มิจฉาชีพโทรหาผู้ดูแล Account อ้างว่าเป็น Support LINE
- ขอ OTP เพื่อ "ยืนยันตัวตน"
- เมื่อได้ OTP ก็เข้า Account ได้ทันที

กฎสำคัญ:
✅ LINE ไม่เคยขอ OTP ทางโทรศัพท์
✅ LINE ไม่เคยขอ Password ทางโทรศัพท์
✅ ห้ามบอก OTP ให้ใคร ไม่ว่าจะอ้างตัวเป็นใคร
```

### Account Recovery Plan

เตรียมแผนรับมือหากถูก Hijack:

```
แผนกู้คืน Account:

ขั้นตอนเร่งด่วน (0-24 ชั่วโมง):
1. เปลี่ยน Password LINE ทันที
2. ตรวจสอบ Linked Devices และ Logout ทุก Session
3. แจ้ง LINE Support ที่ https://help.line.me
4. ตรวจสอบ Broadcast ที่ส่งโดยไม่ได้รับอนุญาต
5. แจ้งผู้ติดตามหากมีการส่งข้อความผิดปกติ

ขั้นตอนติดตาม (24-72 ชั่วโมง):
1. รอ LINE ตรวจสอบและแก้ไข
2. รวบรวมหลักฐาน (Screenshots, Audit Log)
3. ตรวจสอบว่ามีข้อมูลรั่วไหลหรือไม่
4. แจ้งความกับตำรวจถ้าจำเป็น

ขั้นตอนป้องกัน (หลังแก้ไขแล้ว):
1. เปิด 2FA ทันที
2. ตรวจสอบ Operator List ทั้งหมด
3. เปลี่ยน Password ของ Email ที่ผูกกับ Account
4. ทบทวน Security Practices ทั้งหมด
```

### Emergency Contact Information

```
LINE Business Support:
- Help Center: https://help.line.biz
- Email: biz.support@line.me
- LINE Official: @linecorp

สำหรับ Verified Account ที่มี Dedicated Support:
- ติดต่อ Account Manager โดยตรง
- Priority Response ใน 4-8 ชั่วโมง
```

---

## 17.7 API Token Security

### ความเสี่ยงของ API Token

Channel Access Token คือ "กุญแจ" ที่ให้สิทธิ์เต็มที่ในการควบคุม LINE OA ผ่าน API หากหลุดออกไปจะเป็นอันตรายมาก

### Best Practices สำหรับ API Token

```python
# ❌ วิธีที่ผิด: Hardcode ใน Code
LINE_ACCESS_TOKEN = "U+abcdef1234567890..."

# ❌ วิธีที่ผิด: เก็บใน Config File ที่ Push ขึ้น Git
# config.py
TOKEN = "U+abcdef1234567890..."

# ✅ วิธีที่ถูก: ใช้ Environment Variables
import os
LINE_ACCESS_TOKEN = os.getenv('LINE_ACCESS_TOKEN')

if not LINE_ACCESS_TOKEN:
    raise ValueError("LINE_ACCESS_TOKEN is not set!")
```

```bash
# การตั้ง Environment Variables
# Linux/Mac:
export LINE_ACCESS_TOKEN="your_token_here"

# Windows:
set LINE_ACCESS_TOKEN=your_token_here

# Docker:
docker run -e LINE_ACCESS_TOKEN=your_token_here myapp

# .env file (ใช้กับ python-dotenv):
# ไฟล์ .env (ห้าม Commit ขึ้น Git!)
LINE_ACCESS_TOKEN=your_token_here
LINE_CHANNEL_SECRET=your_secret_here
```

```python
# ตัวอย่างการใช้ .env กับ python-dotenv
# pip install python-dotenv

from dotenv import load_dotenv
import os

# โหลด .env file
load_dotenv()

LINE_ACCESS_TOKEN = os.getenv('LINE_ACCESS_TOKEN')
LINE_CHANNEL_SECRET = os.getenv('LINE_CHANNEL_SECRET')

print(f"Token loaded: {'Yes' if LINE_ACCESS_TOKEN else 'No'}")
```

### .gitignore สำหรับ Security

```
# .gitignore - ห้าม Commit ไฟล์เหล่านี้

# Environment files
.env
.env.local
.env.production
*.env

# Key files
*.pem
*.key
*.p12
*.pfx

# Config with credentials
config/credentials.json
config/secrets.yaml

# Python
__pycache__/
*.pyc

# Node.js
node_modules/
```

### การ Rotate API Tokens

API Token ควรถูก Rotate (เปลี่ยน) เป็นระยะเพื่อความปลอดภัย:

```
ควร Rotate Token เมื่อ:
- พนักงานที่รู้ Token ลาออก
- สงสัยว่า Token อาจหลุด
- ตามนโยบาย (เช่น ทุก 90 วัน)
- หลังจาก Incident

ขั้นตอน Token Rotation:
1. สร้าง New Token ใน LINE Developer Console
2. อัปเดต Environment Variables ในทุก Server
3. ทดสอบว่า New Token ทำงานได้
4. Revoke Old Token
5. บันทึกใน Security Log
```

---

## 17.8 Compliance Considerations

### ข้อกำหนดทางกฎหมายที่เกี่ยวข้อง

#### พระราชบัญญัติคุ้มครองข้อมูลส่วนบุคคล (PDPA)

PDPA บังคับใช้กับธุรกิจทุกประเภทที่เก็บรวบรวมข้อมูลส่วนบุคคลของคนไทย

**ข้อมูลส่วนบุคคลที่ LINE OA เก็บ:**
- LINE User ID
- Display Name
- Profile Picture
- Chat History
- Click Behavior
- Location (ถ้าผู้ใช้แชร์)

### PDPA Compliance สำหรับ LINE OA

#### 1. ฐานทางกฎหมาย (Legal Basis)

```
ฐานทางกฎหมายในการประมวลผลข้อมูล:

1. ความยินยอม (Consent) - ผู้ใช้ยินยอมเมื่อเพิ่มเพื่อน
2. สัญญา (Contract) - จำเป็นสำหรับการให้บริการ
3. ประโยชน์อันชอบด้วยกฎหมาย (Legitimate Interest) - วิเคราะห์เพื่อปรับปรุงบริการ
4. การปฏิบัติตามกฎหมาย (Legal Obligation) - เก็บข้อมูลตามที่กฎหมายกำหนด
```

#### 2. Privacy Policy ที่ LINE OA ต้องมี

```markdown
# นโยบายความเป็นส่วนตัว - [ชื่อบริษัท] LINE Official Account

## ข้อมูลที่เราเก็บรวบรวม
เมื่อคุณเพิ่ม [ชื่อบริษัท] เป็นเพื่อนบน LINE เราจะเก็บข้อมูลดังต่อไปนี้:
- LINE User ID (ไม่สามารถระบุตัวตนของคุณได้โดยตรง)
- ข้อมูลโปรไฟล์ที่คุณเปิดเผยต่อสาธารณะบน LINE

## วิธีที่เราใช้ข้อมูล
เราใช้ข้อมูลดังกล่าวเพื่อ:
- ส่งข่าวสาร โปรโมชัน และข้อมูลที่เกี่ยวข้อง
- ปรับปรุงบริการของเรา
- ตอบคำถามและให้บริการลูกค้า

## การเปิดเผยข้อมูล
เราจะไม่เปิดเผยข้อมูลส่วนบุคคลของคุณแก่บุคคลที่สาม ยกเว้น:
- ได้รับความยินยอมจากคุณ
- เป็นไปตามข้อกำหนดทางกฎหมาย

## สิทธิ์ของคุณ
คุณมีสิทธิ์:
- เข้าถึงข้อมูลของคุณ
- แก้ไขข้อมูลที่ไม่ถูกต้อง
- ลบข้อมูลของคุณ (โดยการ Block Account)
- คัดค้านการประมวลผลข้อมูล

## ติดต่อเรา
หากมีคำถามเกี่ยวกับนโยบายนี้ ติดต่อ:
- อีเมล: privacy@yourbusiness.com
- โทร: 02-xxx-xxxx
```

#### 3. Data Retention Policy

```python
# Data Retention Policy Configuration

DATA_RETENTION_POLICY = {
    'chat_history': {
        'retention_period': 365,  # วัน
        'unit': 'days',
        'basis': 'ตามข้อกำหนดทางธุรกิจ',
        'deletion_method': 'permanent_delete'
    },
    'user_analytics': {
        'retention_period': 730,  # 2 ปี
        'unit': 'days',
        'basis': 'Legitimate Interest สำหรับ Business Analysis',
        'deletion_method': 'anonymize'
    },
    'broadcast_logs': {
        'retention_period': 180,  # 6 เดือน
        'unit': 'days',
        'basis': 'ประมวลผลสำเร็จแล้ว',
        'deletion_method': 'aggregate_then_delete'
    },
    'transaction_records': {
        'retention_period': 2555,  # 7 ปี
        'unit': 'days',
        'basis': 'ข้อกำหนดทางบัญชี/ภาษี',
        'deletion_method': 'archive'
    }
}

def should_delete_record(record_type: str, record_date: str) -> dict:
    """
    ตรวจสอบว่าข้อมูลควรถูกลบหรือยัง
    
    Args:
        record_type: ประเภทข้อมูล
        record_date: วันที่สร้างข้อมูล
    
    Returns:
        decision: การตัดสินใจและเหตุผล
    """
    from datetime import datetime, timedelta
    
    policy = DATA_RETENTION_POLICY.get(record_type)
    if not policy:
        return {'should_delete': False, 'reason': 'No policy defined'}
    
    record_datetime = datetime.fromisoformat(record_date)
    retention_days = policy['retention_period']
    expiry_date = record_datetime + timedelta(days=retention_days)
    
    today = datetime.now()
    
    if today >= expiry_date:
        return {
            'should_delete': True,
            'expiry_date': expiry_date.isoformat(),
            'method': policy['deletion_method'],
            'reason': f"เกิน Retention Period {retention_days} วัน"
        }
    else:
        days_remaining = (expiry_date - today).days
        return {
            'should_delete': False,
            'expiry_date': expiry_date.isoformat(),
            'days_remaining': days_remaining,
            'reason': f"ยังอยู่ใน Retention Period (เหลือ {days_remaining} วัน)"
        }
```

#### 4. การตอบสนอง Data Subject Requests (DSR)

```
ประเภทของ DSR และเวลาที่ต้องตอบ:

1. Right of Access (ขอดูข้อมูล)
   - เวลา: ภายใน 30 วัน
   - วิธี: ส่ง Export ข้อมูลทั้งหมดที่เกี่ยวกับบุคคลนั้น

2. Right to Rectification (แก้ไขข้อมูล)
   - เวลา: ภายใน 30 วัน
   - วิธี: อัปเดตข้อมูลที่ไม่ถูกต้อง

3. Right to Erasure (ลบข้อมูล)
   - เวลา: ภายใน 30 วัน
   - วิธี: ลบข้อมูลหรือ Anonymize
   - ข้อยกเว้น: ข้อมูลที่จำเป็นตามกฎหมาย

4. Right to Object (คัดค้านการประมวลผล)
   - เวลา: ภายใน 30 วัน
   - วิธี: หยุดการประมวลผลข้อมูลนั้น
```

---

## 17.9 Security Incident Response

### Incident Response Plan

```
Incident Response Steps:

1. Detection (ตรวจจับ)
   - Monitor Audit Log
   - ตั้ง Alert สำหรับ Login ผิดปกติ
   - ตรวจสอบ Broadcast ที่ไม่ได้อนุมัติ

2. Containment (จำกัดความเสียหาย)
   - เปลี่ยน Password ทันที
   - Revoke API Tokens ที่สงสัย
   - ลบ Operator ที่ไม่ได้รับอนุญาต
   - ปิด Webhook ชั่วคราว (ถ้าจำเป็น)

3. Assessment (ประเมินความเสียหาย)
   - ตรวจสอบว่ามีข้อมูลรั่วไหลหรือไม่
   - ดูว่ามี Broadcast ที่ไม่ได้รับอนุญาตหรือไม่
   - ตรวจสอบการเข้าถึงข้อมูลลูกค้า

4. Notification (แจ้งเหตุการณ์)
   - แจ้ง LINE Support
   - แจ้งผู้บริหาร
   - แจ้งลูกค้าถ้าข้อมูลรั่วไหล (ตาม PDPA ภายใน 72 ชั่วโมง)

5. Recovery (กู้คืน)
   - ตั้ง 2FA ใหม่
   - ตรวจสอบ Security Settings ทั้งหมด
   - เพิ่ม Monitoring

6. Post-Incident Review (ทบทวน)
   - วิเคราะห์สาเหตุ
   - ปรับปรุง Procedure
   - Training พนักงาน
```

### Security Monitoring Dashboard

```python
# ระบบ Monitor Security Events

class SecurityMonitor:
    """
    ระบบ Monitoring สำหรับความปลอดภัย LINE OA
    """
    
    ALERT_THRESHOLDS = {
        'login_failures': 5,       # แจ้งเตือนเมื่อ Login ล้มเหลว 5 ครั้ง
        'permission_denials': 10,  # แจ้งเตือนเมื่อถูกปฏิเสธ Permission 10 ครั้ง
        'broadcast_from_unknown': 1,  # แจ้งเตือนทุกครั้งที่ Broadcast จาก IP ที่ไม่รู้จัก
        'new_operator': 1,         # แจ้งเตือนทุกครั้งที่มี Operator ใหม่
        'api_token_generated': 1   # แจ้งเตือนทุกครั้งที่สร้าง Token ใหม่
    }
    
    def __init__(self, notification_webhook=None):
        self.events = []
        self.alerts = []
        self.notification_webhook = notification_webhook
    
    def record_event(self, event_type: str, details: dict):
        """
        บันทึก Security Event
        """
        from datetime import datetime
        
        event = {
            'timestamp': datetime.now().isoformat(),
            'type': event_type,
            'details': details
        }
        
        self.events.append(event)
        self._check_alerts(event)
    
    def _check_alerts(self, new_event: dict):
        """
        ตรวจสอบว่าควร Trigger Alert หรือไม่
        """
        event_type = new_event['type']
        
        if event_type == 'login_failed':
            # นับจำนวน Login ล้มเหลวใน 10 นาทีที่ผ่านมา
            recent_failures = self._count_recent_events('login_failed', minutes=10)
            if recent_failures >= self.ALERT_THRESHOLDS['login_failures']:
                self._trigger_alert(
                    'brute_force_detected',
                    f'พบ Login ล้มเหลว {recent_failures} ครั้งใน 10 นาที',
                    'high'
                )
        
        elif event_type == 'new_operator_added':
            self._trigger_alert(
                'new_operator',
                f"มี Operator ใหม่: {new_event['details'].get('operator_name', 'Unknown')}",
                'medium'
            )
        
        elif event_type == 'api_token_generated':
            self._trigger_alert(
                'api_token_generated',
                'สร้าง API Token ใหม่',
                'medium'
            )
    
    def _count_recent_events(self, event_type: str, minutes: int) -> int:
        """
        นับ Events ในช่วงเวลาที่กำหนด
        """
        from datetime import datetime, timedelta
        
        cutoff = datetime.now() - timedelta(minutes=minutes)
        
        return sum(
            1 for e in self.events
            if e['type'] == event_type
            and datetime.fromisoformat(e['timestamp']) > cutoff
        )
    
    def _trigger_alert(self, alert_type: str, message: str, severity: str):
        """
        Trigger Security Alert
        """
        from datetime import datetime
        
        alert = {
            'timestamp': datetime.now().isoformat(),
            'type': alert_type,
            'message': message,
            'severity': severity
        }
        
        self.alerts.append(alert)
        
        # ส่ง Notification (ตัวอย่างใช้ LINE Notify หรือ Webhook)
        if self.notification_webhook and severity in ['high', 'critical']:
            self._send_notification(alert)
        
        print(f"🚨 Security Alert [{severity.upper()}]: {message}")
    
    def _send_notification(self, alert: dict):
        """
        ส่ง Alert ไปยัง Notification Channel
        """
        import requests
        
        payload = {
            'text': f"🚨 LINE OA Security Alert\n"
                   f"ระดับ: {alert['severity'].upper()}\n"
                   f"ประเภท: {alert['type']}\n"
                   f"รายละเอียด: {alert['message']}\n"
                   f"เวลา: {alert['timestamp']}"
        }
        
        try:
            requests.post(
                self.notification_webhook,
                json=payload,
                timeout=5
            )
        except Exception as e:
            print(f"ไม่สามารถส่ง Notification: {e}")
    
    def get_security_summary(self) -> dict:
        """
        สรุปสถานะความปลอดภัย
        """
        high_alerts = [a for a in self.alerts if a['severity'] == 'high']
        critical_alerts = [a for a in self.alerts if a['severity'] == 'critical']
        
        return {
            'total_events': len(self.events),
            'total_alerts': len(self.alerts),
            'high_alerts': len(high_alerts),
            'critical_alerts': len(critical_alerts),
            'recent_alerts': self.alerts[-5:],
            'security_score': self._calculate_security_score()
        }
    
    def _calculate_security_score(self) -> int:
        """
        คำนวณ Security Score (0-100)
        """
        score = 100
        
        # หักคะแนนตาม Alert
        for alert in self.alerts:
            if alert['severity'] == 'critical':
                score -= 20
            elif alert['severity'] == 'high':
                score -= 10
            elif alert['severity'] == 'medium':
                score -= 5
        
        return max(0, min(100, score))

# ตัวอย่าง
monitor = SecurityMonitor()

monitor.record_event('login_failed', {'ip': '1.2.3.4', 'user': 'admin@company.com'})
monitor.record_event('login_failed', {'ip': '1.2.3.4', 'user': 'admin@company.com'})
monitor.record_event('login_failed', {'ip': '1.2.3.4', 'user': 'admin@company.com'})
monitor.record_event('new_operator_added', {'operator_name': 'John Doe'})

summary = monitor.get_security_summary()
print(f"Security Score: {summary['security_score']}/100")
print(f"Total Alerts: {summary['total_alerts']}")
```

---

## 17.10 Security Training สำหรับทีม

### หลักสูตร Security Awareness

**Module 1: Password Security (30 นาที)**
- หลักการสร้าง Strong Password
- การใช้ Password Manager
- อย่าแชร์ Password กับใคร

**Module 2: Phishing Awareness (45 นาที)**
- รู้จัก Phishing Email
- สังเกต URL ปลอม
- การตรวจสอบ Identity ของผู้ส่ง

**Module 3: Social Engineering (45 นาที)**
- Tactics ที่นิยมใช้
- วิธีรับมือ
- การ Verify ก่อนทำการใดๆ

**Module 4: Incident Reporting (20 นาที)**
- วิธีรายงาน Incident
- ขั้นตอน Immediate Response
- ช่องทางติดต่อ Security Team

---

## 17.11 แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Security Audit
ตรวจสอบ Security ของ LINE OA ปัจจุบัน:

| รายการ | สถานะ | Action Required |
|--------|-------|----------------|
| 2FA เปิดอยู่ | ✅/❌ | |
| Password แข็งแกร่ง | ✅/❌ | |
| Operator List ล่าสุด | ✅/❌ | |
| API Token ปลอดภัย | ✅/❌ | |
| Privacy Policy มี | ✅/❌ | |
| Audit Log ตั้งค่า | ✅/❌ | |
| Incident Response Plan | ✅/❌ | |

### แบบฝึกหัดที่ 2: RBAC Design
ออกแบบ Role & Permission สำหรับทีมของคุณ:
1. ระบุสมาชิกทีมทั้งหมด
2. กำหนด Role ที่เหมาะสมให้แต่ละคน
3. ตรวจสอบ Access ที่มีอยู่ปัจจุบัน
4. ปรับแก้ให้ตรงกับ Least Privilege Principle

### แบบฝึกหัดที่ 3: Privacy Policy
เขียน Privacy Policy สำหรับ LINE OA ของคุณ:
1. ระบุข้อมูลที่เก็บรวบรวม
2. ระบุวัตถุประสงค์การใช้ข้อมูล
3. ระบุสิทธิ์ของผู้ใช้
4. ระบุช่องทางติดต่อ
5. ให้ทนายความหรือผู้เชี่ยวชาญ PDPA ตรวจสอบ

---

## สรุปบทที่ 17

ในบทนี้เราได้เรียนรู้:
- **Security Best Practices** รวมถึง Password Policy และ Password Manager
- **Two-Factor Authentication** และประเภทต่างๆ ของ 2FA
- **Operator Management** และ Principle of Least Privilege
- **RBAC** การออกแบบ Role-Based Access Control
- **Audit Logs** และการ Implement ระบบ Monitoring
- **การป้องกัน Account Hijacking** และ Recovery Plan
- **API Token Security** และ Best Practices
- **PDPA Compliance** สำหรับธุรกิจไทย
- **Incident Response** Plan และขั้นตอน

**บทต่อไป:** Part 18 - LINE OA กับ E-commerce

---

*อัปเดตล่าสุด: กันยายน 2026*
*เวอร์ชัน: 1.0*
