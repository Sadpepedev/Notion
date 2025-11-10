# Notion Database Auto-Population Setup Guide

## Prerequisites

1. **Python 3.7+** installed on your system
1. **Notion account** with access to the database you want to populate
1. **Internal database** with the scraped social media data

## Setup Instructions

### Step 1: Install Required Packages

```bash
pip install requests python-dotenv

# If using specific database connections:
pip install psycopg2-binary  # For PostgreSQL
pip install mysql-connector-python  # For MySQL
pip install pymongo  # For MongoDB
```

### Step 2: Create a Notion Integration

1. Go to https://www.notion.so/my-integrations
1. Click “New integration”
1. Give it a name (e.g., “FUD Tracker Sync”)
1. Select the workspace
1. Copy the “Internal Integration Token”

### Step 3: Get Your Database ID

1. Open your Notion database in a browser
1. Copy the URL - it looks like:
   `https://www.notion.so/myworkspace/a8b6c3d4e5f67890a1b2c3d4e5f67890?v=...`
1. The database ID is the string after the workspace name and before the `?v=`
   (in this example: `a8b6c3d4e5f67890a1b2c3d4e5f67890`)

### Step 4: Share Database with Integration

1. In your Notion database, click the “…” menu in the top right
1. Click “Add connections”
1. Search for your integration name
1. Click to add it

### Step 5: Configure Environment Variables

Create a `.env` file in the same directory as the script:

```env
NOTION_TOKEN=secret_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
NOTION_DATABASE_ID=a8b6c3d4e5f67890a1b2c3d4e5f67890

# Add your internal database credentials
DB_HOST=localhost
DB_USER=your_user
DB_PASSWORD=your_password
DB_NAME=your_database
```

### Step 6: Modify the Database Connection

Update the `fetch_from_internal_database()` function in the script to connect to your actual database.

Example for PostgreSQL:

```python
import psycopg2
import os

def fetch_from_internal_database():
    conn = psycopg2.connect(
        host=os.getenv("DB_HOST"),
        database=os.getenv("DB_NAME"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD")
    )
    cursor = conn.cursor()
    
    # Adjust this query to match your table structure
    cursor.execute("""
        SELECT 
            uid,
            name,
            status,
            reviewer_name,
            review_date,
            next_follow_up,
            date_added,
            platform,
            socials
        FROM your_table_name
    """)
    
    columns = [desc[0] for desc in cursor.description]
    rows = cursor.fetchall()
    
    # Convert to list of dictionaries
    records = []
    for row in rows:
        records.append(dict(zip(columns, row)))
    
    conn.close()
    return records
```

### Step 7: Run the Script

```bash
python notion_sync_script.py
```

## Automation Options

### Option 1: Cron Job (Linux/Mac)

Add to crontab to run every hour:

```bash
0 * * * * /usr/bin/python3 /path/to/notion_sync_script.py
```

### Option 2: Task Scheduler (Windows)

1. Open Task Scheduler
1. Create Basic Task
1. Set trigger (e.g., daily, hourly)
1. Set action to run Python script

### Option 3: Python Scheduler

Create a scheduler script:

```python
import schedule
import time
from notion_sync_script import main

# Run every 30 minutes
schedule.every(30).minutes.do(main)

while True:
    schedule.run_pending()
    time.sleep(1)
```

## Property Type Mapping

Your Notion database properties map to these API types:

|Your Column   |Notion Property Type|API Format |
|--------------|--------------------|-----------|
|Name          |Title               |`title`    |
|UID           |Text                |`rich_text`|
|Status        |Select              |`select`   |
|Reviewer Name |Text                |`rich_text`|
|Review Date   |Date                |`date`     |
|Next Follow Up|Date                |`date`     |
|Date Added    |Date                |`date`     |
|Platform      |Select              |`select`   |
|Socials       |Text                |`rich_text`|

## Troubleshooting

### Common Issues

1. **401 Unauthorized**: Check your integration token
1. **404 Not Found**: Verify database ID and that integration has access
1. **400 Bad Request**: Check property names match exactly (case-sensitive)
1. **Rate Limiting**: Add delays between requests if syncing many records

### Debug Mode

Add this to see full API responses:

```python
import logging
logging.basicConfig(level=logging.DEBUG)
```

## Advanced Features

### 1. Batch Processing

For large datasets, process in batches:

```python
def sync_in_batches(records, batch_size=10):
    for i in range(0, len(records), batch_size):
        batch = records[i:i+batch_size]
        sync_client.sync_from_database(batch)
        time.sleep(1)  # Rate limiting
```

### 2. Error Recovery

Implement retry logic:

```python
import time
from typing import Callable

def retry_on_error(func: Callable, max_retries=3):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### 3. Incremental Sync

Only sync new/modified records:

```python
def get_records_since(last_sync_date):
    # Query only records modified after last_sync_date
    query = f"SELECT * FROM table WHERE updated_at > '{last_sync_date}'"
    # ... rest of database fetch logic
```

## Security Best Practices

1. Never commit `.env` file to version control
1. Use read-only database credentials when possible
1. Implement logging for audit trails
1. Use HTTPS for all API calls (default with Notion API)
1. Rotate integration tokens periodically

## Support Resources

- [Notion API Documentation](https://developers.notion.com/)
- [API Reference](https://developers.notion.com/reference/intro)
- [Property Types Guide](https://developers.notion.com/reference/property-value-object)
- [Rate Limits](https://developers.notion.com/reference/request-limits)
