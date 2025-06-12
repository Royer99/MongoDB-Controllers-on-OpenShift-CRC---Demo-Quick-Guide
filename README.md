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

1. Start the CRC cluster and wait for it to initialize:

```bash
crc start
```

Once the CRC cluster is ready, you should see output that includes cluster details and confirmation messages.

2. Export the environment to access `oc` commands:

```bash
eval $(crc oc-env)
```

3. Log into OpenShift:

```bash
crc console --credentials
```

2. Install MongoDB Controllers via OperatorHub UI or CLI.

3. Generate the required ConfigMap and Secret directly from Cloud Manager or Ops Manager:

* Navigate to **Kubernetes Setup** under your organization in Cloud Manager/Ops Manager
* Click **"Create New API Keys"** or **"Use Existing API Keys"**
* Click **"Generate Key and YAML"**
* You will receive two YAML files: `config-map.yaml` and `secret.yaml`

4. Apply them to your cluster:

```bash
kubectl apply -f secret.yaml -f config-map.yaml
```

5. Deploy a 3-node ReplicaSet CRD (example):

```yaml
apiVersion: mongodb.com/v1
kind: MongoDBReplicaSet
metadata:
  name: demo-replicaset
  namespace: mongodb
spec:
  members: 3
  version: 6.0.5
  opsManager:
    configMapRef:
      name: my-project
    credentials: organization-secret
  type: ReplicaSet
  persistent: true
  security:
    authentication:
      modes: [SCRAM]
```

6. Create a MongoDB database user (via Ops Manager or AutomationConfig if required).

7. Expose required ports using socat (if running behind NAT or port forwarding):

```bash
sudo socat TCP-LISTEN:443,fork TCP:192.168.130.11:443 &
sudo socat TCP-LISTEN:32017,fork TCP:192.168.130.11:32017 &
```

8. Ensure the following GCP firewall rules are in place:

| Name                     | Direction | Target Tags  | Ports        | Action | Network |
| ------------------------ | --------- | ------------ | ------------ | ------ | ------- |
| `egress`                 | Egress    | opsmgr       | tcp:8080     | Allow  | default |
| `mongodbegresscustom`    | Egress    | openshift    | tcp:32017    | Allow  | default |
| `mongoegress`            | Egress    | openshift    | tcp:27017    | Allow  | default |
| `openshiftegress`        | Egress    | openshift    | tcp:8443     | Allow  | default |
| `default-allow-http`     | Ingress   | http-server  | tcp:80       | Allow  | default |
| `default-allow-https`    | Ingress   | https-server | tcp:443      | Allow  | default |
| `ingress-opsmgr`         | Ingress   | opsmgr       | tcp:8080     | Allow  | default |
| `mongo`                  | Ingress   | openshift    | tcp:27017    | Allow  | default |
| `mongodbcustomport`      | Ingress   | openshift    | tcp:32017    | Allow  | default |
| `openshiftapi`           | Ingress   | openshift    | tcp:6443     | Allow  | default |
| `openshiftconsole`       | Ingress   | openshift    | tcp:8443     | Allow  | default |
| `default-allow-icmp`     | Ingress   | all          | icmp         | Allow  | default |
| `default-allow-internal` | Ingress   | all          | tcp/udp/icmp | Allow  | default |
| `default-allow-rdp`      | Ingress   | all          | tcp:3389     | Allow  | default |
| `default-allow-ssh`      | Ingress   | all          | tcp:22       | Allow  | default |

---

## ✅ Demo Ready

You now have a functional 3-node MongoDB ReplicaSet running on OpenShift CRC managed by MongoDB Controllers.

---

## 📌 Final Cleanup Reminder

Once your demo is complete, don't forget to:

```bash
# Delete the VM
gcloud compute instances delete crc-mongodb-demo --zone=us-central1-c --project=$PROJECT_ID

# Delete the custom image
gcloud compute images delete mongodb-demo-crc --project=$PROJECT_ID
```

This will help avoid unnecessary charges and free up resources.

---

## 📎 Resources

* [MongoDB Kubernetes Operator Docs](https://www.mongodb.com/docs/kubernetes)
* [OpenShift CRC Documentation](https://developers.redhat.com/products/codeready-containers/overview)
* [GCP CLI Reference](https://cloud.google.com/sdk/gcloud)
