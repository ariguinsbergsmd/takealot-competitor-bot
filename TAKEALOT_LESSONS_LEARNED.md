# 🎓 LESSONS LEARNED & GOTCHAS
## Critical Knowledge Transfer for Takealot Tracker Agent

**Date:** January 22, 2026  
**From:** Amazon Dispute Bot Development Team  
**To:** New Takealot Tracker Agent

---

## 🚨 CRITICAL: PostgreSQL Quirks

### Issue: psql Commands Hang Indefinitely

**Problem:**
When running `psql` commands programmatically (from scripts, Node.js, terminal automation), they hang forever waiting for keyboard input.

**Why:**
- PostgreSQL's `psql` client uses a **pager** (`less`, `more`) for large output
- The pager waits for **interactive input** (spacebar, 'q', arrows)
- Even with `-c` flag, paging is NOT disabled
- Even redirecting stderr (`2>&1`) doesn't help

**❌ WRONG (Will Hang):**
```bash
psql -U amazon_user -d takealot_tracker -c "SELECT * FROM products;" 2>&1
psql -U amazon_user -d takealot_tracker -c "SELECT * FROM products;" | cat
```

**✅ CORRECT (Will Work):**
```bash
# Method 1: Environment variable (PREFERRED)
PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -d takealot_tracker -A -t -c "SELECT * FROM products;"

# Method 2: Built-in flag
PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -d takealot_tracker --pset pager=off -A -t -c "SELECT * FROM products;"
```

**Required Flags:**
- `PAGER=''` or `--pset pager=off` - Disables pager
- `-A` - Unaligned output (no padding)
- `-t` - Tuples only (no headers)
- `-x` - Expanded output (for wide tables)

**Document Reference:** See `PSQL-USAGE-RULES.md` in Amazon project for full details.

---

## 🗄️ DATABASE: Create Separate, Don't Share

### ⚠️ CRITICAL CLARIFICATION

**DO NOT use `amazon_disputes` database!**

The connection string in the brief contains credentials for the **PostgreSQL server**, not a shared database.

**What to Do:**

1. **Create your OWN database:**
```bash
# Connect as postgres superuser or amazon_user
PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -h localhost -c "CREATE DATABASE takealot_tracker;"
```

2. **Use these credentials for YOUR database:**
```javascript
// Your connection string
DATABASE_URL=postgresql://amazon_user:amazon_secure_pass_2026@localhost:5432/takealot_tracker
//                         ^^^^^^^^^^^                                            ^^^^^^^^^^^^^^^^
//                         Same user                                              DIFFERENT database!
```

3. **Verify separation:**
```bash
# List all databases
PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -l

# You should see:
# - amazon_disputes  (Amazon bot's database - DON'T TOUCH)
# - takealot_tracker (Your new database - USE THIS)
```

**Why Same Credentials?**
- PostgreSQL has ONE server with multiple databases
- Same user (`amazon_user`) can access multiple databases
- Each database is completely isolated
- Like different folders on the same hard drive

**What If I Accidentally Connect to Wrong Database?**
```bash
# Check which database you're in:
PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -d takealot_tracker -c "SELECT current_database();"
# Should output: takealot_tracker

# If it says amazon_disputes, you're in the wrong place!
# Exit and reconnect to takealot_tracker
```

---

## 🌐 SSH & Network Gotchas

### Issue: SSH Tunnel Dies Silently

**Problem:**
SSH tunnels (for accessing dashboard from Mac) can die without warning, making it seem like the server is down.

**Check if tunnel is alive:**
```bash
ps aux | grep "ssh -L 3001" | grep -v grep
```

**Restart tunnel:**
```bash
# Kill old tunnel
pkill -f "ssh -L 3001"

# Start new tunnel (background)
ssh -L 3001:localhost:3001 ubuntubot@172.0.0.2 -N &
```

**Better: Use autossh (auto-reconnect):**
```bash
# Install on Mac
brew install autossh

# Start persistent tunnel
autossh -M 0 -L 3001:localhost:3001 ubuntubot@172.0.0.2 -N
```

### Issue: Port Already in Use

**Problem:**
PM2 or another process already using port 3001.

**Check what's using a port:**
```bash
# On server
sudo netstat -tuln | grep :3001
# or
sudo lsof -i :3001
```

**Kill process on port:**
```bash
# Find PID
sudo lsof -ti :3001

# Kill it
sudo kill -9 $(sudo lsof -ti :3001)
```

---

## 📦 PM2 Quirks & Best Practices

### Issue: PM2 Doesn't Pick Up .env File

**Problem:**
Creating `.env` file but PM2 doesn't load it.

**Why:**
PM2 doesn't automatically load `.env` files. You must use `env` property in `ecosystem.config.js`.

**✅ CORRECT:**
```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'takealot-scraper',
    script: './src/scraper.js',
    env: {
      DATABASE_URL: 'postgresql://amazon_user:amazon_secure_pass_2026@localhost:5432/takealot_tracker',
      NODE_ENV: 'production',
      PORT: 3001,
    }
  }]
};
```

**Alternative (load .env in code):**
```javascript
// At top of your scraper.js
require('dotenv').config();
```

### Issue: PM2 Cron Not Running

**Problem:**
Set `cron_restart: '0 4 * * *'` but scraper never runs.

**Why:**
`cron_restart` **RESTARTS** the process, not starts it if it's not running.

**✅ CORRECT Setup:**
```javascript
{
  name: 'takealot-scraper',
  script: './src/scraper.js',
  cron_restart: '0 4 * * *',
  autorestart: false,  // Don't restart on exit
  watch: false,
}
```

**How it works:**
1. PM2 starts the process initially
2. Script runs and exits (process stops)
3. At 4 AM, PM2 restarts it (starts the cron)
4. Script runs again and exits
5. Repeat daily

**Verify cron:**
```bash
pm2 list  # Check if process is registered
pm2 logs takealot-scraper --lines 100  # Check for execution
```

### Issue: PM2 Save Doesn't Persist

**Problem:**
Run `pm2 save` but after reboot, processes are gone.

**Solution:**
```bash
# After starting processes
pm2 save

# Enable startup script
pm2 startup
# This gives you a command to run with sudo
# Copy and run that command

# Example output:
# sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u ubuntubot --hp /home/ubuntubot

# Run the provided command, then:
pm2 save
```

---

## 🔄 Next.js Build Issues

### Issue: Build Fails with "Module Not Found"

**Problem:**
Dashboard builds locally but fails on server.

**Common causes:**
1. Missing dependencies in `package.json`
2. Case-sensitive imports (Linux is case-sensitive)
3. Missing `.next` directory

**Fix:**
```bash
cd tracker-dashboard

# Clean install
rm -rf node_modules .next package-lock.json
npm install

# Build with verbose output
npm run build 2>&1 | tee build.log

# Check for errors
cat build.log | grep -i error
```

### Issue: Next.js Static Export vs Server Mode

**Problem:**
Need to decide: static export or server-side rendering?

**Static Export (Simpler):**
```javascript
// next.config.js
module.exports = {
  output: 'export',
  distDir: 'out',
};
```
```bash
npm run build
# Serve with any static server
npx serve out -l 3001
```

**Server Mode (More Features):**
```javascript
// next.config.js (default)
module.exports = {};
```
```bash
npm run build
npm start  # Runs on PORT from env
```

**Recommendation for Takealot:** Use **server mode** for dynamic data.

---

## 🎭 Playwright Challenges

### Issue: Chromium Not Found

**Problem:**
Playwright can't find browser executable.

**Fix:**
```javascript
// Use system Chromium instead of downloading
const { chromium } = require('playwright');
const browser = await chromium.launch({
  executablePath: '/usr/bin/chromium-browser',  // System path
  headless: true,
});
```

**Find system Chromium:**
```bash
which chromium-browser
which chromium
which google-chrome
```

### Issue: Browser Launch Times Out

**Problem:**
`browser.launch()` hangs or times out.

**Causes:**
1. Missing dependencies (fonts, libs)
2. Insufficient memory
3. Too many concurrent browsers

**Fix:**
```bash
# Install missing dependencies
sudo apt-get install -y \
  libx11-xcb1 libxcomposite1 libxdamage1 \
  libxi6 libxtst6 libnss3 libcups2 \
  libxss1 libxrandr2 libasound2 \
  libpangocairo-1.0-0 libatk1.0-0 \
  libatk-bridge2.0-0 libgtk-3-0
```

**In code (increase timeout):**
```javascript
const browser = await chromium.launch({
  timeout: 60000,  // 60 seconds
  headless: true,
});
```

### Issue: DevTools Protocol Error

**Problem:**
Random "Target closed" or "Protocol error" messages.

**Solution:**
Always close pages and browser properly:
```javascript
try {
  const page = await browser.newPage();
  // ... do stuff ...
} catch (error) {
  console.error('Error:', error);
} finally {
  if (page) await page.close();
  if (browser) await browser.close();
}
```

---

## 🔐 Environment Variable Security

### Issue: Credentials in Git

**Problem:**
Accidentally commit passwords to git.

**Prevention:**
```bash
# Create .gitignore FIRST
cat > .gitignore << 'EOF'
.env
.env.local
.env.production
ecosystem.config.js  # If it has passwords
node_modules/
*.log
.DS_Store
EOF

# Verify before first commit
git status
```

**If already committed:**
```bash
# Remove from git but keep local file
git rm --cached .env
git rm --cached ecosystem.config.js

# Create example template
cp .env .env.example
# Edit .env.example to remove actual passwords
# Replace with placeholders like PASSWORD_HERE

git add .env.example .gitignore
git commit -m "Add env example, remove credentials"
```

---

## 📊 Data Validation Lessons

### Issue: Prices Stored as Strings

**Problem:**
Scraping prices as strings ("R1,234.99") breaks comparisons.

**Solution:**
Always parse and store as numbers:
```javascript
// Parse Takealot price format
function parsePrice(priceStr) {
  if (!priceStr) return null;
  
  // Remove "R", spaces, commas
  const cleaned = priceStr.replace(/[R\s,]/g, '');
  
  // Parse to float
  const price = parseFloat(cleaned);
  
  // Validate
  return isNaN(price) || price < 0 ? null : price;
}

// Usage
const regularPrice = parsePrice("R1,234.99");  // Returns: 1234.99
```

**Store in database:**
```sql
regular_price NUMERIC(10,2)  -- Not VARCHAR!
```

### Issue: Rating Format Inconsistencies

**Problem:**
Ratings come as "4.5 out of 5" or "4.5" or 4.5.

**Solution:**
```javascript
function parseRating(ratingStr) {
  if (!ratingStr) return null;
  
  // Extract number
  const match = String(ratingStr).match(/[\d.]+/);
  if (!match) return null;
  
  const rating = parseFloat(match[0]);
  
  // Validate range
  return rating >= 0 && rating <= 5 ? rating : null;
}
```

### Issue: PLID Validation

**Problem:**
Invalid PLIDs cause scraper to fail.

**Solution:**
```javascript
function isValidPLID(plid) {
  // Takealot PLIDs: "PLID" followed by 8 digits
  return /^PLID\d{8}$/.test(plid);
}

// Extract PLID from URL
function extractPLID(url) {
  const match = url.match(/PLID(\d{8})/);
  return match ? `PLID${match[1]}` : null;
}

// Usage
const url = "https://www.takealot.com/product/PLID12345678";
const plid = extractPLID(url);  // "PLID12345678"

if (!isValidPLID(plid)) {
  throw new Error(`Invalid PLID: ${plid}`);
}
```

---

## 🐛 Debugging Tips

### Check Database Connection

```bash
# Quick connection test
PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -d takealot_tracker -c "SELECT version();"
```

### Monitor PM2 Logs in Real-Time

```bash
# All processes
pm2 logs

# Specific process
pm2 logs takealot-scraper --lines 100

# Filter for errors
pm2 logs takealot-scraper --err

# Save logs to file
pm2 logs takealot-scraper --lines 500 > debug.log
```

### Check Memory Usage

```bash
# PM2 memory usage
pm2 status

# System memory
free -h

# If out of memory, restart process
pm2 restart takealot-scraper
```

### Database Query Performance

```sql
-- Check slow queries (if enabled)
SELECT * FROM pg_stat_statements 
ORDER BY total_exec_time DESC 
LIMIT 10;

-- Check table sizes
SELECT 
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables 
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Check indexes
\d products  -- Shows indexes on products table
```

---

## 📈 Performance Lessons

### Issue: Scraping Too Slow

**Problem:**
Scraping 100 products takes 30+ minutes.

**Optimization:**
```javascript
// Bad: Sequential scraping
for (const product of products) {
  await scrapeProduct(product);  // Slow!
}

// Better: Batch with concurrency limit
const pLimit = require('p-limit');
const limit = pLimit(5);  // Max 5 concurrent

const promises = products.map(product => 
  limit(() => scrapeProduct(product))
);

await Promise.all(promises);
```

**Rate limiting:**
```javascript
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Add delay between requests
for (const product of products) {
  await scrapeProduct(product);
  await sleep(1000);  // 1 second delay
}
```

### Issue: Database Connection Pool Exhausted

**Problem:**
"Too many clients" error from PostgreSQL.

**Solution:**
```javascript
// Configure pool properly
const { Pool } = require('pg');
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,  // Maximum connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Always return connections
async function queryDatabase() {
  const client = await pool.connect();
  try {
    const result = await client.query('SELECT * FROM products');
    return result.rows;
  } finally {
    client.release();  // CRITICAL!
  }
}
```

---

## 🎯 Testing Recommendations

### Test with Real Data First

**Don't start with 100 products.**

1. Add 2-3 real Takealot products
2. Run manual scrape
3. Verify data in database
4. Check price formatting
5. Check rating parsing
6. Then scale up

### Test Failure Scenarios

```javascript
// Test with invalid PLID
await addProduct('PLID99999999');  // Should fail gracefully

// Test with deleted product
await addProduct('PLID00000001');  // Should handle 404

// Test with out-of-stock product
await addProduct('PLIDXXXXXXXX');  // Should store in_stock: false
```

### Verify One Snapshot Per Day

```sql
-- Should return 1 per product per day
SELECT product_id, snapshot_date, COUNT(*) 
FROM daily_snapshots 
GROUP BY product_id, snapshot_date 
HAVING COUNT(*) > 1;

-- Should be empty!
```

---

## 🚀 Deployment Checklist

Before going live:

- [ ] Test psql commands with `PAGER=''`
- [ ] Create `takealot_tracker` database (separate from Amazon)
- [ ] Verify connection pooling configured
- [ ] Test Takealot API endpoints with curl
- [ ] Add real products (3-5 for testing)
- [ ] Run manual scrape and verify data
- [ ] Check price parsing (R1,234.99 → 1234.99)
- [ ] Check rating parsing (4.5 out of 5 → 4.5)
- [ ] Verify PLID validation
- [ ] Test PM2 configuration
- [ ] Verify port 3001 is available
- [ ] Test SSH tunnel from Mac
- [ ] Configure cron for 4 AM (not 2 AM!)
- [ ] Test error handling (invalid PLID, network failure)
- [ ] Check logs are being written
- [ ] Verify PM2 log rotation is working
- [ ] Test dashboard loads on port 3001
- [ ] Run `pm2 save` after setup
- [ ] Configure `pm2 startup`

---

## 📞 When Things Go Wrong

**System seems frozen:**
```bash
# Check if PM2 is responsive
pm2 status

# If not, restart PM2
pm2 kill
pm2 start ecosystem.config.js
```

**Database won't connect:**
```bash
# Check PostgreSQL is running
sudo systemctl status postgresql

# Restart if needed
sudo systemctl restart postgresql
```

**Out of disk space:**
```bash
# Check disk usage
df -h

# Find large files
du -sh /home/ubuntubot/* | sort -h

# Clean PM2 logs if huge
pm2 flush
```

**Process won't start:**
```bash
# Check for port conflicts
sudo lsof -i :3001

# Check for syntax errors
node src/scraper.js  # Run directly to see errors
```

---

## 💡 Final Tips

1. **Always use `PAGER=''` with psql** - Cannot stress this enough
2. **Create separate database** - Don't touch `amazon_disputes`
3. **Test with 2-3 products first** - Don't start with 100
4. **Validate data before storing** - Parse prices, ratings, PLIDs
5. **Close browser connections** - Always in try/finally
6. **Use connection pooling** - Don't create new connections each query
7. **Schedule at 4 AM** - Avoid 2 AM (Amazon bot's time)
8. **Monitor logs daily** - `pm2 logs` is your friend
9. **Use SSH tunnel for dashboard** - Don't expose to internet
10. **Document everything** - Future you will thank you

---

**Good luck! You've got all our hard-earned lessons. 🚀**

**Questions?** Check these documents in the Amazon project:
- `PSQL-USAGE-RULES.md` - PostgreSQL gotchas
- `QUESTIONS-ANSWERED.md` - Common issues
- `ecosystem.config.js` - PM2 configuration example
