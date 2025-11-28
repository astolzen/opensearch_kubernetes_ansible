# OpenSearch DNS Analytics Stack

Complete DNS log analytics solution using OpenSearch, Data Prepper, Kafka, and Filebeat for real-time DNS query analysis and geolocation visualization.

## Overview

This project provides automated deployment of a comprehensive DNS analytics platform that:
- Collects DNS logs from Pi-hole via Filebeat
- Streams data through Kafka for scalability
- Processes logs with OpenSearch Data Prepper
- Enriches DNS resolution data with GeoIP information
- Visualizes DNS traffic on interactive maps in OpenSearch Dashboards

## Architecture

```
┌─────────────┐      ┌─────────┐      ┌──────────────┐      ┌────────────┐      ┌──────────────┐
│  Filebeat   │─────▶│  Kafka  │─────▶│ Data Prepper │─────▶│ OpenSearch │─────▶│  Dashboards  │
│  (Pi-hole)  │      │ Topics  │      │  Pipelines   │      │   Indices  │      │     Maps     │
└─────────────┘      └─────────┘      └──────────────┘      └────────────┘      └──────────────┘
  - journals            - journals       - JSON parsing        - journals-*        - DNS queries
  - pihole logs         - pihole         - Grok patterns       - pihole-*          - Blocked domains
                                        - GeoIP enrichment                         - Geographic viz
```

## Components

### 1. OpenSearch 3.3.2
- Single-node deployment
- No security (for lab/internal use)
- Persistent storage (25Gi PVC)
- GeoIP ingest pipeline for DNS resolution data

### 2. OpenSearch Dashboards 3.3.0
- Web UI for visualization
- Ingress configured: `<< Your OpenSearch Dashboard domain >>`
- Index patterns for journals and pihole logs

### 3. Data Prepper 2.12.2
- Kafka source connector (<< Kafka broker IP:port >>)
- Two processing pipelines:
  - **journals-pipeline**: systemd journal logs
  - **pihole-pipeline**: Pi-hole DNS logs
- JSON parsing and field extraction
- Grok pattern matching for DNS queries

### 4. Data Sources
- **Kafka Broker**: << Kafka broker IP:port >> (no auth, no TLS)
- **Topics**:
  - `journals`: Systemd journal logs from filebeat
  - `pihole`: Pi-hole DNS logs from filebeat

## Prerequisites

- Kubernetes cluster with kubectl access
- Ansible installed locally
- Kubeconfig file: `<< Path to your kubeconfig file >>`
- Kafka broker running at << Kafka broker IP:port >>
- Filebeat instances shipping logs to Kafka

## Files

```
kubernetes/opensearch/
├── README.md                    # This file
├── opensearch_create.yml        # Deploy OpenSearch + Dashboards + GeoIP
├── opensearch_delete.yml        # Remove deployment (preserve PV/PVC)
├── data_prepper_config.yaml     # Data Prepper pipeline configuration
└── data_prepper_deploy.yml      # Deploy Data Prepper
```

## Deployment

### Initial Setup

1. **Deploy OpenSearch and Dashboards:**
   ```bash
   ansible-playbook kubernetes/opensearch/opensearch_create.yml
   ```

   This playbook:
   - Creates namespace `opensearch-cluster`
   - Deploys OpenSearch 3.3.2
   - Deploys OpenSearch Dashboards 3.3.0
   - Creates GeoIP ingest pipeline
   - Configures index template for automatic GeoIP enrichment
   - Creates ingress for dashboard access

2. **Deploy Data Prepper:**
   ```bash
   ansible-playbook kubernetes/opensearch/data_prepper_deploy.yml
   ```

   This playbook:
   - Reads pipeline configuration from `data_prepper_config.yaml`
   - Creates ConfigMap with pipelines
   - Deploys Data Prepper with Kafka integration
   - Waits for pod to be ready

### Clean Removal

To remove OpenSearch while preserving data:
```bash
ansible-playbook kubernetes/opensearch/opensearch_delete.yml
```

**Note:** This preserves the PVC `opensearch-data` so you don't lose your data.

## Configuration

### OpenSearch Variables
Located in `opensearch_create.yml`:
```yaml
kubeconfig: "<< Path to your kubeconfig file >>"
opensearch_ns: "opensearch-cluster"
cluster_name: "opensearch-single"
opensearch_version: "3.3.2"
dashboard_version: "3.3.0"
```

### Data Prepper Pipelines
Located in `data_prepper_config.yaml`:

#### Journals Pipeline
- **Source**: Kafka topic `journals`
- **Processing**:
  - JSON parsing from filebeat format
  - Field extraction and enrichment
- **Sink**: OpenSearch index `journals-YYYY.MM.DD`

#### Pihole Pipeline
- **Source**: Kafka topic `pihole`
- **Processing**:
  - JSON parsing from filebeat format
  - DNS query extraction: `query[A]` and `query[AAAA]`
  - Blocked domain detection: `gravity blocked`
  - DNS resolution extraction: `reply`, `cached`, `cached-stale`
  - GeoIP enrichment (via OpenSearch ingest pipeline)
- **Sink**: OpenSearch index `pihole-YYYY.MM.DD`

## Data Fields Extracted

### DNS Query Fields
From messages like: `query[A] www.microsoft.com from << Client IP address >>`
- `dns_queried_system`: Domain being queried (e.g., `www.microsoft.com`)
- `dns_query_from`: IP address making the query (e.g., `<< Client IP address >>`)

### Blocked Query Fields
From messages like: `gravity blocked incoming.telemetry.mozilla.org is 0.0.0.0`
- `dns_queried_system`: Blocked domain
- `dns_query_blocked_response`: Block response (e.g., `0.0.0.0`, `NODATA`)

### DNS Resolution Fields
From messages like: `reply slackb.com is 34.202.253.140`
- `dns_queried_system`: Domain name
- `dns_queried_system_ip`: Resolved IP address
- `dns_queried_system_geoip`: GeoIP information (added by OpenSearch)
  - `continent_name`
  - `country_name`
  - `country_iso_code`
  - `region_name`
  - `city_name`
  - `location`: Coordinates in geo_point format (lat,lon)

## GeoIP Enrichment

The OpenSearch ingest pipeline automatically enriches DNS resolution IPs with geographic data:

1. **Pipeline**: `geoip-pipeline`
   - Extracts location from `dns_queried_system_ip`
   - Adds geographic metadata
   - Converts location to geo_point format for mapping

2. **Index Template**: `pihole-geoip-template`
   - Applies to all `pihole-*` indices
   - Automatically uses `geoip-pipeline`
   - Ensures `location` field is mapped as `geo_point`

## Access Information

### OpenSearch API
```bash
kubectl port-forward svc/opensearch 9200:9200 -n opensearch-cluster
# Then access: http://localhost:9200
```

### OpenSearch Dashboards
- **URL**: http://<< Your OpenSearch Dashboard domain >>
- **Port-forward**: 
  ```bash
  kubectl port-forward svc/opensearch-dashboards 5601:5601 -n opensearch-cluster
  # Then access: http://localhost:5601
  ```

### Data Prepper Metrics
```bash
kubectl port-forward svc/data-prepper 4900:4900 -n opensearch-cluster
# Then access: http://localhost:4900/metrics/prometheus
```

## Creating Visualizations

### DNS Query Map (Geographic Distribution)

1. **Create Index Pattern** (if not exists):
   - Go to **Stack Management** → **Index Patterns**
   - Create pattern: `pihole-*`
   - Time field: `@timestamp`

2. **Create Coordinate Map**:
   - **Visualize** → **Create visualization** → **Coordinate Map**
   - Select index pattern: `pihole-*`
   - **Buckets** → **Add** → **Geo coordinates**
   - **Aggregation**: Geohash
   - **Field**: `dns_queried_system_geoip.location`
   - Click **Update**

3. **Add Metrics** (optional):
   - Terms aggregation on `dns_queried_system.keyword` to see top domains
   - Filters for blocked vs allowed queries

### Sample Queries

**All DNS queries with source IP:**
```
dns_query_from: *
```

**All blocked domains:**
```
dns_query_blocked_response: *
```

**All allowed queries (not blocked):**
```
dns_query_from: * AND NOT _exists_:dns_query_blocked_response
```

**DNS resolutions with geolocation:**
```
_exists_:dns_queried_system_geoip.location
```

**Queries from specific IP:**
```
dns_query_from: "10.0.1.100"
```

**Domains resolved to specific country:**
```
dns_queried_system_geoip.country_name: "United States"
```

## Monitoring

### Check Data Prepper Status
```bash
kubectl logs -f deployment/data-prepper -n opensearch-cluster
```

### Check OpenSearch Indices
```bash
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl -s http://localhost:9200/_cat/indices?v
```

### Check Document Count
```bash
# Journals
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl -s http://localhost:9200/journals-*/_count

# Pihole
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl -s http://localhost:9200/pihole-*/_count
```

### Verify GeoIP Enrichment
```bash
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl -s -X GET "http://localhost:9200/pihole-*/_search?size=1" \
  -H 'Content-Type: application/json' \
  -d '{"query":{"exists":{"field":"dns_queried_system_geoip.location"}}}'
```

## Troubleshooting

### Data Prepper Pod Crashing
1. Check logs: `kubectl logs deployment/data-prepper -n opensearch-cluster`
2. Verify Kafka connectivity: Ensure << Kafka broker IP:port >> is reachable
3. Check ConfigMap: `kubectl get configmap data-prepper-config -n opensearch-cluster -o yaml`

### No GeoIP Data in Visualizations
1. Verify pipeline exists:
   ```bash
   kubectl exec -n opensearch-cluster deployment/opensearch -- \
     curl -s http://localhost:9200/_ingest/pipeline/geoip-pipeline
   ```
2. Check index template:
   ```bash
   kubectl exec -n opensearch-cluster deployment/opensearch -- \
     curl -s http://localhost:9200/_index_template/pihole-geoip-template
   ```
3. Recreate index pattern in Dashboards:
   - Delete old `pihole-*` index pattern
   - Create new one to pick up geo_point field

### OpenSearch Dashboards Not Loading
1. Check pod status: `kubectl get pods -n opensearch-cluster`
2. Check logs: `kubectl logs deployment/opensearch-dashboards -n opensearch-cluster`
3. Verify ingress: `kubectl get ingress -n opensearch-cluster`

### Kafka Connection Issues
1. Test connectivity from Data Prepper pod:
   ```bash
   kubectl exec -n opensearch-cluster deployment/data-prepper -- \
     nc -zv << Kafka broker IP >> 9092
   ```
2. Check Data Prepper logs for Kafka errors
3. Verify Kafka topic exists and has data

## Storage

- **PVC Name**: `opensearch-data`
- **Size**: 25Gi
- **Access Mode**: ReadWriteOnce
- **Location**: `/usr/share/opensearch/data` in OpenSearch pod

**Note**: The delete playbook preserves this PVC to prevent data loss.

## Security Notes

⚠️ **This deployment disables security for simplicity:**
- No authentication required for OpenSearch
- No TLS/SSL encryption
- Suitable for internal/lab environments only

**For production use, enable:**
- OpenSearch Security Plugin
- TLS/SSL certificates
- Authentication (basic auth, LDAP, SAML)
- Network policies
- RBAC

## Performance Tuning

### Increase Data Prepper Resources
Edit `data_prepper_deploy.yml`:
```yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"
```

### Adjust Bulk Size
Edit `data_prepper_config.yaml`:
```yaml
sink:
  - opensearch:
      bulk_size: 10  # Increase for higher throughput
      flush_timeout: 30000  # Decrease for lower latency
```

### Scale OpenSearch
For production, consider:
- Multi-node cluster
- Replica shards
- Index lifecycle management (ILM)
- Hot/warm architecture

## Backup and Recovery

### Backup PVC
```bash
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  tar czf /tmp/opensearch-backup.tar.gz /usr/share/opensearch/data

kubectl cp opensearch-cluster/opensearch-xxxxx:/tmp/opensearch-backup.tar.gz \
  ./opensearch-backup.tar.gz
```

### Restore
```bash
kubectl cp ./opensearch-backup.tar.gz \
  opensearch-cluster/opensearch-xxxxx:/tmp/

kubectl exec -n opensearch-cluster deployment/opensearch -- \
  tar xzf /tmp/opensearch-backup.tar.gz -C /
```

## Version History

- **Initial Release**: OpenSearch 3.3.2, Dashboards 3.3.0, Data Prepper 2.12.2
- GeoIP enrichment with automatic geo_point mapping
- Kafka integration with journals and pihole topics
- DNS query analytics with geolocation visualization

## Contributing

To modify this deployment:
1. Update configuration in YAML files
2. Test changes in dev environment
3. Run playbooks to apply changes
4. Verify with monitoring commands

## License

Internal use - Adjust according to your organization's requirements.

## Support

For issues or questions:
- Check logs: `kubectl logs -n opensearch-cluster <pod-name>`
- Review OpenSearch documentation: https://opensearch.org/docs/latest/
- Check Data Prepper docs: https://opensearch.org/docs/latest/data-prepper/

