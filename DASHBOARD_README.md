# Pi-hole DNS GeoIP Dashboard for OpenSearch

This directory contains dashboard definitions and deployment scripts for visualizing Pi-hole DNS queries with geographic information in OpenSearch Dashboards.

## Overview

The dashboard provides:
- **Interactive world map** showing geographic locations of queried DNS IPs
- **Data table** displaying recent DNS requests with domain, IP, country, and city
- **Top domains** visualization
- **Top countries** pie chart
- **Regional map** showing query distribution by country

## Files

### Dashboard Definitions

1. **`dashboard_pihole_geoip_simple.ndjson`** (Recommended)
   - Uses OpenSearch Dashboards built-in visualizations
   - Works out-of-the-box with standard OpenSearch setup
   - Includes:
     - Coordinate map (tile map) with DNS IP locations
     - Region map showing countries
     - Data table with recent requests
     - Top domains bar chart
     - Top countries pie chart

2. **`dashboard_pihole_geoip.ndjson`** (Advanced)
   - Uses Vega for custom map visualization
   - More customizable but requires Vega support
   - Shows tooltip with domain and IP on map markers

### Deployment Scripts

- **`dashboard_upload.yml`** - Ansible playbook to upload the dashboard to OpenSearch Dashboards

## Prerequisites

### Data Requirements

The dashboard expects data in **pihole-\*** indices with the following fields:

```json
{
  "@timestamp": "2025-11-29T10:30:00Z",
  "dns_queried_system": "example.com",
  "dns_queried_system_ip": "93.184.216.34",
  "dns_queried_system_geoip": {
    "location": "38.0,-97.0",
    "country_name": "United States",
    "country_iso_code": "US",
    "city_name": "Los Angeles",
    "continent_name": "North America",
    "region_name": "California"
  }
}
```

### Index Pattern

The dashboard automatically creates an index pattern for `pihole-*` indices.

### GeoIP Enrichment

Make sure GeoIP enrichment is configured either in:
- **OpenSearch ingest pipeline** (current setup in `opensearch_create.yml`)
- **Data Prepper** (see `data_prepper_config.yaml`)

## Deployment

### Step 1: Ensure OpenSearch and Dashboards are Running

```bash
# Check if pods are running
kubectl get pods -n opensearch-cluster

# You should see:
# - opensearch-xxx (Running)
# - opensearch-dashboards-xxx (Running)
```

### Step 2: Verify You Have Data

```bash
# Check if pihole indices exist with GeoIP data
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl -s "http://localhost:9200/pihole-*/_search?size=1&pretty" | \
  grep -A 5 "dns_queried_system_geoip"
```

### Step 3: Upload the Dashboard

```bash
# Upload the recommended dashboard
ansible-playbook dashboard_upload.yml

# Or specify a different dashboard file
ansible-playbook dashboard_upload.yml -e "dashboard_file=dashboard_pihole_geoip.ndjson"
```

### Step 4: Access the Dashboard

1. **Via Ingress:**
   ```
   http://<< Your OpenSearch Dashboard domain >>
   ```

2. **Via Port-Forward:**
   ```bash
   kubectl port-forward svc/opensearch-dashboards 5601:5601 -n opensearch-cluster
   ```
   Then visit: `http://localhost:5601`

3. **Navigate to Dashboard:**
   - Click on "Dashboard" in the left menu
   - Find "Pi-hole DNS GeoIP Dashboard"
   - Click to open

## Dashboard Layout

```
┌─────────────────────────────────┬─────────────────────────────────┐
│                                 │                                 │
│   Coordinate Map                │   Recent DNS Requests           │
│   (DNS IP Locations)            │   (Table)                       │
│                                 │                                 │
│   Shows dots on world map       │   Domain | IP | Country | City │
│   for each DNS query IP         │   --------------------------------│
│                                 │   example.com | 1.2.3.4 | US    │
│                                 │   ...                           │
│                                 │                                 │
└─────────────────────────────────┴─────────────────────────────────┘
┌──────────────────┬──────────────────┬──────────────────────────────┐
│ Region Map       │ Top Domains      │ Top Countries                │
│ (Countries)      │ (Bar Chart)      │ (Pie Chart)                  │
│                  │                  │                              │
│ Colored by       │ example.com ████ │  [USA: 45%]                  │
│ query count      │ google.com  ███  │  [UK: 20%]                   │
│                  │ github.com  ██   │  [DE: 15%]                   │
└──────────────────┴──────────────────┴──────────────────────────────┘
```

## Dashboard Features

### Time Range Selector
- Located in top-right corner
- Default: Last 24 hours
- Adjustable: Last 15 min, 1 hour, 7 days, 30 days, etc.

### Interactive Map
- **Zoom:** Scroll or use +/- buttons
- **Pan:** Click and drag
- **Tooltips:** Hover over markers to see details
- **Heatmap:** Colored by query density

### Data Table
- **Sortable:** Click column headers
- **Pagination:** Bottom controls
- **Shows:** Domain, IP address, country, city
- **Updates:** Automatically with time range

### Filters
You can add custom filters:
- Click "Add filter" button
- Filter by country, domain, IP, etc.
- Example: Show only queries to `.com` domains

## Customization

### Modify Dashboard

1. Open the dashboard in OpenSearch Dashboards
2. Click "Edit" button (top-right)
3. Modify visualizations:
   - Click gear icon on any panel
   - Adjust settings (colors, sizes, fields)
   - Click "Update"
4. Save changes

### Export Modified Dashboard

```bash
# Export from OpenSearch Dashboards UI:
# Management → Saved Objects → Select dashboard → Export
```

### Change Data Source

To use different indices, edit the `.ndjson` file:

```bash
# Replace all instances of "pihole-*"
sed -i 's/pihole-\*/your-index-\*/g' dashboard_pihole_geoip_simple.ndjson
```

## Troubleshooting

### Dashboard Upload Fails

**Error: "Port 5601 not accessible"**
```bash
# Solution: Check if Dashboards pod is running
kubectl get pods -n opensearch-cluster -l app=opensearch-dashboards

# If not running, redeploy:
ansible-playbook opensearch_create.yml
```

**Error: "No matching index pattern"**
```bash
# Solution: Create index pattern manually
# 1. Go to Dashboards UI
# 2. Management → Index Patterns
# 3. Create index pattern: pihole-*
# 4. Select time field: @timestamp
```

### No Data Showing

**Check 1: Verify indices exist**
```bash
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl -s "http://localhost:9200/_cat/indices/pihole-*?v"
```

**Check 2: Verify GeoIP data exists**
```bash
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl -s "http://localhost:9200/pihole-*/_search?size=1&pretty" | \
  grep "dns_queried_system_geoip"
```

**Check 3: Adjust time range**
- Dashboard defaults to "Last 24 hours"
- Try expanding to "Last 7 days" or "Last 30 days"

### Map Not Displaying

**Issue: Coordinate map shows blank**

**Solution 1:** Check if location data is properly formatted
```bash
# Location should be in format: "lat,lon" (string)
# Example: "37.751,-97.822"
```

**Solution 2:** Verify geo_point mapping
```bash
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl -s "http://localhost:9200/pihole-*/_mapping?pretty" | \
  grep -A 5 "location"

# Should show: "type": "geo_point"
```

### Visualizations Not Loading

**Error: "No results found"**

Possible causes:
1. **No data in time range** - Expand time range
2. **Missing fields** - Check if all required fields exist in your data
3. **GeoIP not enriched** - Verify GeoIP pipeline is working

## Data Flow

```
Pi-hole Logs
    ↓
Kafka Topic (pihole)
    ↓
Data Prepper (optional GeoIP enrichment)
    ↓
OpenSearch (with GeoIP ingest pipeline)
    ↓
pihole-* indices
    ↓
OpenSearch Dashboards
    ↓
Pi-hole DNS GeoIP Dashboard
```

## Performance Tips

### For Large Datasets

1. **Limit time range:** Use shorter periods (last 1 hour instead of 7 days)
2. **Reduce precision:** In coordinate map settings, lower precision level
3. **Add filters:** Filter by specific countries or domains
4. **Use aggregations:** Instead of showing all data points

### Optimize Queries

The dashboard uses aggregations which are generally fast, but for very large datasets:

1. **Use index rollover:** Create daily indices (`pihole-2025.11.29`)
2. **Enable index lifecycle management:** Archive old indices
3. **Add filters:** Pre-filter data in visualization queries

## Related Files

- `opensearch_create.yml` - Creates OpenSearch with GeoIP ingest pipeline
- `data_prepper_config.yaml` - Data Prepper configuration

## Advanced: Creating Custom Visualizations

### Add a New Visualization

1. **In Dashboards UI:**
   - Go to "Visualize"
   - Click "Create visualization"
   - Select visualization type
   - Choose index pattern: `pihole-*`

2. **Configure Metrics and Buckets:**
   - Metric: Count
   - Bucket: Terms aggregation on `dns_queried_system.keyword`

3. **Save and Add to Dashboard:**
   - Save visualization
   - Go to dashboard
   - Edit → Add → Select your visualization

### Example: Top DNS Query Sources

Create a visualization showing top DNS query sources:

```json
{
  "type": "pie",
  "aggs": [
    {
      "type": "count",
      "schema": "metric"
    },
    {
      "type": "terms",
      "field": "dns_query_from.keyword",
      "size": 10,
      "schema": "segment"
    }
  ]
}
```

## Security Considerations

### Current Setup
- **Security disabled** for simplicity (see `opensearch_create.yml`)
- Dashboard accessible without authentication

### Production Setup

For production environments:

1. **Enable OpenSearch Security:**
   ```yaml
   plugins.security.disabled: false
   ```

2. **Configure authentication:**
   - Set up users and roles
   - Use LDAP/SAML/OpenID

3. **Restrict dashboard access:**
   - Create read-only users for dashboards
   - Limit index access by role

4. **Use HTTPS:**
   - Enable TLS for OpenSearch and Dashboards
   - Update ingress with TLS certificates

## Support

For issues or questions:

1. Check the troubleshooting section above
2. Verify all prerequisites are met
3. Check OpenSearch and Dashboards logs:
   ```bash
   kubectl logs -n opensearch-cluster deployment/opensearch
   kubectl logs -n opensearch-cluster deployment/opensearch-dashboards
   ```

## Version Information

- **OpenSearch:** 3.3.2
- **OpenSearch Dashboards:** 3.3.0
- **Dashboard Format:** NDJSON (Newline Delimited JSON)
- **Compatible with:** OpenSearch 2.x and 3.x

## License

This dashboard configuration is provided as-is for use with Pi-hole DNS logging and OpenSearch.

