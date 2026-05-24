# Setup Guide

This guide will walk you through setting up the Databricks Job Error Monitoring System step-by-step.

## Prerequisites

Before you begin, ensure you have:

- [ ] Databricks Workspace with Unity Catalog enabled
- [ ] Admin or user permissions to create:
  - Notebooks
  - Jobs
  - SQL queries and alerts
  - Unity Catalog tables
- [ ] A SQL warehouse for running alerts
- [ ] At least one Databricks job to monitor

## Step 1: Create Unity Catalog Schema

1. Open a SQL editor in Databricks SQL
2. Run the SQL commands from `sql/create_schema.sql`:
   ```sql
   CREATE CATALOG IF NOT EXISTS email_alert;
   CREATE SCHEMA IF NOT EXISTS email_alert.sample_email;
   ```

3. Update catalog/schema names if needed for your organization

## Step 2: Import and Configure the Error Logger Notebook

1. **Import the notebook:**
   - Go to Workspace → Import
   - Upload `notebooks/Error_Logger_Job_Monitor.py`
   - Or use Databricks CLI: `databricks workspace import ...`

2. **Configure the notebook parameters (Cell 3):**
   ```python
   JOB_ID = YOUR_JOB_ID  # Find in your job's URL
   WORKSPACE_ID = "YOUR_WORKSPACE_ID"  # Find in workspace URL
   TABLE_NAME = "email_alert.sample_email.job_error_logs"
   ```

3. **Test the notebook manually:**
   - Attach to a cluster
   - Run all cells
   - Verify it can access your job and fetch run history

## Step 3: Create a Job for the Error Logger

1. **Go to Workflows → Jobs → Create Job**

2. **Configure the job:**
   * **Name**: `Error Logger - Job Monitor`
   * **Task Type**: Notebook
   * **Notebook Path**: Select your imported Error Logger notebook
   * **Cluster**: 
     - Option A: Use an existing cluster
     - Option B: Create a new job cluster (recommended)
       - Node type: Smallest available (e.g., Standard_DS3_v2)
       - Workers: 1-2 workers
   * **Schedule**: 
     - Cron: `*/15 * * * *` (every 15 minutes)
     - Or: `*/30 * * * *` (every 30 minutes)
     - Or: `0 * * * *` (hourly)

3. **Save and run the job once manually** to verify it works

## Step 4: Create SQL Alert Query

1. **Go to Databricks SQL → Queries → Create Query**

2. **Name the query**: `Job Error Log Alert Monitor`

3. **Copy the SQL from** `sql/job_error_log_alert.sql`:
   ```sql
   SELECT 
     workspace_id,
     job_id,
     job_name,
     run_id,
     run_name,
     start_time,
     end_time,
     duration_seconds,
     result_state,
     state_message,
     logged_at
   FROM email_alert.sample_email.job_error_logs
   WHERE logged_at >= CURRENT_TIMESTAMP() - INTERVAL 1 hour
   ORDER BY logged_at DESC
   LIMIT 1;
   ```

4. **Update table name** if you changed the catalog/schema

5. **Save the query**

## Step 5: Configure the SQL Alert

1. **From the query page, click "Create Alert"**

2. **Configure alert settings:**
   * **Alert Name**: `Job Error Alert - Email Notification`
   * **Trigger Condition**:
     - Condition: `Value` or `Rows`
     - Operator: `>`
     - Threshold: `0`
   * **Refresh Schedule**:
     - Schedule: `1 minute` or `5 minutes` (for immediate notifications)

3. **Add Email Destination:**
   * Click "Add Destination" → Email
   * **Recipients**: Enter your email address(es)
   * **Subject**: `🚨 Job Failure Alert - {{job_name}}`
   * **Body**: Click "Custom template" and paste content from `config/email_template.html`:
     ```html
     {{#QUERY_RESULT_ROWS}}
     <html>
     <body style="font-family: Arial, sans-serif; font-size: 14px; color: #333;">
         <h2 style="color: #d9534f;">⚠️ Job Failure Alert</h2>
         
         <p><strong>Job Name:</strong> {{job_name}}</p>
         <p><strong>Job ID:</strong> {{job_id}}</p>
         <p><strong>Run ID:</strong> {{run_id}}</p>
         <p><strong>Workspace ID:</strong> {{workspace_id}}</p>
         
         <p><strong>Start Time:</strong> {{start_time}}</p>
         <p><strong>End Time:</strong> {{end_time}}</p>
         <p><strong>Duration:</strong> {{duration_seconds}} seconds</p>
         <p><strong>Status:</strong> {{result_state}}</p>
         
         <h3 style="color: #d9534f;">Error Message:</h3>
         <p style="background-color: #f8f8f8; padding: 10px; border-left: 3px solid #d9534f;">
             {{state_message}}
         </p>
         
         <hr>
         <p style="font-size: 12px; color: #666;">Logged at: {{logged_at}}</p>
     </body>
     </html>
     {{/QUERY_RESULT_ROWS}}
     ```
   
   **CRITICAL**: The `{{#QUERY_RESULT_ROWS}}...{{/QUERY_RESULT_ROWS}}` wrapper is REQUIRED for variable substitution to work!

4. **Save the alert**

5. **Enable the alert** (toggle switch at the top)

## Step 6: Test the System End-to-End

1. **Trigger a job failure:**
   - Manually run your monitored job and let it fail
   - Or create a test job that intentionally fails

2. **Wait for Error Logger to run** (or run it manually)

3. **Verify error was logged:**
   ```sql
   SELECT * FROM email_alert.sample_email.job_error_logs
   ORDER BY logged_at DESC
   LIMIT 5;
   ```

4. **Wait for alert to trigger** (1-5 minutes based on your schedule)

5. **Check your email** (including spam/junk folder)

## Troubleshooting

### Alert Not Triggering

1. **Check alert is enabled**: Look for green toggle in alert UI
2. **Check refresh schedule**: Make sure it's set to 1-5 minutes
3. **Run query manually**: Verify it returns data
4. **Check alert history**: See evaluation results in alert page

### Email Not Arriving

1. **Check spam/junk folder**
2. **Verify email address** in destination settings
3. **Check template syntax**: Must have `{{#QUERY_RESULT_ROWS}}...{{/QUERY_RESULT_ROWS}}`
4. **Check alert history**: Verify email was actually sent

### Empty Email Fields

* **Most common issue**: Missing `{{#QUERY_RESULT_ROWS}}...{{/QUERY_RESULT_ROWS}}` wrapper
* **Fix**: Wrap your entire HTML template in this Mustache syntax

### Duplicate Errors

* The system automatically prevents duplicates
* If you see duplicates, check the duplicate prevention logic in notebook Cell 10

### No Errors Being Logged

1. **Check JOB_ID**: Verify it matches your monitored job
2. **Check permissions**: Ensure notebook can access job runs via SDK
3. **Check job has failures**: Verify the job actually failed
4. **Run Error Logger manually**: Check output for errors

## Monitoring Multiple Jobs

To monitor multiple jobs:

1. **Option A: Create separate Error Logger notebooks** (one per job)
   - Duplicate the notebook
   - Update JOB_ID in each
   - Create separate jobs for each logger

2. **Option B: Modify the notebook to accept parameters**
   ```python
   # Use Databricks widgets
   dbutils.widgets.text("job_id", "198373878629160")
   JOB_ID = int(dbutils.widgets.get("job_id"))
   ```
   - Create multiple jobs pointing to the same notebook
   - Pass different job_id as parameter

3. **All errors go to the same Delta table** and trigger the same alert

## Customization

### Change Time Window

In `sql/job_error_log_alert.sql`, adjust:
```sql
WHERE logged_at >= CURRENT_TIMESTAMP() - INTERVAL 1 hour  -- Change to 2 hours, 30 minutes, etc.
```

### Add More Fields to Email

1. Add columns to the SELECT in alert query
2. Add corresponding `{{column_name}}` variables in email template

### Filter Specific Jobs

Add filter to alert query:
```sql
WHERE logged_at >= CURRENT_TIMESTAMP() - INTERVAL 1 hour
  AND job_name = 'My Critical Job'
```

## Next Steps

- [ ] Set up monitoring for additional jobs
- [ ] Create Databricks dashboard to visualize error trends
- [ ] Set up Slack/Teams notifications (requires webhook integration)
- [ ] Add error trend analysis queries
- [ ] Schedule regular reviews of error logs

## Support

For issues and questions:
- Check the main [README.md](README.md)
- Review [Databricks SDK Documentation](https://docs.databricks.com/en/dev-tools/sdk-python.html)
- Review [Databricks SQL Alerts Documentation](https://docs.databricks.com/en/sql/user/alerts/index.html)
