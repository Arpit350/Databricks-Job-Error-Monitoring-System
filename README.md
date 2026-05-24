

# Databricks Job Error Monitoring System

A comprehensive error monitoring and alerting system for Databricks jobs that captures detailed error logs, stores them in Delta tables, and sends email notifications with full error details.

## 🎯 Features

- **Automated Error Capture**: Uses Databricks SDK to fetch detailed job run failures
- **Comprehensive Error Details**: Captures workspace_id, job_id, run_id, timestamps, duration, and full error messages
- **Delta Table Storage**: Stores error logs in Unity Catalog Delta table with duplicate prevention
- **Email Alerts**: SQL-based alerts with formatted HTML emails containing all error details
- **Task-Level Error Extraction**: Fetches actual notebook execution errors, not just generic failure messages
- **Duplicate Prevention**: Smart filtering to avoid logging the same error multiple times

## 📊 System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Databricks Jobs (Monitored)                  │
│                  (e.g., Product Validation Job)                 │
└────────────────────────────┬────────────────────────────────────┘
                             │ Fails
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│         Error Logger Monitor Job (Scheduled/Manual)             │
│   - Uses Databricks SDK to fetch run details                   │
│   - Extracts detailed error messages with get_run_output()     │
│   - Checks for duplicates                                       │
│   - Logs NEW errors to Delta table                             │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│        Delta Table: job_error_logs (Unity Catalog)              │
│   workspace_id, job_id, run_id, timestamps, error details      │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│              SQL Alert (Triggers on New Errors)                 │
│   - Queries error logs for recent failures (last 1 hour)       │
│   - Triggers when rows > 0                                      │
│   - Sends formatted HTML email with error details               │
└─────────────────────────────────────────────────────────────────┘
                             ↓
                    📧 Email Notification
```

## 🚀 Quick Start

### Prerequisites

- Databricks Workspace with Unity Catalog enabled
- Python 3.x environment
- `databricks-sdk` package
- Unity Catalog schema for storing error logs
- SQL warehouse for running alerts

### Installation

1. **Clone this repository**
   ```bash
   git clone https://github.com/yourusername/databricks-job-error-monitor.git
   cd databricks-job-error-monitor
   ```

2. **Import the notebook to your Databricks workspace**
   - Upload `notebooks/Error_Logger_Job_Monitor.py` to your workspace
   - Or import it via Databricks CLI/API

3. **Create the Delta table schema**
   ```sql
   CREATE SCHEMA IF NOT EXISTS email_alert.sample_email;
   ```

4. **Import and configure the SQL alert**
   - Import `sql/job_error_log_alert.sql` as a Databricks SQL query
   - Configure the alert settings (see Configuration section)

### Configuration

#### 1. Update the Error Logger Notebook

Edit the notebook and update these parameters:

```python
# Job ID to monitor (find this in your job's URL)
JOB_ID = 198373878629160  # Replace with your job ID

# Your workspace ID (find in workspace URL)
WORKSPACE_ID = "560696151014778"  # Replace with your workspace ID

# Delta table path (customize if needed)
table_name = "email_alert.sample_email.job_error_logs"
```

#### 2. Create a Job for the Error Logger

Create a new Databricks job:
- **Task Type**: Notebook
- **Notebook Path**: Path to your uploaded Error Logger notebook
- **Cluster**: Use your preferred cluster or job compute
- **Schedule**: Every 15-60 minutes (recommended)

#### 3. Configure the SQL Alert

In Databricks SQL:

1. Create a new query with the content from `sql/job_error_log_alert.sql`
2. Update the table name if you changed it:
   ```sql
   FROM email_alert.sample_email.job_error_logs  -- Update this line
   ```
3. Create an alert from the query:
   - **Trigger Condition**: `Value` > 0 or `Rows` > 0
   - **Refresh Schedule**: Every 1-5 minutes for immediate notifications
   - **Destination**: Email
   - **Email Template**: Use content from `config/email_template.html`

## 📁 Repository Structure

```
databricks-job-error-monitor/
├── README.md                          # This file
├── notebooks/
│   └── Error_Logger_Job_Monitor.py   # Main error logging notebook
├── sql/
│   ├── job_error_log_alert.sql        # SQL query for alerts
│   └── create_schema.sql              # Schema creation script
├── config/
│   ├── email_template.html            # HTML email template
│   └── example_config.json            # Example configuration
└── .gitignore
```

## 🔧 How It Works

### 1. Error Logger Notebook

The notebook uses the Databricks SDK to:

- Fetch job configuration and run history
- Filter for failed runs (FAILED, TIMEDOUT states)
- Extract detailed error messages using `get_run_output()`
- Check for duplicate run_ids in the Delta table
- Log only NEW errors to prevent duplicates
- Store comprehensive error information including:
  - Workspace and job identifiers
  - Timestamps and duration
  - Full error messages from notebook execution
  - Task-level errors

### 2. Delta Table Schema

```sql
CREATE TABLE email_alert.sample_email.job_error_logs (
  workspace_id STRING,
  job_id STRING,
  job_name STRING,
  run_id STRING,
  run_name STRING,
  start_time STRING,
  end_time STRING,
  duration_seconds DOUBLE,
  result_state STRING,
  life_cycle_state STRING,
  state_message STRING,
  task_errors STRING,  -- JSON string of task-level errors
  logged_at STRING
);
```

### 3. SQL Alert Query

Queries the error logs table for recent failures:

```sql
SELECT 
  workspace_id, job_id, job_name, run_id, run_name,
  start_time, end_time, duration_seconds,
  result_state, state_message, logged_at
FROM email_alert.sample_email.job_error_logs
WHERE logged_at >= CURRENT_TIMESTAMP() - INTERVAL 1 hour
ORDER BY logged_at DESC
LIMIT 1
```

### 4. Email Template

Uses Mustache template syntax with Databricks SQL Alerts v2:

```html
{{#QUERY_RESULT_ROWS}}
<!-- Your HTML template with {{column_name}} variables -->
{{/QUERY_RESULT_ROWS}}
```

The key is wrapping the template in `{{#QUERY_RESULT_ROWS}}...{{/QUERY_RESULT_ROWS}}` to iterate over query results.

## 📧 Email Notification Format

Email includes:
- **Job Information**: Job name, job ID, run ID, workspace ID
- **Execution Details**: Status, start/end times, duration, logged timestamp
- **Error Details**: Full error message in formatted box

## 🛠️ Troubleshooting

### Emails Not Arriving

1. **Check Alert is Enabled**: Verify the alert toggle is ON
2. **Check Alert Schedule**: Ensure refresh interval is frequent enough (1-5 minutes)
3. **Verify Email Destination**: Confirm email address is correct
4. **Check Template Syntax**: Ensure template is wrapped in `{{#QUERY_RESULT_ROWS}}...{{/QUERY_RESULT_ROWS}}`
5. **Check Query Results**: Run the query manually to verify it returns data
6. **Check Time Window**: Ensure errors are within the query time window (1 hour)

### No Errors Being Logged

1. **Check Job ID**: Verify `JOB_ID` in notebook matches your monitored job
2. **Check Permissions**: Ensure the notebook has access to read job runs
3. **Check Error Logger Schedule**: Verify the Error Logger job is running
4. **Check Table Permissions**: Ensure write permissions to the Delta table

### Duplicate Errors

1. The system automatically prevents duplicates by checking existing run_ids
2. If you see duplicates, verify the duplicate prevention logic in Cell 10

## 📈 Monitoring Best Practices

1. **Schedule the Error Logger**: Run every 15-60 minutes
2. **Set Alert Frequency**: Check every 1-5 minutes for immediate notifications
3. **Adjust Time Window**: Increase `INTERVAL 1 hour` if needed for your use case
4. **Monitor Alert History**: Check alert evaluation history in Databricks SQL
5. **Review Error Logs Regularly**: Query the Delta table for trends

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## 📝 License

This project is licensed under the MIT License.

## 📞 Support

For issues and questions:
- Open an issue in this repository
- Check the Troubleshooting section above
- Review Databricks documentation for SDK and SQL Alerts

## 🔗 Useful Links

- [Databricks SDK Documentation](https://docs.databricks.com/en/dev-tools/sdk-python.html)
- [Databricks SQL Alerts](https://docs.databricks.com/en/sql/user/alerts/index.html)
- [Unity Catalog Documentation](https://docs.databricks.com/en/data-governance/unity-catalog/index.html)

---

**Built for Databricks** | **Powered by Unity Catalog & Delta Lake**
