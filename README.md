# mesh-cnbl-internal-01
combining mesh ingress gateways with internal https LB on GKE using Istio and Gateway APIs. Istio APIs are used within the mesh, and Gateway APIs are used for north/south networking to get requests to the mesh. 
this could be reproduced using Gateway API for everything, but i'll leave that as an exercise for the reader for now.

the Gateway API controller is automatically installed on GKE Autopilot clusters, but must be manually enabled for GKE Standard clusters; see https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-gateways#enable-gateway-existing-cluster

we'll test access using a GCE VM to call mesh-based services, traversing an internal HTTP proxy LB

this also assumes that all necessary APIs have been enabled within the project

### setup

```
# set up envvars
export PROJECT_ID=mesh-internal-http-pxy-test-01 # replace with your project; will also be used as the fleet name
export VPC=default # replace with your vpc
export CLUSTER_SUBNET=default # replace with your subnet
export REGION=us-central1 # replace with your region
export CLUSTER_NAME=my-mesh-cluster # replace with your cluster name
export KUBECTX=gke_${PROJECT_ID}_${REGION}_${CLUSTER_NAME}

gcloud config set project $PROJECT_ID # set your project id to be the default for gcloud commands
```

### create cluster and enable service mesh (CSM)

This is typically done programmatically, but we'll do it manually for now, and just for the cluster we're using for this demo.

```
# create GKE Autopilot cluster - note that this is a public cluster (nodes have public IPs, so not suitable for production)
gcloud beta container --project $PROJECT_ID clusters create-auto $CLUSTER_NAME \
    --region $REGION --release-channel "regular" --tier "standard" --enable-dns-access --enable-ip-access \
    --no-enable-google-cloud-access --network "projects/${PROJECT_ID}/global/networks/${VPC}" --subnetwork "projects/${PROJECT_ID}/regions/${REGION}/subnetworks/${CLUSTER_SUBNET}" \
    --cluster-ipv4-cidr "/17" --binauthz-evaluation-mode=DISABLED --fleet-project=$PROJECT_ID

# enable CSM for the fleet & cluster
gcloud services enable mesh.googleapis.com \
    --project=$PROJECT_ID

gcloud container fleet mesh enable --project $PROJECT_ID

gcloud container fleet mesh update \
    --management automatic \
    --memberships ${CLUSTER_NAME}

# after a few minutes, verify that the service mesh control plane has been deployed in your fleet
gcloud container fleet mesh describe --project $PROJECT_ID
```

### create proxy-only subnet for HTTP(S) LB

```
gcloud compute networks subnets create proxy-only-subnet-${REGION} \
    --project=$PROJECT_ID \
    --purpose=REGIONAL_MANAGED_PROXY \
    --role=ACTIVE \
    --region=$REGION \
    --network=$VPC \
    --range=192.168.0.0/24 # replace with your own range
```

### deploy the ingress gateway pods and service

in this example, i'm just creating the IG service up as ClusterIP, but any service type will work, since the NEGs will just reference the pods anyway

```
# create namespace and label it for injection
kubectl create namespace ingress-gateway
kubectl label namespace ingress-gateway istio-injection=enabled
kubectl apply -k ingress-gateway/variant
```

### deploy the sample workload (whereami) pods and service

```
kubectl create namespace whereami
kubectl label namespace whereami istio-injection=enabled
kubectl apply -k whereami/variant

# also create the Istio virtual service for whereami
kubectl apply -f whereami-virtualservice/
```

### create HTTP load balancer and reference the ingress gateway service

the NEG controller will automatically create the NEGs to create the mapping between the load balancer and the ingress gateway pods

```
# this creates the gateway, httproute, and custom health check objects
kubectl apply -f gateway-api-load-balancer/

# after a few moments, capture the IP address of the internal load balancer
GATEWAY_IP=$(kubectl get gateway internal-http -n ingress-gateway -o jsonpath='{.status.addresses[?(@.type=="IPAddress")].value}')
echo "Gateway IP: $GATEWAY_IP"
```

test by creating a virtual machine in the same project / vpc / region as the load balancer in the console, and log into it 

```
# replace with your gateway IP, and run this command from the VM
curl --header 'Host: whereami.mesh.example.com' http://10.128.0.8
```

result:

```
$ curl --header 'Host: whereami.mesh.example.com' http://10.128.0.8 -s|jq
{
  "cluster_name": "my-mesh-cluster",
  "gce_instance_id": "4236041537522405134",
  "gce_service_account": "mesh-internal-http-pxy-test-01.svc.id.goog",
  "host_header": "whereami.mesh.example.com",
  "metadata": "whereami",
  "node_name": "gk3-my-mesh-cluster-pool-2-973644bc-np7s",
  "pod_ip": "10.59.129.2",
  "pod_name": "whereami-988cc4ddf-pk4xw",
  "pod_name_emoji": "🛝",
  "pod_namespace": "whereami",
  "pod_service_account": "whereami",
  "project_id": "mesh-internal-http-pxy-test-01",
  "timestamp": "2025-01-09T02:54:50",
  "zone": "us-central1-b"
}
```