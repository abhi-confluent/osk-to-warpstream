# Open source kafka to WarpStream Migration Guide

This repository contains the complete setup and configuration for migrating from an open-source Kafka cluster (deployed via Strimzi operator) to WarpStream Kafka using Orbit replication.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Step-by-Step Migration Guide](#step-by-step-migration-guide)
  - [Step 1: Deploy Strimzi Kafka Operator](#step-1-deploy-strimzi-kafka-operator)
  - [Step 2: Deploy Kafka Cluster](#step-2-deploy-kafka-cluster)
  - [Step 3: Create Sample Topics and Data](#step-3-create-sample-topics-and-data)
  - [Step 4: Deploy WarpStream Agent](#step-4-deploy-warpstream-agent)
  - [Step 5: Configure Orbit Migration](#step-5-configure-orbit-migration)
  - [Step 6: Verify Migration](#step-6-verify-migration)
- [Repository Structure](#repository-structure)
- [Troubleshooting](#troubleshooting)
- [References](#references)

## 🎯 Overview

This project demonstrates how to migrate from a self-managed Kafka cluster (using Strimzi operator) to WarpStream. The migration uses Orbit, WarpStream's built-in cluster linking feature that replicates data and metadata between clusters.

**Key Features:**
- Preserves topic configurations, partitions, and offsets
- Replicates consumer group offsets
- Supports incremental migration

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌──────────────────┐          ┌──────────────────────┐     │
│  │  kafka namespace │          │ warpstream-agent     │     │
│  │                  │          │   namespace          │     │
│  │  ┌────────────┐  │          │                      │     │
│  │  │  Strimzi   │  │          │  ┌────────────────┐  │     │
│  │  │  Operator  │  │          │  │ WarpStream     │  │     │
│  │  └────────────┘  │          │  │ Agent (Orbit)  │  │     │
│  │       │          │          │  └────────────────┘  │     │
│  │       ▼          │          │          │           │     │
│  │  ┌────────────┐  │          │          │           │     │
│  │  │  Kafka     │  │◄─────────┼──────────┘           │     │
│  │  │  Cluster   │  │  Replication                    │     │
│  │  │  (KRaft)   │  │          │                      │     |
│  │  └────────────┘  │          |                      │     │
│  │                  │          |                      │     │
│  │  Topics:         │          |Topics:               │     │
│  │  • osk-topic-1   │          |•osk-topic-1(replica) │     │
│  │  • osk-topic-2   │          |•osk-topic-2(replica) │     │
│  │  • osk-topic-3   │          |•osk-topic-3(replica) │     │
│  └──────────────────┘          └──────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │   S3 Bucket     │
                    │ (Object Storage)│
                    └─────────────────┘
```

## 📦 Prerequisites

Before starting, ensure you have:

1. **Kubernetes Cluster** with:
   - `kubectl` configured and connected
   - `helm` v3.x installed
   - Sufficient resources (recommended: at least 4 CPUs and 8GB RAM per node)

2. **S3-Compatible Object Storage**:
   - S3 bucket created for WarpStream
   - IAM user/role with full access to the bucket
   - AWS credentials (Access Key ID and Secret Access Key)

3. **WarpStream Account**:
   - Virtual Cluster ID from WarpStream console
   - Agent API key from WarpStream console

4. **Network Access**:
   - WarpStream agents can reach source Kafka cluster
   - Source Kafka cluster is accessible within Kubernetes

## 🚀 Step-by-Step Migration Guide

### Step 1: Deploy Strimzi Kafka Operator

1. **Add Strimzi Helm repository:**
   ```bash
   helm repo add strimzi https://strimzi.io/charts/
   helm repo update
   ```

2. **Create namespaces:**
   ```bash
   kubectl create namespace strimzi-operator
   kubectl create namespace kafka
   ```

3. **Install Strimzi Cluster Operator:**
   ```bash
   helm install strimzi-operator strimzi/strimzi-kafka-operator \
     --namespace strimzi-operator \
     --set watchNamespaces="{kafka,strimzi-operator}"
   ```

4. **Verify operator is running:**
   ```bash
   kubectl get pods -n strimzi-operator
   # Wait until the operator pod shows Status: Running
   ```

### Step 2: Deploy Kafka Cluster

1. **Apply Kafka cluster configuration:**
   ```bash
   kubectl apply -f k8s/strimzi/kafka-cluster.yaml
   ```

2. **Apply Kafka NodePools:**
   ```bash
   # Create controller node pool (required for KRaft mode)
   kubectl apply -f k8s/strimzi/kafka-controller-nodepool.yaml
   
   # Create broker node pool
   kubectl apply -f k8s/strimzi/kafka-nodepool.yaml
   ```

3. **Wait for cluster to be ready:**
   ```bash
   kubectl get kafka -n kafka
   kubectl get pods -n kafka
   # Wait until all pods show Status: Running and READY: 1/1
   ```

4. **Verify cluster status:**
   ```bash
   kubectl get kafka my-cluster -n kafka
   # Should show READY: True and METADATA STATE: KRaft
   ```

5. **Get bootstrap servers:**
   ```bash
   # The bootstrap server will be: my-cluster-kafka-bootstrap.kafka.svc:9092
   kubectl get svc my-cluster-kafka-bootstrap -n kafka
   ```

### Step 3: Create Sample Topics and Data

1. **Create sample topics using Kafka CLI:**
   ```bash
   # Create a temporary Kafka client pod
   kubectl run kafka-client --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap.kafka.svc:9092 \
     --create --topic osk-topic-1 --partitions 3 --replication-factor 2
   
   kubectl run kafka-client --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap.kafka.svc:9092 \
     --create --topic osk-topic-2 --partitions 3 --replication-factor 2
   
   kubectl run kafka-client --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap.kafka.svc:9092 \
     --create --topic osk-topic-3 --partitions 3 --replication-factor 2
   ```

2. **Verify topics created:**
   ```bash
   kubectl run kafka-client --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap.kafka.svc:9092 --list
   ```

3. **Produce sample JSON messages to topics:**

   **To osk-topic-1:**
   ```bash
   cat <<'EOF' | kubectl run kafka-producer --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap.kafka.svc:9092 --topic osk-topic-1
   {"id": "msg-001", "timestamp": "2025-10-31T06:15:00Z", "source": "user-service", "event": "user_created", "data": {"userId": "12345", "username": "john_doe", "email": "john@example.com"}}
   {"id": "msg-002", "timestamp": "2025-10-31T06:15:30Z", "source": "user-service", "event": "user_updated", "data": {"userId": "12345", "changes": {"email": "john.doe@example.com"}}}
   {"id": "msg-003", "timestamp": "2025-10-31T06:16:00Z", "source": "order-service", "event": "order_placed", "data": {"orderId": "ord-789", "userId": "12345", "amount": 99.99, "items": [{"productId": "prod-1", "quantity": 2}]}}
   {"id": "msg-004", "timestamp": "2025-10-31T06:16:30Z", "source": "payment-service", "event": "payment_processed", "data": {"paymentId": "pay-456", "orderId": "ord-789", "amount": 99.99, "status": "completed"}}
   {"id": "msg-005", "timestamp": "2025-10-31T06:17:00Z", "source": "inventory-service", "event": "stock_updated", "data": {"productId": "prod-1", "previousStock": 100, "newStock": 98, "change": -2}}
   EOF
   ```

   **To osk-topic-2:**
   ```bash
   cat <<'EOF' | kubectl run kafka-producer --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap.kafka.svc:9092 --topic osk-topic-2
   {"id": "msg-101", "timestamp": "2025-10-31T06:20:00Z", "source": "analytics-service", "event": "page_view", "data": {"page": "/products", "userId": "12345", "sessionId": "sess-abc", "duration": 45}}
   {"id": "msg-102", "timestamp": "2025-10-31T06:20:15Z", "source": "analytics-service", "event": "click", "data": {"element": "add-to-cart-button", "userId": "12345", "sessionId": "sess-abc", "page": "/products"}}
   {"id": "msg-103", "timestamp": "2025-10-31T06:20:30Z", "source": "notification-service", "event": "email_sent", "data": {"emailId": "email-321", "to": "john@example.com", "subject": "Order Confirmation", "template": "order_confirmation"}}
   {"id": "msg-104", "timestamp": "2025-10-31T06:21:00Z", "source": "search-service", "event": "search_query", "data": {"query": "laptop", "userId": "12345", "resultsCount": 42, "filters": {"category": "electronics", "priceRange": "500-1000"}}}
   {"id": "msg-105", "timestamp": "2025-10-31T06:21:30Z", "source": "recommendation-service", "event": "recommendation_generated", "data": {"userId": "12345", "recommendations": [{"productId": "prod-2", "score": 0.95}, {"productId": "prod-3", "score": 0.87}]}}
   EOF
   ```

   **To osk-topic-3:**
   ```bash
   cat <<'EOF' | kubectl run kafka-producer --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap.kafka.svc:9092 --topic osk-topic-3
   {"id": "msg-201", "timestamp": "2025-10-31T06:25:00Z", "source": "monitoring-service", "event": "metric_reported", "data": {"metric": "cpu_usage", "value": 65.5, "unit": "percent", "host": "server-01", "tags": {"environment": "production"}}}
   {"id": "msg-202", "timestamp": "2025-10-31T06:25:30Z", "source": "monitoring-service", "event": "alert_triggered", "data": {"alertId": "alert-789", "severity": "warning", "metric": "memory_usage", "value": 85.2, "threshold": 80, "host": "server-02"}}
   {"id": "msg-203", "timestamp": "2025-10-31T06:26:00Z", "source": "log-aggregator", "event": "log_processed", "data": {"logId": "log-456", "level": "ERROR", "message": "Database connection timeout", "service": "api-gateway", "traceId": "trace-xyz"}}
   {"id": "msg-204", "timestamp": "2025-10-31T06:26:30Z", "source": "scheduler-service", "event": "job_completed", "data": {"jobId": "job-123", "jobType": "daily-report", "status": "success", "duration": 120, "recordsProcessed": 5000}}
   {"id": "msg-205", "timestamp": "2025-10-31T06:27:00Z", "source": "audit-service", "event": "access_logged", "data": {"userId": "12345", "action": "DELETE", "resource": "/api/v1/users/67890", "ip": "192.168.1.100", "userAgent": "Mozilla/5.0", "success": true}}
   EOF
   ```

4. **Verify messages were produced (optional):**
   ```bash
   kubectl run kafka-consumer --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap.kafka.svc:9092 \
     --topic osk-topic-1 --from-beginning --max-messages 2
   ```

### Step 4: Deploy WarpStream Agent

1. **Add WarpStream Helm repository:**
   ```bash
   helm repo add warpstream https://warpstreamlabs.github.io/charts
   helm repo update
   ```

2. **Create namespace:**
   ```bash
   kubectl create namespace warpstream-agent
   ```

3. **Create values file:**
   Copy `helm/warpstream/values.yaml.template` to `helm/warpstream/values.yaml` and update with your configuration:
   ```yaml
   config:
     bucketURL: "s3://YOUR_BUCKET_NAME?region=YOUR_REGION"
     agentKey: "YOUR_AGENT_KEY"
     region: "YOUR_REGION"
     virtualClusterID: "YOUR_VIRTUAL_CLUSTER_ID"
   
   extraEnv:
     - name: AWS_ACCESS_KEY_ID
       value: "YOUR_AWS_ACCESS_KEY_ID"
     - name: AWS_SECRET_ACCESS_KEY
       value: "YOUR_AWS_SECRET_ACCESS_KEY"
   
   nodeSelector:
     cloud.google.com/gke-nodepool: "pool-warpstream"  # Adjust based on your node pool
   ```

4. **Install WarpStream agent:**
   ```bash
   helm upgrade --install warpstream-agent warpstream/warpstream-agent \
     --namespace warpstream-agent \
     --values helm/warpstream/values.yaml
   ```

5. **Verify deployment:**
   ```bash
   kubectl get pods -n warpstream-agent
   # Wait until pods show Status: Running and READY: 1/1
   ```

6. **Get WarpStream bootstrap servers:**
   ```bash
   kubectl get svc warpstream-agent -n warpstream-agent
   # Bootstrap server: warpstream-agent.warpstream-agent.svc:9092
   ```

7. **Test WarpStream (optional):**
   ```bash
   # Create a test topic
   kubectl run kafka-client --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-topics.sh --bootstrap-server warpstream-agent.warpstream-agent.svc:9092 \
     --create --topic warpstream-test-topic --partitions 3 --replication-factor 1
   
   # Produce a test message
   echo '{"test": "message"}' | kubectl run kafka-producer --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-console-producer.sh --bootstrap-server warpstream-agent.warpstream-agent.svc:9092 --topic warpstream-test-topic
   ```

### Step 5: Configure Orbit Migration

1. **Prepare Orbit configuration:**
   Copy `orbit/config.yaml.template` to `orbit/config.yaml` and update the source broker configuration as needed:
   ```yaml
   source_bootstrap_brokers:
    - hostname: my-cluster-kafka-bootstrap.kafka.svc
      port: 9092

   source_cluster_credentials:
      # No SASL/TLS required for Strimzi plain listener
      use_tls: false
      tls_insecure_skip_verify: false

   topic_mappings:
      # Match all topics starting with "osk-topic-"
      - source_regex: osk-topic-.*
         destination_prefix: ""
         begin_fetch_at_latest_offset: false
         irreversible_disable_orbit_management: false

   cluster_config:
      copy_source_cluster_configuration: false

   consumer_groups:
      destination_group_prefix: ""
      copy_offsets_enabled: true

   warpstream:
      cluster_fetch_concurrency: 2

   ```

2. **Upload configuration to WarpStream Console:**
   - Log in to your [WarpStream Console](https://console.warpstream.com)
   - Navigate to your Virtual Cluster
   - Go to the **Orbit** section
   - Click **Upload Configuration** or **Create Pipeline**
   - Paste the contents of `orbit/config.yaml`
   - Save and deploy the configuration

3. **Monitor Orbit status:**
   - In the WarpStream Console, check the Orbit dashboard
   - Verify that topics are being discovered and replicated
   - Monitor the replication lag metrics

### Step 6: Verify Migration

1. **Check topics in WarpStream:**
   ```bash
   kubectl run kafka-client --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-topics.sh --bootstrap-server warpstream-agent.warpstream-agent.svc:9092 --list
   # Should show osk-topic-1, osk-topic-2, osk-topic-3
   ```

2. **Compare topic configurations:**
   ```bash
   # Source cluster
   kubectl run kafka-client --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap.kafka.svc:9092 --describe --topic osk-topic-1
   
   # Destination cluster
   kubectl run kafka-client --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-topics.sh --bootstrap-server warpstream-agent.warpstream-agent.svc:9092 --describe --topic osk-topic-1
   ```

3. **Verify message replication:**
   ```bash
   # Consume from WarpStream topic
   kubectl run kafka-consumer --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm -i --tty=false -- \
     bin/kafka-console-consumer.sh --bootstrap-server warpstream-agent.warpstream-agent.svc:9092 \
     --topic osk-topic-1 --from-beginning --max-messages 5
   ```


## 📁 Repository Structure

```
kafka-to-warpstream-migration/
├── README.md                          # This file
├── .gitignore                         # Git ignore rules
├── k8s/                               # Kubernetes manifests
│   ├── strimzi/                       # Strimzi Kafka configurations
│   │   ├── namespace.yaml             # Kafka namespace
│   │   ├── kafka-cluster.yaml         # Kafka cluster definition
│   │   ├── kafka-controller-nodepool.yaml  # Controller node pool (KRaft)
│   │   ├── kafka-nodepool.yaml        # Broker node pool
│   │   └── kafka-user.yaml            # Kafka user for authentication
│   └── warpstream/                    # WarpStream configurations
│       └── namespace.yaml             # WarpStream agent namespace
├── helm/                              # Helm chart values
│   ├── strimzi/
│   │   └── values.yaml                # Strimzi operator values
│   └── warpstream/
│       ├── values.yaml.template       # WarpStream values template
│       └── values.yaml                # WarpStream values (not in git)
└── orbit/                             # Orbit migration configuration
    ├── config.yaml.template           # Orbit configuration template
    ├── config.yaml                    # Orbit configuration (not in git)
    └── migration-guide.md             # Detailed migration guide
```

## 🔧 Troubleshooting

### Strimzi Kafka Issues

**Problem: Cluster stuck in NotReady state**
```bash
# Check cluster status
kubectl describe kafka my-cluster -n kafka

# Common issues:
# - Unsupported Kafka version (must be 4.0.0+ for KRaft)
# - Missing controller NodePool
# - Insufficient resources
```

**Problem: Pods stuck in Pending**
```bash
# Check pod events
kubectl describe pod <pod-name> -n kafka

# Common solutions:
# - Increase node resources
# - Check storage classes
# - Verify PVCs are bound
```

### WarpStream Agent Issues

**Problem: Agent pods crashing**
```bash
# Check logs
kubectl logs -n warpstream-agent <pod-name>

# Common issues:
# - S3 access denied (check IAM permissions)
# - Invalid agent key
# - Network connectivity issues
```

**Problem: Pods not scheduling**
```bash
# Check node selector matches node labels
kubectl get nodes --show-labels | grep nodepool

# Update nodeSelector in values.yaml if needed
```

### Orbit Replication Issues

**Problem: Topics not appearing in WarpStream**
- Verify Orbit configuration is deployed in WarpStream Console
- Check source broker connectivity from WarpStream agents
- Verify topic regex patterns match existing topics

**Problem: High replication lag**
- Increase `cluster_fetch_concurrency` in Orbit config
- Check source cluster load and network latency
- Monitor WarpStream agent resource usage

**Problem: Missing messages**
- Verify `begin_fetch_at_latest_offset: false` to start from beginning
- Check Orbit logs in WarpStream Console
- Verify consumer group offsets if applicable

## 📚 References

### Documentation
- [Strimzi Documentation](https://strimzi.io/docs/)
- [WarpStream Documentation](https://docs.warpstream.com/)
- [WarpStream Orbit Documentation](https://docs.warpstream.com/warpstream/kafka/orbit)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)

### Helm Charts
- [Strimzi Helm Charts](https://strimzi.io/charts/)
- [WarpStream Helm Charts](https://warpstreamlabs.github.io/charts)

### Related Resources
- [Strimzi GitHub Repository](https://github.com/strimzi/strimzi-kafka-operator)
- [WarpStream Console](https://console.warpstream.com)
- [WarpStream Slack Community](https://console.warpstream.com/socials/slack)

## 📝 License

This repository is provided as-is for educational and demonstration purposes.

---

**Note:** This guide assumes you're deploying to a Kubernetes cluster (tested with GKE). Adjust node selectors, storage classes, and other Kubernetes-specific configurations based on your environment.

