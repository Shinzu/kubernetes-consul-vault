# Vault with Consul Backend On Kubernetes

This is a combination of three repos [kelseyhightower/consul-on-kubernetes](https://github.com/kelseyhightower/consul-on-kubernetes), [drud/vault-consul-on-kube](https://github.com/drud/vault-consul-on-kube) and [h2ik/consul-vault-kubernetes](https://github.com/h2ik/consul-vault-kubernetes). 

To the Consul deployment are ACL's added from the last repo.

This first starts with taking the Consul StatefulSets and upgrading them to 1.1.2 and then taking the vault deployments from drud/vault-consul-on-kube and modifing them to work with the statefulset deployments.

Update: I added some manifests to deploy vault also as StatefulSet in addition with automatic initialization and unsealing, this is inspired from [kelseyhightower/vault-on-google-kubernetes-engine](https://github.com/kelseyhightower/vault-on-google-kubernetes-engine) and [sethvargo/vault-on-gke](https://github.com/sethvargo/vault-on-gke). Thanks to both for this.

# Running Consul on Kubernetes

This tutorial will walk you through deploying a three (3) node [Consul](https://www.consul.io) cluster on Kubernetes.

## Overview

* Three (3) node Consul cluster using a [StatefulSet](http://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets)
* Secure communication between Consul members using [TLS and encryption keys](https://www.consul.io/docs/agent/encryption.html)

## Prerequisites

This tutorial leverages features available in Kubernetes 1.10.0 and later.

* [kubernetes](http://kubernetes.io/docs/getting-started-guides/binary_release) 1.10.x

The following clients must be installed on the machine used to follow this tutorial:

* [consul](https://www.consul.io/downloads.html) 1.1.0
* [cfssl](https://pkg.cfssl.org) and [cfssljson](https://pkg.cfssl.org) 1.2

### Create persistance Volumes

In my Setup i use local storage as persistence Volumes for Consul.

First we create a storage class local-storage:

```
kubectl apply -f volumes/storage_class.yaml
```

Now we create the persistenace Volumes, since the local-storage class cannot create this dynamicly you must create this folders manually on each node.

On each node create the Folder structure:

```
mkdir -p /data/storage/consul
```

Create the PV in kubernetes:

```
kubectl apply -f volumes/persistant_volume.yaml
```
 
### Generate TLS Certificates

RPC communication between each Consul member will be encrypted using TLS. Initialize a Certificate Authority (CA):

```
cfssl gencert -initca ca/ca-csr.json | cfssljson -bare ca
```

Create the Consul TLS certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca/ca-config.json \
  -profile=default \
  ca/consul-csr.json | cfssljson -bare consul
```

At this point you should have the following files in the current working directory:

```
ca-key.pem
ca.pem
consul-key.pem
consul.pem
```

### Generate the Consul Gossip Encryption Key

[Gossip communication](https://www.consul.io/docs/internals/gossip.html) between Consul members will be encrypted using a shared encryption key. Generate and store an encrypt key:

```
GOSSIP_ENCRYPTION_KEY=$(consul keygen)
```

### Create the Consul Secret and Configmap

The Consul cluster will be configured using a combination of CLI flags, TLS certificates, and a configuration file, which reference Kubernetes configmaps and secrets.

Store the gossip encryption key and TLS certificates in a Secret:

```
kubectl create secret generic consul \
  --from-literal="gossip-encryption-key=${GOSSIP_ENCRYPTION_KEY}" \
  --from-file=ca.pem \
  --from-file=consul.pem \
  --from-file=consul-key.pem
```

**Depreciated** Create Tokens for the ACL's with uuidgen and put it in the configs/server.json file.

Since Consul 0.9.1 you can bootstrap the ACL Tokens over the ACL API described [here](https://www.consul.io/docs/guides/acl.html#bootstrapping-acls).

ACL are bootstrapped [below](#setup-acl-for-consul) in the ACL Section.

Store the Consul server configuration file in a ConfigMap:

```
kubectl create configmap consul --from-file=configs/server.json
```

### Create the Consul Service

Create a headless service to expose each Consul member internally to the cluster:

```
kubectl create -f services/consul.yaml -f services/consul-http.yaml
```

### Create the Consul StatefulSet

Deploy a three (3) node Consul cluster using a StatefulSet:

```
kubectl create -f statefulsets/consul.yaml
```

Each Consul member will be created one by one. Verify each member is `Running` before moving to the next step.

```
kubectl get pods
```
```
NAME       READY     STATUS    RESTARTS   AGE
consul-0   1/1       Running   0          50s
consul-1   1/1       Running   0          29s
consul-2   1/1       Running   0          15s
```

### Setup ACL for consul

Since we are using ACL with consul we need to setup these.

First create bootstrap master token, this is a managment token:

```
#kubectl exec -it consul-0 -- curl --request PUT http://127.0.0.1:8500/v1/acl/bootstrap
{"ID":"<master token>"}
```

### Set the rules for agent token

On the ACL Page in the WebUI create a new Agent Token acl.(You can use the master token to access this)

```
node "" {
    policy = "write"
}
service "" {
    policy = "read"
}
key "lock/" {
  policy = "write"
}
```

or via ACL API

```
kubectl exec -it consul-0 -- curl \
    --request PUT \
    --header "X-Consul-Token: <master token>" \
    --data '{
        "Name": "Agent Token", 
        "Type": "client", 
        "Rules": "node \"\" { policy = \"write\"} service \"\" { policy = \"read\"} key \"lock/\" { policy = \"write\"}"
        }' http://127.0.0.1:8500/v1/acl/create
{"ID":"<agent token>"}    
```

### Apply ACL to agents

For all 3 consul nodes set the Agent Token via API.

```
kubectl exec -it consul-0 -- curl --request PUT --header "X-CONSUL-TOKEN: <master token>" --data '{"Token": "<agent token>"}' http://localhost:8500/v1/agent/token/acl_agent_token
```
### Update anonymous Token

In order to allow operation like `consul members` works without a token we can allow anonymous some operations

```
kubectl exec -it consul-0 -- curl \
    --request PUT \
    --header "X-Consul-Token: <master token>" \
    --data '{
        "ID": "anonymous", 
        "Type": "client", 
        "Rules": "node \"\" { policy = \"read\"} service \"consul\" { policy = \"read\"} key \"lock/\" { policy = \"write\"}"
        }' http://127.0.0.1:8500/v1/acl/update
{"ID":"anonymous"}
```

### Verification

At this point the Consul cluster has been bootstrapped and is ready for operation. To verify things are working correctly, review the logs for one of the cluster members.

```
kubectl logs consul-0
```

The consul CLI can also be used to check the health of the cluster. In a new terminal start a port-forward to the `consul-0` pod.

```
kubectl port-forward consul-0 8400:8400
```
```
Forwarding from 127.0.0.1:8400 -> 8400
Forwarding from [::1]:8400 -> 8400
```

Run the `consul members` command to view the status of each cluster member.

```
consul members
```
```
Node      Address          Status  Type    Build  Protocol  DC   Segment
consul-0  10.2.3.151:8301  alive   server  1.1.0  2         dc1  <all>
consul-1  10.2.2.199:8301  alive   server  1.1.0  2         dc1  <all>
consul-2  10.2.1.125:8301  alive   server  1.1.0  2         dc1  <all>
```

## Vault deployment
### Create a key that vault will use to access consul (vault-consul-key)

We'll use the consul web UI to create this, which avoids all manner of
quote-escaping problems.(Can also be done via ACL API, see above)

1. Port-forward port 8500 of <consul-0> to local: `kubectl port-forward consul-0 8500`
2. Hit http://localhost:8500/ui with browser.
3. Visit the settings page (gear icon) and enter your acl_master_token.
3. Click "ACL"
4. Add an ACL with name vault-token, type client, rules:
```
key "vault/" {
    policy = "write" 
}
service "vault" {
    policy = "write" 
}
session "" {
    policy = "write" 
}
node "" {
    policy = "write"
}
agent "" {
    policy = "write"
}
```
5. Capture the newly created vault-token and with it (example key here):
``` sh
$ kubectl create secret generic vault-consul-key --from-literal=consul-key=9f34ab90-965c-56c7-37e0-362da75bfad9
```

### TLS setup for exposed vault port

Get key and cert files for the domain vault will be exposed from. You can do this any way
that works for your deployment, including a [self-signed certificate](http://www.akadia.com/services/ssh_test_certificate.html), so long as you have a concatenated full certificate chain
in vault-combined.pem and private key in vault-key.pem :

``` sh
$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca/ca-config.json \
  -profile=default \
  ca/vault-csr.json | cfssljson -bare vault
  
$ cat vault.pem ca.pem > vault-combined.pem

$ kubectl create secret tls vaulttls --cert=vault-combined.pem --key=vault-key.pem
```

### Create services for vault

``` sh
$ kubectl apply -f services/vault-services.yaml
```

### Deploy vault deployment sets
You are now ready to deploy the vault instances:

``` sh
$ kubectl apply -f deployments/vault-1.yaml -f deployments/vault-2.yaml
```

### Apply ACL to agents

We must also apply the Agent Token to the consul container in the Vault pods.

```
kubectl exec -it  <vault pod name> --container consul-agent-client -- curl --request PUT --header "X-CONSUL-TOKEN: <master token>" --data '{"Token": "<agent token>"}' http://localhost:8500/v1/agent/token/acl_agent_token
```

### Vault Initialization

It's easiest to access the vault in its initial setup on the pod itself,
where HTTP port 9000 is exposed for access without https. You can decide
how many keys and the recovery threshold using args to `vault init`

``` sh
$ kubectl exec -it <vault-1*> --container vault -- sh

$ vault operator init
or
$ vault operator init -key-shares=1 -key-threshold=1

```

This provides the key(s) and initial auth token required.

Unseal with

``` sh
$ vault operator unseal
```

(You should not generally use the form `vault unseal <key>` because it probably will leave traces of the key in shell history or elsewhere.)

and auth with
``` sh
$ vault auth
Token (will be hidden): <initial_root_token>
```

Then access <vault-2*> in the exact same way (`kubectl exec -it <vault-2*> --container vault -- sh`) and unseal it.
It will go into standby mode.

## Vault as StatefulSet
You can follow the [above Deployment of Vault](#vault-deployment) until [the tls setup](#tLS-setup-for-exposed-vault-port).

### Additional Steps - Prepare Goolge Cloud Project

The following Instructions are taken from [kelseyhightower/vault-on-google-kubernetes-engine](https://github.com/kelseyhightower/vault-on-google-kubernetes-engine) with some changes.

Of curse you need a google account to proceed.

#### Create a New Project

In this section you will create a new GCP project and enable the APIs required by this tutorial.

Generate a project ID:

```
PROJECT_ID="vault-$(($(date +%s%N)/1000000))"
```

Create a new GCP project:

```
gcloud projects create ${PROJECT_ID} \
  --name "${PROJECT_ID}"
```

[Enable billing](https://cloud.google.com/billing/docs/how-to/modify-project#enable_billing_for_a_new_project) on the new project before moving on to the next step.

Enable the GCP APIs required by this tutorial:

> Note: since we dont need all services, i removed some here. 

```
gcloud services enable \
  cloudapis.googleapis.com \
  cloudkms.googleapis.com \
  iam.googleapis.com \
  --project ${PROJECT_ID}
```

#### Set Configuration

```
COMPUTE_ZONE="us-west1-c"
```

```
COMPUTE_REGION="us-west1"
```

```
GCS_BUCKET_NAME="${PROJECT_ID}-vault-storage"
```

```
KMS_KEY_ID="projects/${PROJECT_ID}/locations/global/keyRings/vault/cryptoKeys/vault-init"
```

#### Create KMS Keyring and Crypto Key

In this section you will create a Cloud KMS [keyring](https://cloud.google.com/kms/docs/object-hierarchy#key_ring) and [cryptographic key](https://cloud.google.com/kms/docs/object-hierarchy#key) suitable for encrypting and decrypting Vault [master keys](https://www.vaultproject.io/docs/concepts/seal.html) and [root tokens](https://www.vaultproject.io/docs/concepts/tokens.html#root-tokens). 

Create the `vault` kms keyring:

```
gcloud kms keyrings create vault \
  --location global \
  --project ${PROJECT_ID}
```

Create the `vault-init` encryption key:

```
gcloud kms keys create vault-init \
  --location global \
  --keyring vault \
  --purpose encryption \
  --project ${PROJECT_ID}
```

#### Create a Google Cloud Storage Bucket

Google Cloud Storage is used to hold encrypted Vault master keys and root tokens.

Create a GCS bucket:

```
gsutil mb -p ${PROJECT_ID} gs://${GCS_BUCKET_NAME}
```

#### Create the Vault IAM Service Account

An [IAM service account](https://cloud.google.com/iam/docs/service-accounts) is used by Vault to access the GCS bucket and KMS encryption key created in the previous sections.

Create the `vault` service account:

```
gcloud iam service-accounts create vault-server \
  --display-name "vault service account" \
  --project ${PROJECT_ID}
```

Grant access to the vault storage bucket:

```
gsutil iam ch \
  serviceAccount:vault-server@${PROJECT_ID}.iam.gserviceaccount.com:objectAdmin \
  gs://${GCS_BUCKET_NAME}
```

```
gsutil iam ch \
  serviceAccount:vault-server@${PROJECT_ID}.iam.gserviceaccount.com:legacyBucketReader \
  gs://${GCS_BUCKET_NAME}
```

Grant access to the `vault-init` KMS encryption key:

```
gcloud kms keys add-iam-policy-binding \
  vault-init \
  --location global \
  --keyring vault \
  --member serviceAccount:vault-server@${PROJECT_ID}.iam.gserviceaccount.com \
  --role roles/cloudkms.cryptoKeyEncrypterDecrypter \
  --project ${PROJECT_ID}
```

#### Create/Download serviceAccount credentials file

This is needed to authentificate to google. Since we dont use google cloud here, we must provide this credentials to the vault-init container, so that he can access google cloud.


```
 gcloud iam service-accounts keys create vault-creds.json --iam-account=vault-server@${PROJECT_ID}.iam.gserviceaccount.com
```

#### Generate TLS Certificates

In this section you will generate the self-signed TLS certificates used to secure communication between Vault clients and servers. 

Generate the Vault TLS certificates:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca/ca-config.json \
  -profile=default \
  ca/vault-csr.json | cfssljson -bare vault
```

#### Deploy Vault

In this section you will deploy the multi-node Vault cluster using a collection of Kubernetes and application configuration files.

> Note: Since i will use nodePort for my service , i removed the loadBalancer part and added `node_ip_addr` to expose `vault`.

Create the `vault` secret to hold the Vault TLS certificates:

```
cat vault.pem ca.pem > vault-combined.pem
```

```
kubectl create secret generic vault \
  --from-file=ca.pem \
  --from-file=vault.pem=vault-combined.pem \
  --from-file=vault-key.pem
```

The `vault` configmap holds the Google Cloud Platform settings required bootstrap the Vault cluster.

Create the `vault` configmap:

```
kubectl create configmap vault \
  --from-literal node_ip_addr=10.0.2.51 \
  --from-literal gcs_bucket_name=${GCS_BUCKET_NAME} \
  --from-literal kms_key_id=${KMS_KEY_ID} \
  --from-literal google_application_credentials="/meta/credentials/vault-creds.json"
```

Create `vault` creds secret(needed for vault-init container):

```
kubectl create secret generic vault-creds --from-file=vault-creds.json
```

#### Create the Vault Servie and StatefulSet

In this section you will create the `vault` service/statefulset used to provision and manage two Vault server instances.

Create the `vault` service:

```
kubectl apply -f services/vault-stateful.yaml
```

Create the `vault` statefulset:

```
kubectl apply -f statefulsets/vault.yaml
```

At this point the multi-node cluster is up and running:

```
kubectl get pods
```
```
$ kubectl get pods -o wide
NAME       READY     STATUS    RESTARTS   AGE       IP           NODE
consul-0   1/1       Running   0          2h        10.2.1.72    worker01
consul-1   1/1       Running   0          2h        10.2.3.213   worker03
consul-2   1/1       Running   0          2h        10.2.2.116   worker02
vault-0    3/3       Running   0          1h        10.2.2.246   worker02
vault-1    3/3       Running   0          1h        10.2.3.110   worker03
```

#### Automatic Initialization and Unsealing

In a typical deployment Vault must be initialized and unsealed before it can be used. In our deployment we are using the [vault-init](https://github.com/sethvargo/vault-init) container to automate the initialization and unseal steps.

```
kubectl logs vault-0 -c vault-init
```
```
2018/04/25 01:52:11 Starting the vault-init service...
2018/04/25 01:52:21 Vault is not initialized. Initializing and unsealing...
2018/04/25 01:52:28 Encrypting unseal keys and the root token...
2018/04/25 01:52:29 Unseal keys written to gs://vault-1524618541915-vault-storage/unseal-keys.json.enc
2018/04/25 01:52:29 Root token written to gs://vault-1524618541915-vault-storage/root-token.enc
2018/04/25 01:52:29 Initialization complete.
2018/04/25 01:52:30 Unseal complete.
2018/04/25 01:52:30 Next check in 10s
```

#### Apply ACL to agents

We must also apply the Agent Token to the consul container in the Vault pods.

```
kubectl exec -it  <vault pod name> --container consul-agent-client -- curl --request PUT --header "X-CONSUL-TOKEN: <master token>" --data '{"Token": "<agent token>"}' http://localhost:8500/v1/agent/token/acl_agent_token
```

#### Logging in

Download and decrypt the root token:

```
export VAULT_TOKEN=$(gsutil cat gs://${GCS_BUCKET_NAME}/root-token.enc | \
  base64 --decode | \
  gcloud kms decrypt \
    --project ${PROJECT_ID} \
    --location global \
    --keyring vault \
    --key vault-init \
    --ciphertext-file - \
    --plaintext-file - 
)
```

```
export VAULT_CACERT="ca.pem"
export VAULT_ADDR="https://10.0.2.51:30820"
```

```
$ vault status
Key                    Value
---                    -----
Seal Type              shamir
Sealed                 false
Total Shares           5
Threshold              3
Version                0.10.4
Cluster Name           vault-cluster-7ead9fe8
Cluster ID             3cc58c0e-dec1-ed48-6b73-43c2b0524ff1
HA Enabled             true
HA Cluster             https://10.2.2.246:8201
HA Mode                standby
Active Node Address    https://10.0.2.51:30820
$ vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_afd92c13    per-token private secret storage
identity/     identity     identity_538cbfbd     identity store
secret/       kv           kv_d155aa32           key/value secret storage
sys/          system       system_9cb0ea17       system endpoints used for control, policy and debugging
```
