# Quick Reference Guide

## 🚀 Quick Commands

### Check Recent Errors
```sql
SELECT 
  job_name,
  run_id,
  result_state,
  start_time,
  logged_at
FROM email_alert.sample_email.job_error_logs
ORDER BY logged_at DESC
LIMIT 10;
```

### Count Errors by Job
```sql
SELECT 
  job_name,
  COUNT(*) as error_count,
  MAX(logged_at) as last_error
FROM email_alert.sample_email.job_error_logs
GROUP BY job_name
ORDER BY error_count DESC;
```

### Get Full Error Details for a Specific Run
```sql
SELECT * 
FROM email_alert.sample_email.job_error_logs
WHERE run_id = 'YOUR_RUN_ID';
```

### Errors in Last 24 Hours
```sql
SELECT 
  job_name,
  run_id,
  state_message,
  logged_at
FROM email_alert.sample_email.job_error_logs
WHERE logged_at >= CURRENT_TIMESTAMP() - INTERVAL 24 hours
ORDER BY logged_at DESC;
```

## 🔧 Configuration Parameters

### Error Logger Notebook
```python
JOB_ID = 198373878629160          # Your monitored job ID
WORKSPACE_ID = "560696151014778"  # Your workspace ID
TABLE_NAME = "email_alert.sample_email.job_error_logs"
```

### Alert Query Time Window
```sql
-- Current: 1 hour
WHERE logged_at >= CURRENT_TIMESTAMP() - INTERVAL 1 hour

-- Options:
INTERVAL 30 minutes
INTERVAL 2 hours
INTERVAL 1 day
```

### Email Template Key Syntax
```html
{{#QUERY_RESULT_ROWS}}
  {{column_name}}  <!-- Displays value from query result -->
{{/QUERY_RESULT_ROWS}}
```

## 📊 Monitoring Schedule Recommendations

| Component | Recommended Schedule | Purpose |
|-----------|---------------------|---------|
| Error Logger Job | Every 15-30 min | Capture failures quickly |
| SQL Alert Refresh | Every 1-5 min | Immediate notification |
| Query Time Window | 1 hour | Catch recent errors |

## 🐛 Troubleshooting Checklist

### Email Not Arriving?
- [ ] Alert is enabled (green toggle)
- [ ] Email address is correct
- [ ] Template has `{{#QUERY_RESULT_ROWS}}...{{/QUERY_RESULT_ROWS}}`
- [ ] Query returns data when run manually
- [ ] Check spam/junk folder
- [ ] Alert refresh schedule is frequent enough

### No Errors Logged?
- [ ] JOB_ID matches your monitored job
- [ ] Job has actually failed
- [ ] Error Logger job is scheduled and running
- [ ] Permissions to access job runs via SDK
- [ ] Table exists and is writable

### Alert Triggering But Empty Email?
- [ ] **Most common**: Template missing `{{#QUERY_RESULT_ROWS}}` wrapper
- [ ] Variable names match column names in SELECT
- [ ] Query returns at least one row

## 📍 Important Files

| File | Purpose |
|------|---------|
| `notebooks/Error_Logger_Job_Monitor.py` | Main error capture logic |
| `sql/job_error_log_alert.sql` | Alert query |
| `config/email_template.html` | Email body template |
| `sql/create_schema.sql` | Initial setup SQL |
| `README.md` | Full documentation |
| `SETUP_GUIDE.md` | Step-by-step setup |

## 🔗 Useful Links

* **Job URL Format**: `https://your-workspace.cloud.databricks.com/?o=WORKSPACE_ID#job/JOB_ID`
* **Run URL Format**: `https://your-workspace.cloud.databricks.com/?o=WORKSPACE_ID#job/JOB_ID/run/RUN_ID`
* **Databricks SDK Docs**: https://docs.databricks.com/en/dev-tools/sdk-python.html
* **SQL Alerts Docs**: https://docs.databricks.com/en/sql/user/alerts/index.html

## 💡 Pro Tips

1. **Start Small**: Monitor one critical job first, then expand
2. **Adjust Time Windows**: Tune based on your job failure frequency
3. **Review Regularly**: Check error trends weekly
4. **Clean Old Logs**: Archive/delete errors older than 90 days
5. **Document Patterns**: Note recurring errors for root cause analysis

## 🆘 Common Issues & Solutions

### Issue: Duplicate Errors in Table
**Solution**: The system auto-prevents duplicates. If you see them:
```python
# Check duplicate prevention logic in notebook Cell 10
existing_run_ids_set = {row.run_id for row in existing_run_ids}
new_failed_runs = [error for error in failed_runs 
                   if str(error['run_id']) not in existing_run_ids_set]
```

### Issue: Alert Not Finding Recent Errors
**Solution**: Increase time window
```sql
-- Change from:
WHERE logged_at >= CURRENT_TIMESTAMP() - INTERVAL 1 hour
-- To:
WHERE logged_at >= CURRENT_TIMESTAMP() - INTERVAL 2 hours
```

### Issue: Error Messages Too Long for Email
**Solution**: Truncate in query
```sql
SELECT 
  SUBSTRING(state_message, 1, 500) as state_message_truncated,
  ...
```

## 📈 Next Steps

1. Monitor system for 1 week
2. Review error patterns
3. Adjust schedules and time windows
4. Add more jobs to monitoring
5. Create error trend dashboards
6. Set up additional notification channels (Slack, Teams)

---

**Need Help?** Check the full [README.md](README.md) or [SETUP_GUIDE.md](SETUP_GUIDE.md)
