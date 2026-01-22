# 🗄️ DATABASE SETUP INSTRUCTIONS

**Status:** ⚠️ **MANUAL SETUP REQUIRED**  
**Reason:** Server requires sudo password for PostgreSQL operations

---

## 🚨 ISSUE ENCOUNTERED

The automated setup hit a blocker: The server requires an interactive sudo password, which cannot be provided through SSH automation.

**Error:**
```
sudo: a terminal is required to read the password
```

---

## ✅ SOLUTION: Manual Setup

You need to run these commands manually on the server with sudo access.

---

## 📝 STEP-BY-STEP INSTRUCTIONS

### **Step 1: SSH into Server**

```bash
ssh ubuntubot@172.0.0.2
# Password: Trash081!
```

### **Step 2: Grant Schema Permissions**

```bash
sudo -u postgres psql -d takealot_tracker << 'SQL'
GRANT ALL ON SCHEMA public TO amazon_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO amazon_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO amazon_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO amazon_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO amazon_user;
SQL
```

### **Step 3: Create Database Tables**

```bash
cd /home/ubuntubot/takealot-tracker
PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -h localhost -d takealot_tracker -f lib/database/schema.sql
```

### **Step 4: Verify Tables Created**

```bash
PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -h localhost -d takealot_tracker -c "\dt"
```

**Expected Output:**
```
                  List of relations
 Schema |         Name          | Type  |    Owner    
--------+-----------------------+-------+-------------
 public | alerts                | table | amazon_user
 public | competitors           | table | amazon_user
 public | price_history         | table | amazon_user
 public | product_variations    | table | amazon_user
 public | products              | table | amazon_user
 public | scrape_logs           | table | amazon_user
```

---

## 🎯 ALTERNATIVE: One-Command Setup

If you have sudo access, run this single command:

```bash
ssh ubuntubot@172.0.0.2 << 'EOF'
cd /home/ubuntubot/takealot-tracker

# Grant permissions
sudo -u postgres psql -d takealot_tracker -c "GRANT ALL ON SCHEMA public TO amazon_user;"
sudo -u postgres psql -d takealot_tracker -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO amazon_user;"
sudo -u postgres psql -d takealot_tracker -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO amazon_user;"

# Create tables
PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -h localhost -d takealot_tracker -f lib/database/schema.sql

# Verify
PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -h localhost -d takealot_tracker -c "\dt"
EOF
```

---

## 📊 DATABASE SCHEMA OVERVIEW

The schema includes:

### **Tables:**
1. **products** - Main product information
2. **product_variations** - Product variations (color, size, etc.)
3. **price_history** - Historical price data
4. **competitors** - Competitor products and pricing
5. **alerts** - Price change alerts
6. **scrape_logs** - Scraping activity logs

### **Indexes:**
- Optimized queries for product lookups
- Fast price history retrieval
- Efficient alert checking

### **Views:**
- `recent_price_changes` - Last 7 days of price changes
- `products_with_alerts` - Products with pending alerts

### **Triggers:**
- Auto-update `updated_at` timestamp on products

---

## ✅ VERIFICATION CHECKLIST

After running the setup:

- [ ] Tables created (6 tables)
- [ ] Indexes created (7 indexes)
- [ ] Views created (2 views)
- [ ] Triggers created (1 trigger)
- [ ] `amazon_user` can query tables
- [ ] `amazon_user` can insert data

**Test Query:**
```bash
PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -h localhost -d takealot_tracker -c "
SELECT table_name FROM information_schema.tables WHERE table_schema = 'public';
"
```

---

## 🚀 ONCE TABLES ARE CREATED

### **Start Development Server:**
```bash
cd /home/ubuntubot/takealot-tracker
npm run dev
```

### **Access Dashboard:**
```
http://172.0.0.2:3001
```

### **Test Scraper:**
1. Open dashboard
2. Enter product ID: `PLID97218102`
3. Click "Scrape Now"
4. View results!

---

## 🐛 TROUBLESHOOTING

### **Error: "permission denied for schema public"**
**Solution:** Run Step 2 (Grant Schema Permissions)

### **Error: "relation does not exist"**
**Solution:** Run Step 3 (Create Database Tables)

### **Error: "FATAL: password authentication failed"**
**Solution:** Check `.env` file has correct password: `amazon_secure_pass_2026`

### **Connection refused**
**Solution:** Ensure PostgreSQL is running:
```bash
sudo systemctl status postgresql
```

---

## 📞 NEED HELP?

**Schema File Location:**
```
/home/ubuntubot/takealot-tracker/lib/database/schema.sql
```

**View Schema Contents:**
```bash
cat /home/ubuntubot/takealot-tracker/lib/database/schema.sql
```

**Database Connection String:**
```
postgresql://amazon_user:amazon_secure_pass_2026@localhost:5432/takealot_tracker
```

---

## ⚡ QUICK TEST

Once tables are created, test with this command:

```bash
cd /home/ubuntubot/takealot-tracker
npx ts-node lib/scrapers/test-scraper.ts
```

This will:
1. ✅ Launch Playwright browser
2. ✅ Scrape two example products
3. ✅ Save to database
4. ✅ Display results

---

**Status:** ⏳ **WAITING FOR MANUAL SETUP**  
**Next Step:** Run the commands above on the server  
**Then:** Everything will be fully operational! 🚀
