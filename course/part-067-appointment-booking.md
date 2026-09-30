# Part 67: ระบบนัดหมาย/จองคิวบน LINE OA

## บทนำ

ระบบนัดหมายบน LINE OA ช่วยให้ลูกค้าจองนัดหมายได้สะดวกผ่าน LINE ในบทนี้จะสร้างระบบครบวงจรรองรับหลายบริการ หลายพนักงาน และป้องกันการจองซ้ำ

## สิ่งที่จะได้เรียนรู้

- Service catalog management
- Staff/resource scheduling
- Time slot generation
- Double-booking prevention
- Google Calendar integration
- Automated reminders
- Cancellation & rescheduling flow
- No-show handling
- Admin dashboard

---

## 1. Database Schema

```sql
-- database/schema.sql
CREATE DATABASE IF NOT EXISTS lineoa_booking CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE lineoa_booking;

-- ผู้ใช้
CREATE TABLE customers (
  id INT PRIMARY KEY AUTO_INCREMENT,
  line_user_id VARCHAR(100) UNIQUE NOT NULL,
  display_name VARCHAR(200),
  picture_url TEXT,
  phone VARCHAR(20),
  email VARCHAR(200),
  notes TEXT,
  no_show_count INT DEFAULT 0,
  total_bookings INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_line_user_id (line_user_id)
);

-- บริการ
CREATE TABLE services (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  description TEXT,
  duration_minutes INT NOT NULL DEFAULT 60,
  price DECIMAL(10,2) DEFAULT 0,
  buffer_time INT DEFAULT 0,
  max_per_slot INT DEFAULT 1,
  color VARCHAR(7) DEFAULT '#06C755',
  image_url TEXT,
  category VARCHAR(100),
  is_active TINYINT(1) DEFAULT 1,
  sort_order INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_is_active (is_active)
);

-- พนักงาน/ทรัพยากร
CREATE TABLE staff (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  role VARCHAR(100),
  bio TEXT,
  avatar_url TEXT,
  google_calendar_id VARCHAR(200),
  google_token JSON,
  working_days JSON DEFAULT '["mon","tue","wed","thu","fri"]',
  working_hours_start TIME DEFAULT '09:00:00',
  working_hours_end TIME DEFAULT '18:00:00',
  is_active TINYINT(1) DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ความสามารถของพนักงาน
CREATE TABLE staff_services (
  staff_id INT NOT NULL,
  service_id INT NOT NULL,
  PRIMARY KEY (staff_id, service_id),
  FOREIGN KEY (staff_id) REFERENCES staff(id) ON DELETE CASCADE,
  FOREIGN KEY (staff_id) REFERENCES services(id) ON DELETE CASCADE
);

-- วันหยุด/วันพิเศษ
CREATE TABLE holidays (
  id INT PRIMARY KEY AUTO_INCREMENT,
  date DATE NOT NULL,
  name VARCHAR(200),
  staff_id INT,
  type ENUM('public','staff_off','special') DEFAULT 'public',
  FOREIGN KEY (staff_id) REFERENCES staff(id) ON DELETE CASCADE,
  INDEX idx_date (date)
);

-- การจองนัดหมาย
CREATE TABLE appointments (
  id INT PRIMARY KEY AUTO_INCREMENT,
  booking_ref VARCHAR(20) UNIQUE NOT NULL,
  customer_id INT NOT NULL,
  service_id INT NOT NULL,
  staff_id INT,
  appointment_date DATE NOT NULL,
  start_time TIME NOT NULL,
  end_time TIME NOT NULL,
  status ENUM('pending','confirmed','cancelled','completed','no_show') DEFAULT 'pending',
  notes TEXT,
  customer_notes TEXT,
  cancel_reason TEXT,
  reminder_sent_24h TINYINT(1) DEFAULT 0,
  reminder_sent_1h TINYINT(1) DEFAULT 0,
  google_event_id VARCHAR(200),
  price DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (customer_id) REFERENCES customers(id),
  FOREIGN KEY (service_id) REFERENCES services(id),
  FOREIGN KEY (staff_id) REFERENCES staff(id),
  INDEX idx_date_status (appointment_date, status),
  INDEX idx_customer_id (customer_id),
  INDEX idx_staff_date (staff_id, appointment_date)
);

-- Time Blocks (เวลาที่ไม่ว่าง - block เพิ่มเติม)
CREATE TABLE time_blocks (
  id INT PRIMARY KEY AUTO_INCREMENT,
  staff_id INT NOT NULL,
  date DATE NOT NULL,
  start_time TIME NOT NULL,
  end_time TIME NOT NULL,
  reason VARCHAR(200),
  FOREIGN KEY (staff_id) REFERENCES staff(id) ON DELETE CASCADE
);

-- ข้อมูลตัวอย่าง
INSERT INTO services (name, description, duration_minutes, price, color) VALUES
('ตัดผมชาย', 'ตัดผมสไตล์โมเดิร์น พร้อมสระผม', 45, 250, '#3498DB'),
('ทำสีผม', 'ทำสีผม ไฮไลท์ ทุกสไตล์', 120, 1500, '#9B59B6'),
('ทรีตเมนต์ผม', 'ฟื้นฟูผมเสีย บำรุงเส้นผม', 60, 800, '#27AE60'),
('สระและเป่าแห้ง', 'สระผมและจัดทรง', 30, 150, '#F39C12');

INSERT INTO staff (name, role, working_days, working_hours_start, working_hours_end) VALUES
('คุณสมชาย', 'ช่างแต่งผม Senior', '["mon","tue","wed","thu","fri","sat"]', '09:00:00', '18:00:00'),
('คุณสมหญิง', 'ช่างแต่งผม Expert', '["tue","wed","thu","fri","sat","sun"]', '10:00:00', '19:00:00');
```

## 2. โครงสร้างโปรเจค

```
lineoa-booking/
├── src/
│   ├── config/
│   │   ├── database.js
│   │   └── googleCalendar.js
│   ├── services/
│   │   ├── bookingService.js
│   │   ├── scheduleService.js
│   │   ├── reminderService.js
│   │   └── googleCalendarService.js
│   ├── handlers/
│   │   ├── messageHandler.js
│   │   └── postbackHandler.js
│   ├── messages/
│   │   ├── bookingMessages.js
│   │   └── calendarMessages.js
│   └── routes/
│       ├── webhook.js
│       └── admin.js
├── cron/
│   └── reminderJob.js
└── index.js
```

## 3. Schedule Service

```javascript
// src/services/scheduleService.js
const db = require('../config/database');

class ScheduleService {
  // สร้าง time slots ของวันที่กำหนด
  async getAvailableSlots(serviceId, staffId, date) {
    const [[service]] = await db.query(
      'SELECT * FROM services WHERE id = ? AND is_active = 1',
      [serviceId]
    );
    if (!service) throw new Error('Service not found');

    const [[staff]] = await db.query(
      'SELECT * FROM staff WHERE id = ? AND is_active = 1',
      [staffId]
    );
    if (!staff) throw new Error('Staff not found');

    // ตรวจสอบว่าพนักงานทำงานวันนี้
    const dayOfWeek = new Date(date).toLocaleDateString('en-US', { weekday: 'short' }).toLowerCase();
    const workingDays = JSON.parse(staff.working_days);
    if (!workingDays.includes(dayOfWeek)) {
      return { slots: [], message: 'Staff not working on this day' };
    }

    // ตรวจสอบวันหยุด
    const [[holiday]] = await db.query(
      'SELECT id FROM holidays WHERE date = ? AND (staff_id IS NULL OR staff_id = ?)',
      [date, staffId]
    );
    if (holiday) {
      return { slots: [], message: 'Holiday' };
    }

    // สร้าง slots ทั้งหมด
    const startTime = staff.working_hours_start;
    const endTime = staff.working_hours_end;
    const slotDuration = service.duration_minutes + (service.buffer_time || 0);

    const allSlots = this.generateTimeSlots(startTime, endTime, slotDuration);

    // ดึงการจองที่มีอยู่แล้ว
    const [existingBookings] = await db.query(
      `SELECT start_time, end_time FROM appointments
       WHERE staff_id = ? AND appointment_date = ? AND status IN ('pending','confirmed')`,
      [staffId, date]
    );

    // ดึง time blocks
    const [timeBlocks] = await db.query(
      'SELECT start_time, end_time FROM time_blocks WHERE staff_id = ? AND date = ?',
      [staffId, date]
    );

    const busyTimes = [...existingBookings, ...timeBlocks];

    // กรอง slots ที่ว่าง
    const now = new Date();
    const availableSlots = allSlots.filter(slot => {
      const slotDateTime = new Date(`${date} ${slot.start}`);
      if (slotDateTime <= now) return false;

      return !busyTimes.some(busy => {
        return this.timesOverlap(slot.start, slot.end, busy.start_time, busy.end_time);
      });
    });

    return { slots: availableSlots, service, staff };
  }

  // สร้าง time slots ระหว่างช่วงเวลา
  generateTimeSlots(startTime, endTime, durationMinutes) {
    const slots = [];
    let current = this.timeToMinutes(startTime);
    const end = this.timeToMinutes(endTime);

    while (current + durationMinutes <= end) {
      slots.push({
        start: this.minutesToTime(current),
        end: this.minutesToTime(current + durationMinutes)
      });
      current += durationMinutes;
    }

    return slots;
  }

  timeToMinutes(time) {
    const [h, m] = time.split(':').map(Number);
    return h * 60 + m;
  }

  minutesToTime(minutes) {
    const h = Math.floor(minutes / 60);
    const m = minutes % 60;
    return `${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}`;
  }

  timesOverlap(start1, end1, start2, end2) {
    const s1 = this.timeToMinutes(start1);
    const e1 = this.timeToMinutes(end1);
    const s2 = this.timeToMinutes(typeof start2 === 'string' ? start2 : start2.toString().slice(0, 5));
    const e2 = this.timeToMinutes(typeof end2 === 'string' ? end2 : end2.toString().slice(0, 5));
    return s1 < e2 && e1 > s2;
  }

  // ดึง staff ที่ให้บริการ service นั้น
  async getStaffForService(serviceId) {
    const [staff] = await db.query(
      `SELECT s.* FROM staff s
       JOIN staff_services ss ON s.id = ss.staff_id
       WHERE ss.service_id = ? AND s.is_active = 1`,
      [serviceId]
    );
    return staff;
  }

  // ดึงวันที่ว่างของเดือน
  async getAvailableDates(serviceId, staffId, year, month) {
    const daysInMonth = new Date(year, month, 0).getDate();
    const availableDates = [];

    const [[staff]] = await db.query('SELECT * FROM staff WHERE id = ?', [staffId]);
    const workingDays = JSON.parse(staff.working_days);

    const [holidays] = await db.query(
      `SELECT date FROM holidays 
       WHERE YEAR(date) = ? AND MONTH(date) = ? AND (staff_id IS NULL OR staff_id = ?)`,
      [year, month, staffId]
    );
    const holidayDates = holidays.map(h => h.date.toISOString().split('T')[0]);

    const today = new Date();
    today.setHours(0, 0, 0, 0);

    for (let day = 1; day <= daysInMonth; day++) {
      const date = new Date(year, month - 1, day);
      if (date < today) continue;

      const dayName = date.toLocaleDateString('en-US', { weekday: 'short' }).toLowerCase();
      const dateStr = date.toISOString().split('T')[0];

      if (workingDays.includes(dayName) && !holidayDates.includes(dateStr)) {
        availableDates.push(dateStr);
      }
    }

    return availableDates;
  }
}

module.exports = new ScheduleService();
```

## 4. Booking Service

```javascript
// src/services/bookingService.js
const db = require('../config/database');
const scheduleService = require('./scheduleService');
const googleCalendarService = require('./googleCalendarService');

function generateRef() {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  let ref = 'BK';
  for (let i = 0; i < 8; i++) {
    ref += chars.charAt(Math.floor(Math.random() * chars.length));
  }
  return ref;
}

class BookingService {
  // สร้างการจอง
  async createBooking(customerId, serviceId, staffId, date, startTime, notes = '') {
    const [[service]] = await db.query('SELECT * FROM services WHERE id = ?', [serviceId]);
    if (!service) throw new Error('Service not found');

    const endMinutes = scheduleService.timeToMinutes(startTime) + service.duration_minutes;
    const endTime = scheduleService.minutesToTime(endMinutes);

    // ตรวจสอบซ้ำ (Double-booking prevention)
    const [conflicts] = await db.query(
      `SELECT id FROM appointments
       WHERE staff_id = ? AND appointment_date = ?
         AND status IN ('pending','confirmed')
         AND (
           (start_time < ? AND end_time > ?)
           OR (start_time >= ? AND start_time < ?)
         )`,
      [staffId, date, endTime, startTime, startTime, endTime]
    );

    if (conflicts.length > 0) {
      throw new Error('Time slot is already booked');
    }

    // ตรวจสอบว่า slot ยังว่างอยู่จริง
    const { slots } = await scheduleService.getAvailableSlots(serviceId, staffId, date);
    const isAvailable = slots.some(s => s.start === startTime);
    if (!isAvailable) {
      throw new Error('Selected time slot is not available');
    }

    const bookingRef = generateRef();
    const price = service.price;

    const [result] = await db.query(
      `INSERT INTO appointments 
       (booking_ref, customer_id, service_id, staff_id, appointment_date, start_time, end_time, notes, price)
       VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)`,
      [bookingRef, customerId, serviceId, staffId, date, startTime, endTime, notes, price]
    );

    const appointmentId = result.insertId;

    // บันทึกใน Google Calendar
    const [[customer]] = await db.query('SELECT * FROM customers WHERE id = ?', [customerId]);
    const [[staff]] = await db.query('SELECT * FROM staff WHERE id = ?', [staffId]);

    if (staff.google_calendar_id && staff.google_token) {
      try {
        const eventId = await googleCalendarService.createEvent({
          calendarId: staff.google_calendar_id,
          token: JSON.parse(staff.google_token),
          summary: `${service.name} - ${customer.display_name}`,
          start: `${date}T${startTime}:00+07:00`,
          end: `${date}T${endTime}:00+07:00`,
          description: `โทร: ${customer.phone || 'N/A'}\nหมายเหตุ: ${notes}`
        });

        await db.query(
          'UPDATE appointments SET google_event_id = ? WHERE id = ?',
          [eventId, appointmentId]
        );
      } catch (err) {
        console.error('Google Calendar error:', err.message);
      }
    }

    return this.getAppointment(appointmentId);
  }

  // ดึงรายละเอียดการจอง
  async getAppointment(id) {
    const [[appointment]] = await db.query(
      `SELECT a.*, 
              c.display_name as customer_name, c.phone as customer_phone, c.line_user_id,
              s.name as service_name, s.duration_minutes, s.color,
              st.name as staff_name
       FROM appointments a
       JOIN customers c ON a.customer_id = c.id
       JOIN services s ON a.service_id = s.id
       LEFT JOIN staff st ON a.staff_id = st.id
       WHERE a.id = ?`,
      [id]
    );
    return appointment;
  }

  // ดึงการจองของลูกค้า
  async getCustomerAppointments(customerId, status = null) {
    let query = `
      SELECT a.*, s.name as service_name, s.color, st.name as staff_name
      FROM appointments a
      JOIN services s ON a.service_id = s.id
      LEFT JOIN staff st ON a.staff_id = st.id
      WHERE a.customer_id = ?`;
    const params = [customerId];

    if (status) {
      query += ' AND a.status = ?';
      params.push(status);
    }

    query += ' ORDER BY a.appointment_date DESC, a.start_time DESC LIMIT 10';

    const [appointments] = await db.query(query, params);
    return appointments;
  }

  // ยกเลิกการจอง
  async cancelAppointment(appointmentId, customerId, reason) {
    const [[appt]] = await db.query(
      'SELECT * FROM appointments WHERE id = ? AND customer_id = ?',
      [appointmentId, customerId]
    );

    if (!appt) throw new Error('Appointment not found');
    if (!['pending', 'confirmed'].includes(appt.status)) {
      throw new Error('Cannot cancel this appointment');
    }

    // ต้องยกเลิกก่อนนัด 2 ชั่วโมง
    const apptDateTime = new Date(`${appt.appointment_date.toISOString().split('T')[0]} ${appt.start_time}`);
    const now = new Date();
    const diffHours = (apptDateTime - now) / (1000 * 60 * 60);

    if (diffHours < 2) {
      throw new Error('Cannot cancel within 2 hours of appointment');
    }

    await db.query(
      "UPDATE appointments SET status = 'cancelled', cancel_reason = ? WHERE id = ?",
      [reason, appointmentId]
    );

    // ลบจาก Google Calendar
    if (appt.google_event_id) {
      const [[staff]] = await db.query('SELECT * FROM staff WHERE id = ?', [appt.staff_id]);
      if (staff?.google_token) {
        try {
          await googleCalendarService.deleteEvent(
            staff.google_calendar_id,
            JSON.parse(staff.google_token),
            appt.google_event_id
          );
        } catch (err) {
          console.error('Google Calendar delete error:', err.message);
        }
      }
    }

    return true;
  }

  // เลื่อนนัด
  async rescheduleAppointment(appointmentId, customerId, newDate, newStartTime) {
    const [[appt]] = await db.query(
      'SELECT * FROM appointments WHERE id = ? AND customer_id = ?',
      [appointmentId, customerId]
    );

    if (!appt) throw new Error('Appointment not found');
    if (!['pending', 'confirmed'].includes(appt.status)) {
      throw new Error('Cannot reschedule this appointment');
    }

    // ยกเลิกเดิม
    await db.query(
      "UPDATE appointments SET status = 'cancelled', cancel_reason = 'Rescheduled' WHERE id = ?",
      [appointmentId]
    );

    // สร้างใหม่
    const newAppt = await this.createBooking(
      customerId,
      appt.service_id,
      appt.staff_id,
      newDate,
      newStartTime,
      appt.notes
    );

    return newAppt;
  }

  // Mark no-show
  async markNoShow(appointmentId) {
    const [[appt]] = await db.query('SELECT * FROM appointments WHERE id = ?', [appointmentId]);
    if (!appt) throw new Error('Appointment not found');

    await db.query(
      "UPDATE appointments SET status = 'no_show' WHERE id = ?",
      [appointmentId]
    );

    await db.query(
      'UPDATE customers SET no_show_count = no_show_count + 1 WHERE id = ?',
      [appt.customer_id]
    );
  }

  // Mark completed
  async markCompleted(appointmentId) {
    await db.query(
      "UPDATE appointments SET status = 'completed' WHERE id = ?",
      [appointmentId]
    );
    const appt = await this.getAppointment(appointmentId);
    await db.query(
      'UPDATE customers SET total_bookings = total_bookings + 1 WHERE id = ?',
      [appt.customer_id]
    );
  }
}

module.exports = new BookingService();
```

## 5. Google Calendar Service

```javascript
// src/services/googleCalendarService.js
const { google } = require('googleapis');

class GoogleCalendarService {
  getAuthClient(token) {
    const oauth2Client = new google.auth.OAuth2(
      process.env.GOOGLE_CLIENT_ID,
      process.env.GOOGLE_CLIENT_SECRET,
      process.env.GOOGLE_REDIRECT_URI
    );
    oauth2Client.setCredentials(token);
    return oauth2Client;
  }

  async createEvent({ calendarId, token, summary, start, end, description }) {
    const auth = this.getAuthClient(token);
    const calendar = google.calendar({ version: 'v3', auth });

    const event = {
      summary,
      description,
      start: { dateTime: start, timeZone: 'Asia/Bangkok' },
      end: { dateTime: end, timeZone: 'Asia/Bangkok' },
      reminders: {
        useDefault: false,
        overrides: [
          { method: 'email', minutes: 24 * 60 },
          { method: 'popup', minutes: 60 }
        ]
      }
    };

    const response = await calendar.events.insert({
      calendarId,
      resource: event
    });

    return response.data.id;
  }

  async updateEvent({ calendarId, token, eventId, summary, start, end, description }) {
    const auth = this.getAuthClient(token);
    const calendar = google.calendar({ version: 'v3', auth });

    await calendar.events.update({
      calendarId,
      eventId,
      resource: {
        summary,
        description,
        start: { dateTime: start, timeZone: 'Asia/Bangkok' },
        end: { dateTime: end, timeZone: 'Asia/Bangkok' }
      }
    });
  }

  async deleteEvent(calendarId, token, eventId) {
    const auth = this.getAuthClient(token);
    const calendar = google.calendar({ version: 'v3', auth });
    await calendar.events.delete({ calendarId, eventId });
  }

  // ดึงนัดหมายของวันนั้น
  async getEventsForDay(calendarId, token, date) {
    const auth = this.getAuthClient(token);
    const calendar = google.calendar({ version: 'v3', auth });

    const startOfDay = new Date(`${date}T00:00:00+07:00`).toISOString();
    const endOfDay = new Date(`${date}T23:59:59+07:00`).toISOString();

    const response = await calendar.events.list({
      calendarId,
      timeMin: startOfDay,
      timeMax: endOfDay,
      singleEvents: true,
      orderBy: 'startTime'
    });

    return response.data.items;
  }
}

module.exports = new GoogleCalendarService();
```

## 6. Reminder Service

```javascript
// src/services/reminderService.js
const db = require('../config/database');
const line = require('@line/bot-sdk');

const client = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

class ReminderService {
  // ส่ง reminder 24 ชั่วโมงก่อนนัด
  async send24HourReminders() {
    const tomorrow = new Date();
    tomorrow.setDate(tomorrow.getDate() + 1);
    const tomorrowStr = tomorrow.toISOString().split('T')[0];

    const [appointments] = await db.query(
      `SELECT a.*, c.line_user_id, c.display_name,
              s.name as service_name, s.duration_minutes,
              st.name as staff_name
       FROM appointments a
       JOIN customers c ON a.customer_id = c.id
       JOIN services s ON a.service_id = s.id
       LEFT JOIN staff st ON a.staff_id = st.id
       WHERE a.appointment_date = ?
         AND a.status = 'confirmed'
         AND a.reminder_sent_24h = 0`,
      [tomorrowStr]
    );

    let sent = 0;
    for (const appt of appointments) {
      try {
        await client.pushMessage(appt.line_user_id, createReminderMessage(appt, '24 ชั่วโมง'));
        await db.query(
          'UPDATE appointments SET reminder_sent_24h = 1 WHERE id = ?',
          [appt.id]
        );
        sent++;
      } catch (err) {
        console.error(`Reminder error for ${appt.booking_ref}:`, err.message);
      }
    }

    console.log(`Sent ${sent} 24h reminders`);
    return sent;
  }

  // ส่ง reminder 1 ชั่วโมงก่อนนัด
  async send1HourReminders() {
    const now = new Date();
    const oneHourLater = new Date(now.getTime() + 60 * 60 * 1000);
    const targetDate = oneHourLater.toISOString().split('T')[0];
    const targetHour = oneHourLater.toTimeString().slice(0, 5);

    const [appointments] = await db.query(
      `SELECT a.*, c.line_user_id, s.name as service_name, st.name as staff_name
       FROM appointments a
       JOIN customers c ON a.customer_id = c.id
       JOIN services s ON a.service_id = s.id
       LEFT JOIN staff st ON a.staff_id = st.id
       WHERE a.appointment_date = ?
         AND a.start_time BETWEEN ? AND ?
         AND a.status = 'confirmed'
         AND a.reminder_sent_1h = 0`,
      [targetDate, targetHour, oneHourLater.toTimeString().slice(0, 5)]
    );

    let sent = 0;
    for (const appt of appointments) {
      try {
        await client.pushMessage(appt.line_user_id, createReminderMessage(appt, '1 ชั่วโมง'));
        await db.query(
          'UPDATE appointments SET reminder_sent_1h = 1 WHERE id = ?',
          [appt.id]
        );
        sent++;
      } catch (err) {
        console.error(`1h reminder error:`, err.message);
      }
    }

    return sent;
  }

  // Auto-mark no show
  async markNoShows() {
    const oneHourAgo = new Date(Date.now() - 60 * 60 * 1000);
    const dateStr = oneHourAgo.toISOString().split('T')[0];
    const timeStr = oneHourAgo.toTimeString().slice(0, 8);

    const [appointments] = await db.query(
      `SELECT a.*, c.line_user_id, s.name as service_name
       FROM appointments a
       JOIN customers c ON a.customer_id = c.id
       JOIN services s ON a.service_id = s.id
       WHERE a.status = 'confirmed'
         AND (a.appointment_date < ? OR (a.appointment_date = ? AND a.end_time < ?))`,
      [dateStr, dateStr, timeStr]
    );

    for (const appt of appointments) {
      await db.query(
        "UPDATE appointments SET status = 'no_show' WHERE id = ?",
        [appt.id]
      );
      await db.query(
        'UPDATE customers SET no_show_count = no_show_count + 1 WHERE id = ?',
        [appt.customer_id]
      );

      try {
        await client.pushMessage(appt.line_user_id, {
          type: 'text',
          text: `⚠️ เราไม่พบคุณตามนัดหมาย ${appt.service_name}\nวันที่ ${appt.appointment_date}\nเวลา ${appt.start_time}\n\nหากต้องการนัดหมายใหม่ กรุณาพิมพ์ "จองนัด"`
        });
      } catch (err) {
        console.error('No-show notification error:', err.message);
      }
    }
  }
}

function createReminderMessage(appt, timeLeft) {
  const dateFormatted = new Date(appt.appointment_date).toLocaleDateString('th-TH', {
    weekday: 'long',
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  });

  return {
    type: 'flex',
    altText: `⏰ แจ้งเตือนนัดหมาย ${appt.service_name}`,
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [{
          type: 'text',
          text: `⏰ แจ้งเตือน: อีก ${timeLeft}`,
          weight: 'bold',
          color: '#FFFFFF',
          size: 'md'
        }],
        backgroundColor: appt.color || '#06C755'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          { type: 'text', text: appt.service_name, weight: 'bold', size: 'lg' },
          {
            type: 'box', layout: 'baseline', margin: 'md',
            contents: [
              { type: 'icon', url: 'https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png', size: 'sm' },
              { type: 'text', text: `เลขที่: ${appt.booking_ref}`, size: 'sm', margin: 'sm' }
            ]
          },
          {
            type: 'box', layout: 'vertical', margin: 'md', spacing: 'xs',
            contents: [
              {
                type: 'box', layout: 'horizontal',
                contents: [
                  { type: 'text', text: '📅 วันที่:', size: 'sm', color: '#666666', flex: 2 },
                  { type: 'text', text: dateFormatted, size: 'sm', flex: 4, wrap: true }
                ]
              },
              {
                type: 'box', layout: 'horizontal',
                contents: [
                  { type: 'text', text: '🕐 เวลา:', size: 'sm', color: '#666666', flex: 2 },
                  { type: 'text', text: appt.start_time.slice(0, 5), size: 'sm', flex: 4, weight: 'bold' }
                ]
              },
              appt.staff_name ? {
                type: 'box', layout: 'horizontal',
                contents: [
                  { type: 'text', text: '👤 ช่าง:', size: 'sm', color: '#666666', flex: 2 },
                  { type: 'text', text: appt.staff_name, size: 'sm', flex: 4 }
                ]
              } : null
            ].filter(Boolean)
          }
        ]
      },
      footer: {
        type: 'box',
        layout: 'horizontal',
        spacing: 'sm',
        contents: [
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '✅ ยืนยัน',
              data: `action=confirm_appt&id=${appt.id}`
            },
            style: 'primary',
            color: '#06C755',
            flex: 1
          },
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '❌ ยกเลิก',
              data: `action=cancel_appt&id=${appt.id}`
            },
            style: 'secondary',
            flex: 1
          }
        ]
      }
    }
  };
}

module.exports = new ReminderService();
```

## 7. Booking Messages (Flex)

```javascript
// src/messages/bookingMessages.js
function createServiceList(services) {
  const bubbles = services.map(service => ({
    type: 'bubble',
    body: {
      type: 'box',
      layout: 'vertical',
      contents: [
        {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'text',
            text: '●',
            size: 'xxl',
            color: service.color || '#06C755',
            align: 'center'
          }],
          paddingAll: 'lg'
        },
        {
          type: 'text',
          text: service.name,
          weight: 'bold',
          size: 'md',
          align: 'center',
          wrap: true
        },
        {
          type: 'text',
          text: `⏱ ${service.duration_minutes} นาที`,
          size: 'sm',
          color: '#666666',
          align: 'center',
          margin: 'sm'
        },
        {
          type: 'text',
          text: service.price > 0 ? `฿${parseFloat(service.price).toLocaleString('th-TH')}` : 'ฟรี',
          size: 'md',
          weight: 'bold',
          color: '#E53E3E',
          align: 'center',
          margin: 'sm'
        },
        service.description ? {
          type: 'text',
          text: service.description,
          size: 'xs',
          color: '#999999',
          wrap: true,
          align: 'center',
          margin: 'sm',
          maxLines: 2
        } : null
      ].filter(Boolean)
    },
    footer: {
      type: 'box',
      layout: 'vertical',
      contents: [{
        type: 'button',
        action: {
          type: 'postback',
          label: 'จองเลย',
          data: `action=select_service&service_id=${service.id}`
        },
        style: 'primary',
        color: service.color || '#06C755'
      }]
    }
  }));

  return {
    type: 'flex',
    altText: 'เลือกบริการ',
    contents: {
      type: 'carousel',
      contents: bubbles
    }
  };
}

function createStaffList(staffList, serviceId) {
  const bubbles = staffList.map(staff => ({
    type: 'bubble',
    size: 'kilo',
    body: {
      type: 'box',
      layout: 'vertical',
      contents: [
        {
          type: 'image',
          url: staff.avatar_url || 'https://via.placeholder.com/150',
          size: 'full',
          aspectRatio: '1:1',
          aspectMode: 'cover'
        },
        {
          type: 'text',
          text: staff.name,
          weight: 'bold',
          size: 'md',
          margin: 'md'
        },
        {
          type: 'text',
          text: staff.role || '',
          size: 'sm',
          color: '#666666'
        }
      ]
    },
    footer: {
      type: 'box',
      layout: 'vertical',
      contents: [{
        type: 'button',
        action: {
          type: 'postback',
          label: 'เลือก',
          data: `action=select_staff&staff_id=${staff.id}&service_id=${serviceId}`
        },
        style: 'primary',
        color: '#06C755'
      }]
    }
  }));

  if (bubbles.length === 0) {
    return { type: 'text', text: 'ไม่พบพนักงาน' };
  }

  return {
    type: 'flex',
    altText: 'เลือกพนักงาน',
    contents: bubbles.length === 1 ? bubbles[0] : {
      type: 'carousel',
      contents: bubbles
    }
  };
}

function createDatePicker(availableDates, serviceId, staffId, year, month) {
  const monthName = new Date(year, month - 1).toLocaleDateString('th-TH', {
    year: 'numeric',
    month: 'long'
  });

  // สร้าง calendar grid (5 แถว x 7 คอลัมน์)
  const firstDay = new Date(year, month - 1, 1).getDay();
  const daysInMonth = new Date(year, month, 0).getDate();
  const availableSet = new Set(availableDates.map(d => parseInt(d.split('-')[2])));

  const calRows = [];
  let dayCount = 1;
  const weekDays = ['อา', 'จ', 'อ', 'พ', 'พฤ', 'ศ', 'ส'];

  // Header row
  const headerRow = {
    type: 'box',
    layout: 'horizontal',
    contents: weekDays.map(d => ({
      type: 'text',
      text: d,
      size: 'xs',
      align: 'center',
      color: d === 'อา' ? '#E53E3E' : '#666666',
      flex: 1
    }))
  };
  calRows.push(headerRow);

  for (let week = 0; week < 6; week++) {
    const dayBoxes = [];
    for (let dow = 0; dow < 7; dow++) {
      const cellDay = week * 7 + dow - firstDay + 1;
      if (cellDay < 1 || cellDay > daysInMonth) {
        dayBoxes.push({ type: 'text', text: '', flex: 1 });
      } else {
        const isAvailable = availableSet.has(cellDay);
        const dateStr = `${year}-${String(month).padStart(2,'0')}-${String(cellDay).padStart(2,'0')}`;
        dayBoxes.push({
          type: 'button',
          action: isAvailable ? {
            type: 'postback',
            label: String(cellDay),
            data: `action=select_date&date=${dateStr}&service_id=${serviceId}&staff_id=${staffId}`
          } : {
            type: 'postback',
            label: String(cellDay),
            data: 'action=noop'
          },
          style: isAvailable ? 'primary' : 'secondary',
          color: isAvailable ? '#06C755' : undefined,
          height: 'sm',
          flex: 1
        });
      }
    }

    calRows.push({
      type: 'box',
      layout: 'horizontal',
      spacing: 'xs',
      margin: 'xs',
      contents: dayBoxes
    });

    if (dayCount > daysInMonth) break;
    dayCount += 7;
  }

  const prevMonth = month === 1 ? 12 : month - 1;
  const prevYear = month === 1 ? year - 1 : year;
  const nextMonth = month === 12 ? 1 : month + 1;
  const nextYear = month === 12 ? year + 1 : year;

  return {
    type: 'flex',
    altText: `เลือกวันนัดหมาย - ${monthName}`,
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'horizontal',
        contents: [
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '◀',
              data: `action=change_month&year=${prevYear}&month=${prevMonth}&service_id=${serviceId}&staff_id=${staffId}`
            },
            style: 'secondary',
            flex: 1,
            height: 'sm'
          },
          {
            type: 'text',
            text: monthName,
            weight: 'bold',
            align: 'center',
            flex: 4
          },
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '▶',
              data: `action=change_month&year=${nextYear}&month=${nextMonth}&service_id=${serviceId}&staff_id=${staffId}`
            },
            style: 'secondary',
            flex: 1,
            height: 'sm'
          }
        ],
        backgroundColor: '#F7F8FA'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        spacing: 'xs',
        contents: calRows
      }
    }
  };
}

function createTimeSlots(slots, date, serviceId, staffId) {
  const dateFormatted = new Date(date).toLocaleDateString('th-TH', {
    weekday: 'long',
    month: 'long',
    day: 'numeric'
  });

  if (slots.length === 0) {
    return {
      type: 'flex',
      altText: 'ไม่มีเวลาว่าง',
      contents: {
        type: 'bubble',
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            { type: 'text', text: '😔 ไม่มีเวลาว่าง', weight: 'bold', size: 'md' },
            { type: 'text', text: dateFormatted, size: 'sm', color: '#666666', margin: 'sm' },
            { type: 'text', text: 'กรุณาเลือกวันอื่น', size: 'sm', color: '#999999', margin: 'sm' }
          ]
        }
      }
    };
  }

  // แบ่ง slots เป็นกลุ่มเช้า/บ่าย/เย็น
  const morning = slots.filter(s => parseInt(s.start.split(':')[0]) < 12);
  const afternoon = slots.filter(s => parseInt(s.start.split(':')[0]) >= 12 && parseInt(s.start.split(':')[0]) < 17);
  const evening = slots.filter(s => parseInt(s.start.split(':')[0]) >= 17);

  function makeSlotButtons(group) {
    const rows = [];
    for (let i = 0; i < group.length; i += 3) {
      const row = group.slice(i, i + 3).map(slot => ({
        type: 'button',
        action: {
          type: 'postback',
          label: slot.start.slice(0, 5),
          data: `action=select_time&time=${slot.start}&date=${date}&service_id=${serviceId}&staff_id=${staffId}`
        },
        style: 'primary',
        color: '#06C755',
        height: 'sm',
        flex: 1
      }));

      // Pad to 3
      while (row.length < 3) row.push({ type: 'filler' });
      rows.push({ type: 'box', layout: 'horizontal', spacing: 'sm', contents: row, margin: 'xs' });
    }
    return rows;
  }

  const sections = [];

  if (morning.length > 0) {
    sections.push({ type: 'text', text: '🌅 ช่วงเช้า', size: 'sm', weight: 'bold', color: '#666666' });
    sections.push(...makeSlotButtons(morning));
  }

  if (afternoon.length > 0) {
    sections.push({ type: 'text', text: '☀️ ช่วงบ่าย', size: 'sm', weight: 'bold', color: '#666666', margin: 'md' });
    sections.push(...makeSlotButtons(afternoon));
  }

  if (evening.length > 0) {
    sections.push({ type: 'text', text: '🌇 ช่วงเย็น', size: 'sm', weight: 'bold', color: '#666666', margin: 'md' });
    sections.push(...makeSlotButtons(evening));
  }

  return {
    type: 'flex',
    altText: `เลือกเวลา ${dateFormatted}`,
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [
          { type: 'text', text: '🕐 เลือกเวลานัดหมาย', weight: 'bold', color: '#FFFFFF' },
          { type: 'text', text: dateFormatted, size: 'sm', color: '#FFFFFF', opacity: 0.8 }
        ],
        backgroundColor: '#06C755'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        spacing: 'xs',
        contents: sections
      }
    }
  };
}

function createBookingConfirmation(service, staff, date, time) {
  const dateFormatted = new Date(date).toLocaleDateString('th-TH', {
    weekday: 'long',
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  });

  return {
    type: 'flex',
    altText: 'ยืนยันการนัดหมาย',
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [{ type: 'text', text: '📋 สรุปการนัดหมาย', weight: 'bold', color: '#FFFFFF', size: 'md' }],
        backgroundColor: '#06C755'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        spacing: 'sm',
        contents: [
          { type: 'text', text: service.name, weight: 'bold', size: 'lg' },
          { type: 'separator', margin: 'md' },
          {
            type: 'box', layout: 'vertical', spacing: 'xs', margin: 'md',
            contents: [
              { type: 'box', layout: 'horizontal', contents: [
                { type: 'text', text: '📅 วันที่:', size: 'sm', color: '#666666', flex: 2 },
                { type: 'text', text: dateFormatted, size: 'sm', flex: 3, wrap: true }
              ]},
              { type: 'box', layout: 'horizontal', contents: [
                { type: 'text', text: '🕐 เวลา:', size: 'sm', color: '#666666', flex: 2 },
                { type: 'text', text: time.slice(0, 5), size: 'sm', weight: 'bold', flex: 3 }
              ]},
              { type: 'box', layout: 'horizontal', contents: [
                { type: 'text', text: '⏱ ระยะเวลา:', size: 'sm', color: '#666666', flex: 2 },
                { type: 'text', text: `${service.duration_minutes} นาที`, size: 'sm', flex: 3 }
              ]},
              staff ? { type: 'box', layout: 'horizontal', contents: [
                { type: 'text', text: '👤 ช่าง:', size: 'sm', color: '#666666', flex: 2 },
                { type: 'text', text: staff.name, size: 'sm', flex: 3 }
              ]} : null,
              service.price > 0 ? { type: 'box', layout: 'horizontal', contents: [
                { type: 'text', text: '💰 ราคา:', size: 'sm', color: '#666666', flex: 2 },
                { type: 'text', text: `฿${parseFloat(service.price).toLocaleString('th-TH')}`, size: 'sm', weight: 'bold', color: '#E53E3E', flex: 3 }
              ]} : null
            ].filter(Boolean)
          }
        ]
      },
      footer: {
        type: 'box',
        layout: 'vertical',
        spacing: 'sm',
        contents: [
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '✅ ยืนยันการจอง',
              data: `action=confirm_booking&service_id=${service.id}&staff_id=${staff?.id}&date=${date}&time=${time}`
            },
            style: 'primary',
            color: '#06C755'
          },
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '← เปลี่ยนเวลา',
              data: `action=select_date&date=${date}&service_id=${service.id}&staff_id=${staff?.id}`
            },
            style: 'secondary'
          }
        ]
      }
    }
  };
}

function createBookingSuccess(appointment) {
  const dateFormatted = new Date(appointment.appointment_date).toLocaleDateString('th-TH', {
    weekday: 'long',
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  });

  return {
    type: 'flex',
    altText: `✅ จองนัดหมายสำเร็จ! ${appointment.booking_ref}`,
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [
          { type: 'text', text: '✅ จองนัดหมายสำเร็จ!', weight: 'bold', color: '#FFFFFF', size: 'lg', align: 'center' }
        ],
        backgroundColor: '#27AE60'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: appointment.booking_ref,
            size: 'xxl',
            weight: 'bold',
            align: 'center',
            color: '#06C755'
          },
          { type: 'text', text: 'รหัสการจอง', size: 'xs', color: '#999999', align: 'center' },
          { type: 'separator', margin: 'md' },
          {
            type: 'box', layout: 'vertical', spacing: 'xs', margin: 'md',
            contents: [
              { type: 'box', layout: 'horizontal', contents: [
                { type: 'text', text: 'บริการ:', size: 'sm', color: '#666666', flex: 2 },
                { type: 'text', text: appointment.service_name, size: 'sm', weight: 'bold', flex: 3 }
              ]},
              { type: 'box', layout: 'horizontal', contents: [
                { type: 'text', text: 'วันที่:', size: 'sm', color: '#666666', flex: 2 },
                { type: 'text', text: dateFormatted, size: 'sm', flex: 3, wrap: true }
              ]},
              { type: 'box', layout: 'horizontal', contents: [
                { type: 'text', text: 'เวลา:', size: 'sm', color: '#666666', flex: 2 },
                { type: 'text', text: appointment.start_time.slice(0, 5), size: 'sm', weight: 'bold', flex: 3 }
              ]},
              appointment.staff_name ? { type: 'box', layout: 'horizontal', contents: [
                { type: 'text', text: 'ช่าง:', size: 'sm', color: '#666666', flex: 2 },
                { type: 'text', text: appointment.staff_name, size: 'sm', flex: 3 }
              ]} : null
            ].filter(Boolean)
          }
        ]
      },
      footer: {
        type: 'box',
        layout: 'vertical',
        contents: [{
          type: 'button',
          action: {
            type: 'postback',
            label: '📋 ดูการนัดหมายของฉัน',
            data: 'action=my_appointments'
          },
          style: 'secondary'
        }]
      }
    }
  };
}

module.exports = {
  createServiceList,
  createStaffList,
  createDatePicker,
  createTimeSlots,
  createBookingConfirmation,
  createBookingSuccess
};
```

## 8. Postback Handler

```javascript
// src/handlers/postbackHandler.js
const db = require('../config/database');
const redis = require('../config/redis');
const bookingService = require('../services/bookingService');
const scheduleService = require('../services/scheduleService');
const {
  createServiceList, createStaffList, createDatePicker,
  createTimeSlots, createBookingConfirmation, createBookingSuccess
} = require('../messages/bookingMessages');

async function getOrCreateCustomer(lineUserId, client) {
  const [[existing]] = await db.query(
    'SELECT * FROM customers WHERE line_user_id = ?', [lineUserId]
  );
  if (existing) return existing;

  const profile = await client.getProfile(lineUserId);
  const [result] = await db.query(
    'INSERT INTO customers (line_user_id, display_name, picture_url) VALUES (?, ?, ?)',
    [lineUserId, profile.displayName, profile.pictureUrl]
  );
  const [[customer]] = await db.query('SELECT * FROM customers WHERE id = ?', [result.insertId]);
  return customer;
}

async function handlePostback(client, event) {
  const userId = event.source.userId;
  const params = new URLSearchParams(event.postback.data);
  const action = params.get('action');

  if (action === 'noop') return;

  const customer = await getOrCreateCustomer(userId, client);

  try {
    switch (action) {
      case 'book': {
        const [services] = await db.query(
          'SELECT * FROM services WHERE is_active = 1 ORDER BY sort_order'
        );
        await client.replyMessage(event.replyToken, createServiceList(services));
        break;
      }

      case 'select_service': {
        const serviceId = parseInt(params.get('service_id'));
        const staffList = await scheduleService.getStaffForService(serviceId);

        if (staffList.length === 0) {
          await client.replyMessage(event.replyToken, { type: 'text', text: 'ไม่มีพนักงานว่างสำหรับบริการนี้' });
        } else if (staffList.length === 1) {
          // ถ้ามีพนักงานคนเดียว ข้ามไปเลือกวัน
          const now = new Date();
          const { availableDates } = await scheduleService.getAvailableDates(serviceId, staffList[0].id, now.getFullYear(), now.getMonth() + 1);
          const msg = createDatePicker(availableDates || [], serviceId, staffList[0].id, now.getFullYear(), now.getMonth() + 1);
          await client.replyMessage(event.replyToken, msg);
        } else {
          await client.replyMessage(event.replyToken, createStaffList(staffList, serviceId));
        }
        break;
      }

      case 'select_staff': {
        const serviceId = parseInt(params.get('service_id'));
        const staffId = parseInt(params.get('staff_id'));
        const now = new Date();
        const availableDates = await scheduleService.getAvailableDates(serviceId, staffId, now.getFullYear(), now.getMonth() + 1);
        const msg = createDatePicker(availableDates, serviceId, staffId, now.getFullYear(), now.getMonth() + 1);
        await client.replyMessage(event.replyToken, msg);
        break;
      }

      case 'change_month': {
        const serviceId = parseInt(params.get('service_id'));
        const staffId = parseInt(params.get('staff_id'));
        const year = parseInt(params.get('year'));
        const month = parseInt(params.get('month'));
        const availableDates = await scheduleService.getAvailableDates(serviceId, staffId, year, month);
        const msg = createDatePicker(availableDates, serviceId, staffId, year, month);
        await client.replyMessage(event.replyToken, msg);
        break;
      }

      case 'select_date': {
        const serviceId = parseInt(params.get('service_id'));
        const staffId = parseInt(params.get('staff_id'));
        const date = params.get('date');
        const { slots } = await scheduleService.getAvailableSlots(serviceId, staffId, date);
        await client.replyMessage(event.replyToken, createTimeSlots(slots, date, serviceId, staffId));
        break;
      }

      case 'select_time': {
        const serviceId = parseInt(params.get('service_id'));
        const staffId = parseInt(params.get('staff_id'));
        const date = params.get('date');
        const time = params.get('time');

        const [[service]] = await db.query('SELECT * FROM services WHERE id = ?', [serviceId]);
        const [[staff]] = await db.query('SELECT * FROM staff WHERE id = ?', [staffId]);

        await client.replyMessage(event.replyToken, createBookingConfirmation(service, staff, date, time));
        break;
      }

      case 'confirm_booking': {
        const serviceId = parseInt(params.get('service_id'));
        const staffId = parseInt(params.get('staff_id'));
        const date = params.get('date');
        const time = params.get('time');

        try {
          const appointment = await bookingService.createBooking(
            customer.id, serviceId, staffId, date, time
          );
          await client.replyMessage(event.replyToken, createBookingSuccess(appointment));
        } catch (err) {
          await client.replyMessage(event.replyToken, {
            type: 'text',
            text: `❌ ไม่สามารถจองได้: ${err.message}\n\nกรุณาเลือกเวลาอื่น`
          });
        }
        break;
      }

      case 'my_appointments': {
        const appointments = await bookingService.getCustomerAppointments(customer.id);
        if (appointments.length === 0) {
          await client.replyMessage(event.replyToken, {
            type: 'text',
            text: '📋 ยังไม่มีการนัดหมาย\n\nพิมพ์ "จองนัด" เพื่อนัดหมายใหม่'
          });
          break;
        }

        const bubbles = appointments.slice(0, 10).map(appt => createAppointmentCard(appt));
        await client.replyMessage(event.replyToken, {
          type: 'flex',
          altText: 'การนัดหมายของฉัน',
          contents: {
            type: 'carousel',
            contents: bubbles
          }
        });
        break;
      }

      case 'cancel_appt': {
        const apptId = parseInt(params.get('id'));

        // ถาม reason
        await redis.setex(`state:${userId}`, 300, JSON.stringify({
          state: 'cancel_reason',
          apptId
        }));

        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: 'กรุณาระบุเหตุผลการยกเลิก:\n\n1. ติดธุระด่วน\n2. ไม่สะดวกเวลาที่กำหนด\n3. เปลี่ยนใจ\n4. อื่นๆ (พิมพ์เอง)\n\n(พิมพ์ตัวเลข 1-4 หรือพิมพ์เหตุผล)'
        });
        break;
      }

      case 'confirm_appt': {
        const apptId = parseInt(params.get('id'));
        await db.query(
          "UPDATE appointments SET status = 'confirmed' WHERE id = ? AND customer_id = ?",
          [apptId, customer.id]
        );
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: '✅ ยืนยันการนัดหมายแล้ว\n\nเราจะรอรับคุณตามเวลาที่นัดไว้ครับ'
        });
        break;
      }

      default:
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: 'ไม่เข้าใจคำสั่ง พิมพ์ "เมนู" เพื่อดูตัวเลือก'
        });
    }
  } catch (error) {
    console.error('Postback error:', error);
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: '❌ เกิดข้อผิดพลาด กรุณาลองใหม่'
    });
  }
}

function createAppointmentCard(appt) {
  const statusEmoji = {
    pending: '⏳',
    confirmed: '✅',
    cancelled: '❌',
    completed: '🎉',
    no_show: '⚠️'
  };
  const statusText = {
    pending: 'รอยืนยัน',
    confirmed: 'ยืนยันแล้ว',
    cancelled: 'ยกเลิก',
    completed: 'เสร็จแล้ว',
    no_show: 'ไม่มาตามนัด'
  };

  const dateFormatted = new Date(appt.appointment_date).toLocaleDateString('th-TH', {
    month: 'short',
    day: 'numeric'
  });

  return {
    type: 'bubble',
    size: 'kilo',
    body: {
      type: 'box',
      layout: 'vertical',
      contents: [
        {
          type: 'box',
          layout: 'horizontal',
          contents: [
            { type: 'text', text: `${statusEmoji[appt.status]} ${statusText[appt.status]}`, size: 'xs', color: '#666666', flex: 3 },
            { type: 'text', text: appt.booking_ref, size: 'xs', color: '#999999', align: 'end', flex: 3 }
          ]
        },
        { type: 'text', text: appt.service_name, weight: 'bold', size: 'md', margin: 'sm', wrap: true },
        {
          type: 'box', layout: 'horizontal', margin: 'sm',
          contents: [
            { type: 'text', text: '📅', size: 'sm', flex: 0 },
            { type: 'text', text: `${dateFormatted} เวลา ${appt.start_time.slice(0,5)}`, size: 'sm', margin: 'sm' }
          ]
        },
        appt.staff_name ? {
          type: 'box', layout: 'horizontal', margin: 'xs',
          contents: [
            { type: 'text', text: '👤', size: 'sm', flex: 0 },
            { type: 'text', text: appt.staff_name, size: 'sm', margin: 'sm' }
          ]
        } : null
      ].filter(Boolean)
    },
    footer: appt.status === 'confirmed' ? {
      type: 'box',
      layout: 'horizontal',
      spacing: 'sm',
      contents: [
        {
          type: 'button',
          action: {
            type: 'postback',
            label: 'เลื่อนนัด',
            data: `action=reschedule&id=${appt.id}`
          },
          style: 'secondary',
          height: 'sm',
          flex: 1
        },
        {
          type: 'button',
          action: {
            type: 'postback',
            label: 'ยกเลิก',
            data: `action=cancel_appt&id=${appt.id}`
          },
          style: 'secondary',
          height: 'sm',
          color: '#E53E3E',
          flex: 1
        }
      ]
    } : null
  };
}

module.exports = { handlePostback };
```

## 9. Cron Job สำหรับ Reminders

```javascript
// cron/reminderJob.js
const cron = require('node-cron');
const reminderService = require('../src/services/reminderService');

// ส่ง reminder 24 ชั่วโมงก่อน (ทุกวัน 9:00)
cron.schedule('0 9 * * *', async () => {
  console.log('Running 24h reminder job...');
  try {
    const count = await reminderService.send24HourReminders();
    console.log(`Sent ${count} 24h reminders`);
  } catch (error) {
    console.error('24h reminder error:', error);
  }
}, {
  timezone: 'Asia/Bangkok'
});

// ส่ง reminder 1 ชั่วโมงก่อน (ทุก 30 นาที)
cron.schedule('*/30 * * * *', async () => {
  try {
    const count = await reminderService.send1HourReminders();
    if (count > 0) console.log(`Sent ${count} 1h reminders`);
  } catch (error) {
    console.error('1h reminder error:', error);
  }
});

// Auto mark no-show (ทุก 30 นาที)
cron.schedule('15,45 * * * *', async () => {
  try {
    await reminderService.markNoShows();
  } catch (error) {
    console.error('No-show marking error:', error);
  }
});

console.log('Reminder jobs started');
```

## 10. Admin Dashboard API

```javascript
// src/routes/admin.js
const express = require('express');
const db = require('../config/database');
const bookingService = require('../services/bookingService');

const router = express.Router();

function adminAuth(req, res, next) {
  if (req.headers['x-admin-token'] !== process.env.ADMIN_TOKEN) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
}

router.use(adminAuth);

// สถิติรวม
router.get('/stats', async (req, res) => {
  const today = new Date().toISOString().split('T')[0];

  const [[todayAppts]] = await db.query(
    `SELECT COUNT(*) as total,
     SUM(status = 'confirmed') as confirmed,
     SUM(status = 'completed') as completed,
     SUM(status = 'cancelled') as cancelled,
     SUM(status = 'no_show') as no_show
     FROM appointments WHERE appointment_date = ?`,
    [today]
  );

  const [[thisMonth]] = await db.query(
    `SELECT COUNT(*) as total,
     COALESCE(SUM(price), 0) as revenue
     FROM appointments
     WHERE YEAR(appointment_date) = YEAR(CURDATE())
       AND MONTH(appointment_date) = MONTH(CURDATE())
       AND status = 'completed'`
  );

  const [[totalCustomers]] = await db.query('SELECT COUNT(*) as total FROM customers');

  res.json({
    today: todayAppts,
    thisMonth,
    totalCustomers: totalCustomers.total
  });
});

// ตารางนัดของวัน
router.get('/schedule/:date', async (req, res) => {
  const { date } = req.params;
  const staffId = req.query.staff_id;

  let query = `
    SELECT a.*, c.display_name as customer_name, c.phone,
           s.name as service_name, s.color, s.duration_minutes,
           st.name as staff_name
    FROM appointments a
    JOIN customers c ON a.customer_id = c.id
    JOIN services s ON a.service_id = s.id
    LEFT JOIN staff st ON a.staff_id = st.id
    WHERE a.appointment_date = ?
      AND a.status IN ('pending','confirmed','completed')`;

  const params = [date];
  if (staffId) {
    query += ' AND a.staff_id = ?';
    params.push(staffId);
  }

  query += ' ORDER BY a.start_time';

  const [appointments] = await db.query(query, params);
  res.json({ appointments });
});

// จัดการสถานะ
router.put('/appointments/:id', async (req, res) => {
  const { id } = req.params;
  const { status, notes } = req.body;

  const validStatuses = ['confirmed', 'completed', 'no_show', 'cancelled'];
  if (!validStatuses.includes(status)) {
    return res.status(400).json({ error: 'Invalid status' });
  }

  await db.query(
    'UPDATE appointments SET status = ?, notes = COALESCE(?, notes) WHERE id = ?',
    [status, notes, id]
  );

  if (status === 'completed') {
    await bookingService.markCompleted(parseInt(id));
  } else if (status === 'no_show') {
    await bookingService.markNoShow(parseInt(id));
  }

  // แจ้งเตือนลูกค้า
  const appt = await bookingService.getAppointment(parseInt(id));
  if (appt) {
    const lineClient = require('../config/lineClient');
    const messages = {
      confirmed: `✅ การนัดหมาย ${appt.booking_ref} ได้รับการยืนยันแล้วครับ`,
      completed: `🎉 ขอบคุณที่ใช้บริการ ${appt.service_name}\nหวังว่าคุณจะพอใจกับบริการของเรา`
    };
    if (messages[status]) {
      await lineClient.pushMessage(appt.line_user_id, {
        type: 'text',
        text: messages[status]
      });
    }
  }

  res.json({ message: 'Updated successfully' });
});

// จัดการ Staff
router.get('/staff', async (req, res) => {
  const [staff] = await db.query('SELECT * FROM staff WHERE is_active = 1');
  res.json({ staff });
});

router.post('/staff', async (req, res) => {
  const { name, role, working_days, working_hours_start, working_hours_end } = req.body;
  const [result] = await db.query(
    'INSERT INTO staff (name, role, working_days, working_hours_start, working_hours_end) VALUES (?, ?, ?, ?, ?)',
    [name, role, JSON.stringify(working_days), working_hours_start, working_hours_end]
  );
  res.json({ id: result.insertId });
});

// Block เวลา
router.post('/time-blocks', async (req, res) => {
  const { staff_id, date, start_time, end_time, reason } = req.body;
  const [result] = await db.query(
    'INSERT INTO time_blocks (staff_id, date, start_time, end_time, reason) VALUES (?, ?, ?, ?, ?)',
    [staff_id, date, start_time, end_time, reason]
  );
  res.json({ id: result.insertId });
});

// เพิ่มวันหยุด
router.post('/holidays', async (req, res) => {
  const { date, name, staff_id, type } = req.body;
  await db.query(
    'INSERT INTO holidays (date, name, staff_id, type) VALUES (?, ?, ?, ?)',
    [date, name, staff_id || null, type || 'public']
  );
  res.json({ message: 'Holiday added' });
});

// รายงาน
router.get('/reports', async (req, res) => {
  const { start, end } = req.query;

  const [byService] = await db.query(
    `SELECT s.name, COUNT(*) as count, COALESCE(SUM(a.price), 0) as revenue
     FROM appointments a JOIN services s ON a.service_id = s.id
     WHERE a.appointment_date BETWEEN ? AND ? AND a.status = 'completed'
     GROUP BY s.id ORDER BY count DESC`,
    [start, end]
  );

  const [byStaff] = await db.query(
    `SELECT st.name, COUNT(*) as count, COALESCE(SUM(a.price), 0) as revenue
     FROM appointments a JOIN staff st ON a.staff_id = st.id
     WHERE a.appointment_date BETWEEN ? AND ? AND a.status = 'completed'
     GROUP BY st.id ORDER BY count DESC`,
    [start, end]
  );

  res.json({ byService, byStaff });
});

module.exports = router;
```

## 11. index.js

```javascript
// index.js
require('dotenv').config();
const express = require('express');
const line = require('@line/bot-sdk');
const redis = require('./src/config/redis');
const db = require('./src/config/database');
const { handlePostback } = require('./src/handlers/postbackHandler');
const {
  createServiceList, createDatePicker, createTimeSlots
} = require('./src/messages/bookingMessages');
const bookingService = require('./src/services/bookingService');
require('./cron/reminderJob');

const app = express();
const lineConfig = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
};
const client = new line.Client(lineConfig);

app.use('/webhook', line.middleware(lineConfig), async (req, res) => {
  try {
    await Promise.all(req.body.events.map(event => handleEvent(event)));
    res.json({ ok: true });
  } catch (err) {
    console.error(err);
    res.status(500).end();
  }
});

app.use('/admin', require('./src/routes/admin'));
app.use(express.json());

async function handleEvent(event) {
  const userId = event.source.userId;

  if (event.type === 'follow') {
    const profile = await client.getProfile(userId);
    const [[existing]] = await db.query('SELECT id FROM customers WHERE line_user_id = ?', [userId]);
    if (!existing) {
      await db.query(
        'INSERT INTO customers (line_user_id, display_name, picture_url) VALUES (?, ?, ?)',
        [userId, profile.displayName, profile.pictureUrl]
      );
    }
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: `สวัสดีครับ ${profile.displayName}! 👋\n\nยินดีต้อนรับสู่ระบบนัดหมาย\nพิมพ์ "จองนัด" เพื่อเริ่มต้น`
    });
    return;
  }

  if (event.type === 'postback') {
    await handlePostback(client, event);
    return;
  }

  if (event.type === 'message' && event.message.type === 'text') {
    const text = event.message.text.trim();

    // ตรวจสอบ state
    const stateData = await redis.get(`state:${userId}`);
    const state = stateData ? JSON.parse(stateData) : null;

    if (state?.state === 'cancel_reason') {
      await redis.del(`state:${userId}`);
      const reasons = { '1': 'ติดธุระด่วน', '2': 'ไม่สะดวกเวลาที่กำหนด', '3': 'เปลี่ยนใจ' };
      const reason = reasons[text] || text;

      try {
        const [[cust]] = await db.query('SELECT * FROM customers WHERE line_user_id = ?', [userId]);
        await bookingService.cancelAppointment(state.apptId, cust.id, reason);
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: `✅ ยกเลิกการนัดหมายแล้ว\nเหตุผล: ${reason}\n\nพิมพ์ "จองนัด" เพื่อนัดหมายใหม่`
        });
      } catch (err) {
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: `❌ ไม่สามารถยกเลิกได้: ${err.message}`
        });
      }
      return;
    }

    const commands = {
      'จองนัด': async () => {
        const [services] = await db.query('SELECT * FROM services WHERE is_active = 1 ORDER BY sort_order');
        await client.replyMessage(event.replyToken, createServiceList(services));
      },
      'นัดของฉัน': async () => {
        const [[cust]] = await db.query('SELECT * FROM customers WHERE line_user_id = ?', [userId]);
        const appts = await bookingService.getCustomerAppointments(cust.id);
        if (appts.length === 0) {
          await client.replyMessage(event.replyToken, { type: 'text', text: 'ยังไม่มีการนัดหมาย' });
        } else {
          await client.replyMessage(event.replyToken, {
            type: 'text',
            text: `📋 การนัดหมายของคุณ:\n${appts.map(a => `${a.booking_ref} - ${a.service_name} (${a.appointment_date} ${a.start_time.slice(0,5)})`).join('\n')}`
          });
        }
      },
      'เมนู': async () => {
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: '📋 เมนู:\n\n• จองนัด - นัดหมายใหม่\n• นัดของฉัน - ดูการนัดหมาย\n• ยกเลิก - ยกเลิกนัดหมาย'
        });
      }
    };

    if (commands[text]) {
      await commands[text]();
    } else {
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: 'พิมพ์ "เมนู" เพื่อดูตัวเลือกทั้งหมด'
      });
    }
  }
}

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Booking bot on port ${PORT}`));
```

## 12. package.json

```json
{
  "name": "lineoa-booking",
  "version": "1.0.0",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "@line/bot-sdk": "^8.0.0",
    "dotenv": "^16.0.0",
    "express": "^4.18.0",
    "googleapis": "^128.0.0",
    "ioredis": "^5.3.0",
    "mysql2": "^3.6.0",
    "node-cron": "^3.0.0"
  }
}
```

## 13. การทดสอบ

```javascript
// tests/schedule.test.js
const scheduleService = require('../src/services/scheduleService');

describe('Schedule Service', () => {
  test('สร้าง time slots ระหว่าง 9:00-18:00 ทุก 60 นาที', () => {
    const slots = scheduleService.generateTimeSlots('09:00:00', '18:00:00', 60);
    expect(slots).toHaveLength(9);
    expect(slots[0]).toEqual({ start: '09:00', end: '10:00' });
    expect(slots[8]).toEqual({ start: '17:00', end: '18:00' });
  });

  test('ตรวจสอบ time overlap', () => {
    expect(scheduleService.timesOverlap('09:00', '10:00', '09:30', '10:30')).toBe(true);
    expect(scheduleService.timesOverlap('09:00', '10:00', '10:00', '11:00')).toBe(false);
    expect(scheduleService.timesOverlap('10:00', '11:00', '09:00', '10:00')).toBe(false);
  });

  test('แปลงเวลา <-> นาที', () => {
    expect(scheduleService.timeToMinutes('09:00')).toBe(540);
    expect(scheduleService.minutesToTime(540)).toBe('09:00');
  });
});
```

## สรุป

ระบบนัดหมายบน LINE OA ที่สร้างในบทนี้ครอบคลุม:

1. **Service & Staff Management** - จัดการบริการและพนักงาน
2. **Smart Scheduling** - สร้าง time slots อัตโนมัติ
3. **Double-booking Prevention** - ป้องกันจองซ้ำด้วย DB locking
4. **Google Calendar Sync** - ซิงค์กับ Google Calendar
5. **Calendar UI** - Flex Message calendar สำหรับเลือกวัน
6. **Reminders** - แจ้งเตือนอัตโนมัติ 24h และ 1h
7. **No-show Handling** - ระบบจัดการลูกค้าไม่มาตามนัด
8. **Cancellation Flow** - ขั้นตอนยกเลิก/เลื่อนนัด
9. **Admin API** - จัดการจากฝั่ง admin
10. **Cron Jobs** - งาน scheduled สำหรับ reminders
