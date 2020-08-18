# secure-network-in-gke

Secure the network of GKE cluster with Calico.

This guide provides examples how to create GKE cluster and install [Calico OSS](https://www.projectcalico.org/) or [Calico Enterprise](https://www.tigera.io/). It provides several examples to secure Kubernetes network using `k8s` and `Calico` network policies.

The `gcloud` CLI is used to create GKE cluster and manage necessary resources. If you don't have it installed, refer to Google documentation to install and configure [Google Cloud SDK](https://cloud.google.com/sdk/install) that includes `gcloud` tool.

For more details about networking options for Kubernetes clusters on GCP refer to [Everything you need to know about Kubernetes networking on Google Cloud](https://www.projectcalico.org/everything-you-need-to-know-about-kubernetes-networking-on-google-cloud/) blog post.

## create GKE cluster with Calico OSS for network policy

GKE provides built-in support for network policy enforcement via Calico OSS. One can [enable Calico network policies in GKE cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/network-policy) by enabling network policy capability during cluster installation or enabling it post install using `--enable-network-policy` option.

>The default GKE networking option, i.e. **without** `--enable-network-policy` setting, uses `kubenet` networking plugin with `Host-Local IPAM`. GKE cluster is configured with `VPC-Native` option which configures the GCP host network to allow cross-host communications for Pods.

>When GKE cluster is created with network policy feature enabled, i.e. **using** `--enable-network-policy` setting, then `Calico CNI` is used with `Host-Local IPAM`. Calico version is managed by GKE and cannot be replaced or upgraded outside of what GKE operator offers.

```bash
# set vars
GOOGLE_ZONE='us-west1-a' # us-west-1{a,b,c} - Oregon location, us-west-2{a,b,c} - Los Angeles location
NODE_LOCATIONS='us-west1-a,us-west1-b,us-west1-c' # one can specify several availability zone locations
GOOGLE_PROJECT='<google-project-name>' # google project name
GOOGLE_MACHINE_TYPE='e2-standard-2' # e.g. n1-standard-2 (local SSD), n2-standard-2 (local SSD), e2-standard-2 (no SSD)
# OS_IMAGE_TYPE='COS' # container optimized OS is default
KUBERNETES_VERSION='1.16.13-gke.1' # https://cloud.google.com/run/docs/gke/cluster-versions
NUM_NODES=2
CLUSTER_NAME='calico-wbnr'
USER_LABEL='calico'
RELEASE_CHANNEL='regular' # rapid, regular, stable
NET_TAGS="${CLUSTER_NAME}"
# provision GKE cluster
gcloud beta container clusters create ${CLUSTER_NAME} \
  --enable-network-policy \
  --project=${GOOGLE_PROJECT} --zone=${GOOGLE_ZONE} \
  --num-nodes=${NUM_NODES} \
  --machine-type=${GOOGLE_MACHINE_TYPE} \
  --cluster-version=${KUBERNETES_VERSION} \
  --release-channel=${RELEASE_CHANNEL} \
  --labels=prefix=${CLUSTER_NAME},created_by=${USER_LABEL} \
  --tags ${NET_TAGS} \
  --quiet --verbosity debug

# get kubeconfig
export KUBECONFIG=$(pwd)/kubeconfig
gcloud beta container clusters get-credentials ${CLUSTER_NAME} --project ${GOOGLE_PROJECT} --zone=${GOOGLE_ZONE} --quiet --verbosity debug
```

## create GKE cluster with Calico Enterprise for network policy

Calico Enterprise can be installed in GKE cluster if the cluster has [Intranode visibility](https://cloud.google.com/kubernetes-engine/docs/how-to/intranode-visibility) option enabled. It adds `GCP netd` networking option with Calico and `host-local IPAM`.

>When `netd` networking option is used, the traffic is always routed to the VPC and then to the destination Pod. This happens even if both source and destination Pods are located on the same host. This allows the Pod-to-Pod communications to be recorded in the GCP flow logs. Calico Enterprise requires `netd` networking option to be enabled in GKE cluster.

Provision GKE cluster using `--enable-intra-node-visibility` option.

```bash
# set vars
GOOGLE_ZONE='us-west1-a' # us-west-1{a,b,c} - Oregon location, us-west-2{a,b,c} - Los Angeles location
NODE_LOCATIONS='us-west1-a,us-west1-b' # one can specify several availability zone locations
GOOGLE_PROJECT='<google-project-name>' # google project name
GOOGLE_MACHINE_TYPE='e2-standard-2' # e.g. n1-standard-2 (local SSD), n2-standard-2 (local SSD), e2-standard-2 (no SSD)
# OS_IMAGE_TYPE='COS' # container optimized OS is default
KUBERNETES_VERSION='1.16.13-gke.1' # https://cloud.google.com/run/docs/gke/cluster-versions
NUM_NODES=2
CLUSTER_NAME='calient-wbnr'
USER_LABEL='calico'
RELEASE_CHANNEL='regular' # rapid, regular, stable
NET_TAGS="${CLUSTER_NAME}"
NET_NAME='wbnr-vpc'

# create dedicated VPN
gcloud compute networks create ${NET_NAME} --project=${GOOGLE_PROJECT} --subnet-mode=custom --bgp-routing-mode=regional
# create subnets
# subnet range must not overlap with other subnets for selected region
gcloud compute networks subnets create ${NET_NAME}-subnet1 --project=${GOOGLE_PROJECT} --range=10.8.0.0/16 --network=${NET_NAME} --region=us-west1 --enable-private-ip-google-access
gcloud compute networks subnets create ${NET_NAME}-subnet2 --project=${GOOGLE_PROJECT} --range=10.9.0.0/16 --network=${NET_NAME} --region=us-west2 --enable-private-ip-google-access
# get network reference
NET_REF=$(gcloud compute networks list --filter="name=${NET_NAME}" --format="value(selfLink)")
# get subnet reference
# e.g. --subnetwork "projects/test-project/regions/us-west2/subnetworks/subnet2"
SUBNET_NAME="${NET_NAME}-subnet1"
SUB_REF=$(gcloud compute networks subnets list --filter="name=${SUBNET_NAME} AND network=${NET_REF}" --format="value(selfLink)")

# provision GKE cluster
gcloud beta container clusters create ${CLUSTER_NAME} \
  --enable-intra-node-visibility \
  --project=${GOOGLE_PROJECT} --zone=${GOOGLE_ZONE} \
  --num-nodes=${NUM_NODES} \
  --machine-type=${GOOGLE_MACHINE_TYPE} \
  --cluster-version=${KUBERNETES_VERSION} \
  --release-channel=${RELEASE_CHANNEL} \
  --network "${NET_REF}" \
  --subnetwork "${SUB_REF}" \
  --labels=prefix=${CLUSTER_NAME},created_by=${USER_LABEL} \
  --tags ${NET_TAGS} \
  --quiet --verbosity debug

# get kubeconfig
export KUBECONFIG=$(pwd)/kubeconfig
gcloud beta container clusters get-credentials ${CLUSTER_NAME} --project ${GOOGLE_PROJECT} --zone=${GOOGLE_ZONE} --quiet --verbosity debug
```

Install Calico Enterprise following Tigera [installation documentation](https://docs.tigera.io/getting-started/kubernetes/managed-public-cloud/gke).

Create a service account (e.g. `jane`) to [configure UI access](https://docs.tigera.io/getting-started/cnx/create-user-login) and manage Calico Enterprise resources.

```bash
# create service account and configure role binding for it
kubectl create sa jane -n default
kubectl create clusterrolebinding jane-access --clusterrole tigera-network-admin --serviceaccount default:jane
```

Deploy Calico Enterprise Manager NodePort service as a simple option to access the Enterprise Manager UI.

```bash
kubectl apply -f calico/tigera-manager-np-svc.yml
```

Create firewall rule for Calico Enterprise Manager NodePort service.

>The command below uses `CLUSTER_NAME` variable to assign a network tag to the firewall rule. This variable was used to create GKE cluster and assign the network tag to the cluster instances. If required, modify or add other tags that can be used in firewall rules.

```bash
FW_PORT=31443 # nodePort to access Calico Enterprise Manager service
FW_RULE="${NET_NAME}-allow-ingress-${FW_PORT}"
gcloud compute firewall-rules create ${FW_RULE} \
  --description="${NET_NAME} allow Enterprise Manager service nodePort ${FW_PORT}" --direction=INGRESS --priority=1000 \
  --network=${NET_NAME} --action=ALLOW --rules=tcp:${FW_PORT} \
  --source-ranges=0.0.0.0/0 --target-tags=${CLUSTER_NAME}
```

## explore node networking configuration

Regardless of which networking option you choose for GKE cluster, you can SSH into a GKE node and explore its Kubernetes networking configuration.

```bash
# open SSH port
FW_PORT=22
FW_RULE="${CLUSTER_NAME}-allow-ingress-${FW_PORT}"
gcloud compute firewall-rules create ${FW_RULE} \
  --description="${CLUSTER_NAME} allow port ${FW_PORT}" --direction=INGRESS --priority=1000 \
  --network=${NET_NAME} --action=ALLOW --rules=tcp:${FW_PORT} \
  --source-ranges=0.0.0.0/0 --target-tags=${CLUSTER_NAME}

# get node name

# SSH into the node
NODE_NAME='gke-xxxxx' # get node name from GCP console
ZONE='us-west1-a'
gcloud compute ssh ${NODE_NAME} --zone ${ZONE}

# once in the SSH session, list CNI configuration files
ls /etc/cni/net.d
```

## clean up resources

Delete GKE cluster and created resources.

```bash
GOOGLE_ZONE='us-west1-a'
GOOGLE_PROJECT='<google-project-name>'
CLUSTER_NAME='calient-wbnr'
# cleanup cluster resources
gcloud beta container clusters delete $CLUSTER_NAME \
  --project ${GOOGLE_PROJECT} --zone $GOOGLE_ZONE \
  --async --quiet --verbosity debug || true

# delete firewall rule
gcloud compute firewall-rules delete ${FW_RULE} --quiet

# delete created subnets
gcloud compute networks subnets delete "${NET_NAME}-subnet1" --region us-west1 --quiet

# delete vpc
gcloud compute networks delete ${NET_NAME} --quiet
```

## demo scenarios

The demo scenarios use several PODs and a few policy examples. The folders `30-*` and `35-*` are intended for Calico Enterprise scenarios as they use a custom policy tier.

Use [calicoctl](https://docs.projectcalico.org/getting-started/clis/calicoctl/install) to work with policies in `Calico OSS` cluster. You can use `kubectl` for `Calico Enterprise` cluster.

### `Calico OSS` demo scenario

>Calico OSS demo scenario can be used in both Calico OSS and Calico Enterprise clusters.

- Deploy sample application.

  ```bash
  # deploy app components
  kubectl apply -f demo/10-app/
  # view deployed components
  kubectl -n demo get pod
  ```

  The sample app consists of `nginx` deployment, `centos` standalone POD, and `netshoot` standalone POD. The `nginx` instances serve static HTML page. The `centos` instance queries an external resource, `www.google.com`. The `netshoot` instance queries the `nginx` service with name `nginx-svc`.

- Attach to the log stream of `centos` and `netshoot` PODs to confirm that both processes can get HTTP 200 response. Then deploy Kubernetes default deny policy for `demo` namespace.

  ```bash
  # attach to PODs log streams
  kubectl -n demo logs -f centos
  kubectl -n demo logs -f netshoot
  # deploy k8s default deny policy
  kubectl apply -f demo/20-default-deny/policy-default-deny-k8s.yaml
  ```

  After the policy is applied, both `centos` and `netshoot` processes should not be able to query the targeted resources.

- Allow `centos` to query external resource.

  ```bash
  # deploy centos allow policy
  DATASTORE_TYPE=kubernetes calicoctl apply -f demo/25-sample-policy/allow-centos-egress.yaml
  ```

  Once policy is applied, the `centos` POD should be able to access `www.google.com` resource.

- Allow DNS lookups for entire `demo` namespace and `nginx` service access for `netshoot` component.

  ```bash
  # allow cluster DNS lookups
  DATASTORE_TYPE=kubernetes calicoctl apply -f demo/25-sample-policy/allow-cluster-dns.yaml
  # allow nginx ingress
  DATASTORE_TYPE=kubernetes calicoctl apply -f demo/25-sample-policy/allow-nginx-ingress.yaml
  # allow netshoot egress
  DATASTORE_TYPE=kubernetes calicoctl apply -f demo/25-sample-policy/allow-port80-egress.yaml
  ```

  Once all three policies are applied, the `netshoot` POD should be able to get response from the `nginx-svc` cluster service.

### `Calico Enterprise` demo scenario

>Calico Enterprise demo scenario can be used only in Calico Enterprise clusters as it uses enterprise features.

- Deploy sample application.

  ```bash
  # deploy app components
  kubectl apply -f demo/10-app/
  # view deployed components
  kubectl -n demo get pod
  ```

  The sample app consists of `nginx` deployment, `centos` standalone POD, and `netshoot` standalone POD. The `nginx` instances serve static HTML page. The `centos` instance queries an external resource, `www.google.com`. The `netshoot` instance queries the `nginx` service with name `nginx-svc`.

- Deploy a new policy tier.

  ```bash
  # deploy security tier
  kubectl apply -f demo/30-calient-tier/
  ```

- Attach to the log stream of `centos` and `netshoot` PODs to confirm that both processes can get HTTP 200 response. Then deploy Calico default deny policy for `demo` namespace.

  ```bash
  # attach to PODs log streams
  kubectl -n demo logs -f centos
  kubectl -n demo logs -f netshoot
  # deploy Calico default deny policy
  kubectl apply -f demo/35-dns-policy/policy-default-deny-calico.yaml
  ```

  Once default deny policy takes affect, both `centos` and `netshoot` instances, should not be able to reach the targeted resources.

- Deploy DNS policy to allow access to external resource

  ```bash
  # deploy network sets resources
  kubectl apply -f demo/35-dns-policy/global-netset.yaml
  kubectl apply -f demo/35-dns-policy/public-ip-netset.yaml
  # deploy DNS policy
  kubectl apply -f demo/35-dns-policy/policy-allow-dns-netset-egress.yaml
  ```

  Deploy k8s network policy to allow `netshoot` pod access `nginx-svc` service

  ```bash
  # allow nginx ingress
  kubectl apply -f demo/25-sample-policy/allow-nginx-ingress.yaml
  # allow netshoot egress
  kubectl apply -f demo/25-sample-policy/allow-port80-egress.yaml
  ```

- Test `www.google.com` access

  ```bash
  # curl www.google.com from centos POD
  kubectl -n demo exec -t $(kubectl get pod -l app=centos -n demo -o jsonpath='{.items[*].metadata.name}') -- curl -ILs http://www.google.com -m 3 | grep -i http
  # curl www.google.com from netshoot POD
  kubectl -n demo exec -t $(kubectl get pod -l app=netshoot -n demo -o jsonpath='{.items[*].metadata.name}') -- curl -ILs http://www.google.com -m 3 | grep -i http
  ```

  Try to `curl` any other external resource. You should not be able to access it as `allow-dns-netset-egress` policy explicitly denies access to public IP ranges listed in `public-ip-range` global network set.
