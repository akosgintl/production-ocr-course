# Hands-on Lab!

To ensure everyone can execute this hands-on lab without requiring Azure GPU quota approvals, we implement this validation environment using low-cost CPU instances (Standard_D2s_v3) while preserving the exact production taint-toleration topology and inter-pod networking architecture.

## Prerequisites & Environment Variables

```bash
export LOCATION="eastus" # or your desired location
export RESOURCE_GROUP="tnm-k8s-lab-rg"
export SUBSCRIPTION_ID="XXXXXX"
export CLUSTER_NAME="tnm-lab-cluster"
export SYSTEM_POOL="systempool"
export WORKER_POOL="workerpool"
```

## Authenticate Azure CLI Session

Before proceeding, ensure your Azure CLI session is authenticated and set to the correct active subscription. This step is required for all subsequent resource creation commands.

```bash
az login
az account set --subscription "$SUBSCRIPTION_ID"
az account show --output table
```

Expected: Your intended subscription shows as IsDefault = True (or at least matches $SUBSCRIPTION_ID).

## Infrastructure Setup: AKS & Tainted Node Pools

### 1. Create the Azure Resource Group

```bash
az group create --name $RESOURCE_GROUP --location $LOCATION
```

result:

```bash
{
  "id": "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/tnm-k8s-lab-rg",
  "location": "eastus",
  "managedBy": null,
  "name": "tnm-k8s-lab-rg",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": "Microsoft.Resources/resourceGroups"
}
```

### 2. Provision AKS Cluster (System Node Pool)

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --node-count 1 \
  --nodepool-name $SYSTEM_POOL \
  --node-vm-size Standard_D2s_v3 \
  --generate-ssh-keys
```

### 3. Download AKS cluster credentials for kubectl

```bash
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME
```

### 4. Add Dedicated Workload Node Pool with Custom Taint

```bash
az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name $WORKER_POOL \
  --node-count 1 \
  --node-vm-size Standard_D2s_v3 \
  --node-taints workload=inference:NoSchedule
```

By default, specialized inference nodes are tainted with workload=inference:NoSchedule. This prevents general cluster workloads (e.g., logging daemons, ingress controllers) from consuming compute resources allocated for inference services. Only pods explicitly declaring matching tolerations can schedule onto these nodes.

To ensure the worker node pool has correctly registered and applied the NoSchedule taint, run the following inspection commands:

### 1. Check Node Status and Applied Taints

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type,TAINTS:.spec.taints
```

### 2. Inspect Node Details (Filtered Describe)

```bash
kubectl describe nodes | grep -A 3 "Taints:"
```

Expected Output:

```bash
NAME                                 STATUS   TAINTS
aks-systempool-25843338-vmss000000   Ready    <none>
aks-workerpool-37743923-vmss000000   Ready    [map[effect:NoSchedule key:workload value:inference]]

Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  aks-systempool-25843338-vmss000000
--
Taints:             workload=inference:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  aks-workerpool-37743923-vmss000000
```

## Microservice Deployment: API & ClusterIP Protection

We deploy an API server Pod requesting POSIX shared memory (/dev/shm), CPU resource limits, and a matching toleration. To enforce zero-trust network boundaries, the API is exposed strictly via an internal ClusterIP Service.

⚠️ Note: Save the manifest below as api-deployment.yaml in your local working directory before running kubectl apply.

```yaml
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ocr-inference-api
  labels:
    app: ocr-inference-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ocr-inference-api
  template:
    metadata:
      labels:
        app: ocr-inference-api
    spec:
      # Toleration allowing Pod to cross the workload=inference taint barrier
      tolerations:
        - key: "workload"
          operator: "Equal"
          value: "inference"
          effect: "NoSchedule"
      containers:
        - name: api-server
          image: hashicorp/http-echo:latest
          args:
            - "-text={\"status\":\"success\",\"service\":\"document-intelligence-api\",\"mode\":\"cpu-lab\",\"shm_mounted\":true}"
            - "-listen=:8000"
          ports:
            - containerPort: 8000
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "2Gi"
          # POSIX Shared Memory Mount (/dev/shm)
          volumeMounts:
            - mountPath: /dev/shm
              name: dshm
      volumes:
        - name: dshm
          emptyDir:
            medium: Memory
            sizeLimit: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: ocr-inference-service
spec:
  type: ClusterIP # Internal Virtual IP (Zero public internet exposure)
  selector:
    app: ocr-inference-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```

Now, let's apply the manifest and inspect the internal endpoint.

### 1. Apply deployment and service manifests

```bash
kubectl apply -f api-deployment.yaml
```

### 2. Verify service status and internal ClusterIP allocation

```bash
kubectl get service ocr-inference-service
```

Expected Output:

```bash
NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
ocr-inference-service   ClusterIP   10.0.194.88    <none>        80/TCP    12s
```

Batch Execution: Inter-Pod Communication via K8s Job
We launch a batch Job inside the cluster network. The Job resolves the internal K8s DNS name (http://ocr-inference-service.default.svc.cluster.local:80), executes the HTTP request, and logs the execution metrics.

⚠️ Note: Save the manifest below as batch-job.yaml in your local working directory before running kubectl apply.

```yaml
# batch-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: document-batch-job
spec:
  ttlSecondsAfterFinished: 120
  template:
    metadata:
      name: batch-job-runner
    spec:
      restartPolicy: Never
      containers:
        - name: job-runner
          image: curlimages/curl:latest
          command:
            - "sh"
            - "-c"
            - |
              echo "🚀 Starting Batch Document Processing Job..."
              echo "📡 Connecting to internal ClusterIP DNS: http://ocr-inference-service.default.svc.cluster.local:80"
              RESPONSE=$(curl -s http://ocr-inference-service.default.svc.cluster.local:80)
              echo "✅ Response received from internal API Pod:"
              echo "$RESPONSE"
              echo "🏁 Job completed successfully!"
```

Now, let's apply the manifest and stream the execution logs.

### 1. Submit the batch job

```bash
kubectl apply -f batch-job.yaml
```

### 2. Confirm job creation and execution status

```bash
kubectl get job document-batch-job
```

### 3. Stream execution logs from the job runner

```bash
kubectl logs -f job/document-batch-job
```

Expected Output:

```bash
job.batch/document-batch-job created

NAME                 STATUS     COMPLETIONS   DURATION   AGE
document-batch-job   Complete   1/1           4s         11s

🚀 Starting Batch Document Processing Job...
📡 Connecting to internal ClusterIP DNS: http://ocr-inference-service.default.svc.cluster.local:80
✅ Response received from internal API Pod:
{"status":"success","service":"document-intelligence-api","mode":"cpu-lab","shm_mounted":true}
🏁 Job completed successfully!
```

## Teardown & Resource Cleanup

Once validation is complete, purge the Kubernetes manifests and delete the Azure Resource Group to prevent ongoing infrastructure billing:

### 1. Delete Kubernetes workloads and services

```bash
kubectl delete -f batch-job.yaml
kubectl delete -f api-deployment.yaml
```

### 2. Purge Azure Resource Group asynchronously

```bash
az group delete --name $RESOURCE_GROUP --yes --no-wait
```

Pro Tip: The --no-wait flag allows the Azure control plane to process resource deletion asynchronously, returning control of your terminal session immediately.