## Handling the Missing Year

Since the netlogon log format doesn't include the year, I will consider a few options:

### Option 1: Ingest Pipeline with Script Processor (Recommended)

Use a script processor to prepend the current year before parsing the date:

```
PUT _ingest/pipeline/netlogon-parse
{
  "description": "Parse netlogon debug logs with missing year",
  "processors": [
    {
      "set": {
        "field": "log.original",
        "copy_from": "message",
        "description": "Preserve original message"
      }
    },
    {
      "dissect": {
        "field": "message",
        "pattern": "%{timestamp} %{log_content}"
      }
    },
    {
      "script": {
        "description": "Add year to timestamp with rollover handling",
        "lang": "painless",
        "source": """
          int currentYear = ZonedDateTime.ofInstant(
            Instant.ofEpochMilli(System.currentTimeMillis()), 
            ZoneId.of('UTC')
          ).getYear();
          
          int currentMonth = ZonedDateTime.ofInstant(
            Instant.ofEpochMilli(System.currentTimeMillis()), 
            ZoneId.of('UTC')
          ).getMonthValue();
          
          // Extract log month from timestamp (first 2 chars)
          int logMonth = Integer.parseInt(ctx['timestamp'].substring(0, 2));
          
          // If current month is Jan (1) but log shows Dec (12), use previous year
          if (currentMonth == 1 && logMonth == 12) {
            currentYear = currentYear - 1;
          }
          
          ctx['timestamp_with_year'] = currentYear + "/" + ctx['timestamp'];
        """
      }
    },
    {
      "date": {
        "field": "timestamp_with_year",
        "formats": ["yyyy/MM/dd HH:mm:ss"],
        "target_field": "@timestamp"
      }
    },
    {
      "set": {
        "field": "message",
        "copy_from": "log_content",
        "description": "Replace message with parsed log content"
      }
    },
    {
      "remove": {
        "field": ["timestamp", "timestamp_with_year", "log_content"],
        "ignore_missing": true
      }
    }
  ]
}
```

### Option 2: Handle Year Rollover Edge Case

If you're ingesting logs around December/January, you might want smarter logic:

```
{
  "script": {
    "lang": "painless",
    "source": """
      int currentYear = ZonedDateTime.ofInstant(
        Instant.ofEpochMilli(System.currentTimeMillis()), 
        ZoneId.of('UTC')
      ).getYear();
      
      int currentMonth = ZonedDateTime.ofInstant(
        Instant.ofEpochMilli(System.currentTimeMillis()), 
        ZoneId.of('UTC')
      ).getMonthValue();
      
      // Extract log month from timestamp (first 2 chars)
      int logMonth = Integer.parseInt(ctx['timestamp'].substring(0, 2));
      
      // If current month is Jan (1) but log shows Dec (12), use previous year
      if (currentMonth == 1 && logMonth == 12) {
        currentYear = currentYear - 1;
      }
      
      ctx['timestamp_with_year'] = currentYear + "/" + ctx['timestamp'];
    """
  }
}
```

---

## Filebeat Dissect vs Elasticsearch Ingest Pipeline

**My recommendation: Use Elasticsearch Ingest Pipeline** for this use case.

Here's why:

| Factor | Filebeat Processors | Elasticsearch Ingest Pipeline |
|--------|---------------------|-------------------------------|
| **Complex logic** | Limited (dissect is pattern-based only) | Full scripting support (Painless) |
| **Date manipulation** | No native way to inject year | Script processor handles this easily |
| **Centralized management** | Need to redeploy Filebeat configs | Update pipeline once, applies to all |
| **Debugging** | Harder to test/debug | Easy to test with `_simulate` API |
| **Performance** | Offloads work from ES | Adds load to ingest nodes |
| **Flexibility** | Dissect fails on format variations | Grok + conditionals handle edge cases |

### When Filebeat Processing Makes Sense

Use Filebeat processors when you're doing simple field additions, dropping events, or basic transformations that don't require scripting.

### Recommended Architecture for Your Case

```
# filebeat.yml - keep it simple
filebeat.inputs:
  - type: filestream
    id: netlogon-logs
    paths:
      - /path/to/netlogon.log
    fields:
      log_type: netlogon
    fields_under_root: true

output.elasticsearch:
  hosts: ["your-es-host:9200"]
  pipeline: "netlogon-parse"  # Let ES handle the parsing
```

Then let the ingest pipeline handle all the heavy lifting—date manipulation, field extraction, and any conditional logic for log variations.
