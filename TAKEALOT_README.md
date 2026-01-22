# 📦 Takealot Product & Competitor Tracker - Documentation Package

**Date:** January 22, 2026  
**Status:** Ready for Development  
**Target Server:** 172.0.0.2 (shared with Amazon Dispute Bot)

---

## 📖 READ THESE IN ORDER

### 1️⃣ **START HERE:** `TAKEALOT_LESSONS_LEARNED.md`
**Critical gotchas and solutions from Amazon bot development**

Contains:
- 🚨 PostgreSQL psql hanging issue (MUST READ!)
- 🗄️ Database setup clarification (create separate DB!)
- 📦 PM2 quirks and solutions
- 🎭 Playwright challenges and fixes
- 📊 Data validation patterns
- 🐛 Debugging tips
- ✅ Pre-deployment checklist

**Why first?** Prevents you from hitting the same issues we did.

---

### 2️⃣ **MAIN REFERENCE:** `TAKEALOT_COMPETITOR_TRACKER_PROJECT_BRIEF.md`
**Complete project specifications and infrastructure details**

Contains:
- 🖥️ Server access (IP, SSH, credentials)
- 📊 Database setup (PostgreSQL)
- 🏗️ Architecture overview
- 🌐 Takealot API documentation
- 🎭 Playwright usage strategy
- 📦 PM2 configuration templates
- 🚀 Deployment guide
- 📚 Quick reference

**Why second?** Your complete implementation guide.

---

### 3️⃣ **OPTIONAL:** `TAKEALOT_BRIEF_CHANGELOG.md`
**What was changed from original brief**

Contains:
- Line-by-line changes
- Before/after comparisons
- Rationale for updates

**Why optional?** Context for reviewers, not needed for implementation.

---

## 🎯 Quick Start (TL;DR)

```bash
# 1. Connect to server
ssh ubuntubot@172.0.0.2

# 2. Create project directory
mkdir -p /home/ubuntubot/takealot-tracker
cd /home/ubuntubot/takealot-tracker

# 3. Create separate database (⚠️ NOT amazon_disputes!)
PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -h localhost -c "CREATE DATABASE takealot_tracker;"

# 4. Initialize project
npm init -y
npm install pg express dotenv playwright

# 5. Read the full brief for next steps
# See: TAKEALOT_COMPETITOR_TRACKER_PROJECT_BRIEF.md
```

---

## ⚠️ CRITICAL WARNINGS

### 1. PostgreSQL Commands
**ALWAYS use `PAGER=''` or commands will hang:**
```bash
# ❌ WRONG (will hang)
psql -U amazon_user -d takealot_tracker -c "SELECT * FROM products;"

# ✅ CORRECT
PAGER='' psql -U amazon_user -d takealot_tracker -A -t -c "SELECT * FROM products;"
```

### 2. Database Separation
**Create YOUR OWN database, don't use Amazon's:**
- ❌ `amazon_disputes` - Amazon bot's database (DON'T TOUCH)
- ✅ `takealot_tracker` - Your new database (CREATE THIS)

Same PostgreSQL server, different databases = complete isolation.

### 3. Port Conflicts
**Use port 3001, not 3000:**
- Port 3000: Amazon dashboard (taken)
- Port 3001: Takealot dashboard (use this)

### 4. Scheduling Conflicts
**Run at 4 AM, not 2 AM:**
- 2:00 AM: Amazon bot scrapes
- 4:00 AM: Takealot tracker scrapes (use this)

---

## 📂 File Structure Overview

```
/home/ubuntubot/takealot-tracker/          ← Your project
├── package.json
├── ecosystem.config.js                    ← PM2 config (port 3001, 4 AM cron)
├── .env                                   ← Environment variables
├── src/
│   ├── scraper.js                         ← Main scraper
│   ├── api-client.js                      ← Takealot API wrapper
│   ├── database.js                        ← PostgreSQL operations
│   └── cli.js                             ← CLI interface
├── config/
│   └── database.sql                       ← Schema setup
├── tracker-dashboard/                     ← Next.js UI (optional)
└── logs/                                  ← PM2 logs (auto-managed)

/home/ubuntubot/amazon-disputer/            ← Amazon bot (DON'T MODIFY)
```

---

## 🗄️ Database Details

### PostgreSQL Server (Shared)
```
Host: localhost
Port: 5432
User: amazon_user
Password: amazon_secure_pass_2026
```

### Databases (Separate)
```
amazon_disputes  ← Amazon bot's database (DON'T TOUCH)
takealot_tracker ← Your database (CREATE & USE)
```

### Your Connection String
```
postgresql://amazon_user:amazon_secure_pass_2026@localhost:5432/takealot_tracker
```

---

## 🚀 High-Level Architecture

```
┌─────────────────────────────────────────────┐
│  Takealot Public APIs (Preferred)           │
│  - api.takealot.com/product-details/{PLID} │
│  - api.takealot.com/product-reviews/{PLID} │
└─────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────┐
│  Playwright (API Discovery Only)            │
│  - Find APIs during build phase             │
│  - Fallback if API missing data             │
└─────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────┐
│  Scraper Service (4 AM Daily)               │
│  - Fetch all active products                │
│  - Store daily snapshots                    │
│  - Track price changes                      │
└─────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────┐
│  PostgreSQL (takealot_tracker DB)           │
│  - products table                           │
│  - daily_snapshots table                    │
│  - competitors table                        │
└─────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────┐
│  Next.js Dashboard (Port 3001)              │
│  - View products                            │
│  - Price history charts                     │
│  - Competitor comparison                    │
└─────────────────────────────────────────────┘
```

---

## 🎓 Key Lessons from Amazon Bot Development

### What We Learned the Hard Way:

1. **PostgreSQL psql hangs without `PAGER=''`**
   - Wasted hours debugging this
   - See: `TAKEALOT_LESSONS_LEARNED.md` section 1

2. **Database confusion**
   - Almost broke Amazon bot by using wrong database
   - Now documented clearly: separate databases!

3. **PM2 cron_restart quirks**
   - Doesn't work like regular cron
   - Needs `autorestart: false`

4. **Price/rating parsing**
   - Must parse "R1,234.99" → 1234.99 (number)
   - Must validate 0-5 rating range

5. **Playwright memory leaks**
   - Always close browser in try/finally
   - Use system Chromium, not download

6. **Testing at scale**
   - Start with 2-3 products
   - Then scale to 100+
   - Don't start big!

**All details in:** `TAKEALOT_LESSONS_LEARNED.md`

---

## ✅ Pre-Development Checklist

Before writing any code:

- [ ] Read `TAKEALOT_LESSONS_LEARNED.md` (30 min)
- [ ] Read `TAKEALOT_COMPETITOR_TRACKER_PROJECT_BRIEF.md` (60 min)
- [ ] SSH into server and verify infrastructure
- [ ] Create `takealot_tracker` database
- [ ] Test Takealot APIs with curl
- [ ] Review Amazon bot's `ecosystem.config.js` for reference
- [ ] Understand PM2 log rotation already configured

---

## 🆘 Getting Help

### Common Issues:

**psql hangs:**
→ See: `TAKEALOT_LESSONS_LEARNED.md` - PostgreSQL Quirks

**Wrong database:**
→ See: `TAKEALOT_LESSONS_LEARNED.md` - Database Setup

**PM2 cron not running:**
→ See: `TAKEALOT_LESSONS_LEARNED.md` - PM2 Quirks

**Playwright errors:**
→ See: `TAKEALOT_LESSONS_LEARNED.md` - Playwright Challenges

**Price parsing broken:**
→ See: `TAKEALOT_LESSONS_LEARNED.md` - Data Validation

### Debug Commands:

```bash
# Check database
PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -l

# Check PM2
pm2 status
pm2 logs --lines 100

# Check ports
sudo netstat -tuln | grep ':300'

# Check server resources
free -h
df -h
```

---

## 📦 What's Included

This documentation package contains everything you need:

✅ Complete server infrastructure details  
✅ Database setup and credentials  
✅ Hard-earned lessons and gotchas  
✅ API documentation  
✅ PM2 configuration templates  
✅ Deployment guide  
✅ Debugging tips  
✅ Quick reference guide  

**No ambiguity. No conflicts. Ready to build.**

---

## 🎯 Success Criteria

Your project is complete when:

- [ ] Can add products via CLI
- [ ] Can link competitors
- [ ] Scrapes data from Takealot APIs
- [ ] Stores daily snapshots (one per product per day)
- [ ] Runs automatically at 4 AM daily
- [ ] Dashboard shows products and price history
- [ ] No conflicts with Amazon bot
- [ ] Uses separate database (`takealot_tracker`)
- [ ] Uses port 3001 (not 3000)
- [ ] Handles errors gracefully
- [ ] Logs are manageable (PM2 rotation working)

---

## 🚀 Ready to Start?

1. Read `TAKEALOT_LESSONS_LEARNED.md` (FIRST!)
2. Read `TAKEALOT_COMPETITOR_TRACKER_PROJECT_BRIEF.md` (SECOND!)
3. SSH into server
4. Create database
5. Start coding!

**Estimated Development Time:** 3-5 days for experienced developer

**Questions?** All answers are in the documentation. Use Ctrl+F to search!

---

**Good luck! 🎉**

**You have all our hard-earned knowledge. Avoid our mistakes. Build something great!**
