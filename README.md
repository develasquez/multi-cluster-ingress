# This is not an official Google project.

This script is for educational purposes only, is not certified and is not recommended for production environments.

## Copyright 2021 Google LLC
Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0
Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

---


# Multi Cluster Ingress Demo

## Documentation

https://cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress-setup

![diagram](/mci-diagram.png)

## Set Vars

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
export PROJECT_NUMBER=$(gcloud projects list \
    --filter=${PROJECT_ID} --format="value(PROJECT_NUMBER)")

export REGION_1=us-central1
export REGION_2=europe-west1
export ZONE=b
export SUBNET_RANGE_1=10.128.0.0/20
export SUBNET_RANGE_2=10.130.0.0/20
```

## Enable Apis

```bash
​​gcloud services enable \
    orgpolicy.googleapis.com \
    anthos.googleapis.com \
    multiclusteringress.googleapis.com \
    multiclusterservicediscovery.googleapis.com \
    gkehub.googleapis.com \
    container.googleapis.com \
    --project=$PROJECT_ID
```


## Create VPCs and Subnets

```bash
gcloud compute networks create fleet-vpc \
--project=$PROJECT_ID \
--subnet-mode=custom \
--mtu=1460 \
--bgp-routing-mode=regional

gcloud compute networks subnets create us-fleet-subnet \
--project=$PROJECT_ID \
--range=$SUBNET_RANGE_1 \
--network=fleet-vpc \
--region=$REGION_1

gcloud compute networks subnets create eu-fleet-subnet \
--project=$PROJECT_ID \
--range=$SUBNET_RANGE_2 \
--network=fleet-vpc \
--region=$REGION_2


gcloud compute networks subnets create asia-fleet-subnet \
--project=$PROJECT_ID \
--range=$SUBNET_RANGE_3 \
--network=fleet-vpc \
--region=$REGION_3
``` 


## Enable Organization Policies 


```bash
cat > os_login.yaml << ENDOFFILE
name: projects/$PROJECT_NUMBER/policies/compute.requireOsLogin
spec:
  rules:
  - enforce: false
ENDOFFILE

gcloud org-policies set-policy os_login.yaml 

cat > shieldedVm.yaml << ENDOFFILE
name: projects/$PROJECT_NUMBER/policies/compute.requireShieldedVm
spec:
  rules:
  - enforce: false
ENDOFFILE

gcloud org-policies set-policy shieldedVm.yaml 

cat > vmCanIpForward.yaml << ENDOFFILE
name: projects/$PROJECT_NUMBER/policies/compute.vmCanIpForward
spec:
  rules:
  - allowAll: true
ENDOFFILE

gcloud org-policies set-policy vmCanIpForward.yaml


cat > vmExternalIpAccess.yaml << ENDOFFILE
name: projects/$PROJECT_NUMBER/policies/compute.vmExternalIpAccess
spec:
  rules:
  - allowAll: true
ENDOFFILE

gcloud org-policies set-policy vmExternalIpAccess.yaml

cat > restrictVpcPeering.yaml << ENDOFFILE
name: projects/$PROJECT_NUMBER/policies/compute.restrictVpcPeering
spec:
  rules:
  - allowAll: true
ENDOFFILE

gcloud org-policies set-policy restrictVpcPeering.yaml
```

## Create Clusters in US and EU

```bash
gcloud beta container clusters create "gke-us" \
--zone "${REGION_1}-$ZONE" \
--machine-type "n1-standard-2" \
--disk-size "10" \
--num-nodes "3" \
--enable-private-nodes \
--master-ipv4-cidr "172.16.0.0/28" \
--release-channel=stable \
--enable-ip-alias \
--workload-pool=$PROJECT_ID.svc.id.goog \
--network "projects/$PROJECT_ID/global/networks/fleet-vpc" \
--subnetwork "projects/$PROJECT_ID/regions/$REGION/subnetworks/us-fleet-subnet" \
--node-locations "${REGION_1}-$ZONE"

gcloud beta container clusters create "gke-eu" \
--zone "${REGION_2}-$ZONE" \
--machine-type "n1-standard-2" \
--disk-size "10" \
--num-nodes "3" \
--enable-private-nodes \
--master-ipv4-cidr "172.18.0.0/28" \
--release-channel=stable \
--enable-ip-alias \
--workload-pool=$PROJECT_ID.svc.id.goog \
--network "projects/$PROJECT_ID/global/networks/fleet-vpc" \
--subnetwork "projects/$PROJECT_ID/regions/$REGION/subnetworks/eu-fleet-subnet" \
--node-locations "${REGION_2}-$ZONE"
```



## Enable cluster config from your local environment

```bash
export MY_IP=$(curl ipinfo.io/ip)
gcloud container clusters update "gke-us" \
    --enable-master-authorized-networks \
    --master-authorized-networks $MY_IP/32 \
    --zone ${REGION_1}-${ZONE}

gcloud container clusters update "gke-eu" \
    --enable-master-authorized-networks \
    --master-authorized-networks $MY_IP/32 \
    --zone ${REGION_2}-${ZONE}
```


## Config Credentials in your local environment

```bash
gcloud container clusters get-credentials gke-us \
    --zone=${REGION_1}-$ZONE \
    --project=$PROJECT_ID

gcloud container clusters get-credentials gke-eu \
    --zone=${REGION_2}-$ZONE \
    --project=$PROJECT_ID

kubectl config rename-context gke_${PROJECT_ID}_${REGION_1}-${ZONE}_gke-us gke-us
kubectl config rename-context gke_${PROJECT_ID}_${REGION_2}-${ZONE}_gke-eu gke-eu
```


## Enroll clusters as members in Workload-identity

```bash
gcloud container hub memberships register gke-us \
    --gke-cluster ${REGION_1}-${ZONE}/gke-us \
    --enable-workload-identity \
    --project=$PROJECT_ID

gcloud container hub memberships register gke-eu \
    --gke-cluster ${REGION_2}-${ZONE}/gke-eu \
    --enable-workload-identity \
    --project=$PROJECT_ID

gcloud container hub memberships list --project=$PROJECT_ID
```



## Enable Ingress

```bash
gcloud beta container hub ingress enable \
  --config-membership=gke-us
```

## Create and Apply K8s Manifests

```yaml
cat > namespace.yaml << ENDOFFILE
apiVersion: v1
kind: Namespace
metadata:
  name: zoneprinter
ENDOFFILE
```

```bash
kubectl config use-context gke-us

kubectl apply -f namespace.yaml

kubectl config use-context gke-eu

kubectl apply -f namespace.yaml
```

```yaml
cat > deploy.yaml << ENDOFFILE
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zone-ingress
  namespace: zoneprinter
  labels:
    app: zoneprinter
spec:
  selector:
    matchLabels:
      app: zoneprinter
  template:
    metadata:
      labels:
        app: zoneprinter
    spec:
      containers:
      - name: frontend
        image: gcr.io/google-samples/zone-printer:0.2
        ports:
        - containerPort: 8080
ENDOFFILE
```

```bash
kubectl config use-context gke-us

kubectl apply -f deploy.yaml

kubectl config use-context gke-eu

kubectl apply -f deploy.yaml
```

```yaml
cat > mcs.yaml << ENDOFFILE
apiVersion: networking.gke.io/v1
kind: MultiClusterService
metadata:
  name: zone-mcs
  namespace: zoneprinter
spec:
  template:
    spec:
      selector:
        app: zoneprinter
      ports:
      - name: web
        protocol: TCP
        port: 8080
        targetPort: 8080
ENDOFFILE
```

```bash
kubectl config use-context gke-us

kubectl apply -f mcs.yaml
```

```yaml
cat > mci.yaml << ENDOFFILE
apiVersion: networking.gke.io/v1
kind: MultiClusterIngress
metadata:
  name: zone-ingress
  namespace: zoneprinter
spec:
  template:
    spec:
      backend:
        serviceName: zone-mcs
        servicePort: 8080
ENDOFFILE
```

```bash
kubectl config use-context gke-us

kubectl apply -f mci.yaml
```

## Test your Multi Cluster Ingress

```bash
kubectl describe mci zone-ingress -n zoneprinter
```
Copy VIP address from result

```bash
curl <INGRESS_VIP>/ping
```

## Test scaling down nodes in any cluster

```bash
while true; do sleep 0.1; curl http://<INGRESS_VIP>/ping; echo -e '\n';done
```


## [OPTIONAL] Test from other location (VM in some region)

```bash
gcloud compute networks create other-vpc \
--project=$PROJECT_ID \
--subnet-mode=custom \
--mtu=1460 \
--bgp-routing-mode=regional

export SUBNET_RANGE_3=172.16.0.0/20

# set your desired region 
export REGION_3=europe-north1

gcloud compute networks subnets create other-vpc-subnet \
--project=$PROJECT_ID \
--range=$SUBNET_RANGE_3 \
--network=other-vpc \
--region=$REGION_3

export MY_IP=$(curl ipinfo.io/ip)

gcloud compute firewall-rules create ssh \
--action allow \
--rules tcp:22 \
--network=other-vpc \
--target-tags=ssh \
--source-ranges=$MY_IP/32

gcloud compute instances create other-vm --project=$PROJECT_ID \
 --zone=${REGION_3}-${ZONE} \
 --machine-type=f1-micro \
 --subnet=other-vpc-subnet \
 --tags=ssh 

gcloud compute ssh other-vm --zone=${REGION_3}-${ZONE}

curl <INGRESS_VIP>/ping
```

