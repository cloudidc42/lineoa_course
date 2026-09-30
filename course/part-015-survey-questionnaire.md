# Part 15: Survey & Questionnaire - เก็บข้อมูลลูกค้าอย่างชาญฉลาด

## บทนำ

การเก็บข้อมูลลูกค้าผ่าน Survey หรือ Questionnaire ใน LINE OA เป็นวิธีที่มีประสิทธิภาพสูงสุดในการทำความเข้าใจพฤติกรรม ความต้องการ และความพึงพอใจของลูกค้า เมื่อเทียบกับช่องทางอื่น LINE มี Response Rate ที่สูงกว่าเพราะผู้ใช้ใช้ LINE เป็นส่วนหนึ่งของชีวิตประจำวันอยู่แล้ว บทนี้จะสอนวิธีสร้าง Survey, กลยุทธ์การกระจาย, การวิเคราะห์ข้อมูล และการนำผลลัพธ์ไปใช้ประโยชน์

---

## 15.1 ภาพรวมระบบ Survey ใน LINE OA

### ช่องทางสร้าง Survey ใน LINE

| ช่องทาง | คำอธิบาย | ข้อดี | ข้อจำกัด |
|--------|----------|-------|---------|
| LINE OA Built-in Survey | ฟีเจอร์ใน LINE OA Manager | ง่าย, ไม่ต้องพัฒนา | จำกัด Customization |
| LIFF App | Web App ใน LINE | Flexible, Design ได้เอง | ต้องพัฒนา |
| Google Forms + LINE | Google Forms + แชร์ผ่าน LINE | ง่าย, ฟรี | ประสบการณ์แยกจาก LINE |
| Chatbot Survey | ถาม-ตอบผ่าน Chatbot | Conversational, Engaging | ซับซ้อนกว่า |
| Typeform/SurveySparrow | Third-party + LINE Share | Professional, สวยงาม | มีค่าใช้จ่าย |

### เมื่อไหร่ควรทำ Survey

```
✓ หลัง Purchase: วัด Customer Satisfaction (CSAT)
✓ หลังบริการ: วัด Service Quality
✓ ประจำเดือน: Net Promoter Score (NPS)
✓ ก่อน Launch: Validate Product Concept
✓ หลัง Event: Event Feedback
✓ ประจำปี: Annual Customer Survey
✓ Exit Survey: ลูกค้าที่ Unfollow
```

---

## 15.2 ประเภทคำถามใน Survey

### 15.2.1 Multiple Choice (ตัวเลือก)

**เมื่อไหรควรใช้:**
- คำถามที่มีคำตอบที่ชัดเจน
- ต้องการ Quantitative Data
- ต้องการวิเคราะห์ได้ง่าย

**ตัวอย่าง:**

```
คุณพบ LINE OA ของเราได้อย่างไร?

○ เพื่อนแนะนำ
○ โฆษณาบน Social Media
○ ค้นหาใน LINE
○ เว็บไซต์ของเรา
○ อื่นๆ
```

```python
multiple_choice_question = {
    "type": "multiple_choice",
    "id": "q1",
    "text": "คุณพบ LINE OA ของเราได้อย่างไร?",
    "required": True,
    "allow_multiple": False,  # เลือกได้คำตอบเดียว
    "options": [
        {"id": "opt1", "label": "เพื่อนแนะนำ", "value": "friend"},
        {"id": "opt2", "label": "โฆษณาบน Social Media", "value": "social_ads"},
        {"id": "opt3", "label": "ค้นหาใน LINE", "value": "line_search"},
        {"id": "opt4", "label": "เว็บไซต์ของเรา", "value": "website"},
        {"id": "opt5", "label": "อื่นๆ", "value": "other", "has_other_text": True}
    ]
}
```

### 15.2.2 Text Response (ข้อความ)

**เมื่อไหรควรใช้:**
- ต้องการความคิดเห็นเชิงลึก
- คำถาม Open-ended
- ต้องการ Qualitative Data

**ตัวอย่าง:**

```
อะไรคือสิ่งที่คุณชอบมากที่สุดเกี่ยวกับสินค้าของเรา?
(กรุณาอธิบายโดยละเอียด)

[ช่องกรอกข้อความ]
```

```python
text_question = {
    "type": "text",
    "id": "q2",
    "text": "อะไรคือสิ่งที่คุณชอบมากที่สุดเกี่ยวกับสินค้าของเรา?",
    "placeholder": "พิมพ์คำตอบของคุณที่นี่...",
    "required": False,
    "max_length": 500,
    "min_length": 10
}
```

### 15.2.3 Rating Scale (คะแนน)

**เมื่อไหรควรใช้:**
- วัด Satisfaction
- NPS (Net Promoter Score)
- เปรียบเทียบระหว่างกลุ่ม

**ตัวอย่างแบบต่างๆ:**

```
1. Rating 1-5 (ดาว)
โดยรวมคุณพอใจกับบริการของเราแค่ไหน?
★☆☆☆☆  ★★☆☆☆  ★★★☆☆  ★★★★☆  ★★★★★

2. Rating 1-10 (NPS)
คุณจะแนะนำเราให้เพื่อนมากน้อยแค่ไหน?
1  2  3  4  5  6  7  8  9  10

3. Likert Scale
ฉันพอใจกับคุณภาพสินค้า
○ ไม่เห็นด้วยอย่างยิ่ง
○ ไม่เห็นด้วย
○ เป็นกลาง
○ เห็นด้วย
○ เห็นด้วยอย่างยิ่ง
```

```python
rating_question = {
    "type": "rating",
    "id": "q3",
    "text": "คุณจะแนะนำเราให้เพื่อนมากน้อยแค่ไหน?",
    "subtext": "0 = ไม่แนะนำเลย, 10 = แนะนำแน่นอน",
    "min_value": 0,
    "max_value": 10,
    "min_label": "ไม่แนะนำ",
    "max_label": "แนะนำแน่นอน",
    "required": True
}
```

### 15.2.4 Checkbox (เลือกได้หลายข้อ)

```python
checkbox_question = {
    "type": "checkbox",
    "id": "q4",
    "text": "คุณซื้อสินค้าประเภทใดจากเราบ้าง? (เลือกได้มากกว่า 1 ข้อ)",
    "options": [
        {"id": "cat1", "label": "เสื้อผ้าผู้หญิง"},
        {"id": "cat2", "label": "เสื้อผ้าผู้ชาย"},
        {"id": "cat3", "label": "กระเป๋าและเครื่องประดับ"},
        {"id": "cat4", "label": "รองเท้า"},
        {"id": "cat5", "label": "ของตกแต่งบ้าน"}
    ],
    "min_selections": 1,
    "max_selections": 5,
    "required": True
}
```

### 15.2.5 Dropdown

```python
dropdown_question = {
    "type": "dropdown",
    "id": "q5",
    "text": "คุณอยู่ในกลุ่มอายุใด?",
    "options": [
        {"id": "age1", "label": "ต่ำกว่า 18 ปี"},
        {"id": "age2", "label": "18-24 ปี"},
        {"id": "age3", "label": "25-34 ปี"},
        {"id": "age4", "label": "35-44 ปี"},
        {"id": "age5", "label": "45-54 ปี"},
        {"id": "age6", "label": "55 ปีขึ้นไป"},
        {"id": "age7", "label": "ไม่ระบุ"}
    ],
    "required": False
}
```

---

## 15.3 การสร้าง Survey ใน LINE OA Manager

### ขั้นตอนการสร้าง Survey

```
1. เข้า LINE OA Manager
2. คลิก "Chat" > "Survey" หรือ "Research"
3. คลิก "+ Create Survey"
4. ตั้งชื่อ Survey
5. เพิ่มคำถาม:
   คลิก "+ Add Question"
   เลือกประเภทคำถาม
   กรอกรายละเอียด
6. กำหนดการแสดงผล:
   - ส่งทันที
   - กำหนดเวลา
   - Trigger-based
7. บันทึกและ Publish
```

### Survey Builder ด้วย Chatbot

```python
class SurveyBot:
    """
    สร้าง Survey ผ่าน Chatbot Interface
    """
    
    def __init__(self, access_token):
        self.token = access_token
        self.active_surveys = {}  # user_id: survey_state
        self.headers = {
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json"
        }
    
    def start_survey(self, user_id, survey_config):
        """
        เริ่ม Survey สำหรับผู้ใช้
        """
        
        self.active_surveys[user_id] = {
            "survey_id": survey_config["id"],
            "questions": survey_config["questions"],
            "current_index": 0,
            "answers": {},
            "started_at": datetime.now().isoformat()
        }
        
        # ส่งคำทักทายและคำถามแรก
        intro_text = (
            f"สวัสดีครับ/ค่ะ 😊\n\n"
            f"ขอเชิญตอบ{survey_config['name']}\n"
            f"ใช้เวลาประมาณ {survey_config.get('estimated_minutes', 2-3)} นาที\n\n"
            f"ข้อมูลของคุณจะถูกเก็บเป็นความลับและใช้เพื่อปรับปรุงบริการเท่านั้น"
        )
        
        self._send_text(user_id, intro_text)
        self._send_question(user_id)
    
    def _send_question(self, user_id):
        """ส่งคำถามถัดไป"""
        
        state = self.active_surveys.get(user_id)
        if not state:
            return
        
        questions = state["questions"]
        current_idx = state["current_index"]
        
        if current_idx >= len(questions):
            self._complete_survey(user_id)
            return
        
        question = questions[current_idx]
        question_num = current_idx + 1
        total = len(questions)
        
        # สร้างข้อความคำถาม
        question_header = f"คำถามที่ {question_num}/{total}"
        
        if question["type"] == "multiple_choice":
            self._send_multiple_choice(user_id, question, question_header)
        elif question["type"] == "text":
            self._send_text_question(user_id, question, question_header)
        elif question["type"] == "rating":
            self._send_rating_question(user_id, question, question_header)
    
    def _send_multiple_choice(self, user_id, question, header):
        """ส่งคำถาม Multiple Choice"""
        
        import requests
        
        # สร้าง Quick Reply
        items = []
        for opt in question["options"]:
            items.append({
                "type": "action",
                "action": {
                    "type": "message",
                    "label": opt["label"][:20],  # จำกัด 20 ตัวอักษร
                    "text": f"answer:{question['id']}:{opt['value']}"
                }
            })
        
        message = {
            "type": "text",
            "text": f"📋 {header}\n\n{question['text']}",
            "quickReply": {
                "items": items[:13]  # LINE จำกัด 13 Quick Reply
            }
        }
        
        requests.post(
            "https://api.line.me/v2/bot/message/push",
            headers=self.headers,
            json={"to": user_id, "messages": [message]}
        )
    
    def _send_rating_question(self, user_id, question, header):
        """ส่งคำถาม Rating"""
        
        import requests
        
        min_val = question.get("min_value", 1)
        max_val = question.get("max_value", 10)
        
        # สร้าง Quick Reply สำหรับ Rating
        items = []
        for i in range(min_val, max_val + 1):
            if question.get("max_value") <= 5:
                label = "⭐" * i + "☆" * (max_val - i)
            else:
                label = str(i)
            
            items.append({
                "type": "action",
                "action": {
                    "type": "message",
                    "label": label[:20],
                    "text": f"answer:{question['id']}:{i}"
                }
            })
        
        message_text = (
            f"⭐ {header}\n\n"
            f"{question['text']}\n\n"
            f"({question.get('min_label', min_val)} ← → {question.get('max_label', max_val)})"
        )
        
        message = {
            "type": "text",
            "text": message_text,
            "quickReply": {"items": items}
        }
        
        requests.post(
            "https://api.line.me/v2/bot/message/push",
            headers=self.headers,
            json={"to": user_id, "messages": [message]}
        )
    
    def handle_answer(self, user_id, answer_text):
        """
        รับและประมวลผลคำตอบ
        """
        
        state = self.active_surveys.get(user_id)
        if not state:
            return False
        
        # Parse คำตอบ
        if answer_text.startswith("answer:"):
            parts = answer_text.split(":", 2)
            if len(parts) == 3:
                question_id = parts[1]
                answer_value = parts[2]
                
                state["answers"][question_id] = answer_value
                state["current_index"] += 1
                
                # ส่งคำถามถัดไป
                self._send_question(user_id)
                return True
        
        # Free text answer
        current_question = state["questions"][state["current_index"]]
        if current_question["type"] == "text":
            state["answers"][current_question["id"]] = answer_text
            state["current_index"] += 1
            self._send_question(user_id)
            return True
        
        return False
    
    def _complete_survey(self, user_id):
        """ส่ง Survey เสร็จสิ้น"""
        
        state = self.active_surveys.pop(user_id, None)
        if not state:
            return
        
        # บันทึกผลลัพธ์
        self._save_survey_results(user_id, state)
        
        # ส่งข้อความขอบคุณ
        thank_you_text = (
            "🙏 ขอบคุณที่ตอบแบบสอบถาม!\n\n"
            "ข้อมูลของคุณมีค่ามากสำหรับเรา\n"
            "เราจะนำไปปรับปรุงผลิตภัณฑ์และบริการให้ดียิ่งขึ้น\n\n"
            "🎁 เป็นการขอบคุณ เราขอมอบส่วนลด 10% สำหรับการสั่งซื้อครั้งถัดไป\n"
            "CODE: THANKS10\n"
            "หมดอายุ: 7 วัน"
        )
        
        self._send_text(user_id, thank_you_text)
    
    def _save_survey_results(self, user_id, state):
        """บันทึกผลการสำรวจลงฐานข้อมูล"""
        # Implementation จะขึ้นกับ Database ที่ใช้
        print(f"Saving survey for {user_id}: {state['answers']}")
    
    def _send_text(self, user_id, text):
        """ส่งข้อความ"""
        import requests
        requests.post(
            "https://api.line.me/v2/bot/message/push",
            headers=self.headers,
            json={"to": user_id, "messages": [{"type": "text", "text": text}]}
        )
```

---

## 15.4 กลยุทธ์การกระจาย Survey

### 15.4.1 วิธีการกระจาย Survey

#### 1. Broadcast Survey

```python
def broadcast_survey(access_token, survey_link, target_group="all"):
    """
    ส่ง Survey ผ่าน Broadcast
    
    Args:
        target_group: "all", "active_30d", "purchasers"
    """
    
    import requests
    
    survey_message = {
        "type": "flex",
        "altText": "เชิญร่วมตอบแบบสอบถาม",
        "contents": {
            "type": "bubble",
            "hero": {
                "type": "image",
                "url": "https://cdn.example.com/survey-banner.jpg",
                "size": "full",
                "aspectRatio": "20:9",
                "aspectMode": "cover"
            },
            "body": {
                "type": "box",
                "layout": "vertical",
                "contents": [
                    {
                        "type": "text",
                        "text": "📋 ช่วยเราปรับปรุงได้ไหม?",
                        "weight": "bold",
                        "size": "xl"
                    },
                    {
                        "type": "text",
                        "text": "ใช้เวลาเพียง 2 นาที แลกกับส่วนลด 10%!",
                        "size": "sm",
                        "color": "#555555",
                        "wrap": True,
                        "margin": "sm"
                    },
                    {
                        "type": "box",
                        "layout": "horizontal",
                        "margin": "md",
                        "contents": [
                            {
                                "type": "text",
                                "text": "⏱️ 2 นาที",
                                "size": "xs",
                                "color": "#888888"
                            },
                            {
                                "type": "text",
                                "text": "🎁 ได้รับ 10% off",
                                "size": "xs",
                                "color": "#1DB446",
                                "align": "end"
                            }
                        ]
                    }
                ]
            },
            "footer": {
                "type": "box",
                "layout": "vertical",
                "contents": [
                    {
                        "type": "button",
                        "style": "primary",
                        "color": "#1DB446",
                        "action": {
                            "type": "uri",
                            "label": "ตอบแบบสอบถาม",
                            "uri": survey_link
                        }
                    }
                ]
            }
        }
    }
    
    requests.post(
        "https://api.line.me/v2/bot/message/broadcast",
        headers={
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json"
        },
        json={"messages": [survey_message]}
    )
```

#### 2. Triggered Survey (หลังซื้อ)

```python
def trigger_post_purchase_survey(user_id, order_id, access_token, delay_hours=2):
    """
    ส่ง Survey อัตโนมัติหลังจากซื้อสินค้า X ชั่วโมง
    """
    
    import requests
    import time
    
    # รอ X ชั่วโมงก่อนส่ง
    time.sleep(delay_hours * 3600)
    
    survey_message = {
        "type": "text",
        "text": (
            f"😊 ขอบคุณที่สั่งซื้อออร์เดอร์ #{order_id} นะคะ!\n\n"
            f"สินค้าถึงมือแล้วหรือยังคะ? คุณรู้สึกอย่างไรกับสินค้าและบริการของเรา?\n\n"
            f"กรุณาให้คะแนน:\n"
            f"1 ⭐ - แย่มาก\n"
            f"2 ⭐⭐ - แย่\n"
            f"3 ⭐⭐⭐ - พอใช้\n"
            f"4 ⭐⭐⭐⭐ - ดี\n"
            f"5 ⭐⭐⭐⭐⭐ - ดีมาก"
        ),
        "quickReply": {
            "items": [
                {"type": "action", "action": {"type": "message", "label": "⭐ 1", "text": f"rating:{order_id}:1"}},
                {"type": "action", "action": {"type": "message", "label": "⭐⭐ 2", "text": f"rating:{order_id}:2"}},
                {"type": "action", "action": {"type": "message", "label": "⭐⭐⭐ 3", "text": f"rating:{order_id}:3"}},
                {"type": "action", "action": {"type": "message", "label": "⭐⭐⭐⭐ 4", "text": f"rating:{order_id}:4"}},
                {"type": "action", "action": {"type": "message", "label": "⭐⭐⭐⭐⭐ 5", "text": f"rating:{order_id}:5"}}
            ]
        }
    }
    
    requests.post(
        "https://api.line.me/v2/bot/message/push",
        headers={
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json"
        },
        json={"to": user_id, "messages": [survey_message]}
    )
```

#### 3. NPS Survey (รายเดือน/รายไตรมาส)

```python
def send_nps_survey(active_users, access_token):
    """
    ส่ง Net Promoter Score Survey ให้ลูกค้า Active
    """
    
    import requests
    
    nps_message = {
        "type": "text",
        "text": (
            "❓ คำถามสำคัญ 1 ข้อ!\n\n"
            "คุณจะแนะนำเราให้เพื่อนหรือครอบครัวมากน้อยแค่ไหน?\n"
            "(0 = ไม่แนะนำเลย, 10 = แนะนำแน่นอน)"
        ),
        "quickReply": {
            "items": [
                {"type": "action", "action": {"type": "message", "label": str(i), "text": f"nps:{i}"}}
                for i in range(0, 11)
            ][:13]
        }
    }
    
    # ส่ง Multicast
    for i in range(0, len(active_users), 500):
        batch = active_users[i:i+500]
        requests.post(
            "https://api.line.me/v2/bot/message/multicast",
            headers={
                "Authorization": f"Bearer {access_token}",
                "Content-Type": "application/json"
            },
            json={"to": batch, "messages": [nps_message]}
        )
```

---

## 15.5 การเก็บและวิเคราะห์ข้อมูล Survey

### 15.5.1 การเก็บข้อมูลใน Database

```python
import sqlite3
import json
from datetime import datetime

def setup_survey_database():
    """
    สร้าง Database Schema สำหรับ Survey
    """
    
    conn = sqlite3.connect("survey_data.db")
    cursor = conn.cursor()
    
    # ตาราง Surveys
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS surveys (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            survey_key TEXT UNIQUE NOT NULL,
            name TEXT NOT NULL,
            description TEXT,
            questions JSON NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            expires_at TIMESTAMP,
            is_active BOOLEAN DEFAULT 1
        )
    """)
    
    # ตาราง Responses
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS survey_responses (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            survey_id INTEGER,
            user_id TEXT NOT NULL,
            answers JSON NOT NULL,
            completed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            completion_time_seconds INTEGER,
            FOREIGN KEY (survey_id) REFERENCES surveys(id)
        )
    """)
    
    # ตาราง Individual Answers
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS answers (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            response_id INTEGER,
            question_id TEXT NOT NULL,
            answer_value TEXT,
            answer_text TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (response_id) REFERENCES survey_responses(id)
        )
    """)
    
    conn.commit()
    return conn

def save_survey_response(conn, survey_id, user_id, answers, completion_time=None):
    """
    บันทึกคำตอบของผู้ใช้
    """
    
    cursor = conn.cursor()
    
    # บันทึก Response
    cursor.execute(
        """INSERT INTO survey_responses (survey_id, user_id, answers, completion_time_seconds)
           VALUES (?, ?, ?, ?)""",
        (survey_id, user_id, json.dumps(answers, ensure_ascii=False), completion_time)
    )
    
    response_id = cursor.lastrowid
    
    # บันทึก Individual Answers
    for question_id, answer in answers.items():
        if isinstance(answer, (int, float)):
            cursor.execute(
                "INSERT INTO answers (response_id, question_id, answer_value) VALUES (?, ?, ?)",
                (response_id, question_id, str(answer))
            )
        else:
            cursor.execute(
                "INSERT INTO answers (response_id, question_id, answer_text) VALUES (?, ?, ?)",
                (response_id, question_id, answer)
            )
    
    conn.commit()
    return response_id
```

### 15.5.2 การวิเคราะห์ข้อมูล

```python
class SurveyAnalytics:
    """
    วิเคราะห์ข้อมูลจาก Survey
    """
    
    def __init__(self, db_connection):
        self.db = db_connection
    
    def calculate_nps(self, survey_id, question_id):
        """
        คำนวณ Net Promoter Score
        
        NPS = % Promoters - % Detractors
        - Promoters: Score 9-10
        - Passives: Score 7-8
        - Detractors: Score 0-6
        """
        
        cursor = self.db.cursor()
        cursor.execute(
            """SELECT CAST(answer_value AS INTEGER) as score
               FROM answers a
               JOIN survey_responses sr ON a.response_id = sr.id
               WHERE sr.survey_id = ? AND a.question_id = ?
               AND answer_value IS NOT NULL""",
            (survey_id, question_id)
        )
        
        scores = [row[0] for row in cursor.fetchall()]
        
        if not scores:
            return {"nps": None, "message": "ไม่มีข้อมูล"}
        
        total = len(scores)
        promoters = sum(1 for s in scores if s >= 9)
        passives = sum(1 for s in scores if 7 <= s <= 8)
        detractors = sum(1 for s in scores if s <= 6)
        
        nps = (promoters / total - detractors / total) * 100
        
        # ประเมิน NPS
        if nps >= 70:
            category = "Excellent"
            interpretation = "ยอดเยี่ยม! ลูกค้าพอใจมาก"
        elif nps >= 50:
            category = "Good"
            interpretation = "ดี ยังมีพื้นที่พัฒนา"
        elif nps >= 0:
            category = "Needs Improvement"
            interpretation = "ต้องปรับปรุง"
        else:
            category = "Poor"
            interpretation = "วิกฤต! ต้องแก้ไขทันที"
        
        return {
            "nps": round(nps, 1),
            "total_responses": total,
            "promoters": promoters,
            "promoters_pct": round(promoters / total * 100, 1),
            "passives": passives,
            "passives_pct": round(passives / total * 100, 1),
            "detractors": detractors,
            "detractors_pct": round(detractors / total * 100, 1),
            "category": category,
            "interpretation": interpretation
        }
    
    def analyze_multiple_choice(self, survey_id, question_id, options):
        """
        วิเคราะห์คำถาม Multiple Choice
        """
        
        cursor = self.db.cursor()
        cursor.execute(
            """SELECT answer_value, COUNT(*) as count
               FROM answers a
               JOIN survey_responses sr ON a.response_id = sr.id
               WHERE sr.survey_id = ? AND a.question_id = ?
               GROUP BY answer_value
               ORDER BY count DESC""",
            (survey_id, question_id)
        )
        
        results = cursor.fetchall()
        total_responses = sum(r[1] for r in results)
        
        if total_responses == 0:
            return {"total": 0, "breakdown": []}
        
        # แมป Value กับ Label
        option_map = {opt["value"]: opt["label"] for opt in options}
        
        breakdown = []
        for value, count in results:
            label = option_map.get(value, value)
            pct = count / total_responses * 100
            breakdown.append({
                "value": value,
                "label": label,
                "count": count,
                "percentage": round(pct, 1)
            })
        
        return {
            "total": total_responses,
            "breakdown": breakdown,
            "most_common": breakdown[0] if breakdown else None
        }
    
    def analyze_text_responses(self, survey_id, question_id, top_n=20):
        """
        วิเคราะห์ Text Responses (Simple Word Frequency)
        """
        
        cursor = self.db.cursor()
        cursor.execute(
            """SELECT answer_text
               FROM answers a
               JOIN survey_responses sr ON a.response_id = sr.id
               WHERE sr.survey_id = ? AND a.question_id = ?
               AND answer_text IS NOT NULL""",
            (survey_id, question_id)
        )
        
        texts = [row[0] for row in cursor.fetchall() if row[0]]
        
        if not texts:
            return {"total": 0, "texts": [], "common_words": []}
        
        # Simple Word Frequency
        word_freq = {}
        stop_words = {"ของ", "และ", "ที่", "ใน", "ไม่", "ได้", "มาก", "เป็น", "ก็", "ว่า", "กับ", "ให้"}
        
        for text in texts:
            words = text.split()
            for word in words:
                word = word.strip(".,!?;:()")
                if len(word) > 1 and word not in stop_words:
                    word_freq[word] = word_freq.get(word, 0) + 1
        
        sorted_words = sorted(word_freq.items(), key=lambda x: x[1], reverse=True)
        
        return {
            "total": len(texts),
            "texts": texts[:10],  # ตัวอย่าง 10 คำตอบ
            "common_words": [
                {"word": w, "count": c} 
                for w, c in sorted_words[:top_n]
            ]
        }
    
    def calculate_csat(self, survey_id, question_id, max_score=5):
        """
        คำนวณ Customer Satisfaction Score
        CSAT = (จำนวนคนที่ให้คะแนน 4-5) / ทั้งหมด * 100
        """
        
        cursor = self.db.cursor()
        cursor.execute(
            """SELECT CAST(answer_value AS REAL) as score
               FROM answers a
               JOIN survey_responses sr ON a.response_id = sr.id
               WHERE sr.survey_id = ? AND a.question_id = ?
               AND answer_value IS NOT NULL""",
            (survey_id, question_id)
        )
        
        scores = [row[0] for row in cursor.fetchall()]
        
        if not scores:
            return None
        
        total = len(scores)
        avg_score = sum(scores) / total
        
        # CSAT: คนที่ให้คะแนนสูง (4-5 จาก 5 หรือ 8-10 จาก 10)
        threshold = max_score * 0.7  # 70% ขึ้นไป
        satisfied = sum(1 for s in scores if s >= threshold)
        csat = satisfied / total * 100
        
        return {
            "csat": round(csat, 1),
            "avg_score": round(avg_score, 2),
            "total_responses": total,
            "satisfied_count": satisfied,
            "max_score": max_score
        }

# ตัวอย่างการใช้งาน
def generate_survey_report(survey_id, db_path):
    """
    สร้าง Full Report สำหรับ Survey
    """
    
    conn = sqlite3.connect(db_path)
    analytics = SurveyAnalytics(conn)
    
    # NPS
    nps_result = analytics.calculate_nps(survey_id, "q_recommend")
    
    # CSAT
    csat_result = analytics.calculate_csat(survey_id, "q_satisfaction")
    
    # Multiple Choice
    source_result = analytics.analyze_multiple_choice(
        survey_id, "q_source",
        [
            {"value": "friend", "label": "เพื่อนแนะนำ"},
            {"value": "social", "label": "Social Media"},
            {"value": "search", "label": "ค้นหา"}
        ]
    )
    
    # Text
    feedback_result = analytics.analyze_text_responses(survey_id, "q_feedback")
    
    report = {
        "survey_id": survey_id,
        "generated_at": datetime.now().isoformat(),
        "nps": nps_result,
        "csat": csat_result,
        "source_breakdown": source_result,
        "text_feedback": feedback_result
    }
    
    return report
```

---

## 15.6 การใช้ข้อมูล Survey สำหรับ Audience Segmentation

### การแบ่งกลุ่มจาก Survey Data

```python
class AudienceSegmenter:
    """
    แบ่งกลุ่มผู้ใช้จาก Survey Data
    """
    
    def __init__(self, db):
        self.db = db
    
    def segment_by_nps(self, survey_id):
        """
        แบ่งกลุ่มตาม NPS Score
        """
        
        cursor = self.db.cursor()
        cursor.execute(
            """SELECT sr.user_id, CAST(a.answer_value AS INTEGER) as nps_score
               FROM answers a
               JOIN survey_responses sr ON a.response_id = sr.id
               WHERE sr.survey_id = ? AND a.question_id = 'q_nps'""",
            (survey_id,)
        )
        
        users_by_segment = {
            "promoters": [],   # 9-10
            "passives": [],    # 7-8
            "detractors": []   # 0-6
        }
        
        for user_id, score in cursor.fetchall():
            if score >= 9:
                users_by_segment["promoters"].append(user_id)
            elif score >= 7:
                users_by_segment["passives"].append(user_id)
            else:
                users_by_segment["detractors"].append(user_id)
        
        return users_by_segment
    
    def create_targeted_campaigns(self, segments, access_token):
        """
        สร้าง Campaign ที่ตรงกลุ่มเป้าหมาย
        """
        
        import requests
        
        campaigns = {
            "promoters": {
                "message": (
                    "😊 คุณเป็นแฟนตัวยงของเราแล้ว!\n\n"
                    "เพื่อเป็นการขอบคุณ ขอมอบ Exclusive Code สำหรับคุณ:\n"
                    "CODE: LOYAL20 (ลด 20%)\n\n"
                    "และอย่าลืมแนะนำเราให้เพื่อนๆ นะคะ 😊\n"
                    "ทุกการแนะนำ คุณจะได้รับ Credit 50 บาท!"
                )
            },
            "passives": {
                "message": (
                    "📊 ขอบคุณสำหรับ Feedback ของคุณ!\n\n"
                    "เราอยากรู้ว่าเราจะปรับปรุงอะไรได้บ้าง\n"
                    "มีอะไรอยากให้เราปรับปรุงไหมคะ?\n\n"
                    "พิมพ์ความคิดเห็นได้เลยค่ะ 👇"
                )
            },
            "detractors": {
                "message": (
                    "😔 เราเสียใจมากที่ประสบการณ์ของคุณไม่ดี\n\n"
                    "ทีมงานของเราอยากพูดคุยกับคุณโดยตรง\n"
                    "เพื่อแก้ไขปัญหาและปรับปรุงบริการ\n\n"
                    "กรุณาทักมาหาเราได้เลยค่ะ\n"
                    "หรือโทร: 02-xxx-xxxx (ฟรี)"
                )
            }
        }
        
        for segment_name, user_ids in segments.items():
            if not user_ids:
                continue
            
            message = campaigns[segment_name]["message"]
            
            # ส่ง Multicast ตาม Segment
            for i in range(0, len(user_ids), 500):
                batch = user_ids[i:i+500]
                requests.post(
                    "https://api.line.me/v2/bot/message/multicast",
                    headers={
                        "Authorization": f"Bearer {access_token}",
                        "Content-Type": "application/json"
                    },
                    json={
                        "to": batch,
                        "messages": [{"type": "text", "text": message}]
                    }
                )
            
            print(f"Sent to {len(user_ids)} {segment_name}")
```

---

## 15.7 Best Practices สำหรับ Response Rate

### กลยุทธ์เพิ่ม Response Rate

```
1. รางวัลสำหรับการตอบ
   ✓ Discount Code (10-15%)
   ✓ สะสม Stamp Card
   ✓ ของรางวัลจับฉลาก
   ✓ Credit / Points

2. ความสั้นกระชับ
   ✓ ไม่เกิน 5 คำถาม สำหรับ Quick Survey
   ✓ บอกเวลาที่ใช้ตอบ
   ✓ Progress Bar ("คำถาม 2 จาก 5")

3. เวลาที่ส่ง
   ✓ วันจันทร์-พุธ เวลา 10:00-11:00
   ✓ หลัง Purchase ทันที (ความรู้สึกยังสด)
   ✓ หลีกเลี่ยงวันหยุดนักขัตฤกษ์

4. การ Follow Up
   ✓ ส่ง Reminder 1 ครั้ง หลัง 2-3 วัน
   ✓ ไม่ส่ง Reminder มากกว่า 2 ครั้ง

5. ความโปร่งใส
   ✓ บอกว่าใช้ข้อมูลทำอะไร
   ✓ รับรองความลับของข้อมูล
   ✓ แสดงว่าเคยนำ Feedback ไปปรับปรุงอะไร
```

### Benchmark Response Rates

| ช่องทาง | Response Rate เฉลี่ย |
|--------|---------------------|
| LINE OA (มีรางวัล) | 25-40% |
| LINE OA (ไม่มีรางวัล) | 8-15% |
| Email Survey | 5-15% |
| SMS Survey | 10-25% |
| Phone Survey | 15-30% |
| In-App Survey | 10-20% |

```python
def calculate_response_rate_benchmark(responses, total_sent):
    """
    เปรียบเทียบ Response Rate กับ Benchmark
    """
    
    rate = responses / total_sent * 100 if total_sent > 0 else 0
    
    benchmarks = [
        {"level": "ต่ำกว่า Benchmark", "min": 0, "max": 8},
        {"level": "พอกับ Benchmark", "min": 8, "max": 15},
        {"level": "สูงกว่า Benchmark", "min": 15, "max": 25},
        {"level": "ดีมาก", "min": 25, "max": 40},
        {"level": "ยอดเยี่ยม", "min": 40, "max": 100}
    ]
    
    for b in benchmarks:
        if b["min"] <= rate < b["max"]:
            performance = b["level"]
            break
    else:
        performance = "N/A"
    
    recommendations = []
    if rate < 8:
        recommendations = [
            "เพิ่มรางวัลสำหรับผู้ตอบ",
            "ลดจำนวนคำถาม",
            "ปรับเวลาส่งให้เหมาะสม",
            "เพิ่มความชัดเจนว่าใช้เวลาเท่าไหร่"
        ]
    elif rate < 15:
        recommendations = [
            "ลองเพิ่ม Incentive เล็กน้อย",
            "ส่ง Reminder 1 ครั้ง"
        ]
    
    return {
        "response_rate": round(rate, 1),
        "performance": performance,
        "recommendations": recommendations
    }
```

---

## 15.8 ตัวอย่าง Survey Templates

### Template 1: Post-Purchase Satisfaction Survey

```python
post_purchase_survey = {
    "id": "post_purchase_v1",
    "name": "ความพึงพอใจหลังการซื้อ",
    "estimated_minutes": 2,
    "incentive": "รับส่วนลด 10% สำหรับการสั่งซื้อครั้งถัดไป",
    "questions": [
        {
            "id": "q1",
            "type": "rating",
            "text": "สินค้าตรงกับที่โฆษณาไว้มากน้อยแค่ไหน?",
            "min_value": 1,
            "max_value": 5,
            "min_label": "ไม่ตรงเลย",
            "max_label": "ตรงมาก",
            "required": True
        },
        {
            "id": "q2",
            "type": "rating",
            "text": "คุณพอใจกับความเร็วในการจัดส่งมากน้อยแค่ไหน?",
            "min_value": 1,
            "max_value": 5,
            "required": True
        },
        {
            "id": "q3",
            "type": "multiple_choice",
            "text": "บรรจุภัณฑ์สินค้าเป็นอย่างไร?",
            "options": [
                {"id": "p1", "label": "สวยงาม ประทับใจมาก", "value": "excellent"},
                {"id": "p2", "label": "ดี ป้องกันสินค้าได้ดี", "value": "good"},
                {"id": "p3", "label": "พอใช้", "value": "ok"},
                {"id": "p4", "label": "ต้องปรับปรุง", "value": "poor"}
            ],
            "required": True
        },
        {
            "id": "q4",
            "type": "text",
            "text": "มีอะไรที่เราทำได้ดีกว่านี้ไหม?",
            "placeholder": "พิมพ์ความคิดเห็นของคุณ...",
            "required": False
        },
        {
            "id": "q5",
            "type": "rating",
            "text": "คุณจะซื้อสินค้าจากเราอีกครั้งไหม?",
            "min_value": 1,
            "max_value": 5,
            "min_label": "ไม่ซื้อ",
            "max_label": "ซื้อแน่นอน",
            "required": True
        }
    ]
}
```

### Template 2: Service Quality Survey

```python
service_quality_survey = {
    "id": "service_quality_v1",
    "name": "คุณภาพการบริการ",
    "estimated_minutes": 3,
    "questions": [
        {
            "id": "q1",
            "type": "rating",
            "text": "โดยรวมคุณพอใจกับบริการของเราแค่ไหน?",
            "min_value": 1,
            "max_value": 5,
            "required": True
        },
        {
            "id": "q2",
            "type": "multiple_choice",
            "text": "พนักงานบริการคุณดีแค่ไหน?",
            "options": [
                {"id": "s1", "label": "ดีมาก มืออาชีพ", "value": "excellent"},
                {"id": "s2", "label": "ดี เป็นมิตร", "value": "good"},
                {"id": "s3", "label": "พอใช้", "value": "ok"},
                {"id": "s4", "label": "ต้องปรับปรุง", "value": "poor"},
                {"id": "s5", "label": "ไม่ได้ติดต่อกับพนักงาน", "value": "na"}
            ],
            "required": True
        },
        {
            "id": "q3",
            "type": "checkbox",
            "text": "อะไรที่คุณชอบมากที่สุดเกี่ยวกับเรา? (เลือกได้มากกว่า 1 ข้อ)",
            "options": [
                {"id": "f1", "label": "คุณภาพสินค้า"},
                {"id": "f2", "label": "ราคาที่เหมาะสม"},
                {"id": "f3", "label": "การบริการ"},
                {"id": "f4", "label": "ความรวดเร็ว"},
                {"id": "f5", "label": "ความสะดวกในการสั่ง"},
                {"id": "f6", "label": "การสื่อสาร/ตอบสนอง"}
            ],
            "required": True
        },
        {
            "id": "q4",
            "type": "rating",
            "text": "คุณจะแนะนำเราให้เพื่อนไหม? (0-10)",
            "min_value": 0,
            "max_value": 10,
            "min_label": "ไม่แนะนำ",
            "max_label": "แนะนำแน่นอน",
            "required": True
        },
        {
            "id": "q5",
            "type": "text",
            "text": "คำแนะนำหรือความคิดเห็นเพิ่มเติม",
            "required": False
        }
    ]
}
```

### Template 3: Product Concept Test

```python
product_concept_survey = {
    "id": "concept_test_v1",
    "name": "ทดสอบแนวคิดสินค้าใหม่",
    "estimated_minutes": 5,
    "questions": [
        {
            "id": "q1",
            "type": "multiple_choice",
            "text": "เราคิดจะเปิดตัว [ชื่อสินค้า] ราคา [ราคา] บาท คุณมีความคิดเห็นอย่างไร?",
            "options": [
                {"id": "i1", "label": "สนใจมาก จะซื้อทันที", "value": "very_interested"},
                {"id": "i2", "label": "สนใจ อาจพิจารณาซื้อ", "value": "interested"},
                {"id": "i3", "label": "ไม่ค่อยสนใจ", "value": "not_interested"},
                {"id": "i4", "label": "ไม่สนใจเลย", "value": "not_at_all"}
            ],
            "required": True
        },
        {
            "id": "q2",
            "type": "multiple_choice",
            "text": "ราคา [ราคา] บาท รู้สึกอย่างไร?",
            "options": [
                {"id": "p1", "label": "ถูกมาก คุ้มมาก", "value": "too_cheap"},
                {"id": "p2", "label": "ราคาเหมาะสม", "value": "just_right"},
                {"id": "p3", "label": "แพงเล็กน้อย", "value": "slightly_expensive"},
                {"id": "p4", "label": "แพงเกินไป", "value": "too_expensive"}
            ],
            "required": True
        },
        {
            "id": "q3",
            "type": "multiple_choice",
            "text": "คุณจะซื้อสินค้านี้จากที่ไหน?",
            "options": [
                {"id": "c1", "label": "ร้านค้าออนไลน์ (LINE, Shopee, Lazada)"},
                {"id": "c2", "label": "เว็บไซต์ของแบรนด์"},
                {"id": "c3", "label": "หน้าร้านโดยตรง"},
                {"id": "c4", "label": "ไม่แน่ใจ"}
            ],
            "required": True
        },
        {
            "id": "q4",
            "type": "text",
            "text": "มีฟีเจอร์หรือคุณสมบัติอะไรที่คุณอยากให้เพิ่มในสินค้านี้บ้าง?",
            "required": False
        }
    ]
}
```

---

## 15.9 LIFF Survey App

### สร้าง Survey ด้วย LIFF (LINE Front-end Framework)

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>แบบสอบถาม</title>
    <script src="https://static.line-scdn.net/liff/edge/2/sdk.js"></script>
    <style>
        body {
            font-family: 'Helvetica Neue', Arial, sans-serif;
            background: #f5f5f5;
            margin: 0;
            padding: 16px;
        }
        
        .survey-container {
            max-width: 480px;
            margin: 0 auto;
            background: white;
            border-radius: 12px;
            padding: 24px;
            box-shadow: 0 2px 8px rgba(0,0,0,0.1);
        }
        
        .survey-title {
            font-size: 20px;
            font-weight: bold;
            color: #333;
            margin-bottom: 8px;
        }
        
        .progress-bar {
            background: #eee;
            height: 4px;
            border-radius: 2px;
            margin-bottom: 24px;
        }
        
        .progress-fill {
            background: #06C755;
            height: 100%;
            border-radius: 2px;
            transition: width 0.3s ease;
        }
        
        .question {
            margin-bottom: 24px;
        }
        
        .question-text {
            font-size: 16px;
            font-weight: 600;
            color: #222;
            margin-bottom: 12px;
        }
        
        .option-button {
            display: block;
            width: 100%;
            padding: 12px 16px;
            margin-bottom: 8px;
            border: 2px solid #e0e0e0;
            border-radius: 8px;
            background: white;
            text-align: left;
            cursor: pointer;
            transition: all 0.2s;
            font-size: 15px;
        }
        
        .option-button:hover {
            border-color: #06C755;
            background: #f0fff4;
        }
        
        .option-button.selected {
            border-color: #06C755;
            background: #e8f8ef;
            color: #06C755;
            font-weight: 600;
        }
        
        .rating-container {
            display: flex;
            gap: 8px;
            flex-wrap: wrap;
        }
        
        .rating-btn {
            width: 44px;
            height: 44px;
            border: 2px solid #e0e0e0;
            border-radius: 8px;
            background: white;
            cursor: pointer;
            font-size: 15px;
            font-weight: 600;
            transition: all 0.2s;
        }
        
        .rating-btn.selected {
            background: #06C755;
            border-color: #06C755;
            color: white;
        }
        
        .text-input {
            width: 100%;
            padding: 12px;
            border: 2px solid #e0e0e0;
            border-radius: 8px;
            font-size: 15px;
            min-height: 100px;
            resize: vertical;
            box-sizing: border-box;
        }
        
        .text-input:focus {
            border-color: #06C755;
            outline: none;
        }
        
        .nav-buttons {
            display: flex;
            gap: 12px;
            margin-top: 24px;
        }
        
        .btn-next {
            flex: 1;
            padding: 14px;
            background: #06C755;
            color: white;
            border: none;
            border-radius: 8px;
            font-size: 16px;
            font-weight: bold;
            cursor: pointer;
        }
        
        .btn-next:disabled {
            background: #ccc;
            cursor: not-allowed;
        }
        
        .btn-prev {
            padding: 14px 20px;
            background: white;
            color: #666;
            border: 2px solid #e0e0e0;
            border-radius: 8px;
            font-size: 16px;
            cursor: pointer;
        }
        
        .completion-screen {
            text-align: center;
            padding: 40px 0;
        }
        
        .completion-icon {
            font-size: 60px;
            margin-bottom: 16px;
        }
    </style>
</head>
<body>
    <div class="survey-container" id="surveyContainer">
        <div class="survey-title">📋 แบบสอบถามความพึงพอใจ</div>
        <p style="color: #888; font-size: 14px; margin-bottom: 16px;">ใช้เวลาประมาณ 2 นาที</p>
        
        <div class="progress-bar">
            <div class="progress-fill" id="progressFill" style="width: 20%"></div>
        </div>
        
        <div id="questionContainer"></div>
        
        <div class="nav-buttons">
            <button class="btn-prev" id="btnPrev" onclick="prevQuestion()" style="display:none">← ย้อนกลับ</button>
            <button class="btn-next" id="btnNext" onclick="nextQuestion()">ถัดไป →</button>
        </div>
    </div>

    <script>
        const questions = [
            {
                id: "q1",
                type: "rating",
                text: "โดยรวมคุณพอใจกับสินค้าของเราแค่ไหน?",
                min: 1, max: 5,
                minLabel: "ไม่พอใจ", maxLabel: "พอใจมาก"
            },
            {
                id: "q2",
                type: "multiple_choice",
                text: "คุณพบเราได้อย่างไร?",
                options: [
                    {value: "friend", label: "เพื่อนแนะนำ"},
                    {value: "social", label: "Social Media"},
                    {value: "search", label: "ค้นหาเอง"},
                    {value: "other", label: "อื่นๆ"}
                ]
            },
            {
                id: "q3",
                type: "text",
                text: "มีอะไรที่เราควรปรับปรุงบ้างไหม?",
                placeholder: "พิมพ์คำแนะนำของคุณ...",
                required: false
            }
        ];
        
        let currentIndex = 0;
        let answers = {};
        let userId = null;
        
        // Initialize LIFF
        async function initLiff() {
            await liff.init({ liffId: "YOUR_LIFF_ID" });
            if (liff.isLoggedIn()) {
                const profile = await liff.getProfile();
                userId = profile.userId;
            }
            renderQuestion();
        }
        
        function renderQuestion() {
            const q = questions[currentIndex];
            const container = document.getElementById('questionContainer');
            const progress = ((currentIndex + 1) / questions.length) * 100;
            
            document.getElementById('progressFill').style.width = progress + '%';
            
            let html = `<div class="question"><div class="question-text">${currentIndex + 1}. ${q.text}</div>`;
            
            if (q.type === 'multiple_choice') {
                q.options.forEach(opt => {
                    const selected = answers[q.id] === opt.value ? 'selected' : '';
                    html += `<button class="option-button ${selected}" onclick="selectOption('${q.id}', '${opt.value}', this)">${opt.label}</button>`;
                });
            } else if (q.type === 'rating') {
                html += `<div class="rating-container">`;
                for (let i = q.min; i <= q.max; i++) {
                    const selected = answers[q.id] == i ? 'selected' : '';
                    html += `<button class="rating-btn ${selected}" onclick="selectRating('${q.id}', ${i}, this)">${i}</button>`;
                }
                html += `</div>`;
                if (q.minLabel || q.maxLabel) {
                    html += `<div style="display:flex; justify-content:space-between; margin-top:8px; font-size:12px; color:#888;">
                        <span>${q.minLabel || ''}</span><span>${q.maxLabel || ''}</span></div>`;
                }
            } else if (q.type === 'text') {
                const value = answers[q.id] || '';
                html += `<textarea class="text-input" id="textAnswer" placeholder="${q.placeholder || ''}" 
                    onchange="answers['${q.id}'] = this.value">${value}</textarea>`;
            }
            
            html += '</div>';
            container.innerHTML = html;
            
            // Show/hide prev button
            document.getElementById('btnPrev').style.display = currentIndex > 0 ? 'block' : 'none';
            
            // Update next button
            const btnNext = document.getElementById('btnNext');
            if (currentIndex === questions.length - 1) {
                btnNext.textContent = 'ส่งคำตอบ ✓';
            } else {
                btnNext.textContent = 'ถัดไป →';
            }
        }
        
        function selectOption(questionId, value, button) {
            answers[questionId] = value;
            document.querySelectorAll('.option-button').forEach(b => b.classList.remove('selected'));
            button.classList.add('selected');
        }
        
        function selectRating(questionId, value, button) {
            answers[questionId] = value;
            document.querySelectorAll('.rating-btn').forEach(b => b.classList.remove('selected'));
            button.classList.add('selected');
        }
        
        function nextQuestion() {
            const q = questions[currentIndex];
            
            // Validate
            if (q.required !== false && !answers[q.id]) {
                alert('กรุณาตอบคำถามนี้ก่อนดำเนินการต่อ');
                return;
            }
            
            if (q.type === 'text') {
                const textarea = document.getElementById('textAnswer');
                if (textarea) answers[q.id] = textarea.value;
            }
            
            if (currentIndex < questions.length - 1) {
                currentIndex++;
                renderQuestion();
            } else {
                submitSurvey();
            }
        }
        
        function prevQuestion() {
            if (currentIndex > 0) {
                currentIndex--;
                renderQuestion();
            }
        }
        
        async function submitSurvey() {
            // ส่งข้อมูลไป Server
            const response = await fetch('/api/survey/submit', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({
                    userId: userId,
                    answers: answers,
                    completedAt: new Date().toISOString()
                })
            });
            
            // แสดงหน้าขอบคุณ
            document.getElementById('surveyContainer').innerHTML = `
                <div class="completion-screen">
                    <div class="completion-icon">🎉</div>
                    <h2>ขอบคุณสำหรับคำตอบ!</h2>
                    <p>ข้อมูลของคุณมีค่ามากสำหรับเรา</p>
                    <p style="color: #06C755; font-weight: bold;">🎁 คูปองส่วนลด 10% ส่งไปใน LINE แล้วค่ะ</p>
                    <button onclick="liff.closeWindow()" style="margin-top: 20px; padding: 12px 24px; background: #06C755; color: white; border: none; border-radius: 8px; font-size: 16px; cursor: pointer;">
                        ปิดหน้าต่าง
                    </button>
                </div>
            `;
        }
        
        initLiff();
    </script>
</body>
</html>
```

---

## 15.10 แบบฝึกหัด

### แบบฝึกหัดที่ 1: NPS Survey Campaign

**โจทย์:** สร้าง NPS Survey Campaign สำหรับร้านอาหารที่ส่งให้ลูกค้าทุกเดือน

**ข้อกำหนด:**
1. ส่งให้ลูกค้าที่มียอดซื้อในเดือนนั้น
2. ส่ง Reminder 1 ครั้งหลัง 3 วัน
3. คำนวณ NPS และแบ่งกลุ่มลูกค้า
4. ส่ง Follow-up ตาม Segment

```python
def monthly_nps_campaign(month, year, db, access_token):
    """
    TODO: Monthly NPS Campaign
    1. ค้นหาลูกค้าที่ซื้อในเดือนนั้น
    2. ส่ง NPS Survey
    3. หลัง 3 วัน ส่ง Reminder ให้คนที่ยังไม่ตอบ
    4. คำนวณ NPS
    5. แบ่ง Segment และส่ง Follow-up
    """
    pass
```

### แบบฝึกหัดที่ 2: Survey Analytics Dashboard

**โจทย์:** สร้างฟังก์ชันที่สรุปผล Survey ออกมาเป็น Text Report ที่ส่งได้ทาง LINE

```python
def create_survey_summary_report(survey_id, db):
    """
    TODO: สร้าง Report สรุปที่:
    1. แสดง Response Rate
    2. แสดง NPS หรือ CSAT Score
    3. แสดง Top 3 Positive Feedback
    4. แสดง Top 3 Areas for Improvement
    5. Format เป็นข้อความที่อ่านง่าย
    """
    pass
```

---

## 15.11 สรุปบทที่ 15

### Key Takeaways

1. **Survey Types**: Multiple Choice, Text, Rating แต่ละแบบเหมาะกับวัตถุประสงค์ต่างกัน
2. **Response Rate**: เพิ่มด้วย Incentive, ความสั้น และการส่งในเวลาที่เหมาะสม
3. **NPS**: วัดความภักดีของลูกค้า (Promoters - Detractors)
4. **Segmentation**: ใช้ข้อมูล Survey เพื่อส่ง Campaign ที่ตรงกลุ่ม
5. **LIFF**: สร้าง Survey ที่สวยงามและ Interactive ได้ด้วย LIFF

### Survey Best Practices

```
✓ ไม่เกิน 5 คำถาม สำหรับ Quick Survey
✓ มอบ Incentive ที่มีมูลค่า
✓ บอกเวลาที่ใช้ตอบ
✓ ส่งทันทีหลังเหตุการณ์ (Post-purchase, Post-service)
✓ ส่ง Reminder 1 ครั้ง
✓ นำ Feedback ไปปรับปรุงและแจ้งให้ลูกค้าทราบ
✓ วิเคราะห์ Trend รายเดือน
✓ ตอบสนอง Detractors ทันที
```

### Survey Question Quality Checklist

```
□ คำถามชัดเจน ไม่คลุมเครือ
□ ไม่ Leading Question (ชี้นำคำตอบ)
□ ตัวเลือกครอบคลุมทุกกรณี
□ มี "ไม่แน่ใจ/ไม่ทราบ" เมื่อจำเป็น
□ ลำดับคำถามเป็น Logical Flow
□ ทดสอบกับกลุ่มเล็กก่อน Launch
□ มี Mobile-friendly Design
```

---

## สรุป Course Parts 11-15

### สิ่งที่ได้เรียนในส่วนนี้

| Part | หัวข้อ | Skill ที่ได้ |
|------|--------|-----------|
| 11 | Rich Messages | สร้าง Interactive Image Message |
| 12 | Card Messages | นำเสนอสินค้าแบบ Professional |
| 13 | Coupons & Rewards | สร้าง Loyalty Program |
| 14 | Timeline Posts | สร้าง Content Strategy |
| 15 | Survey | เก็บข้อมูลและวิเคราะห์ |

### Next Steps

หลังจากเรียนจบ Parts 11-15 แล้ว ผู้เรียนควร:

1. **สร้าง Rich Message** สำหรับโปรโมชั่นล่าสุด
2. **วางแผน Content Calendar** สำหรับเดือนถัดไป
3. **ตั้งระบบ Coupon** พร้อม Reminder อัตโนมัติ
4. **ทำ NPS Survey** เพื่อวัด Customer Loyalty
5. **วิเคราะห์ข้อมูล** และปรับกลยุทธ์

---

*จบ Part 15 - ยินดีด้วยที่เรียนจบ Parts 11-15 แล้ว!*
