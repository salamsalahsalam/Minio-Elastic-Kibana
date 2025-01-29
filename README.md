# First: 

### **📄 README: Elasticsearch ILM Policy for Log Retention**  

## **🔹 Overview**  
This document explains the **Index Lifecycle Management (ILM) policy** used to manage log data in **Elasticsearch**. The policy ensures that:  
✅ Logs stay in the **hot phase for 10 minutes** for active writes.  
✅ Logs move to the **warm phase**, become **read-only**, and get optimized after 10 minutes.  
✅ Logs are **deleted after 25 minutes** to free up storage.  

---

## **🔹 ILM Phases Explained**  
| **Phase**  | **Duration** | **Actions Taken** |
|------------|------------|-------------------|
| **Hot**    | 0 → 10 min  | Data is actively written (no rollover) |
| **Warm**   | 10 → 25 min | Merges shards into 1, sets index to read-only |
| **Delete** | 25 min      | Deletes the index to free storage |

---

## **🔹 ILM Policy Configuration**  
To apply this policy, use the following request in **Kibana Dev Tools** or **cURL**:

```json
PUT _ilm/policy/my-lifecycle-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0m",
        "actions": {}
      },
      "warm": {
        "min_age": "10m",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          },
          "readonly": {}
        }
      },
      "delete": {
        "min_age": "25m",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

---

## **🔹 How This Works**  
1. **Hot Phase (0-10 min)**  
   - The index starts in the hot phase.  
   - **No rollover is applied**, so the index remains active for writes.  

2. **Warm Phase (10-25 min)**  
   - The index moves to **warm storage** after 10 minutes.  
   - **Forcemerge** optimizes performance by merging all shards into one.  
   - The index becomes **read-only** to prevent further writes.  

3. **Delete Phase (After 25 min)**  
   - The index is automatically **deleted** after 25 minutes to free up disk space.  

---

## **🔹 Applying the Policy to an Index**  
To apply this ILM policy to a new index:  

```json
PUT remat-logs-000001
{
  "settings": {
    "index.lifecycle.name": "my-lifecycle-policy"
  }
}
```

✅ This ensures the **index follows the ILM policy** and transitions through the phases automatically.  

---

## **🔹 Summary**  
- **No rollover** occurs in the hot phase.  
- Data moves to warm after **10 minutes**, becomes read-only, and is optimized.  
- Data is **deleted after 25 minutes**.  
- ILM automates **storage management** to keep Elasticsearch efficient.  

Would you like me to add **shard allocation settings** for different node types? 😊 

## For documentation about how to make index life cycle managmment and index template for managing daily logs : https://github.com/salamsalahsalam/Minio-Elastic-Kibana/blob/main/ELastic%26kibana.pptx


# Second Configure Elasticsearch with MinIO for Snapshots
This repository contains steps and configurations to set up Elasticsearch with MinIO as a snapshot repository in a Kubernetes environment.

## For snapshots and minio Documentation: https://github.com/salamsalahsalam/Minio-Elastic-Kibana/blob/main/Elasticsearch_MinIO_Configuration.docx

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
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.3   # Replace <version> with your Elasticsearch version
    command:
      - sh
      - -c
      - |
        if ! /usr/share/elasticsearch/bin/elasticsearch-plugin list | grep -q repository-s3; then
        echo "S3 plugin not found. Installing..."
        /usr/share/elasticsearch/bin/elasticsearch-plugin install --batch repository-s3
        else
        echo "S3 plugin is already installed. Skipping installation."
        fi
    volumeMounts:
      - name: elasticsearch-plugins
        mountPath: /usr/share/elasticsearch/plugins

extraVolumeMounts:
  - name: plugins
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
![image](https://github.com/user-attachments/assets/fed7ee27-7941-45b3-82c9-50ff26d1d4d6)

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

#### Using the MinIO Console to create bucket and access keys

Log into the MinIO console, navigate to "Buckets," and create a new bucket named `elasticsearch-snapshots`.
![image](https://github.com/user-attachments/assets/aad091d1-c27b-44a4-b41d-57afe45f979d)


You can also create access keys directly from the console if needed.
![image](https://github.com/user-attachments/assets/a5fbd5cb-f2d3-4351-9925-e0bac49a11f6)


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
![image](https://github.com/user-attachments/assets/87cf1c78-f326-4f25-85b8-7f50808c10a2)


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
![image](https://github.com/user-attachments/assets/c67fee55-55a1-4cae-925a-b13d98239edd)

## Troubleshooting

- **Path not accessible:** Ensure MinIO is reachable from all Elasticsearch nodes.
  ```bash
  curl http://<minio-ip>:9000
  ```

- **Access denied:** Verify access and secret keys are correctly configured.



