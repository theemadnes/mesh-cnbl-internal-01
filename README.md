# mesh-cnbl-internal-01
combining mesh ingress gateways with internal https LB on GKE using Istio and Gateway APIs. Istio APIs are used within the mesh, and Gateway APIs are used for north/south networking to get requests to the mesh. 
this could be reproduced using Gateway API for everything, but i'll leave that as an exercise for the reader for now.

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