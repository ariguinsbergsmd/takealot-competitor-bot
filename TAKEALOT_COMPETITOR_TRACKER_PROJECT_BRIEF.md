# 🎯 PROJECT BRIEF: Takealot Product & Competitor Tracker

**Date:** January 22, 2026  
**Platform:** Ubuntu Desktop Environment (Shared Server)  
**Purpose:** Daily product tracking and competitor price monitoring system

---

## 🖥️ SERVER INFRASTRUCTURE (EXISTING - DO NOT REINSTALL)

### **Server Access Details**

| Property | Value |
|----------|-------|
| **IP Address** | `172.0.0.2` |
| **Hostname** | `ari-ubuntu` |
| **SSH Username** | `ubuntubot` |
| **SSH Password** | Contact system admin (passwordless SSH key preferred) |
| **OS** | Ubuntu Desktop Environment |
| **Architecture** | x86_64 |

**SSH Connection:**
```bash
ssh ubuntubot@172.0.0.2
# or if SSH config is set up:
ssh ari-ubuntu
```

### **Already Installed Services** ✅

| Service | Version | Port | Purpose | Status |
|---------|---------|------|---------|--------|
| **PostgreSQL** | 14+ | 5432 | Primary database | Running |
| **Redis** | Latest | 6379 | Queue management (BullMQ) | Running |
| **Node.js** | 18+ | - | Runtime environment | Installed |
| **PM2** | Latest | - | Process manager | Configured |
| **Chromium** | Latest | - | Headless browser | Installed |
| **Playwright** | Latest | - | Browser automation | Installed |

**⚠️ DO NOT install duplicate services** - Use what's already there.

### **Port Allocation**

| Port | Service | Owner | Notes |
|------|---------|-------|-------|
| **3000** | Next.js Dashboard | Amazon Dispute Bot | In use |
| **3001** | **AVAILABLE** | Takealot Tracker | Use this for your dashboard |
| **5432** | PostgreSQL | System | Shared database server |
| **6379** | Redis | System | Shared Redis server |

**Choose port 3001 for your web dashboard to avoid conflicts.**

### **Existing PostgreSQL Database**

**Connection Details:**
```javascript
{
  user: 'amazon_user',
  password: 'amazon_secure_pass_2026',
  host: 'localhost',
  port: 5432,
  database: 'amazon_disputes' // Amazon project database - DO NOT USE
}
```

**⚠️ Create a separate database for Takealot Tracker:**
```sql
-- Connect as postgres user first
sudo -u postgres psql

-- Create new database
CREATE DATABASE takealot_tracker;

-- Grant permissions to existing user
GRANT ALL PRIVILEGES ON DATABASE takealot_tracker TO amazon_user;
```

**⚠️ CRITICAL: Create YOUR OWN Database**

**DO NOT use `amazon_disputes` database!** The credentials above are for the PostgreSQL **server**, not a shared database.

**Your connection string (DIFFERENT database):**
```
postgresql://amazon_user:amazon_secure_pass_2026@localhost:5432/takealot_tracker
//         ^^^^^^^^^^^^ Same user/password        ^^^^^^^^^^^^^^^^ YOUR database (NEW)
```

**Create your database:**
```bash
# Connect and create
PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -h localhost -c "CREATE DATABASE takealot_tracker;"

# Verify it was created
PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -l
# Should show both:
#   - amazon_disputes  (Amazon bot - DON'T TOUCH)
#   - takealot_tracker (Your database - USE THIS)
```

**Why same credentials?**
- PostgreSQL = ONE server, MULTIPLE databases
- Same user can access different databases
- Each database is completely isolated
- Like different folders on same computer

**See:** `TAKEALOT_LESSONS_LEARNED.md` for complete database setup guide.

### **Existing Directory Structure**

```
/home/ubuntubot/
├── amazon-disputer/              ← Amazon Dispute Bot (DO NOT MODIFY)
│   ├── worker.js
│   ├── ecosystem.config.js
│   ├── dispute-dashboard/
│   ├── scrapers/
│   ├── browser-data/             ← Shared browser sessions
│   ├── logs/                     ← PM2 logs
│   └── node_modules/
│
└── takealot-tracker/             ← YOUR PROJECT GOES HERE
    ├── src/
    ├── config/
    ├── data/
    ├── logs/
    └── tracker-dashboard/
```

**⚠️ Install your project at:** `/home/ubuntubot/takealot-tracker/`

### **Shared Resources You Can Use**

#### 1. **Browser Sessions** (for Playwright)
- Location: `/home/ubuntubot/amazon-disputer/browser-data/`
- Contains authenticated browser profiles
- Read-only access recommended
- Create your own at `/home/ubuntubot/takealot-tracker/browser-data/` if needed

#### 2. **PM2 Process Manager**
Already configured with log rotation (`pm2-logrotate`):
- Max log size: 10 MB
- Retention: 30 days
- Compression: gzip enabled

**Register your services with PM2:**
```bash
pm2 start ecosystem.config.js
pm2 save
pm2 startup  # Enable auto-start on reboot
```

#### 3. **Redis for Job Queues**
If you need background job processing (optional):
```javascript
const Redis = require('ioredis');
const redis = new Redis({
  host: 'localhost',
  port: 6379,
  // No password required on local server
});
```

---

## 📋 PROJECT OVERVIEW

Build a service that runs on Ubuntu Desktop to track Takealot products and their competitors. The system will scrape product data daily and store historical data for price tracking, review monitoring, and competitive analysis.

**Coexistence with Amazon Dispute Bot:**
- Both projects run on the same server
- Separate databases, separate ports
- Shared infrastructure (PostgreSQL, Redis, Chromium)
- No interference or conflicts

---

## 🎯 CORE REQUIREMENTS

### 1. Product Management
- ✅ Add Takealot products to track (by URL or PLID)
- ✅ Link competitor products to each tracked product
- ✅ Remove products from tracking
- ✅ View list of all tracked products

### 2. Data Collection (Daily Scraping)

For each tracked product, collect:
- **Product Name** (title)
- **Star Rating** (average rating, e.g., 4.2/5)
- **Number of Reviews** (total review count)
- **Regular Price** (original/strikethrough price)
- **Selling Price** (current/discounted price)
- **Availability** (in stock / out of stock)
- **Timestamp** (when scraped)

### 3. Competitor Tracking
- Each product can have multiple competitors
- Competitors are also Takealot products
- Same data collected for competitors
- Store relationship between main product and competitors

### 4. Historical Data Storage
- Store data daily (one snapshot per day)
- Enable time-series analysis
- Track price changes over time
- Track review count growth
- Detect pricing patterns

---

## 🔧 TECHNICAL REQUIREMENTS

### Platform
- **OS:** Ubuntu Desktop Environment (172.0.0.2)
- **Runtime:** Node.js 18+ ✅ Already installed
- **Database:** PostgreSQL ✅ Use existing server (new database: `takealot_tracker`)
- **Process Manager:** PM2 ✅ Already configured
- **Scheduler:** PM2 cron_restart or systemd timer

### Database Choice

**PostgreSQL (RECOMMENDED)** ✅
- Already installed and configured on server
- Better performance for time-series data
- Supports JSON columns for flexible data
- Connection pooling built-in
- Concurrent access for dashboard + scraper

**Connection String:**
```
postgresql://amazon_user:amazon_secure_pass_2026@localhost:5432/takealot_tracker
```

**SQLite (Not Recommended)**
- File locking issues with concurrent access
- No network access for remote monitoring
- Harder to backup/replicate
- Use PostgreSQL instead

### Architecture
```
┌─────────────────────────────────────────────┐
│         User Interface (CLI/Web)            │
│  - Add/Remove Products                      │
│  - Link Competitors                         │
│  - View Reports                             │
└─────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────┐
│         Scraper Service                     │
│  - Fetch product data from Takealot        │
│  - Use public APIs (preferred)             │
│  - Use Playwright MCP (fallback)           │
└─────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────┐
│         Data Storage Layer                  │
│  - Products table                           │
│  - Competitors table                        │
│  - DailySnapshots table                     │
│  - Historical price/review data             │
└─────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────┐
│         Scheduler (cron/systemd)            │
│  - Runs daily at specified time             │
│  - Scrapes all tracked products             │
│  - Stores snapshots                         │
└─────────────────────────────────────────────┘
```

---

## 📊 DATABASE SCHEMA (PostgreSQL)

**Create the database first:**
```sql
-- Run as postgres user or with admin rights
CREATE DATABASE takealot_tracker;
GRANT ALL PRIVILEGES ON DATABASE takealot_tracker TO amazon_user;
```

### Table: `products`
```sql
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  plid VARCHAR(20) UNIQUE NOT NULL,        -- Takealot PLID
  name TEXT NOT NULL,                       -- Product name
  url TEXT NOT NULL,                        -- Full Takealot URL
  is_competitor BOOLEAN DEFAULT FALSE,      -- Is this a competitor product?
  parent_product_id INTEGER,                -- Links to main product
  added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_scraped_at TIMESTAMP,
  active BOOLEAN DEFAULT TRUE,              -- Still tracking?
  FOREIGN KEY (parent_product_id) REFERENCES products(id) ON DELETE CASCADE
);

CREATE INDEX idx_products_plid ON products(plid);
CREATE INDEX idx_products_active ON products(active);
CREATE INDEX idx_products_parent ON products(parent_product_id);
```

### Table: `daily_snapshots`
```sql
CREATE TABLE daily_snapshots (
  id SERIAL PRIMARY KEY,
  product_id INTEGER NOT NULL,
  snapshot_date DATE NOT NULL,              -- YYYY-MM-DD
  product_name TEXT,                        -- Name at time of snapshot
  star_rating NUMERIC(3,2),                 -- e.g., 4.25
  review_count INTEGER,                     -- Total reviews
  regular_price NUMERIC(10,2),              -- Original price
  selling_price NUMERIC(10,2),              -- Current/sale price
  discount_percentage NUMERIC(5,2),         -- Calculated discount
  in_stock BOOLEAN,                         -- Availability
  scraped_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  raw_data JSONB,                           -- Store full API response for debugging
  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
  UNIQUE(product_id, snapshot_date)         -- One snapshot per product per day
);

CREATE INDEX idx_snapshots_product ON daily_snapshots(product_id);
CREATE INDEX idx_snapshots_date ON daily_snapshots(snapshot_date);
CREATE INDEX idx_snapshots_product_date ON daily_snapshots(product_id, snapshot_date);
```

### Table: `competitors`
```sql
CREATE TABLE competitors (
  id SERIAL PRIMARY KEY,
  main_product_id INTEGER NOT NULL,
  competitor_product_id INTEGER NOT NULL,
  added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (main_product_id) REFERENCES products(id) ON DELETE CASCADE,
  FOREIGN KEY (competitor_product_id) REFERENCES products(id) ON DELETE CASCADE,
  UNIQUE(main_product_id, competitor_product_id)
);

CREATE INDEX idx_competitors_main ON competitors(main_product_id);
CREATE INDEX idx_competitors_comp ON competitors(competitor_product_id);
```

### Table: `scrape_logs` (Optional but Recommended)
```sql
CREATE TABLE scrape_logs (
  id SERIAL PRIMARY KEY,
  run_id UUID DEFAULT gen_random_uuid(),    -- Group snapshots from same scrape run
  product_id INTEGER,
  status VARCHAR(20),                       -- 'success', 'failed', 'skipped'
  error_message TEXT,
  execution_time_ms INTEGER,                -- Performance tracking
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE SET NULL
);

CREATE INDEX idx_scrape_logs_run ON scrape_logs(run_id);
CREATE INDEX idx_scrape_logs_status ON scrape_logs(status);
CREATE INDEX idx_scrape_logs_created ON scrape_logs(created_at);
```

---

## 🌐 TAKEALOT PUBLIC API REFERENCE

### ⚠️ CRITICAL: Use Public APIs First

Before using Playwright MCP to scrape pages, **ALWAYS try the public APIs first**. Takealot has documented public APIs that are faster, more reliable, and less fragile than web scraping.

### Base URL
```
https://api.takealot.com
```

### Available Endpoints

#### 1. **Product Details API**
```
GET https://api.takealot.com/rest/v-1-10-0/product-details/{PLID}
```

**Response includes:**
- Product name (`title`)
- Brand
- Price info (`selling_price`, `list_price`)
- Availability (`available_for_sale`)
- Images
- Description
- Specifications
- Category info

**Example:**
```bash
curl "https://api.takealot.com/rest/v-1-10-0/product-details/PLID12345678"
```

#### 2. **Product Reviews API**
```
GET https://api.takealot.com/rest/v-1-10-0/product-reviews/{PLID}
```

**Parameters:**
- `sort=MostRecent` or `MostHelpful`
- `start=0` (pagination offset)
- `rows=50` (page size, max 50)
- `filter=1` (1-star), `2`, `3`, `4`, `5`, or empty for all

**Response includes:**
- Total review count (`results_total`)
- Average rating (`average_rating`)
- Individual reviews array
- Rating distribution

**Example:**
```bash
curl "https://api.takealot.com/rest/v-1-10-0/product-reviews/PLID12345678?sort=MostRecent&start=0&rows=50"
```

#### 3. **Product Search API**
```
GET https://api.takealot.com/rest/v-1-10-0/productlines/search
```

**Parameters:**
- `qsearch={query}` (search term)
- `rows=50` (results per page)
- `start=0` (offset)
- `sort=BestSeller`, `PriceAsc`, `PriceDesc`, `NewArrivals`, `Rating`

**Use case:** Find competitor products by search term

---

## 🎭 PLAYWRIGHT FOR API DISCOVERY (Build Phase Only)

### ⚠️ CRITICAL: Playwright Usage Strategy

**Phase 1: Build/Development (Use Playwright)**
- Use Playwright MCP to **discover** Takealot's public APIs
- Open product pages in browser
- Inspect Network tab for API calls
- Reverse-engineer the endpoints
- Document the API structure
- **Goal:** Find the APIs, don't scrape HTML

**Phase 2: Production (Use APIs)**
- Call the APIs directly (faster, more reliable)
- Only fallback to Playwright if specific data is missing
- Minimize browser automation in production

### How to Use Playwright MCP for API Discovery

**Step 1: Launch browser and navigate**
```javascript
// Use Playwright MCP (NOT Playwright Test)
await page.goto('https://www.takealot.com/product/PLID12345678');
```

**Step 2: Open DevTools Network tab**
```javascript
// In your actual browser (Chrome/Firefox), not in code:
// 1. Press F12 to open DevTools
// 2. Go to Network tab
// 3. Refresh the page
// 4. Look for XHR/Fetch requests
// 5. Find requests to api.takealot.com
```

**Step 3: Document the API calls**
```javascript
// Example of what you'll find:
// GET https://api.takealot.com/rest/v-1-10-0/product-details/PLID12345678
// GET https://api.takealot.com/rest/v-1-10-0/product-reviews/PLID12345678?start=0&rows=50
```

**Step 4: Test the APIs directly**
```bash
# Once discovered, test with curl:
curl "https://api.takealot.com/rest/v-1-10-0/product-details/PLID12345678"
```

**Step 5: Implement in your code**
```javascript
// Use fetch/axios instead of Playwright for production:
const response = await fetch(
  `https://api.takealot.com/rest/v-1-10-0/product-details/${plid}`
);
const data = await response.json();
```

### When to Use Playwright in Production

**Fallback scenarios ONLY:**
1. API doesn't return a specific field you need
2. API rate limits you (add delays first)
3. API structure changes and breaks your code

**Playwright scraping approach (fallback):**
```javascript
// Example: If API doesn't provide stock status
const page = await browser.newPage();
await page.goto(`https://www.takealot.com/product/${plid}`);

// Extract from page
const inStock = await page.$eval('[data-ref="buybox-add-to-cart"]', 
  el => !el.disabled
);

await page.close();
```

### Shared Playwright Resources

**Browser Profiles:**
- Amazon project has browser sessions at: `/home/ubuntubot/amazon-disputer/browser-data/`
- These are for Amazon.co.za (not Takealot)
- Create your own if needed: `/home/ubuntubot/takealot-tracker/browser-data/`

**Chromium Installation:**
- Already installed system-wide
- Use in your code:
```javascript
const { chromium } = require('playwright');
const browser = await chromium.launch({
  headless: true,
  executablePath: '/usr/bin/chromium-browser', // System Chromium
});
```

### Important: Playwright MCP vs Playwright Test

**✅ Use: Playwright MCP (Model Context Protocol)**
- For browser automation
- For data extraction
- Returns data to your application

**❌ Don't Use: Playwright Test**
- Testing framework (expect/assert)
- Not for production scraping
- Wrong tool for this job

---

## 🔄 DAILY SCRAPING WORKFLOW

### 1. Scheduled Execution

**Option A: PM2 Cron Restart (Recommended)**
```javascript
// In ecosystem.config.js
{
  name: 'takealot-scraper',
  script: './src/scraper.js',
  cron_restart: '0 4 * * *',  // Daily at 4 AM (Amazon bot runs at 2 AM)
  autorestart: false,
  watch: false,
}
```

**Option B: Separate Cron Job**
```bash
# crontab -e
0 4 * * * cd /home/ubuntubot/takealot-tracker && node src/scraper.js >> logs/cron.log 2>&1
```

**Why 4 AM?**
- Amazon Dispute Bot runs at 2 AM
- Avoids resource contention
- Low server load time
- After Takealot's own updates (usually midnight-2 AM)

### 2. Scraping Process

```
FOR EACH active product IN products table:
  
  1. Fetch product data using Public API
     - GET product-details/{PLID}
     - Extract: name, prices, availability
  
  2. Fetch review data using Public API
     - GET product-reviews/{PLID}
     - Extract: rating, review_count
  
  3. IF API fails OR data incomplete:
     - Fallback to Playwright MCP
     - Navigate to product page
     - Extract missing data
  
  4. Calculate discount percentage:
     - discount = ((regular_price - selling_price) / regular_price) * 100
  
  5. Insert snapshot into daily_snapshots table:
     - product_id
     - snapshot_date = TODAY
     - All collected data
  
  6. Update products.last_scraped_at = NOW()
  
  7. Log results (success/failure)

END FOR
```

### 3. Error Handling
- Retry failed requests (max 3 attempts)
- Log failures to error log
- Send notification if critical product fails
- Continue with other products

---

## 🖥️ USER INTERFACE (CLI)

### Command Structure

```bash
# Add a product to track
./tracker add-product https://www.takealot.com/product/PLID12345678

# Add a competitor to an existing product
./tracker add-competitor --main=PLID12345678 --competitor=PLID87654321

# List all tracked products
./tracker list-products

# View product history
./tracker history PLID12345678

# View price comparison with competitors
./tracker compare PLID12345678

# Remove a product
./tracker remove-product PLID12345678

# Run manual scrape (outside of schedule)
./tracker scrape-now

# View scraping logs
./tracker logs
```

### Example Output

```
$ ./tracker list-products

Tracked Products:
══════════════════════════════════════════════════════

1. Samsung Galaxy S24 Ultra (PLID12345678)
   Current Price: R18,999 (was R21,999) [-14%]
   Rating: 4.6/5 (1,234 reviews)
   Last Scraped: 2026-01-22 02:00:00
   Competitors: 3
   
   Competitors:
   - iPhone 15 Pro Max (PLID87654321) - R22,499 [+18%]
   - Xiaomi 14 Pro (PLID11112222) - R16,999 [-11%]
   - Google Pixel 8 Pro (PLID33334444) - R19,999 [+5%]

2. Sony WH-1000XM5 Headphones (PLID55556666)
   Current Price: R5,499 (was R6,999) [-21%]
   Rating: 4.8/5 (892 reviews)
   Last Scraped: 2026-01-22 02:00:00
   Competitors: 2
```

---

## 📈 REPORTING & ANALYTICS

### Key Reports to Generate

1. **Price History Chart**
   - Show price over time (last 30/60/90 days)
   - Highlight price drops
   - Show competitor prices on same chart

2. **Price Alert**
   - Notify when price drops below threshold
   - Notify when competitor undercuts your product

3. **Review Growth**
   - Track review count over time
   - Calculate review velocity (reviews per day)
   - Monitor rating changes

4. **Competitive Position**
   - Where does your product rank on price?
   - Rating comparison with competitors
   - Price difference percentage

5. **Best Time to Buy**
   - Identify recurring price patterns
   - Suggest when prices typically drop

---

## 🛠️ IMPLEMENTATION STEPS

### Phase 1: Setup & Basic Infrastructure
1. ✅ Set up Ubuntu environment
2. ✅ Install Node.js/Python
3. ✅ Set up database (SQLite)
4. ✅ Create database schema
5. ✅ Test Takealot Public API access

### Phase 2: Core Scraping Logic
1. ✅ Build API client for Takealot
2. ✅ Implement product-details fetching
3. ✅ Implement product-reviews fetching
4. ✅ Add error handling and retries
5. ✅ Test with sample PLIDs

### Phase 3: Playwright MCP Fallback
1. ✅ Set up Playwright MCP
2. ✅ Investigate page structure for missing data
3. ✅ Implement fallback scraping logic
4. ✅ Test fallback scenarios

### Phase 4: Product Management
1. ✅ CLI command: add-product
2. ✅ CLI command: add-competitor
3. ✅ CLI command: remove-product
4. ✅ CLI command: list-products
5. ✅ Validate PLIDs before adding

### Phase 5: Scheduling & Automation
1. ✅ Create scraping script
2. ✅ Set up cron job for daily execution
3. ✅ Add logging system
4. ✅ Add notification system (email/Slack)

### Phase 6: Reporting
1. ✅ CLI command: history (view snapshots)
2. ✅ CLI command: compare (competitor comparison)
3. ✅ Generate price charts (ASCII or export to CSV)
4. ✅ Price alert system

### Phase 7: Web UI (Recommended)

**Use Next.js (Same as Amazon Dashboard)**
- Consistent with existing project
- Server-side rendering
- Easy to deploy with PM2

**Setup:**
```bash
cd /home/ubuntubot/takealot-tracker
npx create-next-app@latest tracker-dashboard
cd tracker-dashboard
npm install pg chart.js date-fns
npm run build
```

**PM2 Configuration:**
```javascript
// Add to ecosystem.config.js
{
  name: 'takealot-dashboard',
  cwd: './tracker-dashboard',
  script: 'npm',
  args: 'start',
  env: {
    NODE_ENV: 'production',
    PORT: 3001,  // Different from Amazon dashboard (3000)
    DATABASE_URL: 'postgresql://amazon_user:amazon_secure_pass_2026@localhost:5432/takealot_tracker',
  },
  instances: 1,
  exec_mode: 'cluster',
}
```

**Access:**
- Local: `http://172.0.0.2:3001`
- Via SSH tunnel from Mac: `ssh -L 3001:localhost:3001 ubuntubot@172.0.0.2`
- Then open: `http://localhost:3001`

1. ⏳ Build Next.js dashboard
2. ⏳ Display price charts with Chart.js
3. ⏳ Product management via web
4. ⏳ Real-time scraping status

---

## 🔐 BEST PRACTICES

### Rate Limiting
- Add delay between API requests (500ms - 1s)
- Don't scrape the same product multiple times per day
- Respect Takealot's servers

### Error Handling
- Log all errors with context
- Implement exponential backoff for retries
- Don't fail entire batch if one product fails

### Data Quality
- Validate prices (must be positive numbers)
- Validate ratings (0-5 range)
- Handle missing/null values gracefully

### Storage Optimization
- Index frequently queried columns (plid, snapshot_date)
- Archive old snapshots after 1 year (optional)
- Regularly vacuum/optimize database

### Security
- Don't commit API keys to git
- Use environment variables for config
- Restrict database file permissions

---

## 📦 DELIVERABLES

### Required Files
```
/home/ubuntubot/takealot-tracker/
├── README.md                           # Project documentation
├── package.json                        # Node.js dependencies
├── ecosystem.config.js                 # PM2 configuration
├── .env.example                        # Environment template
├── src/
│   ├── scraper.js                     # Main scraping logic
│   ├── api-client.js                  # Takealot API wrapper
│   ├── playwright-fallback.js         # Browser automation (fallback only)
│   ├── database.js                    # PostgreSQL operations
│   ├── cli.js                         # CLI interface
│   └── scheduler.js                   # Scheduling logic (if not using PM2 cron)
├── config/
│   └── database.sql                   # Schema setup script
├── tracker-dashboard/                  # Next.js web UI (optional but recommended)
│   ├── app/
│   ├── components/
│   ├── lib/
│   └── public/
├── logs/                               # Application logs (PM2 managed)
│   ├── scraper.log
│   └── dashboard.log
└── scripts/
    ├── setup-database.sh              # Initialize PostgreSQL database
    ├── test-api.sh                    # Test Takealot APIs
    └── run-manual-scrape.sh           # Manual scraping script
```

### PM2 Ecosystem Configuration

**Create:** `/home/ubuntubot/takealot-tracker/ecosystem.config.js`
```javascript
module.exports = {
  apps: [
    {
      name: 'takealot-dashboard',
      cwd: './tracker-dashboard',
      script: 'npm',
      args: 'start',
      env: {
        NODE_ENV: 'production',
        PORT: 3001,
        DATABASE_URL: 'postgresql://amazon_user:amazon_secure_pass_2026@localhost:5432/takealot_tracker',
      },
      instances: 1,
      exec_mode: 'cluster',
      watch: false,
      max_memory_restart: '300M',
      error_file: './logs/dashboard-error.log',
      out_file: './logs/dashboard-out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    },
    {
      name: 'takealot-scraper',
      cwd: './',
      script: './src/scraper.js',
      env: {
        NODE_ENV: 'production',
        DATABASE_URL: 'postgresql://amazon_user:amazon_secure_pass_2026@localhost:5432/takealot_tracker',
      },
      instances: 1,
      exec_mode: 'fork',
      cron_restart: '0 4 * * *',  // Daily at 4 AM
      autorestart: false,
      watch: false,
      error_file: './logs/scraper-error.log',
      out_file: './logs/scraper-out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    },
  ],
};
```

**Start services:**
```bash
cd /home/ubuntubot/takealot-tracker
pm2 start ecosystem.config.js
pm2 save  # Save PM2 configuration
```

### Environment Variables

**Create:** `/home/ubuntubot/takealot-tracker/.env`
```bash
# Database
DATABASE_URL=postgresql://amazon_user:amazon_secure_pass_2026@localhost:5432/takealot_tracker

# Server
NODE_ENV=production
PORT=3001

# Scraping
SCRAPE_DELAY_MS=1000
MAX_RETRIES=3
USER_AGENT="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36"

# Optional: Redis (if using job queues)
REDIS_URL=redis://localhost:6379
```

### Documentation
- ✅ README with setup instructions
- ✅ API documentation (endpoints used)
- ✅ Database schema documentation
- ✅ CLI usage guide
- ✅ Troubleshooting guide

---

## 🎯 SUCCESS CRITERIA

### Functional Requirements
- ✅ Can add/remove products via CLI
- ✅ Can link competitors to products
- ✅ Scrapes all required data points
- ✅ Stores daily snapshots correctly
- ✅ Runs automatically every day
- ✅ Generates useful reports

### Performance Requirements
- ⚡ Scrapes 100 products in < 10 minutes
- ⚡ API requests complete in < 2 seconds each
- ⚡ Database queries return in < 100ms

### Reliability Requirements
- ✅ Handles network failures gracefully
- ✅ Logs all errors for debugging
- ✅ Continues scraping if one product fails
- ✅ Sends alerts for critical failures

---

## 🚨 IMPORTANT REMINDERS FOR THE AGENT

### ⚠️ READ FIRST: Lessons Learned Document

**BEFORE starting development, read:**
```
TAKEALOT_LESSONS_LEARNED.md
```

This document contains:
- Critical PostgreSQL quirks (psql hanging issue)
- Database setup clarification (create separate DB!)
- PM2 gotchas and solutions
- Playwright challenges and fixes
- Data validation patterns
- Performance optimizations
- Debugging tips
- Complete deployment checklist

**Key lessons:**
- Always use `PAGER=''` with psql commands
- Create `takealot_tracker` database (DON'T use `amazon_disputes`)
- Test with 2-3 products first, not 100
- Parse prices/ratings properly before storing
- Close browser connections in try/finally blocks

### Server Infrastructure Rules

1. **DO NOT install duplicate services**
   - PostgreSQL ✅ already running
   - Redis ✅ already running
   - Node.js ✅ already installed
   - PM2 ✅ already configured
   - Chromium ✅ already installed

2. **Use separate database**
   - Create: `takealot_tracker` database
   - DO NOT use: `amazon_disputes` database
   - Same credentials, different database

3. **Use port 3001 for dashboard**
   - Port 3000 is taken by Amazon Dashboard
   - Port 3001 is available

4. **Install at correct location**
   - Your path: `/home/ubuntubot/takealot-tracker/`
   - DO NOT modify: `/home/ubuntubot/amazon-disputer/`

5. **Log management**
   - PM2 log rotation already configured
   - Logs auto-rotate at 10 MB
   - 30 days retention

### API Discovery Strategy

1. **Use Playwright ONLY for discovering APIs**
   - Open product pages in browser
   - Inspect Network tab
   - Document API endpoints
   - Test APIs with curl

2. **Production: Use APIs directly**
   - Call `api.takealot.com` endpoints
   - Faster than browser automation
   - More reliable
   - Less resource-intensive

3. **Playwright fallback for missing data**
   - Only if API doesn't have the data
   - Minimize browser automation
   - Use existing Chromium installation

### Database Best Practices

1. **Use connection pooling**
   ```javascript
   const { Pool } = require('pg');
   const pool = new Pool({
     connectionString: process.env.DATABASE_URL,
     max: 20,
     idleTimeoutMillis: 30000,
   });
   ```

2. **One snapshot per day**
   - Use UNIQUE constraint: `(product_id, snapshot_date)`
   - Prevent duplicates
   - Use `ON CONFLICT DO UPDATE` for upserts

3. **Index for performance**
   - Index on `plid`, `snapshot_date`, `product_id`
   - Already provided in schema above

### Scheduling Best Practices

1. **Run at 4 AM (not 2 AM)**
   - Amazon bot runs at 2 AM
   - Avoid resource contention
   - Separate time slots

2. **Use PM2 cron_restart**
   - Built-in scheduling
   - Better than separate cron jobs
   - Logs managed by PM2

3. **Rate limiting**
   - 1 second delay between API calls
   - Respect Takealot's servers
   - Exponential backoff on errors

### Testing Before Deployment

1. **Test API connectivity**
   ```bash
   curl "https://api.takealot.com/rest/v-1-10-0/product-details/PLID12345678"
   ```

2. **Test database connection**
   ```bash
   PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -d takealot_tracker -c "SELECT version();"
   ```

3. **Test with small dataset first**
   - Add 2-3 products
   - Run manual scrape
   - Verify data stored correctly
   - Then scale up

### Code Quality

1. **Error handling**
   - Try-catch all API calls
   - Log errors with context
   - Continue on single product failure

2. **Data validation**
   - Validate PLIDs (format check)
   - Validate prices (positive numbers)
   - Validate ratings (0-5 range)

3. **Documentation**
   - Comment all functions
   - Document API responses
   - Create README with setup guide

---

## 📞 SUPPORT & REFERENCES

### Takealot URLs
- **Product Page:** `https://www.takealot.com/product/PLID{PLID}`
- **API Base:** `https://api.takealot.com`
- **Search:** `https://www.takealot.com/all?qsearch={query}`

### API Documentation
See **TAKEALOT PUBLIC API REFERENCE** section above for complete endpoint details.

### Example PLIDs for Testing
- PLID12345678 (use real PLIDs from actual products)
- Find PLIDs from any Takealot product URL
- Format: `PLID` followed by 8 digits

---

## ✅ FINAL CHECKLIST

Before considering the project complete:

- [ ] Can add products via CLI
- [ ] Can link competitors
- [ ] Successfully fetches data via API
- [ ] Playwright MCP fallback works
- [ ] Daily snapshots stored correctly
- [ ] Cron job configured and tested
- [ ] CLI commands all functional
- [ ] Error handling implemented
- [ ] Logging system works
- [ ] Documentation complete
- [ ] README with setup guide
- [ ] Tested with 10+ real products
- [ ] Price comparison working
- [ ] Historical data viewable
- [ ] No hardcoded credentials

---

**Project Status:** 📋 READY TO START  
**Estimated Duration:** 3-5 days  
**Priority:** High  
**Complexity:** Medium

---

## 🎬 GETTING STARTED

### Pre-Deployment Checklist

Before you start coding, verify server infrastructure:

```bash
# Connect to server
ssh ubuntubot@172.0.0.2

# Check PostgreSQL
pg_isready -h localhost -p 5432
# Expected: localhost:5432 - accepting connections

# Check Redis
redis-cli ping
# Expected: PONG

# Check Node.js
node --version
# Expected: v18.x.x or higher

# Check PM2
pm2 list
# Should show amazon-disputer services running

# Check available ports
sudo netstat -tuln | grep ':300'
# 3000 taken, 3001 available

# Create your project directory
mkdir -p /home/ubuntubot/takealot-tracker
cd /home/ubuntubot/takealot-tracker
```

### Step-by-Step Implementation

Agent, begin by:

1. **Initialize Node.js project**
   ```bash
   cd /home/ubuntubot/takealot-tracker
   npm init -y
   npm install pg express dotenv playwright
   ```

2. **Create database**
   ```bash
   # ⚠️ CRITICAL: Use PAGER='' to prevent hanging!
   PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -h localhost -c "CREATE DATABASE takealot_tracker;"
   
   # Verify it was created (not amazon_disputes!)
   PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -l | grep takealot
   ```

3. **Set up schema**
   - Copy the PostgreSQL schema from above
   - Save to `config/database.sql`
   - Run (with PAGER=''):
   ```bash
   PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -d takealot_tracker -f config/database.sql
   ```

4. **Test Takealot Public APIs**
   - Use curl to test endpoints
   - Document response structure
   - Create `src/api-client.js` wrapper

5. **Build CLI interface**
   - `src/cli.js` with commander or yargs
   - Implement `add-product` command first
   - Test adding a real product

6. **Implement scraper**
   - `src/scraper.js` for main logic
   - Use API client (not Playwright)
   - Store snapshots in database

7. **Set up PM2**
   - Create `ecosystem.config.js`
   - Test dashboard on port 3001
   - Configure cron for 4 AM

8. **Build web dashboard** (optional)
   - Next.js on port 3001
   - Display products and price charts
   - Show historical data

### Testing Workflow

```bash
# Add a test product
node src/cli.js add-product https://www.takealot.com/product/PLID12345678

# Run manual scrape
node src/scraper.js

# Check database
PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -d takealot_tracker -A -t -c "SELECT * FROM products;"
PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -d takealot_tracker -A -t -c "SELECT * FROM daily_snapshots ORDER BY scraped_at DESC LIMIT 5;"

# Start PM2 services
pm2 start ecosystem.config.js
pm2 logs takealot-dashboard

# Test dashboard
curl http://localhost:3001
# Or via SSH tunnel from your Mac:
# ssh -L 3001:localhost:3001 ubuntubot@172.0.0.2
# Then open http://localhost:3001 in browser
```

### Deployment Commands

```bash
# Full deployment sequence
cd /home/ubuntubot/takealot-tracker

# Install dependencies
npm install

# Setup database
bash scripts/setup-database.sh

# Test API connectivity
bash scripts/test-api.sh

# Build dashboard (if using Next.js)
cd tracker-dashboard && npm run build && cd ..

# Start all services
pm2 start ecosystem.config.js
pm2 save

# Check status
pm2 list
pm2 logs --lines 50

# Monitor first scrape
pm2 logs takealot-scraper --lines 100
```

### SSH Access from Mac

```bash
# From your Mac, create SSH tunnel for dashboard access
ssh -L 3001:localhost:3001 ubuntubot@172.0.0.2 -N

# In another terminal or browser, open:
open http://localhost:3001
```

### Monitoring & Maintenance

```bash
# Check PM2 status
pm2 status

# View logs
pm2 logs takealot-scraper --lines 50
pm2 logs takealot-dashboard --lines 50

# Restart services
pm2 restart takealot-scraper
pm2 restart takealot-dashboard

# Database size check
PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -d takealot_tracker -A -t -c "
SELECT 
  pg_size_pretty(pg_database_size('takealot_tracker')) as db_size,
  (SELECT count(*) FROM products) as products_count,
  (SELECT count(*) FROM daily_snapshots) as snapshots_count;
"

# Clean old snapshots (keep 1 year)
PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -d takealot_tracker -c "
DELETE FROM daily_snapshots 
WHERE snapshot_date < CURRENT_DATE - INTERVAL '1 year';
"
```

**Good luck! 🚀**

---

## 📚 QUICK REFERENCE

### 📖 Essential Reading
**BEFORE starting, read these documents:**
1. **`TAKEALOT_LESSONS_LEARNED.md`** - Critical quirks, gotchas, and solutions
2. **`PSQL-USAGE-RULES.md`** (in Amazon project) - PostgreSQL command usage
3. This project brief (you're reading it)

### Server Details
| Item | Value |
|------|-------|
| **IP Address** | `172.0.0.2` |
| **SSH User** | `ubuntubot` |
| **Project Path** | `/home/ubuntubot/takealot-tracker/` |
| **Database** | `takealot_tracker` (PostgreSQL) |
| **DB User** | `amazon_user` |
| **DB Password** | `amazon_secure_pass_2026` |
| **Dashboard Port** | `3001` (3000 is taken) |
| **Scrape Time** | `4:00 AM` (2 AM is taken) |

### Connection Strings
```bash
# PostgreSQL
postgresql://amazon_user:amazon_secure_pass_2026@localhost:5432/takealot_tracker

# Redis (optional)
redis://localhost:6379

# SSH from Mac
ssh ubuntubot@172.0.0.2

# SSH Tunnel (dashboard access from Mac)
ssh -L 3001:localhost:3001 ubuntubot@172.0.0.2 -N
```

### Essential Commands
```bash
# Database access (⚠️ Always use PAGER='' to prevent hanging!)
PAGER='' PGPASSWORD='amazon_secure_pass_2026' psql -U amazon_user -d takealot_tracker

# PM2 management
pm2 list
pm2 logs takealot-scraper --lines 50
pm2 restart takealot-scraper
pm2 save

# Service status
pm2 status
systemctl status postgresql
systemctl status redis-server
```

### Files You'll Create
```
/home/ubuntubot/takealot-tracker/
├── package.json
├── ecosystem.config.js          ← PM2 config
├── .env                          ← Environment variables
├── src/
│   ├── scraper.js               ← Main scraper (runs at 4 AM)
│   ├── api-client.js            ← Takealot API wrapper
│   ├── database.js              ← PostgreSQL queries
│   └── cli.js                   ← CLI commands
├── config/
│   └── database.sql             ← Schema setup
├── tracker-dashboard/           ← Next.js UI (port 3001)
└── logs/                        ← PM2 logs (auto-managed)
```

### Don't Touch
```
/home/ubuntubot/amazon-disputer/     ← Amazon Dispute Bot (existing project)
Port 3000                             ← Amazon Dashboard
Port 5432                             ← PostgreSQL (shared, different DB)
Port 6379                             ← Redis (shared)
Database: amazon_disputes             ← Amazon's database
2:00 AM schedule                      ← Amazon's scrape time
```

### Coexistence Strategy
- ✅ Same server, different directories
- ✅ Same PostgreSQL, different databases
- ✅ Same Redis server (optional use)
- ✅ Different ports (3001 vs 3000)
- ✅ Different schedules (4 AM vs 2 AM)
- ✅ Separate PM2 processes
- ✅ Separate log files

---

**Project Status:** 📋 READY TO START  
**Estimated Duration:** 3-5 days  
**Priority:** High  
**Complexity:** Medium  
**Agent:** New agent (not the Amazon bot agent)

---

**Last Updated:** January 22, 2026  
**Document Version:** 2.0 (Enhanced with server infrastructure details)
