# 🚀 TAKEALOT COMPETITOR TRACKER - IMPLEMENTATION PLAN
**Date:** January 22, 2026  
**Status:** Ready for Approval  
**Prepared By:** AI Development Agent

---

## 📋 EXECUTIVE SUMMARY

Based on comprehensive documentation review and server infrastructure testing, this plan outlines a phased approach to building the Takealot Product & Competitor Tracker system on the shared Ubuntu server (172.0.0.2).

### ✅ Server Access Confirmed
- **Connection:** SSH to `ubuntubot@172.0.0.2` - ✅ SUCCESSFUL
- **Server:** ARI-UBUNTU, Ubuntu 24.04, 1 day uptime
- **Services Running:** PostgreSQL 16.11, Redis 7.0.15, PM2 installed
- **Development Stack:** Node.js v20.20.0, npm 10.8.2, Python 3.12.3

### ⚠️ Critical Issues Identified
1. **Database User Missing:** `ubuntubot` PostgreSQL role doesn't exist
2. **PM2 Configuration:** Needs proper setup for new project
3. **Lessons Learned:** Must implement psql pager fixes from Amazon bot experience

---

## 🎯 PROJECT GOALS

### Primary Objectives
1. **Daily Product Tracking** - Monitor Takealot product prices, stock, reviews
2. **Competitor Monitoring** - Track competitor pricing across similar products
3. **Automated Alerts** - Price change notifications and competitor insights
4. **Data Analytics** - Historical trends and competitive analysis
5. **API Integration** - WhatsApp notifications for critical updates

### Success Metrics
- Track 50+ products daily with <5min update cycle
- 99%+ uptime for monitoring service
- <10s response time for API queries
- Zero data loss with automated backups

---

## 🏗️ ARCHITECTURE OVERVIEW

### Technology Stack Decision

**Recommendation: Node.js/TypeScript (Aligned with Amazon Bot)**

**Rationale:**
- ✅ Proven on same server (Amazon Disputer running successfully)
- ✅ PM2 already configured and working
- ✅ Playwright experience from Amazon bot directly applicable
- ✅ Team has recent Node.js/PostgreSQL patterns documented
- ✅ Easier code reuse and maintenance
- ❌ Python would require separate PM2 config and ecosystem management

### System Components

```
┌─────────────────────────────────────────────────────────┐
│                    TAKEALOT TRACKER                      │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐    ┌──────────────┐   ┌────────────┐ │
│  │   Scraper    │───▶│  Processor   │──▶│  Database  │ │
│  │  (Playwright)│    │  (Business   │   │(PostgreSQL)│ │
│  │              │    │   Logic)     │   │            │ │
│  └──────────────┘    └──────────────┘   └────────────┘ │
│         │                    │                  │        │
│         │                    ▼                  │        │
│         │            ┌──────────────┐           │        │
│         └───────────▶│ Cache/Queue  │◀──────────┘        │
│                      │   (Redis)    │                    │
│                      └──────────────┘                    │
│                             │                            │
│                             ▼                            │
│                   ┌──────────────────┐                  │
│                   │  Notification    │                  │
│                   │    Service       │                  │
│                   │  (WhatsApp API)  │                  │
│                   └──────────────────┘                  │
└─────────────────────────────────────────────────────────┘
```

---

## 📁 PROJECT STRUCTURE

```
~/takealot-tracker/
├── src/
│   ├── scrapers/
│   │   ├── product-scraper.ts       # Main product page scraper
│   │   ├── competitor-scraper.ts    # Competitor product scraper
│   │   ├── search-scraper.ts        # Search results scraper
│   │   └── base-scraper.ts          # Common scraping utilities
│   ├── database/
│   │   ├── connection.ts            # PostgreSQL connection manager
│   │   ├── models/
│   │   │   ├── Product.ts
│   │   │   ├── PriceHistory.ts
│   │   │   ├── Competitor.ts
│   │   │   └── Alert.ts
│   │   └── migrations/
│   │       └── init.sql
│   ├── processors/
│   │   ├── price-analyzer.ts        # Price trend analysis
│   │   ├── competitor-matcher.ts    # Match products to competitors
│   │   └── alert-generator.ts       # Generate price alerts
│   ├── services/
│   │   ├── notification.ts          # WhatsApp/Email notifications
│   │   ├── cache.ts                 # Redis caching layer
│   │   └── scheduler.ts             # Cron job management
│   ├── api/
│   │   ├── routes/
│   │   │   ├── products.ts
│   │   │   ├── competitors.ts
│   │   │   └── analytics.ts
│   │   └── server.ts
│   ├── config/
│   │   ├── database.ts
│   │   ├── scraper.ts
│   │   └── environment.ts
│   └── utils/
│       ├── logger.ts
│       ├── retry.ts
│       └── validators.ts
├── scripts/
│   ├── setup-database.sh            # Database initialization
│   ├── test-scraper.js              # Scraper testing
│   └── backup.sh                    # Backup automation
├── tests/
│   ├── unit/
│   └── integration/
├── logs/
├── .env.example
├── .gitignore
├── package.json
├── tsconfig.json
├── ecosystem.config.js              # PM2 configuration
└── README.md
```

---

## 📊 DATABASE SCHEMA

### Core Tables

```sql
-- Products being tracked
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    takealot_id VARCHAR(50) UNIQUE NOT NULL,
    name TEXT NOT NULL,
    brand VARCHAR(255),
    category VARCHAR(255),
    current_price DECIMAL(10,2),
    original_price DECIMAL(10,2),
    stock_status VARCHAR(50),
    rating DECIMAL(3,2),
    review_count INTEGER,
    product_url TEXT,
    image_url TEXT,
    last_checked TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Price history for trend analysis
CREATE TABLE price_history (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(id) ON DELETE CASCADE,
    price DECIMAL(10,2) NOT NULL,
    stock_status VARCHAR(50),
    checked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    price_change DECIMAL(10,2),
    percent_change DECIMAL(5,2)
);

-- Competitor products
CREATE TABLE competitors (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(id) ON DELETE CASCADE,
    competitor_name VARCHAR(255) NOT NULL,
    competitor_product_id VARCHAR(50),
    competitor_price DECIMAL(10,2),
    competitor_url TEXT,
    similarity_score DECIMAL(3,2),
    last_checked TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Price alerts
CREATE TABLE alerts (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(id) ON DELETE CASCADE,
    alert_type VARCHAR(50) NOT NULL, -- 'price_drop', 'price_increase', 'competitor_lower'
    threshold_value DECIMAL(10,2),
    previous_value DECIMAL(10,2),
    current_value DECIMAL(10,2),
    message TEXT,
    is_sent BOOLEAN DEFAULT FALSE,
    sent_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Scraping logs
CREATE TABLE scrape_logs (
    id SERIAL PRIMARY KEY,
    scraper_type VARCHAR(50),
    target_id VARCHAR(100),
    status VARCHAR(50), -- 'success', 'failed', 'partial'
    products_found INTEGER,
    duration_ms INTEGER,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for performance
CREATE INDEX idx_products_takealot_id ON products(takealot_id);
CREATE INDEX idx_price_history_product_checked ON price_history(product_id, checked_at DESC);
CREATE INDEX idx_competitors_product ON competitors(product_id);
CREATE INDEX idx_alerts_unsent ON alerts(is_sent, created_at) WHERE is_sent = FALSE;
CREATE INDEX idx_scrape_logs_created ON scrape_logs(created_at DESC);
```

---

## 🔧 IMPLEMENTATION PHASES

### **PHASE 1: Foundation & Setup (Days 1-2)**

#### 1.1 Server Setup
- [ ] Create database user and database
- [ ] Set up project directory structure
- [ ] Configure environment variables
- [ ] Test database connectivity

**Commands:**
```bash
# On server as ubuntubot
mkdir -p ~/takealot-tracker
cd ~/takealot-tracker

# Database setup (will need postgres user password)
sudo -u postgres psql << EOF
CREATE USER takealot_user WITH PASSWORD 'SecurePassword123!';
CREATE DATABASE takealot_tracker OWNER takealot_user;
GRANT ALL PRIVILEGES ON DATABASE takealot_tracker TO takealot_user;
EOF
```

#### 1.2 Project Initialization
- [ ] Initialize Node.js project
- [ ] Install dependencies (TypeScript, Playwright, PostgreSQL client, Redis)
- [ ] Set up TypeScript configuration
- [ ] Create basic folder structure

**Dependencies:**
```json
{
  "dependencies": {
    "playwright": "^1.40.0",
    "pg": "^8.11.0",
    "redis": "^4.6.0",
    "dotenv": "^16.3.0",
    "express": "^4.18.0",
    "node-cron": "^3.0.0",
    "winston": "^3.11.0",
    "axios": "^1.6.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "@types/node": "^20.10.0",
    "@types/express": "^4.17.0",
    "@types/pg": "^8.10.0",
    "ts-node": "^10.9.0",
    "nodemon": "^3.0.0"
  }
}
```

#### 1.3 Critical Fixes Implementation
- [ ] Implement psql pager fix (PAGER=cat) everywhere
- [ ] Create database connection wrapper with proper error handling
- [ ] Set up logging system (Winston)

---

### **PHASE 2: Core Scraping Engine (Days 3-5)**

#### 2.1 Base Scraper Infrastructure
- [ ] Create Playwright browser manager (reuse from Amazon bot)
- [ ] Implement retry logic with exponential backoff
- [ ] Add user-agent rotation
- [ ] Set up proxy support (if needed)

#### 2.2 Product Page Scraper
- [ ] Parse product details from Takealot product pages
- [ ] Extract price, stock, ratings, reviews
- [ ] Handle variations and bundles
- [ ] Validate data before database insertion

**Target URL Pattern:**
```
https://www.takealot.com/[product-name]/PLID[product-id]
Example: https://www.takealot.com/samsung-galaxy-s23/PLID94045678
```

#### 2.3 Search & Competitor Scraper
- [ ] Scrape search results for competitor products
- [ ] Implement product matching algorithm (fuzzy matching by name/brand)
- [ ] Extract competitor pricing
- [ ] Store competitor relationships

#### 2.4 Data Storage
- [ ] Implement database models
- [ ] Create product insertion/update logic
- [ ] Store price history on every check
- [ ] Handle duplicates and conflicts

---

### **PHASE 3: Business Logic & Analytics (Days 6-8)**

#### 3.1 Price Analysis Engine
- [ ] Calculate price trends (7-day, 30-day averages)
- [ ] Detect price drops/increases
- [ ] Identify pricing patterns
- [ ] Generate competitor price comparisons

#### 3.2 Alert System
- [ ] Define alert rules:
  - Price drops > 10%
  - Competitor prices lower by > 5%
  - Out of stock → back in stock
  - Price increased after drop (buy signal)
- [ ] Alert generation logic
- [ ] Alert deduplication (don't spam)

#### 3.3 Redis Caching Layer
- [ ] Cache product data (5-minute TTL)
- [ ] Queue scraping jobs
- [ ] Rate limiting for Takealot requests
- [ ] Store temporary scrape data

---

### **PHASE 4: Automation & Scheduling (Days 9-10)**

#### 4.1 Cron Jobs
- [ ] Set up node-cron scheduler
- [ ] Schedule product checks every 6 hours
- [ ] Schedule competitor scans daily
- [ ] Schedule cleanup jobs (old logs, old price history)

#### 4.2 PM2 Configuration
```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'takealot-tracker',
    script: './dist/index.js',
    instances: 1,
    autorestart: true,
    watch: false,
    max_memory_restart: '500M',
    env: {
      NODE_ENV: 'production',
      TZ: 'Africa/Johannesburg'
    },
    error_file: './logs/pm2-error.log',
    out_file: './logs/pm2-out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z'
  }]
};
```

#### 4.3 Monitoring & Health Checks
- [ ] Implement health check endpoint
- [ ] Monitor scraper success rates
- [ ] Alert on service failures
- [ ] Log all operations

---

### **PHASE 5: API & Notifications (Days 11-12)**

#### 5.1 REST API
- [ ] `GET /api/products` - List tracked products
- [ ] `GET /api/products/:id` - Product details with history
- [ ] `POST /api/products` - Add product to tracking
- [ ] `DELETE /api/products/:id` - Remove product
- [ ] `GET /api/competitors/:productId` - Get competitors
- [ ] `GET /api/analytics` - Price trends dashboard data

#### 5.2 WhatsApp Notifications
- [ ] Integrate WhatsApp Business API
- [ ] Send price drop alerts
- [ ] Send competitor alerts
- [ ] Format messages with product links

#### 5.3 Dashboard (Optional - Future Enhancement)
- [ ] Simple HTML dashboard for viewing data
- [ ] Charts for price trends
- [ ] Competitor comparison tables

---

### **PHASE 6: Testing & Deployment (Days 13-14)**

#### 6.1 Testing
- [ ] Unit tests for scrapers
- [ ] Integration tests for database operations
- [ ] End-to-end test: Add product → Scrape → Store → Alert
- [ ] Load testing (100+ products)

#### 6.2 Deployment
- [ ] Deploy to server via Git
- [ ] Start with PM2
- [ ] Configure auto-restart on failure
- [ ] Set up log rotation

#### 6.3 Monitoring & Optimization
- [ ] Monitor memory usage
- [ ] Optimize database queries
- [ ] Fine-tune scraping intervals
- [ ] Review and adjust alert thresholds

---

## ⚠️ CRITICAL LESSONS LEARNED (From Amazon Bot)

### 1. PostgreSQL psql Pager Issue 🚨
**Problem:** `psql` commands hang waiting for keyboard input  
**Solution:** ALWAYS use `PAGER=cat` or `-P pager=off`

```javascript
// WRONG
const { stdout } = await exec('psql -U takealot_user -d takealot_tracker -c "SELECT * FROM products;"');

// CORRECT
const { stdout } = await exec('PAGER=cat psql -U takealot_user -d takealot_tracker -c "SELECT * FROM products;"');
```

### 2. Database Setup Clarity
**Lesson:** Create SEPARATE database for Takealot tracker  
**Action:** `takealot_tracker` database, NOT reusing `amazon_disputes`

### 3. PM2 Process Management
**Lesson:** Use unique app names in PM2  
**Action:** Name it `takealot-tracker` (not `amazon-disputer`)

### 4. Playwright Browser Management
**Lesson:** Reuse browser contexts, close properly, handle crashes  
**Action:** Implement browser pool with health checks

### 5. Data Validation
**Lesson:** Validate ALL scraped data before database insertion  
**Action:** TypeScript interfaces + runtime validation (Zod/Joi)

### 6. Rate Limiting
**Lesson:** Don't hammer Takealot servers  
**Action:** 2-5 second delays between requests, respect robots.txt

### 7. Error Handling
**Lesson:** Network errors are common  
**Action:** Retry logic with exponential backoff, log all failures

---

## 📦 DELIVERABLES

### Code Deliverables
1. ✅ Complete TypeScript/Node.js application
2. ✅ Database schema and migrations
3. ✅ PM2 configuration file
4. ✅ Environment configuration template
5. ✅ Comprehensive logging
6. ✅ API documentation
7. ✅ Deployment scripts

### Documentation Deliverables
1. ✅ Setup guide
2. ✅ API reference
3. ✅ Scraper configuration guide
4. ✅ Troubleshooting guide
5. ✅ Maintenance procedures

### Operational Deliverables
1. ✅ Automated backups
2. ✅ Health monitoring
3. ✅ Alert system
4. ✅ Log rotation

---

## 🎯 SUCCESS CRITERIA

### Technical Criteria
- [ ] Successfully track 50+ products with 99%+ uptime
- [ ] Price updates every 6 hours
- [ ] API response time < 500ms for queries
- [ ] Zero database deadlocks or data loss
- [ ] Scraper success rate > 95%

### Business Criteria
- [ ] Accurate price tracking with < 1% error rate
- [ ] Competitor matching accuracy > 80%
- [ ] Timely alerts (< 10 minutes from price change)
- [ ] Automated daily reports
- [ ] Easy product addition (< 30 seconds)

---

## ⏱️ TIMELINE ESTIMATE

| Phase | Duration | Effort | Priority |
|-------|----------|--------|----------|
| Phase 1: Foundation | 2 days | 16 hours | Critical |
| Phase 2: Scraping | 3 days | 24 hours | Critical |
| Phase 3: Analytics | 3 days | 24 hours | High |
| Phase 4: Automation | 2 days | 16 hours | High |
| Phase 5: API | 2 days | 16 hours | Medium |
| Phase 6: Testing | 2 days | 16 hours | Critical |
| **TOTAL** | **14 days** | **112 hours** | |

**Buffer:** +3 days for unexpected issues  
**Total Estimated:** 17 calendar days

---

## 💰 RESOURCE REQUIREMENTS

### Infrastructure (Existing - No Cost)
- ✅ Ubuntu server (shared)
- ✅ PostgreSQL 16
- ✅ Redis 7
- ✅ Node.js v20

### New Services (Required)
- 🔔 WhatsApp Business API (if not existing)
  - Alternative: Twilio WhatsApp ($0.005/message)
  - Alternative: Email notifications (free)

### Development Tools (Free)
- ✅ VS Code / Cursor
- ✅ Git
- ✅ Playwright (free)

---

## 🔐 SECURITY CONSIDERATIONS

### Credentials Management
- [ ] Store all credentials in `.env` file (never commit)
- [ ] Use strong database passwords
- [ ] Rotate API keys regularly

### Data Privacy
- [ ] No personal data collection
- [ ] Only public Takealot data
- [ ] Comply with robots.txt
- [ ] Respect rate limits

### Server Security
- [ ] Firewall configuration (only necessary ports)
- [ ] Regular security updates
- [ ] Backup encryption
- [ ] Limited SSH access

---

## 📊 MONITORING & MAINTENANCE

### Daily Checks
- [ ] PM2 process status (`pm2 status`)
- [ ] Error log review (`tail -f ~/takealot-tracker/logs/error.log`)
- [ ] Scraper success rate

### Weekly Maintenance
- [ ] Database backup verification
- [ ] Disk space check
- [ ] Performance metrics review
- [ ] Update dependencies (if needed)

### Monthly Tasks
- [ ] Clean old price history (keep 90 days)
- [ ] Analyze trends and optimize
- [ ] Review and adjust alert thresholds
- [ ] Security updates

---

## 🚀 NEXT STEPS FOR APPROVAL

### Immediate Actions (Upon Approval)
1. **Confirm database credentials** - Need postgres user access to create `takealot_user`
2. **Approve technology stack** - Node.js/TypeScript recommended
3. **Prioritize features** - Which features are MVP vs nice-to-have?
4. **Set timeline expectations** - 14-17 days realistic?

### Questions for Stakeholder
1. **Product List:** Do you have an initial list of products to track?
2. **Notification Preference:** WhatsApp, Email, or both?
3. **Update Frequency:** Is 6-hour intervals acceptable? (Can be adjusted)
4. **Competitor Definition:** How should we identify competitors? (Same brand? Similar price? Category?)
5. **Alert Thresholds:** What price drop % should trigger alerts? (Recommended: 10%)

---

## 📞 CONTACT & SUPPORT

**Project Lead:** AI Development Agent  
**Server Administrator:** System Admin (password access: Trash081!)  
**Server Location:** 172.0.0.2 (ari-ubuntu)

---

## ✅ APPROVAL SIGN-OFF

This implementation plan is ready for review and approval.

**Prepared By:** AI Development Agent  
**Date:** January 22, 2026  
**Status:** 🟡 PENDING APPROVAL

**Approver Signature:** _________________  
**Date Approved:** _________________

---

**Once approved, development will commence immediately with Phase 1: Foundation & Setup.**
