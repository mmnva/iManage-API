# iManage-API
Universal API info
# iManage Document Sync Solution

A comprehensive Python solution for downloading documents from iManage Work, filtering by custom fields, and storing metadata in PostgreSQL.

## Features

- **OAuth2 Authentication**: Secure authentication with iManage Work
- **Custom Field Filtering**: Dynamic filtering based on custom fields that can change over time
- **PostgreSQL Storage**: Efficient storage using JSONB for flexible custom fields
- **Batch Processing**: Handle large document sets with configurable batch sizes
- **Download Management**: Optional file downloads with local storage
- **Scheduling**: Automated sync operations on schedule
- **Reporting**: Generate reports on synced documents and custom field usage

## Why PostgreSQL?

After comparing Apache Cassandra, DuckDB, and PostgreSQL:

|Aspect            |PostgreSQL (Chosen)             |DuckDB                 |Cassandra              |
|------------------|--------------------------------|-----------------------|-----------------------|
|**Custom Fields** |JSONB perfect for dynamic fields|Good JSON support      |Limited flexibility    |
|**Querying**      |Full SQL with JSON operators    |Full SQL               |Limited CQL            |
|**Setup**         |Moderate complexity             |Very simple            |Complex cluster setup  |
|**Python Support**|Excellent (psycopg2)            |Good                   |Good                   |
|**Use Case Fit**  |✅ Best for this use case        |Good for analytics only|Overkill for this scale|

PostgreSQL with JSONB provides the perfect balance of:

- Flexibility for changing custom fields
- Powerful querying capabilities
- Proven reliability
- Easy integration with Python

## Installation

### 1. Prerequisites

- Python 3.8+
- PostgreSQL 12+
- iManage Work account with API access

### 2. Install Dependencies

```bash
pip install -r requirements.txt
```

### 3. Setup PostgreSQL

#### Option A: Using Docker (Recommended for Development)

```bash
docker-compose up -d
```

#### Option B: Manual Setup

```bash
# Create database
createdb imanage_docs

# Run setup script
psql -d imanage_docs -f postgres_setup.sql
```

### 4. Configure

1. Copy the example configuration:

```bash
cp config.yaml.example config.yaml
```

1. Edit `config.yaml` with your iManage credentials:

```yaml
imanage:
  server: "your.imanage.server.com"
  username: "your_username@domain.com"
  password: "your_password"
  client_id: "your_client_id"
  client_secret: "your_client_secret"
```

1. Or use environment variables:

```bash
export IMANAGE_SERVER=your.imanage.server.com
export IMANAGE_USERNAME=your_username@domain.com
export IMANAGE_PASSWORD=your_password
export IMANAGE_CLIENT_ID=your_client_id
export IMANAGE_CLIENT_SECRET=your_client_secret
```

## Usage

### Basic Sync

```python
from imanage_downloader import iManageConfig, DatabaseConfig, iManageClient, DocumentDatabase, DocumentSyncManager

# Configure
imanage_config = iManageConfig(
    server="your.server.com",
    username="username",
    password="password",
    client_id="client_id",
    client_secret="client_secret",
    library_id="Active"
)

db_config = DatabaseConfig()

# Initialize
client = iManageClient(imanage_config)
database = DocumentDatabase(db_config)
sync_manager = DocumentSyncManager(client, database)

# Authenticate and connect
client.authenticate()
database.connect()
database.create_tables()

# Sync with custom field filters
filters = {
    "custom1": "Legal",
    "custom2": "Contract",
    "type": "wordx"
}

sync_manager.sync_documents(
    filters=filters,
    download_files=True
)
```

### Command Line Usage

```bash
# Basic sync with default configuration
python enhanced_sync_script.py --sync

# Sync with custom field filters
python enhanced_sync_script.py --sync --custom-field custom1 Legal --custom-field custom2 Contract

# Sync and download files
python enhanced_sync_script.py --sync --download

# Sync recent changes (last 24 hours)
python enhanced_sync_script.py --recent 24

# Generate report
python enhanced_sync_script.py --report

# Run scheduled sync (every hour)
python enhanced_sync_script.py --schedule
```

### Querying the Database

```sql
-- Find documents by custom fields
SELECT * FROM documents 
WHERE custom_fields @> '{"custom1": "Legal", "custom2": "Contract"}'::jsonb;

-- Get custom field distribution
SELECT 
    custom_fields->>'custom1' as custom1_value,
    COUNT(*) as document_count
FROM documents
WHERE custom_fields ? 'custom1'
GROUP BY custom_fields->>'custom1';

-- Find recently modified documents with specific custom fields
SELECT id, name, custom_fields, edit_date 
FROM documents 
WHERE custom_fields @> '{"custom1": "Legal"}'::jsonb 
  AND edit_date > CURRENT_TIMESTAMP - INTERVAL '7 days';

-- Use the built-in function
SELECT * FROM search_custom_fields('{"custom1": "Legal"}'::jsonb);
```

## Custom Fields Handling

The solution automatically handles dynamic custom fields:

1. **Storage**: All custom fields are stored in a JSONB column
1. **Indexing**: GIN index enables efficient queries
1. **Flexibility**: New custom fields are automatically captured
1. **Querying**: Use PostgreSQL’s JSON operators for powerful searches

Example custom field structure in database:

```json
{
  "custom1": "Legal",
  "custom2": "Contract",
  "custom3": "2024",
  "custom1_description": "Department",
  "custom2_description": "Document Type"
}
```

## API Rate Limiting

The solution includes built-in rate limiting considerations:

- Configurable batch sizes
- Retry logic with exponential backoff
- Token refresh on expiration

## Error Handling

- Automatic retry for failed downloads
- Transaction-based database updates
- Comprehensive logging
- Sync history tracking

## Performance Optimization

1. **Batch Processing**: Process documents in configurable batches
1. **Selective Downloads**: Option to sync metadata only
1. **Incremental Sync**: Sync only recently changed documents
1. **Database Indexes**: Optimized for common query patterns

## Database Schema

### Main Tables

- `documents`: Stores document metadata with JSONB custom fields
- `sync_history`: Tracks sync operations
- `custom_field_definitions`: Optional tracking of field usage

### Views

- `recent_documents`: Quick access to recent documents
- `custom_field_stats`: Statistics on custom field usage

## Security Considerations

1. **Credentials**: Never commit credentials to version control
1. **Environment Variables**: Use for production deployments
1. **SSL/TLS**: Ensure secure connections to iManage
1. **Database Security**: Use strong passwords and limit access

## Troubleshooting

### Common Issues

1. **Authentication Failed**
- Verify credentials
- Check user type (must be Virtual)
- Confirm OAuth2 password grant is enabled
1. **Connection Timeout**
- Check network connectivity
- Verify server URL
- Check SSL certificate settings
1. **Custom Fields Not Found**
- Ensure fields are included in profile_fields
- Check field names match exactly
- Verify user has access to fields

### Logging

Check `imanage_sync.log` for detailed information:

```bash
tail -f imanage_sync.log
```

## Advanced Features

### Scheduled Syncs

Use cron for scheduled syncs:

```bash
# Sync every hour
0 * * * * /usr/bin/python3 /path/to/enhanced_sync_script.py --recent 1

# Daily full sync at 2 AM
0 2 * * * /usr/bin/python3 /path/to/enhanced_sync_script.py --sync
```

### Custom Field Monitoring

Track custom field changes over time:

```python
# Get custom field evolution
cursor.execute("""
    SELECT 
        DATE(last_updated) as date,
        jsonb_object_keys(custom_fields) as field,
        COUNT(DISTINCT custom_fields->>jsonb_object_keys(custom_fields)) as unique_values
    FROM documents
    GROUP BY DATE(last_updated), jsonb_object_keys(custom_fields)
    ORDER BY date DESC, field;
""")
```

## Contributing

1. Fork the repository
1. Create a feature branch
1. Add tests for new functionality
1. Submit a pull request

## License

MIT License - See LICENSE file for details

## Support

For issues or questions:

1. Check the troubleshooting section
1. Review logs for error messages
1. Consult iManage REST API documentation
1. Contact your iManage administrator for API access issues
