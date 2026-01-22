# 🛠️ Manual Setup Steps Required

## 🚨 CRITICAL: Database Schema Setup

The Next.js application has been built successfully, but **database tables do not exist yet**. You need sudo access to complete the setup.

---

## Step 1: SSH to Server with Sudo Access

```bash
ssh ubuntubot@172.0.0.2
# Enter password: Trash081!
```

---

## Step 2: Grant PostgreSQL Permissions

Run these commands to grant the `amazon_user` permissions on the `takealot_tracker` database:

```bash
sudo -u postgres psql
```

In the PostgreSQL prompt:

```sql
-- Connect to takealot_tracker database
\c takealot_tracker

-- Grant all privileges on schema public
GRANT ALL ON SCHEMA public TO amazon_user;

-- Grant create privileges
ALTER DATABASE takealot_tracker OWNER TO amazon_user;

-- Grant future table permissions
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO amazon_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO amazon_user;

-- Exit
\q
```

---

## Step 3: Apply Database Schema

Now apply the schema as the `amazon_user`:

```bash
cd ~/takealot-tracker
export PGPASSWORD='amazon_secure_pass_2026'
psql -U amazon_user -d takealot_tracker -f lib/database/schema.sql
```

### Verify Tables Were Created:

```bash
psql -U amazon_user -d takealot_tracker -c "\dt"
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

## Step 4: Start Development Server (Test Mode)

```bash
cd ~/takealot-tracker
npm run dev
```

**Access Dashboard:** Open browser to `http://172.0.0.2:3001`

### Test Scraping:
1. Click "Test Example Product" buttons on the dashboard
2. Or paste a Takealot product ID (e.g., `PLID92991176`)
3. Click "Scrape Product"

---

## Step 5: Deploy to Production with PM2

Once testing is successful:

```bash
# Stop development server (Ctrl+C)

# Build production bundle (already done)
npm run build

# Start with PM2
pm2 start ecosystem.config.js

# Save PM2 configuration
pm2 save

# Set PM2 to auto-start on server reboot
pm2 startup
# Follow the command output to complete startup configuration
```

### PM2 Management Commands:

```bash
# Check status
pm2 status

# View logs
pm2 logs takealot-tracker

# Restart application
pm2 restart takealot-tracker

# Stop application
pm2 stop takealot-tracker

# Monitor in real-time
pm2 monit
```

---

## Step 6: Verify Production Deployment

1. **Check Process:**
   ```bash
   pm2 status
   ```

2. **Check Logs:**
   ```bash
   pm2 logs takealot-tracker --lines 50
   ```

3. **Test Dashboard:**
   - Open: `http://172.0.0.2:3001`
   - Verify all features work

4. **Test API Endpoints:**
   ```bash
   curl http://localhost:3001/api/products
   curl http://localhost:3001/api/analytics
   ```

---

## 🎯 Quick Validation Checklist

- [ ] PostgreSQL permissions granted to `amazon_user`
- [ ] Database schema applied (6 tables created)
- [ ] Tables verified with `\dt` command
- [ ] Development server starts without errors
- [ ] Dashboard accessible at http://172.0.0.2:3001
- [ ] Product scraping works (test with example products)
- [ ] PM2 process running
- [ ] PM2 auto-startup configured
- [ ] Production dashboard accessible
- [ ] API endpoints responding correctly

---

## 📝 Configuration Summary

| Setting | Value |
|---------|-------|
| **Server IP** | 172.0.0.2 |
| **Port** | 3001 |
| **Database** | takealot_tracker |
| **DB User** | amazon_user |
| **DB Password** | amazon_secure_pass_2026 |
| **Project Path** | /home/ubuntubot/takealot-tracker |
| **PM2 Process** | takealot-tracker |

---

## 🐛 Troubleshooting

### Issue: "relation does not exist" errors
**Solution:** Schema not applied. Follow Step 2 and Step 3 above.

### Issue: Permission denied for schema public
**Solution:** Grant permissions in Step 2.

### Issue: Port 3001 already in use
**Solution:** Check for existing processes:
```bash
sudo lsof -i :3001
# Kill the process if needed
sudo kill -9 <PID>
```

### Issue: PM2 process crashes
**Solution:** Check logs:
```bash
pm2 logs takealot-tracker
# Check for database connection errors
```

### Issue: Database connection refused
**Solution:** Verify PostgreSQL is running:
```bash
sudo systemctl status postgresql
sudo systemctl start postgresql
```

---

## 🔄 Next Steps After Setup

1. **Add More Products:** Use the dashboard to add Takealot products to track
2. **Set Up Cron Jobs:** Implement scheduled scraping (every 6 hours)
3. **Configure Alerts:** Set up price drop notifications
4. **Add Competitors:** Link competitor products for price comparison
5. **Monitor Performance:** Use PM2 monitoring and logs

---

## 📞 Support

If you encounter issues:
1. Check the logs: `pm2 logs takealot-tracker`
2. Verify database connection: `psql -U amazon_user -d takealot_tracker -c "SELECT version();"`
3. Check process status: `pm2 status`
4. Review `.env` file for correct credentials

---

**Last Updated:** $(date)
**Status:** Ready for manual database setup ✅
