# Part 19: Customer Service Setup - การตั้งค่าระบบบริการลูกค้า

## บทนำ

การบริการลูกค้าที่ดีผ่าน LINE OA สามารถสร้างความแตกต่างระหว่างธุรกิจที่ประสบความสำเร็จและที่ล้มเหลวได้ ในยุคที่ผู้บริโภคคาดหวัง Response ที่รวดเร็วและ Personalized ระบบ Customer Service ที่มีประสิทธิภาพจึงเป็นสิ่งจำเป็น

บทนี้จะครอบคลุมการออกแบบและ Implement ระบบ Customer Service ที่สมบูรณ์ผ่าน LINE OA ตั้งแต่การ Assign Chat ไปจนถึงการวัดความพึงพอใจ

---

## 19.1 การออกแบบระบบ Customer Service

### Customer Service Model สำหรับ LINE OA

```
Customer Service Model:

Level 1: Bot/Auto-Reply (อัตโนมัติ)
├── FAQ Bot
├── Welcome Message
├── After-Hours Response
└── Basic Inquiry Handling

Level 2: Agent Support (เจ้าหน้าที่)
├── Complex Inquiries
├── Complaint Handling
├── Custom Requests
└── VIP Customer Support

Level 3: Specialist/Manager (ผู้เชี่ยวชาญ)
├── Escalated Complaints
├── Refund/Compensation
├── Technical Issues
└── Media/PR Concerns
```

### การกำหนด Scope ของ Customer Service

ก่อนตั้งค่าระบบ ต้องตอบคำถามเหล่านี้:

| คำถาม | ตัวอย่างคำตอบ |
|-------|-------------|
| ช่วงเวลาให้บริการ | จันทร์-เสาร์ 8:00-20:00 น. |
| ภาษาที่ให้บริการ | ไทย, อังกฤษ |
| ประเภทปัญหาที่รับ | สินค้า, การจัดส่ง, การชำระเงิน, เทคนิค |
| จำนวน Agent | 3 คน (กลางวัน), 1 คน (เย็น) |
| Response Time Target | ภายใน 15 นาที (ในเวลาทำการ) |
| Escalation Policy | ปัญหาที่แก้ไขไม่ได้ใน 30 นาที → Supervisor |

---

## 19.2 Chat Assignment System

### ระบบ Assign Chat อัตโนมัติ

LINE OA มีระบบ Chat Assignment สำหรับ Multi-Agent Support:

#### วิธี Assignment ใน LINE OA Manager

**1. การตั้งค่า Manual Assignment:**
1. เข้า LINE OA Manager → Chat Settings
2. เปิด "Chat" feature
3. ตั้งค่า Assignment Rules
4. กำหนด Default Assignee

**2. Round-Robin Assignment:**
```
Chat ใหม่จาก ลูกค้า A
      ↓
ระบบดูว่า Agent ใดพร้อม
      ↓
Agent 1 (5 chats), Agent 2 (4 chats), Agent 3 (6 chats)
      ↓
Assign ไปที่ Agent 2 (จำนวน Chat น้อยที่สุด)
```

**3. Skill-Based Assignment:**
```
ลูกค้าส่งข้อความ: "มีปัญหาเรื่องการชำระเงิน"
      ↓
Bot ตรวจจับ Keyword "ชำระเงิน"
      ↓
Assign ไปที่ Agent ทีม Finance
```

### Custom Chat Assignment ด้วย API

```python
# ระบบ Chat Assignment อัตโนมัติ

import json
import requests
from datetime import datetime, time
from typing import Optional, List, Dict
from collections import defaultdict

class ChatAssignmentSystem:
    """
    ระบบ Assign Chat อัตโนมัติสำหรับ LINE OA
    """
    
    def __init__(self, line_token: str):
        self.line_token = line_token
        self.headers = {
            'Authorization': f'Bearer {line_token}',
            'Content-Type': 'application/json'
        }
        
        # Agent Registry
        self.agents = {}
        # Active Chats
        self.active_chats = defaultdict(dict)
        # Chat Queue
        self.queue = []
    
    def register_agent(self, agent_id: str, name: str, skills: List[str],
                      shift_start: str, shift_end: str, max_chats: int = 5):
        """
        ลงทะเบียน Agent
        
        Args:
            agent_id: รหัส Agent
            name: ชื่อ Agent
            skills: ทักษะ (เช่น ['payment', 'shipping', 'general'])
            shift_start: เวลาเริ่มงาน (HH:MM)
            shift_end: เวลาเลิกงาน (HH:MM)
            max_chats: จำนวน Chat สูงสุดที่รับได้พร้อมกัน
        """
        self.agents[agent_id] = {
            'id': agent_id,
            'name': name,
            'skills': skills,
            'shift_start': shift_start,
            'shift_end': shift_end,
            'max_chats': max_chats,
            'active_chats': 0,
            'status': 'offline',  # offline, online, busy, break
            'total_handled': 0,
            'avg_handle_time': 0
        }
    
    def update_agent_status(self, agent_id: str, status: str):
        """
        อัปเดตสถานะ Agent
        """
        if agent_id in self.agents:
            self.agents[agent_id]['status'] = status
    
    def get_available_agents(self, required_skill: str = None) -> List[dict]:
        """
        หา Agent ที่พร้อมรับ Chat
        
        Args:
            required_skill: ทักษะที่ต้องการ (optional)
        
        Returns:
            available_agents: list ของ Agent ที่พร้อม
        """
        now = datetime.now().time()
        available = []
        
        for agent_id, agent in self.agents.items():
            # ตรวจสอบสถานะ
            if agent['status'] not in ['online']:
                continue
            
            # ตรวจสอบ Shift
            shift_start = datetime.strptime(agent['shift_start'], '%H:%M').time()
            shift_end = datetime.strptime(agent['shift_end'], '%H:%M').time()
            
            if not (shift_start <= now <= shift_end):
                continue
            
            # ตรวจสอบ Capacity
            if agent['active_chats'] >= agent['max_chats']:
                continue
            
            # ตรวจสอบ Skill (ถ้ากำหนด)
            if required_skill and required_skill not in agent['skills']:
                continue
            
            available.append(agent)
        
        return available
    
    def detect_chat_category(self, message: str) -> str:
        """
        ตรวจจับประเภทปัญหาจากข้อความ
        
        Returns:
            category: ประเภทปัญหา
        """
        message_lower = message.lower()
        
        keywords_map = {
            'payment': ['ชำระ', 'จ่าย', 'โอน', 'payment', 'บัตร', 'บัญชี'],
            'shipping': ['จัดส่ง', 'ส่ง', 'พัสดุ', 'tracking', 'ขนส่ง', 'delivery'],
            'return': ['คืน', 'เปลี่ยน', 'return', 'refund', 'เงินคืน'],
            'product': ['สินค้า', 'ไซส์', 'สี', 'stock', 'product'],
            'account': ['บัญชี', 'รหัส', 'login', 'account', 'ลืม'],
            'complaint': ['ไม่พอใจ', 'ผิดหวัง', 'แย่', 'ร้องเรียน', 'complaint']
        }
        
        for category, keywords in keywords_map.items():
            if any(kw in message_lower for kw in keywords):
                return category
        
        return 'general'
    
    def assign_chat(self, customer_id: str, initial_message: str) -> Optional[dict]:
        """
        Assign Chat ให้ Agent ที่เหมาะสม
        
        Args:
            customer_id: LINE User ID ของลูกค้า
            initial_message: ข้อความแรกของลูกค้า
        
        Returns:
            assignment: ข้อมูลการ Assign
        """
        # ตรวจจับประเภทปัญหา
        category = self.detect_chat_category(initial_message)
        
        # หา Agent ที่พร้อม
        available = self.get_available_agents(required_skill=category)
        
        if not available:
            # ถ้าไม่มี Specialist ให้ลอง General
            available = self.get_available_agents()
        
        if not available:
            # ไม่มี Agent - เพิ่มลงคิว
            self.queue.append({
                'customer_id': customer_id,
                'message': initial_message,
                'category': category,
                'queued_at': datetime.now().isoformat(),
                'queue_position': len(self.queue) + 1
            })
            return None
        
        # เลือก Agent ที่มี Active Chat น้อยที่สุด (Load Balancing)
        agent = min(available, key=lambda a: a['active_chats'])
        
        # Assign Chat
        self.active_chats[customer_id] = {
            'agent_id': agent['id'],
            'agent_name': agent['name'],
            'category': category,
            'assigned_at': datetime.now().isoformat(),
            'status': 'active',
            'messages': []
        }
        
        # อัปเดต Agent
        self.agents[agent['id']]['active_chats'] += 1
        
        assignment = {
            'customer_id': customer_id,
            'agent': agent,
            'category': category,
            'queue_position': None
        }
        
        return assignment
    
    def close_chat(self, customer_id: str, resolution: str = 'resolved'):
        """
        ปิด Chat หลังการบริการ
        
        Args:
            customer_id: LINE User ID
            resolution: วิธีแก้ไข (resolved, escalated, abandoned)
        """
        if customer_id not in self.active_chats:
            return False
        
        chat = self.active_chats[customer_id]
        agent_id = chat['agent_id']
        
        # อัปเดตสถิติ Agent
        self.agents[agent_id]['active_chats'] -= 1
        self.agents[agent_id]['total_handled'] += 1
        
        # คำนวณ Handle Time
        assigned_at = datetime.fromisoformat(chat['assigned_at'])
        handle_time = (datetime.now() - assigned_at).seconds / 60  # นาที
        
        # อัปเดต Average Handle Time
        total = self.agents[agent_id]['total_handled']
        current_avg = self.agents[agent_id]['avg_handle_time']
        new_avg = ((current_avg * (total - 1)) + handle_time) / total
        self.agents[agent_id]['avg_handle_time'] = round(new_avg, 1)
        
        # บันทึก History
        chat['closed_at'] = datetime.now().isoformat()
        chat['handle_time'] = round(handle_time, 1)
        chat['resolution'] = resolution
        chat['status'] = 'closed'
        
        del self.active_chats[customer_id]
        
        # ตรวจสอบ Queue
        if self.queue:
            next_in_queue = self.queue.pop(0)
            self.assign_chat(next_in_queue['customer_id'], next_in_queue['message'])
        
        return True
    
    def get_realtime_stats(self) -> dict:
        """
        ดูสถิติ Real-time ของ Customer Service
        """
        online_agents = sum(1 for a in self.agents.values() if a['status'] == 'online')
        busy_agents = sum(1 for a in self.agents.values() if a['active_chats'] > 0)
        
        return {
            'total_agents': len(self.agents),
            'online_agents': online_agents,
            'busy_agents': busy_agents,
            'active_chats': len(self.active_chats),
            'queue_length': len(self.queue),
            'agents_detail': [
                {
                    'name': a['name'],
                    'status': a['status'],
                    'active_chats': a['active_chats'],
                    'avg_handle_time': a['avg_handle_time']
                }
                for a in self.agents.values()
            ]
        }

# ตัวอย่าง
system = ChatAssignmentSystem('YOUR_LINE_TOKEN')

# ลงทะเบียน Agents
system.register_agent('agent_001', 'สมชาย', ['payment', 'general'], '08:00', '17:00', max_chats=5)
system.register_agent('agent_002', 'สมหญิง', ['shipping', 'return', 'general'], '08:00', '17:00', max_chats=5)
system.register_agent('agent_003', 'สมศักดิ์', ['general', 'complaint'], '12:00', '21:00', max_chats=4)

# Set Status
system.update_agent_status('agent_001', 'online')
system.update_agent_status('agent_002', 'online')
system.update_agent_status('agent_003', 'online')

# Assign Chat
assignment = system.assign_chat('Uxxx001', 'อยากสอบถามเรื่องการชำระเงิน')
if assignment:
    print(f"Assigned to: {assignment['agent']['name']}")
    print(f"Category: {assignment['category']}")
else:
    print("ไม่มี Agent ว่าง กรุณารอคิว")

# ดูสถิติ
stats = system.get_realtime_stats()
print(f"\nReal-time Stats:")
print(f"Online Agents: {stats['online_agents']}")
print(f"Active Chats: {stats['active_chats']}")
print(f"Queue: {stats['queue_length']}")
```

---

## 19.3 Response Time Guidelines

### SLA (Service Level Agreement) สำหรับ LINE OA

#### การกำหนด Response Time Targets

| Priority | ประเภทปัญหา | First Response | Resolution |
|---------|-----------|---------------|-----------|
| P1 - Critical | ข้อมูลส่วนตัวรั่วไหล, ระบบ Down | < 15 นาที | < 4 ชั่วโมง |
| P2 - High | ปัญหาชำระเงิน, ออเดอร์เสีย | < 30 นาที | < 24 ชั่วโมง |
| P3 - Medium | คำถามทั่วไป, สอบถามสินค้า | < 2 ชั่วโมง | < 48 ชั่วโมง |
| P4 - Low | Feedback, ข้อเสนอแนะ | < 24 ชั่วโมง | Best Effort |

#### Metrics ที่ต้องติดตาม

```
Customer Service KPIs:

1. First Response Time (FRT)
   - คำนวณ: เวลาที่ลูกค้าส่งข้อความ → เวลาที่ Agent ตอบกลับครั้งแรก
   - เป้าหมาย: < 15 นาทีสำหรับ 90% ของ Chat

2. Average Handle Time (AHT)
   - คำนวณ: เวลาตั้งแต่รับ Chat จนปิด Chat
   - เป้าหมาย: < 10 นาทีสำหรับปัญหาทั่วไป

3. Resolution Rate (RR)
   - คำนวณ: (จำนวน Chat ที่แก้ไขได้ / Chat ทั้งหมด) × 100%
   - เป้าหมาย: > 85%

4. Customer Satisfaction Score (CSAT)
   - คำนวณ: คะแนน Survey จากลูกค้า
   - เป้าหมาย: > 4.0/5.0

5. First Contact Resolution (FCR)
   - คำนวณ: ปัญหาที่แก้ได้ใน Contact แรก / ทั้งหมด
   - เป้าหมาย: > 70%
```

### Monitoring Response Time

```python
# ระบบ Monitor Response Time

from datetime import datetime, timedelta
from typing import Dict, List

class ResponseTimeMonitor:
    """
    ติดตาม Response Time ของ Customer Service
    """
    
    def __init__(self, sla_targets: Dict[str, int]):
        """
        Args:
            sla_targets: dict ของ {priority: minutes}
        """
        self.sla_targets = sla_targets
        self.chat_timings = {}
        self.sla_breaches = []
    
    def record_customer_message(self, chat_id: str, priority: str = 'P3'):
        """
        บันทึกเวลาที่ลูกค้าส่งข้อความ
        """
        self.chat_timings[chat_id] = {
            'customer_message_at': datetime.now(),
            'priority': priority,
            'first_response_at': None,
            'closed_at': None,
            'sla_deadline': datetime.now() + timedelta(minutes=self.sla_targets.get(priority, 120))
        }
    
    def record_agent_response(self, chat_id: str):
        """
        บันทึกเวลาที่ Agent ตอบ
        """
        if chat_id not in self.chat_timings:
            return
        
        timing = self.chat_timings[chat_id]
        
        if timing['first_response_at'] is None:
            timing['first_response_at'] = datetime.now()
            
            # คำนวณ FRT
            frt = (timing['first_response_at'] - timing['customer_message_at']).seconds / 60
            timing['first_response_time_minutes'] = round(frt, 1)
            
            # ตรวจสอบ SLA
            sla_target = self.sla_targets.get(timing['priority'], 120)
            if frt > sla_target:
                self.sla_breaches.append({
                    'chat_id': chat_id,
                    'priority': timing['priority'],
                    'frt_minutes': frt,
                    'sla_target_minutes': sla_target,
                    'breach_time': datetime.now().isoformat()
                })
    
    def get_chats_near_sla_breach(self, warning_minutes: int = 5) -> List[dict]:
        """
        หา Chats ที่ใกล้จะเกิน SLA
        
        Args:
            warning_minutes: จำนวนนาทีก่อน SLA ที่จะแจ้งเตือน
        
        Returns:
            at_risk_chats: Chats ที่ต้องการความสนใจด่วน
        """
        now = datetime.now()
        at_risk = []
        
        for chat_id, timing in self.chat_timings.items():
            if timing['first_response_at'] is not None:
                continue  # ตอบแล้ว
            
            time_until_breach = (timing['sla_deadline'] - now).seconds / 60
            
            if time_until_breach <= warning_minutes:
                at_risk.append({
                    'chat_id': chat_id,
                    'priority': timing['priority'],
                    'waiting_minutes': (now - timing['customer_message_at']).seconds / 60,
                    'minutes_until_breach': time_until_breach
                })
        
        return sorted(at_risk, key=lambda x: x['minutes_until_breach'])
    
    def generate_sla_report(self) -> dict:
        """
        สรุป SLA Performance
        """
        total_chats = len(self.chat_timings)
        responded_chats = sum(1 for t in self.chat_timings.values() if t.get('first_response_at'))
        
        frt_values = [
            t['first_response_time_minutes']
            for t in self.chat_timings.values()
            if t.get('first_response_time_minutes')
        ]
        
        avg_frt = sum(frt_values) / len(frt_values) if frt_values else 0
        
        sla_compliance = (
            (responded_chats - len(self.sla_breaches)) / responded_chats * 100
            if responded_chats > 0 else 100
        )
        
        return {
            'total_chats': total_chats,
            'responded': responded_chats,
            'sla_breaches': len(self.sla_breaches),
            'sla_compliance_rate': round(sla_compliance, 1),
            'avg_first_response_time': round(avg_frt, 1),
            'pending_chats': total_chats - responded_chats,
            'breach_details': self.sla_breaches[-10:]  # 10 ล่าสุด
        }

# ตัวอย่าง
monitor = ResponseTimeMonitor({
    'P1': 15,
    'P2': 30,
    'P3': 120,
    'P4': 1440
})

# จำลองการทำงาน
monitor.record_customer_message('CHAT-001', 'P2')
monitor.record_customer_message('CHAT-002', 'P3')
monitor.record_customer_message('CHAT-003', 'P1')

# Agent ตอบ CHAT-001
monitor.record_agent_response('CHAT-001')

# ดู Chats ที่ต้องการด่วน
at_risk = monitor.get_chats_near_sla_breach(warning_minutes=25)
print("Chats ที่ต้องการความสนใจด่วน:", at_risk)

# สร้าง Report
report = monitor.generate_sla_report()
print(f"\nSLA Report:")
print(f"Total Chats: {report['total_chats']}")
print(f"SLA Compliance: {report['sla_compliance_rate']}%")
print(f"Avg FRT: {report['avg_first_response_time']} นาที")
```

---

## 19.4 Response Templates

### การสร้าง Template Message ที่มีประสิทธิภาพ

Template Message ช่วยให้ Agent ตอบคำถามได้เร็วและสม่ำเสมอ

#### Template Categories

**1. Greeting Templates**

```
[เวลาเช้า] 
"สวัสดีตอนเช้าครับ! ยินดีให้บริการครับ ขอทราบว่ามีเรื่องอะไรให้ช่วยไหมครับ?"

[เวลาบ่าย]
"สวัสดีตอนบ่ายครับ! ยินดีให้บริการครับ มีอะไรให้ช่วยไหมครับ?"

[เวลาเย็น]
"สวัสดีตอนเย็นครับ! ยินดีให้บริการครับ มีเรื่องอะไรให้ช่วยไหมครับ?"

[ทั่วไป - ใช้ได้ตลอดเวลา]
"สวัสดีครับ/ค่ะ [ชื่อลูกค้า] ยินดีให้บริการครับ/ค่ะ มีอะไรให้ช่วยไหมครับ/ค่ะ?"
```

**2. Hold Templates**

```
[กำลังตรวจสอบ]
"ขอโทษนะครับ ขอเวลาสักครู่ในการตรวจสอบข้อมูลให้ครับ ใช้เวลาประมาณ [X] นาที"

[รอนาน]
"ขออภัยที่ต้องรอนานนะครับ ยังดำเนินการอยู่ครับ รบกวนรออีกสักครู่นะครับ"

[รอ Supervisor]
"ขอโทษครับ เรื่องนี้ต้องให้ผู้จัดการช่วยดูครับ กรุณารอสักครู่นะครับ"
```

**3. Resolution Templates**

```
[แก้ไขได้แล้ว]
"ดำเนินการเรียบร้อยแล้วครับ/ค่ะ [อธิบายสิ่งที่ทำ] ถ้ามีคำถามเพิ่มเติมสอบถามได้เลยนะครับ/ค่ะ"

[ต้องรอผล]
"ได้รับเรื่องของคุณแล้วครับ/ค่ะ จะดำเนินการให้ภายใน [X ชั่วโมง/วัน] แล้วจะแจ้งให้ทราบนะครับ/ค่ะ"

[ปิด Chat]
"ขอบคุณมากครับ/ค่ะ ถ้ามีเรื่องอะไรอีก ทักมาได้เลยนะครับ/ค่ะ ขอให้วันดีครับ/ค่ะ 😊"
```

**4. Apology Templates**

```
[ขออโทษทั่วไป]
"ขออภัยในความไม่สะดวกด้วยนะครับ/ค่ะ เราจะรีบดำเนินการให้โดยเร็วที่สุดครับ/ค่ะ"

[ขอโทษล่าช้า]
"ขออภัยที่ตอบช้าครับ/ค่ะ ขณะนี้มีลูกค้าติดต่อมาจำนวนมาก ขอบคุณที่รอนะครับ/ค่ะ"

[ขอโทษสินค้ามีปัญหา]
"ขออภัยอย่างสูงที่สินค้าที่ได้รับไม่เป็นไปตามที่คาดหวังครับ/ค่ะ เราจะแก้ไขให้ทันทีครับ/ค่ะ"
```

### Template Management System

```python
# ระบบจัดการ Response Templates

import json
from typing import Optional, List, Dict
from datetime import datetime

class TemplateManager:
    """
    ระบบจัดการ Response Templates สำหรับ Customer Service
    """
    
    def __init__(self):
        self.templates: Dict[str, dict] = {}
        self.usage_stats: Dict[str, int] = {}
        self._load_default_templates()
    
    def _load_default_templates(self):
        """
        โหลด Default Templates
        """
        default_templates = {
            'greet_morning': {
                'name': 'ทักทายตอนเช้า',
                'category': 'greeting',
                'content': 'สวัสดีตอนเช้าครับ! ยินดีให้บริการครับ มีอะไรให้ช่วยไหมครับ? 😊',
                'variables': [],
                'tags': ['greeting', 'morning']
            },
            'greet_general': {
                'name': 'ทักทายทั่วไป',
                'category': 'greeting',
                'content': 'สวัสดีครับ/ค่ะ {{customer_name}} ยินดีให้บริการครับ/ค่ะ',
                'variables': ['customer_name'],
                'tags': ['greeting']
            },
            'hold_checking': {
                'name': 'กำลังตรวจสอบ',
                'category': 'hold',
                'content': 'ขอโทษนะครับ ขอเวลาสักครู่ในการตรวจสอบข้อมูลให้ครับ ใช้เวลาประมาณ {{minutes}} นาทีครับ',
                'variables': ['minutes'],
                'tags': ['hold', 'checking']
            },
            'order_check_result': {
                'name': 'ผลการตรวจสอบออเดอร์',
                'category': 'order',
                'content': 'ออเดอร์ #{{order_id}} ของคุณ {{customer_name}}\nสถานะ: {{status}}\n{{details}}\n\nหากมีคำถามเพิ่มเติม สอบถามได้เลยครับ',
                'variables': ['order_id', 'customer_name', 'status', 'details'],
                'tags': ['order', 'status']
            },
            'apology_general': {
                'name': 'ขออภัยทั่วไป',
                'category': 'apology',
                'content': 'ขออภัยในความไม่สะดวกด้วยนะครับ/ค่ะ เราจะรีบดำเนินการให้โดยเร็วที่สุดครับ/ค่ะ',
                'variables': [],
                'tags': ['apology']
            },
            'close_chat': {
                'name': 'ปิด Chat',
                'category': 'closing',
                'content': 'ขอบคุณที่ใช้บริการครับ/ค่ะ {{customer_name}} ถ้ามีเรื่องอะไรอีก ทักมาได้เลยนะครับ/ค่ะ ขอให้วันดีครับ/ค่ะ 😊',
                'variables': ['customer_name'],
                'tags': ['closing']
            },
            'refund_approved': {
                'name': 'อนุมัติ Refund',
                'category': 'refund',
                'content': 'ดำเนินการคืนเงินจำนวน {{amount}} บาทให้คุณ {{customer_name}} แล้วครับ/ค่ะ จะได้รับเงินภายใน {{days}} วันทำการครับ/ค่ะ',
                'variables': ['amount', 'customer_name', 'days'],
                'tags': ['refund', 'approved']
            }
        }
        
        for key, template in default_templates.items():
            self.templates[key] = template
            self.usage_stats[key] = 0
    
    def add_template(self, key: str, name: str, category: str, content: str,
                    variables: List[str] = None, tags: List[str] = None) -> bool:
        """
        เพิ่ม Template ใหม่
        """
        self.templates[key] = {
            'name': name,
            'category': category,
            'content': content,
            'variables': variables or [],
            'tags': tags or [],
            'created_at': datetime.now().isoformat()
        }
        self.usage_stats[key] = 0
        return True
    
    def get_template(self, key: str, variables: Dict[str, str] = None) -> Optional[str]:
        """
        ดึง Template และแทนที่ Variables
        
        Args:
            key: Template Key
            variables: dict ของ variable values
        
        Returns:
            rendered_template: ข้อความที่แทนค่าแล้ว
        """
        template = self.templates.get(key)
        if not template:
            return None
        
        content = template['content']
        
        # แทนที่ Variables
        if variables:
            for var_name, var_value in variables.items():
                content = content.replace(f'{{{{{var_name}}}}}', str(var_value))
        
        # เพิ่มการนับการใช้งาน
        self.usage_stats[key] = self.usage_stats.get(key, 0) + 1
        
        return content
    
    def search_templates(self, query: str = None, category: str = None,
                        tags: List[str] = None) -> List[dict]:
        """
        ค้นหา Templates
        """
        results = []
        
        for key, template in self.templates.items():
            # Filter by category
            if category and template['category'] != category:
                continue
            
            # Filter by tags
            if tags and not any(tag in template.get('tags', []) for tag in tags):
                continue
            
            # Filter by query
            if query:
                query_lower = query.lower()
                if (query_lower not in template['name'].lower() and
                    query_lower not in template['content'].lower()):
                    continue
            
            results.append({
                'key': key,
                'name': template['name'],
                'category': template['category'],
                'content_preview': template['content'][:80] + '...' if len(template['content']) > 80 else template['content'],
                'usage_count': self.usage_stats.get(key, 0)
            })
        
        # เรียงตามการใช้งาน
        return sorted(results, key=lambda x: x['usage_count'], reverse=True)
    
    def get_top_templates(self, n: int = 10) -> List[dict]:
        """
        ดึง Templates ที่ใช้บ่อยที่สุด
        """
        sorted_templates = sorted(
            self.usage_stats.items(),
            key=lambda x: x[1],
            reverse=True
        )[:n]
        
        return [
            {
                'key': key,
                'name': self.templates[key]['name'],
                'usage_count': count
            }
            for key, count in sorted_templates
        ]

# ตัวอย่าง
tm = TemplateManager()

# ใช้ Template
message = tm.get_template('order_check_result', {
    'order_id': 'ORD-2024-001',
    'customer_name': 'คุณสมชาย',
    'status': 'กำลังจัดส่ง',
    'details': 'คาดว่าจะถึงวันพุธนี้'
})
print("Message:", message)

# ค้นหา Template
results = tm.search_templates(category='greeting')
print("\nGreeting Templates:", [r['name'] for r in results])

# Top Templates
top = tm.get_top_templates(5)
print("\nTop Templates:", [(t['name'], t['usage_count']) for t in top])
```

---

## 19.5 Escalation Procedures

### การออกแบบ Escalation Matrix

Escalation Matrix กำหนดว่าปัญหาใดต้องส่งต่อให้ใคร:

```
Escalation Matrix:

Level 1 → Level 2:
├── ปัญหาที่ Agent แก้ไขไม่ได้ใน 15 นาที
├── ลูกค้าขอพูดกับ Supervisor
├── ปัญหาเกี่ยวกับการ Refund เกิน 500 บาท
└── ลูกค้าแสดงความโกรธอย่างรุนแรง

Level 2 → Level 3:
├── ปัญหาที่ Supervisor แก้ไขไม่ได้ใน 30 นาที
├── Refund เกิน 5,000 บาท
├── ปัญหาทางกฎหมาย
└── ลูกค้าขู่จะนำเสนอข่าว/ร้องเรียนหน่วยงาน

Level 3 → Special Team:
├── Data Breach
├── ปัญหา PR ขนาดใหญ่
├── คดีความ
└── ปัญหาที่ส่งผลต่อลูกค้าหลายราย
```

### Escalation Implementation

```python
# ระบบ Escalation

class EscalationSystem:
    """
    ระบบ Escalation สำหรับ Customer Service
    """
    
    ESCALATION_RULES = [
        {
            'condition': 'time_exceeded',
            'threshold_minutes': 15,
            'from_level': 1,
            'to_level': 2,
            'auto': True,
            'message': 'Chat เกิน 15 นาทีโดยไม่มีการแก้ไข'
        },
        {
            'condition': 'refund_amount',
            'threshold_baht': 500,
            'from_level': 1,
            'to_level': 2,
            'auto': False,
            'message': 'ยอด Refund เกิน 500 บาท'
        },
        {
            'condition': 'customer_angry',
            'keywords': ['โกรธ', 'เดือด', 'ฟ้อง', 'ร้องเรียน', 'แย่มาก', 'รับผิดชอบ', 'ทนายความ'],
            'from_level': 1,
            'to_level': 2,
            'auto': True,
            'message': 'ลูกค้าแสดงอารมณ์รุนแรง'
        }
    ]
    
    def __init__(self):
        self.escalation_log = []
        self.pending_escalations = {}
    
    def check_escalation_needed(self, chat_id: str, chat_data: dict, message: str = '') -> Optional[dict]:
        """
        ตรวจสอบว่าควร Escalate หรือไม่
        
        Args:
            chat_id: รหัส Chat
            chat_data: ข้อมูล Chat ปัจจุบัน
            message: ข้อความล่าสุดจากลูกค้า
        
        Returns:
            escalation_info: ข้อมูลการ Escalate (หรือ None ถ้าไม่ต้อง)
        """
        current_level = chat_data.get('current_level', 1)
        
        for rule in self.ESCALATION_RULES:
            if rule['from_level'] != current_level:
                continue
            
            trigger = False
            reason = ''
            
            if rule['condition'] == 'time_exceeded':
                # ตรวจสอบเวลา
                assigned_at = datetime.fromisoformat(chat_data['assigned_at'])
                elapsed = (datetime.now() - assigned_at).seconds / 60
                if elapsed > rule['threshold_minutes']:
                    trigger = True
                    reason = f"Chat เกิน {rule['threshold_minutes']} นาที (ผ่านมา {elapsed:.0f} นาที)"
            
            elif rule['condition'] == 'customer_angry':
                # ตรวจสอบ Keywords ในข้อความ
                if message and any(kw in message for kw in rule.get('keywords', [])):
                    trigger = True
                    reason = f"พบ Keyword ที่บ่งชี้ความไม่พอใจ: '{message[:50]}'"
            
            elif rule['condition'] == 'refund_amount':
                # ตรวจสอบจำนวนเงิน Refund
                refund_amount = chat_data.get('refund_amount', 0)
                if refund_amount > rule.get('threshold_baht', 0):
                    trigger = True
                    reason = f"Refund {refund_amount:,} บาท เกิน {rule['threshold_baht']:,} บาท"
            
            if trigger:
                return {
                    'chat_id': chat_id,
                    'from_level': current_level,
                    'to_level': rule['to_level'],
                    'reason': reason,
                    'auto': rule['auto'],
                    'timestamp': datetime.now().isoformat()
                }
        
        return None
    
    def escalate(self, chat_id: str, escalation_info: dict, notify_agent: bool = True) -> bool:
        """
        ดำเนิน Escalation
        """
        # บันทึก Log
        self.escalation_log.append(escalation_info)
        
        # บันทึกว่า Chat ถูก Escalate แล้ว
        self.pending_escalations[chat_id] = {
            'escalation': escalation_info,
            'escalated_at': datetime.now().isoformat(),
            'status': 'pending'
        }
        
        print(f"🚨 Escalation: Chat {chat_id}")
        print(f"   จาก Level {escalation_info['from_level']} → Level {escalation_info['to_level']}")
        print(f"   เหตุผล: {escalation_info['reason']}")
        
        return True
    
    def get_escalation_stats(self) -> dict:
        """
        สรุปสถิติ Escalation
        """
        total = len(self.escalation_log)
        
        by_reason = {}
        for entry in self.escalation_log:
            reason_type = entry.get('reason', 'Unknown')[:30]
            by_reason[reason_type] = by_reason.get(reason_type, 0) + 1
        
        return {
            'total_escalations': total,
            'pending': len(self.pending_escalations),
            'by_reason': by_reason,
            'escalation_rate': f"{(total / max(1, total + 100)) * 100:.1f}%"
        }

# ตัวอย่าง
escalation_system = EscalationSystem()

# จำลอง Chat ที่นานเกินไป
from datetime import datetime, timedelta
old_chat = {
    'assigned_at': (datetime.now() - timedelta(minutes=20)).isoformat(),
    'current_level': 1,
    'refund_amount': 0
}

result = escalation_system.check_escalation_needed('CHAT-001', old_chat, '')
if result:
    escalation_system.escalate('CHAT-001', result)

# จำลอง Chat ที่ลูกค้าโกรธ
angry_chat = {
    'assigned_at': datetime.now().isoformat(),
    'current_level': 1,
    'refund_amount': 0
}

angry_message = "แย่มากเลย สินค้าพัง ต้องการเงินคืนด่วน ไม่ได้จะฟ้องนะ"
result = escalation_system.check_escalation_needed('CHAT-002', angry_chat, angry_message)
if result:
    escalation_system.escalate('CHAT-002', result)
```

---

## 19.6 Customer Satisfaction Tracking

### CSAT Survey Implementation

```python
# ระบบส่ง CSAT Survey หลังปิด Chat

class CSATSurvey:
    """
    ระบบ CSAT Survey ผ่าน LINE
    """
    
    def __init__(self, line_token: str):
        self.line_token = line_token
        self.surveys = {}
        self.responses = []
        
        self.headers = {
            'Authorization': f'Bearer {line_token}',
            'Content-Type': 'application/json'
        }
    
    def send_csat_survey(self, user_id: str, chat_id: str, agent_name: str):
        """
        ส่ง CSAT Survey หลังปิด Chat
        """
        survey_id = f"CSAT-{chat_id}"
        
        self.surveys[survey_id] = {
            'chat_id': chat_id,
            'user_id': user_id,
            'agent_name': agent_name,
            'sent_at': datetime.now().isoformat(),
            'responded': False
        }
        
        message = {
            "type": "flex",
            "altText": "ขอความคิดเห็นเกี่ยวกับการบริการ",
            "contents": {
                "type": "bubble",
                "body": {
                    "type": "box",
                    "layout": "vertical",
                    "spacing": "md",
                    "contents": [
                        {
                            "type": "text",
                            "text": "คุณพึงพอใจกับการบริการของเราแค่ไหน?",
                            "weight": "bold",
                            "size": "lg",
                            "wrap": True
                        },
                        {
                            "type": "text",
                            "text": f"ผู้ให้บริการ: {agent_name}",
                            "color": "#666666",
                            "size": "sm"
                        },
                        {
                            "type": "box",
                            "layout": "horizontal",
                            "margin": "xl",
                            "spacing": "sm",
                            "contents": [
                                {
                                    "type": "button",
                                    "action": {
                                        "type": "postback",
                                        "label": "😡 1",
                                        "data": f"csat={survey_id}&score=1"
                                    },
                                    "height": "sm"
                                },
                                {
                                    "type": "button",
                                    "action": {
                                        "type": "postback",
                                        "label": "😕 2",
                                        "data": f"csat={survey_id}&score=2"
                                    },
                                    "height": "sm"
                                },
                                {
                                    "type": "button",
                                    "action": {
                                        "type": "postback",
                                        "label": "😐 3",
                                        "data": f"csat={survey_id}&score=3"
                                    },
                                    "height": "sm"
                                },
                                {
                                    "type": "button",
                                    "action": {
                                        "type": "postback",
                                        "label": "😊 4",
                                        "data": f"csat={survey_id}&score=4"
                                    },
                                    "height": "sm"
                                },
                                {
                                    "type": "button",
                                    "style": "primary",
                                    "color": "#00B900",
                                    "action": {
                                        "type": "postback",
                                        "label": "😍 5",
                                        "data": f"csat={survey_id}&score=5"
                                    },
                                    "height": "sm"
                                }
                            ]
                        }
                    ]
                }
            }
        }
        
        response = requests.post(
            "https://api.line.me/v2/bot/message/push",
            headers=self.headers,
            json={'to': user_id, 'messages': [message]}
        )
        
        return response.status_code == 200
    
    def record_response(self, survey_id: str, score: int, comment: str = ''):
        """
        บันทึกคำตอบ Survey
        """
        if survey_id not in self.surveys:
            return False
        
        survey = self.surveys[survey_id]
        survey['responded'] = True
        
        response_entry = {
            'survey_id': survey_id,
            'chat_id': survey['chat_id'],
            'user_id': survey['user_id'],
            'agent_name': survey['agent_name'],
            'score': score,
            'comment': comment,
            'responded_at': datetime.now().isoformat()
        }
        
        self.responses.append(response_entry)
        
        # ถ้าคะแนนต่ำ (1-2) ให้แจ้ง Supervisor
        if score <= 2:
            print(f"⚠️ Low CSAT Alert: Score {score}/5 for Chat {survey['chat_id']}")
            print(f"   Agent: {survey['agent_name']}")
            if comment:
                print(f"   Comment: {comment}")
        
        return True
    
    def get_csat_report(self, agent_id: str = None) -> dict:
        """
        สรุป CSAT Report
        """
        responses = self.responses
        if agent_id:
            responses = [r for r in responses if r.get('agent_id') == agent_id]
        
        if not responses:
            return {'status': 'no_data'}
        
        scores = [r['score'] for r in responses]
        avg_score = sum(scores) / len(scores)
        
        # คำนวณ CSAT (% ที่ให้คะแนน 4-5)
        satisfied = sum(1 for s in scores if s >= 4)
        csat_rate = (satisfied / len(scores)) * 100
        
        # Distribution
        distribution = {i: scores.count(i) for i in range(1, 6)}
        
        return {
            'total_responses': len(responses),
            'avg_score': round(avg_score, 2),
            'csat_rate': round(csat_rate, 1),
            'score_distribution': distribution,
            'detractors': sum(1 for s in scores if s <= 2),
            'neutrals': sum(1 for s in scores if s == 3),
            'promoters': sum(1 for s in scores if s >= 4)
        }

# ตัวอย่าง
csat = CSATSurvey('YOUR_LINE_TOKEN')

# จำลองการตอบ Survey
csat.surveys['CSAT-001'] = {
    'chat_id': 'CHAT-001',
    'user_id': 'Uxxx001',
    'agent_name': 'สมชาย',
    'sent_at': datetime.now().isoformat(),
    'responded': False
}

csat.record_response('CSAT-001', 5, 'บริการดีมาก ตอบเร็ว')
csat.record_response('CSAT-001', 4, '')

report = csat.get_csat_report()
print(f"CSAT Report:")
print(f"Average Score: {report['avg_score']}/5")
print(f"CSAT Rate: {report['csat_rate']}%")
```

---

## 19.7 SLA Management

### Dashboard สำหรับ SLA Monitoring

```python
# SLA Dashboard

class SLADashboard:
    """
    Dashboard สำหรับติดตาม SLA
    """
    
    SLA_TARGETS = {
        'first_response_time': {
            'target_minutes': 15,
            'warning_at': 10,  # แจ้งเตือนเมื่อเหลือ 5 นาที
            'critical_at': 13
        },
        'resolution_time': {
            'target_minutes': 60,
            'warning_at': 45,
            'critical_at': 55
        },
        'csat_score': {
            'target': 4.0,
            'warning_at': 3.5,
            'critical_at': 3.0
        }
    }
    
    def __init__(self):
        self.metrics_history = []
        self.current_metrics = {}
    
    def update_metrics(self, new_metrics: dict):
        """
        อัปเดต Metrics ปัจจุบัน
        """
        self.current_metrics = {
            **new_metrics,
            'updated_at': datetime.now().isoformat()
        }
        self.metrics_history.append(self.current_metrics.copy())
    
    def get_sla_status(self) -> dict:
        """
        ดูสถานะ SLA ทั้งหมด
        """
        status = {}
        
        for metric_name, targets in self.SLA_TARGETS.items():
            current_value = self.current_metrics.get(metric_name)
            
            if current_value is None:
                status[metric_name] = {
                    'value': None,
                    'status': 'unknown',
                    'target': targets['target_minutes'] if 'target_minutes' in targets else targets['target']
                }
                continue
            
            target = targets.get('target_minutes') or targets.get('target')
            
            # Determine status
            if metric_name == 'csat_score':
                # Higher is better
                if current_value >= target:
                    health = 'healthy'
                elif current_value >= targets['warning_at']:
                    health = 'warning'
                else:
                    health = 'critical'
            else:
                # Lower is better (time-based)
                if current_value <= targets['warning_at']:
                    health = 'healthy'
                elif current_value <= targets['critical_at']:
                    health = 'warning'
                else:
                    health = 'critical'
            
            status[metric_name] = {
                'value': current_value,
                'target': target,
                'status': health,
                'percentage': min(100, (current_value / target) * 100) if target else None
            }
        
        return status
    
    def generate_daily_report(self) -> str:
        """
        สร้างรายงานประจำวัน
        """
        sla_status = self.get_sla_status()
        
        report_lines = [
            "=" * 50,
            f"📊 SLA Daily Report - {datetime.now().strftime('%d/%m/%Y')}",
            "=" * 50,
            ""
        ]
        
        for metric, data in sla_status.items():
            icon = "✅" if data['status'] == 'healthy' else ("⚠️" if data['status'] == 'warning' else "🔴")
            
            if metric == 'first_response_time':
                report_lines.append(f"{icon} First Response Time: {data['value']} นาที (เป้า: {data['target']} นาที)")
            elif metric == 'resolution_time':
                report_lines.append(f"{icon} Resolution Time: {data['value']} นาที (เป้า: {data['target']} นาที)")
            elif metric == 'csat_score':
                report_lines.append(f"{icon} CSAT Score: {data['value']}/5.0 (เป้า: {data['target']}/5.0)")
        
        return "\n".join(report_lines)

# ตัวอย่าง
dashboard = SLADashboard()

dashboard.update_metrics({
    'first_response_time': 8.5,
    'resolution_time': 42,
    'csat_score': 4.2
})

print(dashboard.generate_daily_report())
```

---

## 19.8 Shift-Based Operator Coverage

### การวางแผน Shift สำหรับ Customer Service

```
ตัวอย่าง Shift Schedule (ร้านค้า E-commerce):

เวลาให้บริการ: 08:00 - 22:00 น. (7 วัน)

Shift A (Morning): 08:00 - 14:00 น.
├── Agent 1 (Lead): ทักษะ General, Payment
├── Agent 2: ทักษะ Shipping, Return
└── Bot Support: FAQ, Auto-Reply

Shift B (Afternoon): 14:00 - 20:00 น.
├── Agent 3 (Lead): ทักษะ General, Complaint
├── Agent 4: ทักษะ Payment, Technical
└── Bot Support: FAQ, Auto-Reply

Shift C (Evening): 20:00 - 22:00 น. (Limited)
├── Agent 5 (Part-time): ทักษะ General
└── Bot Support: FAQ, Auto-Reply + Escalation

นอกเวลาทำการ: 22:00 - 08:00 น.
└── Bot Only: Auto-Reply + Next Day Callback
```

### Shift Handover Protocol

```
ขั้นตอน Handover ระหว่าง Shift:

1. 30 นาทีก่อนสิ้นสุด Shift:
   - หยุดรับ Chat ใหม่
   - สรุปสถานะ Active Chats

2. เวลา Handover:
   - Brief Meeting 10-15 นาที
   - ส่งต่อ Active Chats พร้อมบันทึก
   - แจ้ง Pending Issues

3. เอกสาร Handover:
   - รายการ Chat ที่ส่งต่อ
   - Issues ที่ค้างอยู่
   - ประกาศสำคัญ
   - จำนวนคิวที่รอ

ตัวอย่าง Handover Note:
---
วันที่: 15/01/2024, 14:00 น.
Shift A → Shift B

Active Chats ที่ส่งต่อ:
1. CHAT-042 (คุณสมชาย): รอคำตอบจาก Warehouse เรื่องสต็อก
   → Follow up ใน 30 นาที

2. CHAT-055 (คุณสมหญิง): ร้องเรียนสินค้าชำรุด
   → รออนุมัติ Refund 1,200 บาทจาก Supervisor

Pending Tasks:
- Reply คุณวิชัย เรื่อง Invoice (Email ส่งแล้ว รอ 2 ชั่วโมง)

สถิติ Shift A:
- Chat รับ: 45
- แก้ไขได้: 42 (93.3%)
- CSAT: 4.4/5.0
- FRT เฉลี่ย: 8.2 นาที
---
```

---

## 19.9 แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: SLA กำหนด
1. วิเคราะห์ช่วงเวลาที่ลูกค้าติดต่อมามากที่สุด
2. กำหนด SLA Targets ที่สมเหตุสมผลสำหรับธุรกิจของคุณ
3. สร้าง Escalation Matrix

### แบบฝึกหัดที่ 2: Template Creation
1. รวบรวมคำถามที่พบบ่อยที่สุด 20 ข้อ
2. สร้าง Template Response สำหรับแต่ละข้อ
3. ทดสอบ Template กับ Chat จริง
4. ปรับปรุงตาม Feedback

### แบบฝึกหัดที่ 3: CSAT Setup
1. ออกแบบ CSAT Survey สำหรับ LINE OA ของคุณ
2. ตั้งค่า Auto-Send Survey หลังปิด Chat
3. ตั้ง Alert สำหรับคะแนนต่ำ (< 3)
4. สร้าง Weekly CSAT Report

---

## สรุปบทที่ 19

ในบทนี้เราได้เรียนรู้:
- **Chat Assignment System** การ Assign Chat อัตโนมัติตาม Skill
- **Response Time Guidelines** SLA Targets และการ Monitor
- **Response Templates** การสร้างและจัดการ Template Library
- **Escalation Procedures** Matrix และ Implementation
- **CSAT Tracking** การวัด Customer Satisfaction
- **SLA Management** Dashboard และ Reporting
- **Shift Coverage** การวางแผน Shift และ Handover Protocol

**บทต่อไป:** Part 20 - Marketing Strategies

---

*อัปเดตล่าสุด: กันยายน 2026*
*เวอร์ชัน: 1.0*
