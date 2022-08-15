# mcss-service-mesh-demo

## Overview

## Demo 

### Before you begin
Enable the right APIs for your project

```
gcloud services enable \
autopush-multiclusterservicediscovery.sandbox.googleapis.com \
gkeconnect-autopush.sandbox.googleapis.com \
gkehub.googleapis.com \
cloudresourcemanager.googleapis.com \
trafficdirector.googleapis.com \
dns.googleapis.com \
--project=$USER-gke-dev
```

### Enabling MCS on your GKE cluster

1. Create a cluster with workload identity enabled. 
```
export ZONE="us-central1-c"
export PROJECT_ID="$USER-gke-dev"
export PROJECT_NUMBER="$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')"

gcloud container clusters create mcss-demo-cluster \
    --workload-pool=${PROJECT_ID}.svc.id.goog \
    --zone=${ZONE} \
    --machine-type=e2-standard-4 \
    --num-nodes=3
```

2. Enable the MCS feature for your project's fleet and grant the required Identity and Access Management (IAM) permissions for MCS Importer
```
gcloud container hub multi-cluster-services enable \
    --project $USER-gke-dev

gcloud projects add-iam-policy-binding $USER-gke-dev \
    --member "serviceAccount:$USER-gke-dev.svc.id.goog[gke-mcs/gke-mcs-importer]" \
    --role "roles/compute.networkViewer"
```

3. Register your GKE cluster to the fleet  
```
gcloud container hub memberships register mcss-demo-cluster \
--project=${PROJECT_ID} \
--gke-cluster=${ZONE}/mcss-demo-cluster \
--enable-workload-identity
```

4. Authorize k8s cluster access via GKE Hub
```
gcloud container clusters get-credentials mcss-demo-cluster --zone=${ZONE}
kubectl create clusterrolebinding hw-codelab-admin \
    --user=gkehub-codelab@$USER-gke-dev.iam.gserviceaccount.com --clusterrole=cluster-admin
kubectl create clusterrole hw-codelab-impersonator \
    --verb=impersonate --resource=users \
    --resource-name=gkehub-codelab@$USER-gke-dev.iam.gserviceaccount.com
kubectl create clusterrolebinding hw-codelab-impersonator \
    --serviceaccount gke-connect:connect-agent-sa  \
    --clusterrole=hw-codelab-impersonator
```

5. Create the namespace mcss-demo.
```
kubectl create ns mcss-demo
```

### Create the statefulset with readiness sidecar container

Create the StatefulSet and other necessary configurations. 
Note: The `SERVICE_NAME` and `MEMBERSHIP_NAME` [environment variables](https://kubernetes.io/docs/tasks/inject-data-application/define-interdependent-environment-variables/#define-an-environment-dependent-variable-for-a-container) in the readiness container must match the service name in `service.yaml` and the membership name used to register your GKE Cluster to the fleet. 
```
kubectl apply -f configmap.yaml  -n mcss-demo
kubectl apply -f service.yaml  -n mcss-demo
kubectl apply -f sidecar-role-binding.yaml -n mcss-demo
kubectl apply -f statefulset.yaml  -n mcss-demo 
```

Initialize the Redis cluster
```
kubectl exec -it redis-0 -c redis -- redis-cli --cluster create --cluster-replicas 1 \
  redis-0.redis-cluster:6379 redis-1.redis-cluster:6379 redis-2.redis-cluster:6379 \
  redis-3.redis-cluster:6379 redis-4.redis-cluster:6379 redis-5.redis-cluster:6379
```


#### Readiness Sidecar Overview 
The readiness sidecar in `statefulset.yaml` runs in a sidecar container along side each statefulset replica. The readiness sidecar container uses [podReadinessGates](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-readiness-gate). The podReadinessGates is defined in the statefulsets pod template spec. PodReadinessGates allow you to specify a list of additional custom conditions that the kubelet evaluates for Pod readiness. Readiness gates are determined by the current state of status.condition fields for the Pod. If Kubernetes cannot find such a condition in the status.conditions field of a Pod, the status of the condition is defaulted to `"False"`. `statfulset.yaml` defines a podReadinessGate `"www.example.com/feature-1"`. The Pod is evaluated to be Ready only when both the following statements apply:
- All containers in the Pod are ready.
- All conditions specified in readinessGates are True.

The readiness sidecar container marks the condition `"www.example.com/feature-1"` to `"True"` once the following conditions are satisfied.  Note that these conditions are for statefulset migrations that use multi-cluster services. To use your own service mesh, you would define these conditions for yourself: 
1. This pod has the correct DNS resolution: `dig -x ${POD_IP} +short == ${HOSTNAME}.${SERVICE_NAME}.${NAMESPACE}.svc.cluster.local.`. 
2. The pod MultiCluster Services domain name resolves to the pod's IP:  `dig ${HOSTNAME}.${MEMBERSHIP_NAME}.${SERVICE_NAME}.${NAMESPACE}.svc.clusterset.local == POD_IP`

If at any point during statefulset migration these conditions are false, the Pod will be marked as not Ready, and the statefulset migration will stop until these conditions are satisfied once again. The kubectl patch command does not support patching object status. To set these status.conditions for the pod, applications and operators should use the PATCH action. You can use a [Kubernetes client library](https://kubernetes.io/docs/reference/using-api/client-libraries/) to write code that sets custom Pod conditions for Pod readiness.


`POD_IP`, `NAMESPACE`, `SERVICE_NAME`, and `MEMBERSHIP_NAME` are passed to the readiness container as [environment variables](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/). StatefulSet derives its `HOSTNAME` from the name of the StatefulSet and the ordinal of the Pod. The pattern for the constructed hostname is `$(statefulset name)-$(ordinal)` ([source](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#stable-network-id)). 

#### Using Readiness Gates 
We can see the statefulset controller is waiting to create the next replica because replica-0 is not ready because the `"www.example.com/feature-1"` condition has not been set to True.
```
kubectl get pods -o wide -n mcss-demo 
NAME      READY   STATUS    RESTARTS   AGE   IP           NODE                                               NOMINATED NODE   READINESS GATES
redis-0   2/2     Running   0          19s   10.68.1.10   gke-mcss-demo-cluster-default-pool-8aa9db9a-zndk   <none>           1/2

kubectl get pods redis-0 -n mcss-demo -o yaml
kind: Pod
...
spec:
  readinessGates:
  - conditionType: "www.example.com/feature-1"
... 
status:
  conditions:
    - lastProbeTime: null
      lastTransitionTime: "2022-08-15T22:26:24Z"
      status: "True"
      type: Initialized
    - lastProbeTime: null
      lastTransitionTime: "2022-08-15T22:26:24Z"
      message: corresponding condition of pod readiness gate "www.example.com/feature-1"
        does not exist.
      reason: ReadinessGatesNotReady
      status: "False"
      type: Ready
```

Once the conditions above have been satisfied, redis-0 replica is marked as Ready, and the StatefulSet controller proceeds to create replica redis-1. 
``` 
kubectl get pods -o wide -n mcss-demo 
NAME      READY   STATUS    RESTARTS   AGE   IP           NODE                                               NOMINATED NODE   READINESS GATES
redis-0   2/2     Running   0          87s   10.68.1.10   gke-mcss-demo-cluster-default-pool-8aa9db9a-zndk   <none>           2/2
redis-1   2/2     Running   0          34s   10.68.0.7    gke-mcss-demo-cluster-default-pool-8aa9db9a-x4d4   <none>           1/2

kubectl get pods redis-0 -n mcss-demo -o yaml
kind: Pod
...
spec:
  readinessGates:
  - conditionType: "www.example.com/feature-1"
... 
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2022-08-16T01:33:35Z"
    status: "True"
    type: www.example.com/feature-1
  - lastProbeTime: null
    lastTransitionTime: "2022-08-16T01:33:35Z"
    status: "True"
    type: Ready
```

