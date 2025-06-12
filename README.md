# MongoDB Controllers on OpenShift (CRC) - Demo Quick Guide

This repository provides step-by-step instructions to run a MongoDB Kubernetes Controller demo on OpenShift using CodeReady Containers (CRC). The environment is pre-baked in a GCP image stored in a shared bucket.

---

## 🧰 Prerequisites

* GCP Project with Compute Engine enabled
* Access to GCS bucket: `ra-opsmanager-daisy-bkt-us-central1`
* `gcloud` CLI installed and authenticated
* Permissions to create custom images, VMs, and resource policies

---

## 📥 Step 1: Import the Image from GCS

```bash
export IMAGE_NAME=mongodb-demo-crc
export BUCKET_NAME=ra-opsmanager-daisy-bkt-us-central1
export OBJECT_PATH=daisy-image-export-20250603-01:54:48-mhdwy/outs/image-export-export-disk.tar.gz
export PROJECT_ID=your-cloning-project-id

gcloud compute images create $IMAGE_NAME \
  --source-uri gs://$BUCKET_NAME/$OBJECT_PATH \
  --project=$PROJECT_ID
```

---

## 🖥️ Step 2: Create a VM from the Custom Image

```bash
gcloud compute instances create crc-mongodb-demo \
  --project=$PROJECT_ID \
  --zone=us-central1-c \
  --enable-nested-virtualization \
  --min-cpu-platform="Intel Haswell" \
  --machine-type=n1-standard-8 \
  --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=default \
  --maintenance-policy=MIGRATE \
  --provisioning-model=STANDARD \
  --service-account=1061005126199-compute@developer.gserviceaccount.com \
  --scopes=https://www.googleapis.com/auth/devstorage.read_only,\
https://www.googleapis.com/auth/logging.write,\
https://www.googleapis.com/auth/monitoring.write,\
https://www.googleapis.com/auth/service.management.readonly,\
https://www.googleapis.com/auth/servicecontrol,\
https://www.googleapis.com/auth/trace.append \
  --create-disk=auto-delete=yes,boot=yes,device-name=crc-mongodb-demo,\
image=projects/$PROJECT_ID/global/images/$IMAGE_NAME,mode=rw,size=300,type=pd-balanced \
  --labels=goog-ec-src=vm_add-gcloud \
  --reservation-affinity=any
```


---

## 🔐 Step 3: SSH into the VM

```bash
gcloud compute ssh crc-mongodb-demo --zone=us-central1-c
```

---

## 📦 Step 4: Inside the VM

1. Log into OpenShift:

```bash
crc start
crc console --credentials
```

2. Install MongoDB Controllers via OperatorHub UI or CLI.

3. Create a ConfigMap for Ops Manager settings (example below):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-operator-config
  namespace: mongodb
  labels:
    app.kubernetes.io/name: mongodb-enterprise-operator
    app.kubernetes.io/instance: mongodb
    app.kubernetes.io/component: controller
    app.kubernetes.io/part-of: mongodb-enterprise
    app.kubernetes.io/managed-by: mongodb-enterprise-operator
    app.kubernetes.io/version: v1.0.0
    app.kubernetes.io/created-by: crc-demo
    app.kubernetes.io/ops-manager-url: https://<your-ops-manager-url>
data:
  baseUrl: https://<your-ops-manager-url>
  orgId: <your-org-id>
```

4. Create a secret with your Ops Manager public and private keys:

```bash
kubectl create secret generic mongodb-opsmgr-secret \
  --from-literal=publicKey=<your-public-key> \
  --from-literal=privateKey=<your-private-key> \
  -n mongodb
```

5. Deploy a 3-node ReplicaSet CRD (example):

```yaml
apiVersion: mongodb.com/v1
kind: MongoDB
metadata:
  name: demo-mongodb-cluster-100
  namespace: mongodb
spec:
  members: 3
  version: 8.0.0
  type: ReplicaSet
  security:
    authentication:
      enabled: true
      modes: ["SCRAM"]
  cloudManager:
    configMapRef:
      name: my-project
  credentials: organization-secret
  persistent: true
  podSpec:
    persistence:
      single:
        storage: 10Gi
        storageClass: crc-csi-hostpath-provisioner
    podTemplate:
      spec:
       containers:
        - name: mongodb-enterprise-database
          resources:
            limits:
              cpu: 2
              memory: 1G
            requests:
              cpu: 1
              memory: 1G
            persistence:
              single:
                storage: 10Gi
                storageClass: crc-csi-hostpath-provisioner
```

6. Create a MongoDB database user (via Ops Manager or AutomationConfig if required).

---

## ✅ Demo Ready

You now have a functional 3-node MongoDB ReplicaSet running on OpenShift CRC managed by MongoDB Controllers.

---

## 📎 Resources

* [MongoDB Kubernetes Operator Docs](https://www.mongodb.com/docs/kubernetes)
* [OpenShift CRC Documentation](https://developers.redhat.com/products/codeready-containers/overview)
* [GCP CLI Reference](https://cloud.google.com/sdk/gcloud)
