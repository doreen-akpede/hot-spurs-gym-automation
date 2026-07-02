# 🏋️ Hot Spurs Gym — Membership Management Automation

A fully automated gym membership management system built with **n8n**, **Airtable**, **Fillout**, **Cloudinary**, **Gmail**, and **Telegram**. This project automates the entire member lifecycle — from registration to expiry and renewal — with zero manual intervention required.

---

## 🚀 Project Overview

Hot Spurs Gym is a Lagos-based fitness centre. This automation system handles:

- **New member onboarding** — form submission triggers photo upload, Airtable record creation, and a branded welcome email
- **Renewal reminders** — members are automatically notified 3 days before their membership expires
- **Expiry management** — expired memberships are auto-detected, status updated, and members notified with a renewal link
- **Renewal processing** — when a member renews, their Airtable record is automatically updated with new dates, plan, and amount

---

## 🛠️ Tech Stack

| Tool | Purpose |
|---|---|
| **n8n** | Workflow automation engine (self-hosted on Railway) |
| **PostgreSQL** | Persistent data storage for n8n (Railway) |
| **Airtable** | Member database |
| **Fillout** | Member registration and renewal forms |
| **Cloudinary** | Member profile photo storage |
| **Gmail** | Branded automated emails |
| **Telegram** | Real-time owner notifications |

---

## 📋 Workflows

### Workflow 1 — Member Registration
**Trigger:** Fillout form submission

**Flow:**
```
Fillout Form → Webhook → IF (photo?) → Download Photo → Upload to Cloudinary → Merge → Edit Fields → Create Airtable Record → Wait → Gmail (welcome email) → Telegram (owner alert)
```

**Features:**
- Conditional photo upload via Cloudinary
- Dynamic Member ID generation (e.g. HS-20260630-4821)
- Calendar-based renewal date calculation (Monthly/Quarterly/Annual)
- Dynamic pricing (₦15,000 / ₦40,500 / ₦135,000)
- Branded HTML welcome email with gym contact details

---

### Workflow 2 — 3-Day Renewal Reminder
**Trigger:** Daily schedule (8:00 AM)

**Flow:**
```
Schedule Trigger → Search Airtable → IF (Renewal Date = today + 3 days) → Gmail (reminder email) → Telegram (owner alert)
```

**Features:**
- Runs automatically every morning
- Only processes members expiring in exactly 3 days
- Branded HTML reminder email with renewal CTA

---

### Workflow 3 — Expiry Notification & Renewal Processing
**Trigger:** Daily schedule (8:00 AM) + Fillout renewal form submission

**Chain 1 — Expiry Detection:**
```
Schedule Trigger → Search Airtable → IF (Renewal Date = today) → Update Record (Expired) → Gmail (expiry email + bank details + Renew Now button) → Telegram (owner alert)
```

**Chain 2 — Renewal Processing:**
```
Fillout Renewal Form → Webhook → Search Airtable (by email + Expired status) → Update Record (Active + new dates + new plan + new amount) → Gmail (confirmation email) → Telegram (owner alert)
```

**Features:**
- Auto-flips member status to Expired on due date
- Expiry email includes bank transfer details and Renew Now button
- Renewal form supports plan upgrades/downgrades
- Amount automatically recalculates based on new plan
- Smart search filter prevents duplicate processing

---

## 📧 Email Templates

All emails use a consistent **navy (#1a1a2e) and gold (#c9a84c)** branded design including:
- Member details table
- Dynamic membership information
- Gym contact details and opening hours
- Automated footer

---

## 🗄️ Airtable Schema

**Base:** Hot Spurs Gym  
**Table:** Members

| Field | Type |
|---|---|
| Full Name | Single line text |
| Email | Email |
| Phone | Phone number |
| Membership Type | Single select |
| Member ID | Single line text |
| Start Date | Date |
| Renewal Date | Date |
| Status | Single select (Active/Expired) |
| Payment Status | Single select (Paid/Unpaid) |
| Amount | Currency |
| Photo URL | URL |

---

## 💰 Pricing Structure

| Plan | Monthly Rate | Discount | Total |
|---|---|---|---|
| Monthly | ₦15,000 × 1 | — | ₦15,000 |
| Quarterly | ₦15,000 × 3 | 10% off | ₦40,500 |
| Annual | ₦15,000 × 12 | 25% off | ₦135,000 |

---

## 🔧 Setup Instructions

To import and run these workflows in your own n8n instance:

1. Clone this repository
2. In n8n, go to **Workflows → Import from file**
3. Import each `.json` file
4. Reconnect the following credentials:
   - Airtable Personal Access Token
   - Gmail OAuth2
   - Telegram Bot Token
   - Cloudinary (HTTP Request node)
5. Update your Fillout webhook URLs to point to your n8n instance
6. Activate all three workflows

---

## 🐛 Key Debugging Insights

This project involved resolving several non-obvious bugs during development:

- **Fillout Advanced mode** sending empty payloads (fix: revert to Simple mode)
- **Test vs Production webhook URL** causing silent failures in live submissions
- **Merge node "Choose Branch"** not auto-detecting active input (fix: switch to Append mode)
- **Cloudinary node reference** breaking no-photo path (fix: `.isExecuted` conditional fallback)

See the full debugging case study → *(link to LinkedIn post)*

---

## 👩🏽‍💻 Built By

**Doreen Akpede**  
Built as part of a structured professional AI Automation training programme,
focusing on no-code workflow automation, AI integration, and real-world
client project delivery. 
[LinkedIn](https://www.linkedin.com/in/onohinosen-akpede) | [GitHub](https://github.com/doreendasweet-dot)

---

## 📌 Tools & Integrations

![n8n](https://img.shields.io/badge/n8n-EA4B71?style=flat&logo=n8n&logoColor=white)
![Airtable](https://img.shields.io/badge/Airtable-18BFFF?style=flat&logo=airtable&logoColor=white)
![Gmail](https://img.shields.io/badge/Gmail-D14836?style=flat&logo=gmail&logoColor=white)
![Telegram](https://img.shields.io/badge/Telegram-2CA5E0?style=flat&logo=telegram&logoColor=white)
![Railway](https://img.shields.io/badge/Railway-0B0D0E?style=flat&logo=railway&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=flat&logo=postgresql&logoColor=white)
