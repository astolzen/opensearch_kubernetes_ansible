# Pi-hole DNS GeoIP Dashboard - Quick Start Guide

## TL;DR - Quick Deployment

```bash
# 1. Verify prerequisites
ansible-playbook dashboard_verify.yml

# 2. Upload dashboard
ansible-playbook dashboard_upload.yml

# 3. Access dashboard
# Visit: http://<< Your OpenSearch Dashboard domain >>
# Go to: Dashboard → Pi-hole DNS GeoIP Dashboard
```

## What You Get

### Visual Layout

**Left Side:** Interactive world map showing where DNS queries are going
- Red dots = DNS query IP locations
- Hover to see domain, IP, country, city
- Zoom and pan to explore

**Right Side:** Table of recent DNS requests
- Domain names queried
- IP addresses resolved
- Geographic location (country, city)
- Sortable and filterable

**Bottom Row:** Analytics
- Regional map by country
- Top 10 most queried domains
- Top 10 countries by query count

## Files Created

| File | Purpose |
|------|---------|
| `dashboard_pihole_geoip_simple.ndjson` | **Main dashboard** (recommended) |
| `dashboard_pihole_geoip.ndjson` | Advanced dashboard with Vega |
| `dashboard_upload.yml` | **Deployment playbook** |
| `dashboard_verify.yml` | **Verification playbook** |
| `DASHBOARD_README.md` | Full documentation |
| `DASHBOARD_QUICKSTART.md` | This file |

## Prerequisites Checklist

- [ ] OpenSearch running in `opensearch-cluster` namespace
- [ ] OpenSearch Dashboards running
- [ ] Pi-hole data in `pihole-*` indices
- [ ] GeoIP enrichment enabled (OpenSearch pipeline OR Data Prepper)
- [ ] Data includes `dns_queried_system_geoip.location` field

## Step-by-Step Deployment

### Step 1: Verify Everything is Ready

```bash
ansible-playbook dashboard_verify.yml
```

**Expected output:**
```
✓ OpenSearch is running
✓ OpenSearch Dashboards is running
✓ pihole-* indices exist
✓ GeoIP enriched data exists
✓ GeoIP ingest pipeline configured
✓ geo_point mapping configured
```

**If you see warnings:**
- Missing GeoIP data? Run: `ansible-playbook opensearch_create.yml`
- No pihole indices? Check Data Prepper is running and sending data

### Step 2: Upload Dashboard

```bash
ansible-playbook dashboard_upload.yml
```

**Expected output:**
```
Dashboard uploaded successfully!
```

**Alternative:** Use custom dashboard file
```bash
ansible-playbook dashboard_upload.yml -e "dashboard_file=dashboard_pihole_geoip.ndjson"
```

### Step 3: Access and View

**Option A: Via Ingress**
```
http://<< Your OpenSearch Dashboard domain >>
```

**Option B: Via Port-Forward**
```bash
kubectl port-forward svc/opensearch-dashboards 5601:5601 -n opensearch-cluster
# Visit: http://localhost:5601
```

**Navigate to Dashboard:**
1. Click "Dashboard" icon (left sidebar)
2. Find "Pi-hole DNS GeoIP Dashboard"
3. Click to open

## Dashboard Usage

### Change Time Range

**Top-right corner:** Click time selector
- Quick ranges: Last 15m, 1h, 24h, 7d, 30d
- Custom: Select specific start/end dates

**Default:** Last 24 hours

### Add Filters

**Filter bar (top):** Click "+ Add filter"

**Examples:**
```
dns_queried_system.keyword is "google.com"
dns_queried_system_geoip.country_name.keyword is "United States"
dns_queried_system_ip.keyword is not "127.0.0.1"
```

### Interact with Map

- **Zoom in/out:** Mouse wheel or +/- buttons
- **Pan:** Click and drag
- **Tooltip:** Hover over dots to see details
- **Reset:** Click home icon to reset view

### Export Data

**From table:**
1. Click "Inspect" button
2. Choose "Download CSV" or "Copy to clipboard"

## Common Use Cases

### 1. Find DNS Queries to Specific Country

```
1. Add filter: dns_queried_system_geoip.country_name.keyword is "China"
2. View map to see which domains resolve to IPs in China
3. Check table for full list
```

### 2. Track Most Queried Domains

```
1. Look at "Top Queried Domains" bar chart (bottom)
2. Click on a bar to filter by that domain
3. See geographic distribution on map
```

### 3. Identify Unusual Geographic Patterns

```
1. Zoom out on map to see global view
2. Look for unexpected geographic concentrations
3. Click on area to filter and investigate
```

### 4. Monitor Real-Time DNS Activity

```
1. Set time range to "Last 15 minutes"
2. Enable auto-refresh (top-right)
3. Watch map update as new queries arrive
```

## Troubleshooting Quick Reference

### "No results found"

**Fix:** Expand time range to "Last 7 days" or "Last 30 days"

### Map is blank

**Check 1:** Verify GeoIP data exists
```bash
ansible-playbook dashboard_verify.yml
```

**Check 2:** Look at verification output for warnings

### Dashboard upload fails

**Fix:** Ensure Dashboards pod is running
```bash
kubectl get pods -n opensearch-cluster -l app=opensearch-dashboards
```

### Visualizations show error

**Common cause:** Index pattern not found

**Fix:** Dashboard should auto-create `pihole-*` pattern, but if not:
1. Go to Management → Index Patterns
2. Create pattern: `pihole-*`
3. Time field: `@timestamp`
4. Refresh dashboard

## Data Requirements

### Minimum Required Fields

```json
{
  "@timestamp": "timestamp",
  "dns_queried_system": "domain.com",
  "dns_queried_system_ip": "1.2.3.4",
  "dns_queried_system_geoip": {
    "location": "lat,lon"
  }
}
```

### Optimal Fields (for full features)

```json
{
  "@timestamp": "2025-11-29T10:30:00Z",
  "dns_queried_system": "example.com",
  "dns_queried_system_ip": "93.184.216.34",
  "dns_query_from": "192.168.1.100",
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

## Performance Tips

### For Faster Loading

1. **Shorter time ranges:** Use "Last 1 hour" instead of "Last 30 days"
2. **Add filters:** Pre-filter data before viewing
3. **Lower map precision:** Edit visualization settings

### For Large Datasets

1. **Use aggregations:** Dashboard already uses these by default
2. **Archive old data:** Set up index lifecycle management
3. **Sample data:** Show representative subset

## Customization Ideas

### Add Your Own Visualizations

**Example: DNS Query Timeline**
1. Edit dashboard → Add
2. Create new visualization
3. Type: Line chart
4. X-axis: Date histogram on @timestamp
5. Y-axis: Count
6. Save and add to dashboard

**Example: Query Type Distribution**
1. Create pie chart
2. Slice: Terms on dns_query_type (if you have this field)
3. Shows: A vs AAAA vs CNAME distribution

### Modify Colors

1. Edit dashboard
2. Click gear icon on visualization
3. Options → Color schema
4. Choose different color palette

### Change Layout

1. Edit dashboard
2. Drag panels to resize/reposition
3. Click and drag from panel header
4. Save when done

## Integration with GeoIP Setup

### Current Setup (OpenSearch Ingest Pipeline)

**Dashboard works with:**
- GeoIP enrichment in OpenSearch (current setup)
- Configured via `opensearch_create.yml`
- Enrichment happens at index time

### Alternative Setup (Data Prepper)

**Dashboard also works with:**
- GeoIP enrichment in Data Prepper
- Configured via `data_prepper_config.yaml`
- Enrichment happens before indexing

**No changes needed to dashboard** - it works with both!

## Next Steps

After dashboard is working:

1. **Set up alerts:** Create monitors for unusual patterns
2. **Share dashboard:** Export and share with team
3. **Create reports:** Schedule automated PDF reports
4. **Add more visualizations:** Expand with custom charts

## Support Commands

```bash
# Check if dashboard exists
kubectl exec -n opensearch-cluster deployment/opensearch-dashboards -- \
  curl -s "http://localhost:5601/api/saved_objects/_find?type=dashboard" | \
  grep "pihole"

# View dashboard logs
kubectl logs -n opensearch-cluster deployment/opensearch-dashboards

# Check OpenSearch data
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl -s "http://localhost:9200/pihole-*/_search?size=1&pretty"
```

## Related Documentation

- **Full Guide:** `DASHBOARD_README.md` - Complete documentation
- **Data Prepper:** `data_prepper_config.yaml` - Pipeline configuration

---

**Dashboard Version:** 1.0  
**Compatible with:** OpenSearch 3.3.x, OpenSearch Dashboards 3.3.x  
**Last Updated:** 2025-11-29

