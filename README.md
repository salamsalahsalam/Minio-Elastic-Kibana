# Configure Elasticsearch with MinIO for Snapshots

This repository contains steps and configurations to set up Elasticsearch with MinIO as a snapshot repository in a Kubernetes environment.


## Prerequisites

Before proceeding, ensure you have the following:

- A running Kubernetes cluster with sufficient resources.
- Helm installed on your local machine.
- Elasticsearch installed in the cluster using Helm.
- MinIO installed in the cluster or as a standalone Docker container.

## Installation and Configuration

### Step 1: Install the S3 Plugin Using Init Container

Ensure that the `repository-s3` plugin is installed on all Elasticsearch nodes using an init container. Update your `values.yaml` for Elasticsearch with the following configuration:

```yaml
extraInitContainers:
  - name: install-s3-plugin
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.3
    command: ["sh", "-c"]
    args:
      - bin/elasticsearch-plugin install --batch repository-s3;
    volumeMounts:
      - name: elasticsearch-plugins
        mountPath: /usr/share/elasticsearch/plugins
extraVolumes:
  - name: elasticsearch-plugins
    emptyDir: {}
```

Apply the updated Helm chart to your Elasticsearch deployment:

```bash
helm upgrade --install elasticsearch elastic/elasticsearch -f values.yaml
```
![image](https://github.com/user-attachments/assets/ce47aaf7-bea0-468c-afdc-ee47b7ce4901)

### Step 2: Install MinIO Using Helm

Deploy MinIO to your Kubernetes cluster using the following command:

```bash
helm repo add minio https://charts.min.io/
helm install minio minio/minio \
  --set accessKey=minioadmin \
  --set secretKey=minioadmin \
  --set resources.requests.memory=512Mi \
  --set replicas=1 \
  --set persistence.enabled=false \
  --set mode=standalone \
  --set rootUser=rootuser,rootPassword=rootpass123
```

Retrieve the MinIO service information:

```bash
kubectl get svc
```

### Step 3: Create a MinIO Bucket and Access Keys

#### Using `mc` CLI

1. Configure an alias for your MinIO instance:

   ```bash
   mc alias set myminio http://<minio-ip>:9000 <access-key> <secret-key>
   ```

2. Create a bucket for Elasticsearch snapshots:

   ```bash
   mc mb myminio/elasticsearch-snapshots
   ```

#### Using the MinIO Console

Log into the MinIO console, navigate to "Buckets," and create a new bucket named `elasticsearch-snapshots`. You can also create access keys directly from the console if needed.

### Step 4: Configure Elasticsearch Snapshot Repository

Use the following API request to register a snapshot repository in Elasticsearch:

```http
PUT /_snapshot/minio-s3-repo
{
  "type": "s3",
  "settings": {
    "bucket": "elasticsearch-snapshots",
    "endpoint": "http://<minio-ip>:9000",
    "protocol": "http",
    "path_style_access": "true",
    "access_key": "<minio-access-key>",
    "secret_key": "<minio-secret-key>"
  }
}
```

### Step 5: Verify and Test Snapshots

#### Create an Index

```http
PUT /test-index
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "timestamp": {
        "type": "date"
      }
    }
  }
}
```

#### Add Documents to the Index

```http
POST /test-index/_doc
{
  "name": "Document 1",
  "timestamp": "2025-01-24T12:00:00"
}

POST /test-index/_doc
{
  "name": "Document 2",
  "timestamp": "2025-01-24T13:00:00"
}
```

#### Create a Snapshot

```http
PUT /_snapshot/minio-s3-repo/snapshot-test
{
  "indices": "test-index",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

#### Verify the Snapshot

```http
GET /_snapshot/minio-s3-repo/snapshot-test
```

#### Check in MinIO

Log into your MinIO instance and verify that the snapshot files are present in the `elasticsearch-snapshots` bucket.

---

## Troubleshooting

- **Path not accessible:** Ensure MinIO is reachable from all Elasticsearch nodes.
  ```bash
  curl http://<minio-ip>:9000
  ```

- **Access denied:** Verify access and secret keys are correctly configured.



