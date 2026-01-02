# OpenSearch DNS Analytics Stack - Setup Guide

## 📋 Prerequisites Checklist

Before deploying, ensure you have:

- [ ] Kubernetes cluster (RKE2, K3s, or similar)
- [ ] `kubectl` configured with cluster access
- [ ] Ansible installed with `kubernetes.core` collection
- [ ] Kafka broker accessible from cluster
- [ ] Ingress controller deployed (Traefik, nginx, etc.)
- [ ] DNS entry or hosts file entry for OpenSearch Dashboards

## ⚙️ Configuration Steps

### 1. Update Kubeconfig Path

In **all playbook files** (`opensearch_create.yml`, `opensearch_delete.yml`, `data_prepper_deploy.yml`):

```yaml
vars:
  kubeconfig: "<< Path to your kubeconfig file >>"
  # Example: "/home/user/.kube/config"
  # Or: "/etc/rancher/rke2/rke2.yaml"
```

### 2. Configure Ingress Domain

In `opensearch_create.yml`:

```yaml
spec:
  rules:
    - host: << Your OpenSearch Dashboard domain >>
      # Example: opensearch-dashboard.example.com
```

Update your DNS or `/etc/hosts`:
```
<your-ingress-ip> opensearch-dashboard.example.com
```

### 3. Configure Kafka Connection

In `data_prepper_config.yaml`, update **both pipelines**:

```yaml
journals-pipeline:
  source:
    kafka:
      bootstrap_servers:
        - "<< Kafka broker IP:port >>"
        # Example: "192.168.1.100:9092"

pihole-pipeline:
  source:
    kafka:
      bootstrap_servers:
        - "<< Kafka broker IP:port >>"
        # Example: "192.168.1.100:9092"
```

### 4. Verify Kafka Topics

Ensure these topics exist in your Kafka cluster:
- `journals` - for systemd journal logs
- `pihole` - for Pi-hole DNS logs

Create them if needed:
```bash
kafka-topics.sh --create --topic journals --bootstrap-server << Kafka broker IP:port >>
kafka-topics.sh --create --topic pihole --bootstrap-server << Kafka broker IP:port >>
```

## 🚀 Deployment

### Step 1: Deploy OpenSearch
```bash
ansible-playbook opensearch_create.yml
```

This creates:
- Namespace: `opensearch-cluster`
- PersistentVolumeClaim (25Gi)
- OpenSearch deployment
- OpenSearch Dashboards
- GeoIP ingest pipeline for DNS IP enrichment
- Ingress for dashboard access

### Step 2: Deploy Data Prepper
```bash
ansible-playbook data_prepper_deploy.yml
```

This creates:
- ConfigMap with pipeline configuration
- Data Prepper deployment
- Service for metrics endpoint

### Step 3: Verify Deployment
```bash
# Check all pods are running
kubectl get pods -n opensearch-cluster

# Expected output:
# opensearch-xxxxxxxxx-xxxxx              1/1     Running
# opensearch-dashboards-xxxxxxxxx-xxxxx   1/1     Running
# data-prepper-xxxxxxxxx-xxxxx            1/1     Running
```

## 🔍 Access & Verification

### Access OpenSearch Dashboards
Visit: `http://<< Your OpenSearch Dashboard domain >>`

Or use port-forward:
```bash
kubectl port-forward svc/opensearch-dashboards 5601:5601 -n opensearch-cluster
# Visit: http://localhost:5601
```

### Check Data Ingestion

1. **Via kubectl exec**:
```bash
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl -s "http://localhost:9200/_cat/indices?v"
```

You should see indices like:
- `journals-2024.11.26`
- `pihole-2024.11.26`

2. **Via Dashboards**:
- Go to "Dev Tools" → Console
- Run: `GET _cat/indices?v`

### Query Sample Logs

**Journald logs**:
```json
GET journals-*/_search
{
  "size": 10,
  "sort": [{"@timestamp": "desc"}]
}
```

**Pihole DNS logs with GeoIP**:
```json
GET pihole-*/_search
{
  "query": {
    "exists": {
      "field": "dns_queried_system_geoip"
    }
  },
  "size": 10
}
```

## 🔧 Customization

### Adjust Resource Limits

In `opensearch_create.yml`, modify:
```yaml
resources:
  requests:
    memory: "2Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "2000m"
```

### Change Storage Size

In `opensearch_create.yml`:
```yaml
spec:
  resources:
    requests:
      storage: 25Gi  # Change this value
```

### Modify Data Retention

Create an Index State Management (ISM) policy in OpenSearch Dashboards to auto-delete old indices.

### Add More Pipelines

Edit `data_prepper_config.yaml` to add new Kafka topics:
```yaml
your-pipeline:
  source:
    kafka:
      topics:
        - name: your-topic
          group_id: data-prepper-your-topic
      bootstrap_servers:
        - "<< Kafka broker IP:port >>"
  sink:
    - opensearch:
        hosts: 
          - "http://opensearch.opensearch-cluster.svc.cluster.local:9200"
        index: "your-index-%{yyyy.MM.dd}"
```

Then redeploy:
```bash
ansible-playbook data_prepper_deploy.yml
```

## 🧹 Cleanup

### Delete Everything (keeps PVC)
```bash
ansible-playbook opensearch_delete.yml
```

### Delete PVC Manually (if needed)
```bash
kubectl delete pvc opensearch-data -n opensearch-cluster
```

### Delete Namespace
```bash
kubectl delete namespace opensearch-cluster
```

## 🐛 Troubleshooting

### Data Prepper Not Processing Logs

1. **Check Data Prepper logs**:
```bash
kubectl logs -f deployment/data-prepper -n opensearch-cluster
```

2. **Verify Kafka connectivity**:
```bash
kubectl exec -n opensearch-cluster deployment/data-prepper -- \
  nc -zv << Kafka broker IP >> 9092
```

3. **Check pipeline metrics**:
```bash
kubectl port-forward svc/data-prepper 4900:4900 -n opensearch-cluster
curl http://localhost:4900/metrics/prometheus
```

### OpenSearch Pod CrashLoopBackOff

1. **Check logs**:
```bash
kubectl logs deployment/opensearch -n opensearch-cluster
```

2. **Common issue - vm.max_map_count**:
The `sysctl` initContainer should handle this, but verify:
```bash
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  cat /proc/sys/vm/max_map_count
# Should be: 262144
```

### GeoIP Not Working

1. **Verify pipeline exists**:
```bash
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl "http://localhost:9200/_ingest/pipeline/geoip-pipeline?pretty"
```

2. **Check if template is applied**:
```bash
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl "http://localhost:9200/_index_template/pihole-geoip-template?pretty"
```

3. **Manually test GeoIP**:
```json
POST _ingest/pipeline/geoip-pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "dns_queried_system_ip": "8.8.8.8"
      }
    }
  ]
}
```

## 📊 Monitoring

### View Data Prepper Metrics
```bash
kubectl port-forward svc/data-prepper 4900:4900 -n opensearch-cluster
curl http://localhost:4900/list
curl http://localhost:4900/metrics/prometheus
```

### Monitor Index Size
```bash
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl "http://localhost:9200/_cat/indices?v&s=store.size:desc"
```

### Check Cluster Health
```bash
kubectl exec -n opensearch-cluster deployment/opensearch -- \
  curl "http://localhost:9200/_cluster/health?pretty"
```

## 🔐 Production Hardening

For production deployments, consider:

1. **Enable OpenSearch security plugin**
   - Remove `DISABLE_SECURITY_PLUGIN: "true"`
   - Configure users, roles, and TLS

2. **Add resource quotas**
   - Set namespace resource limits
   - Configure pod disruption budgets

3. **Implement backups**
   - Use OpenSearch snapshot/restore
   - Schedule regular backups

4. **Configure TLS**
   - Add cert-manager
   - Enable HTTPS on ingress
   - Use TLS for OpenSearch inter-node communication

5. **High Availability**
   - Deploy 3+ OpenSearch nodes
   - Use StatefulSet instead of Deployment
   - Configure replication

## 📚 Next Steps

- [Create custom dashboards in OpenSearch Dashboards](https://opensearch.org/docs/latest/dashboards/)
- [Set up alerting](https://opensearch.org/docs/latest/monitoring-plugins/alerting/)
- [Configure Index State Management](https://opensearch.org/docs/latest/im-plugin/ism/)
- [Optimize search performance](https://opensearch.org/docs/latest/tuning-your-cluster/)































