# Red Hat Advanced Console Management(RHACM)

## Pre-reqs

### Generating HMAC keys (one time only)

```bash
gcloud auth login ${USER}@ford.com --no-launch-browser --force

GCP_PROJECT='prj-caas-rhacm-pp-p-ae3a'
GCP_PROJECT_NUMBER='816516246174'
GCP_SERVICE_ACCOUNT_NAME='acm-pp-gcs-hmac'
VAULT_MANANGED_SA='vault816516246174-a-1645123663@prj-c-vault-831c.iam.gserviceaccount.com'

mkdir -p ${GITOPS_REPO}/rhacm/ref/gcp-hmac-keys

kubectl create secret generic thanos-hmac --from-env-file <(
    gsutil hmac create -p ${GCP_PROJECT} acm-pp-gcs-hmac@${GCP_PROJECT}.iam.gserviceaccount.com |
        sed -e 's/Access ID: */ACCESS_KEY=/' -e 's/Secret: */SECRET_KEY=/'
) --dry-run=client -o yaml >${GITOPS_REPO}/rhacm/ref/gcp-hmac-keys/${GCP_PROJECT}.yaml
```
```
# For Prod Instance
GCP_PROJECT='prj-caas-rhacm-pd-p-22a6'
GCP_PROJECT_NUMBER='300055182250'
GCP_GCS_ADMIN_SERVICE_ACCOUT="acm-pd-admin"
GCP_GCS_HMAC_SERVICE_ACCOUNT_NAME='acm-pd-gcs-hmac'
GCP_VAULT_SERVICE_ACCOUNT_NAME='acm-pd-gcs-sa'
VAULT_MANANGED_SA='vault300055182250-a-1647973284@prj-c-vault-831c.iam.gserviceaccount.com'
```

### Make bucket (one time only)

```bash
# For Preprod
gsutil -i acm-pp-gcs-hmac@${GCP_PROJECT}.iam.gserviceaccount.com mb -p ${GCP_PROJECT} -c standard -l NAM4 gs://thanos-pp111

# For Prod
gsutil -i ${GCP_GCS_ADMIN_SERVICE_ACCOUT}@${GCP_PROJECT}.iam.gserviceaccount.com mb -p ${GCP_PROJECT} -c standard -l NAM4 gs://thanos-pd111"

```

## Install and setup HUB Cluster

Unless otherwise noted, these steps are to be run on the hub cluster (i.e. pp111/pd111)

### Install ACM Operator

```bash
GITOPS_REPO="${HOME}/workspace/containers/platform-gitops"
#OVERLAY="preprod"
#OVERLAY="prod"
kustomize build ${GITOPS_REPO}/rhacm/overlays/${OVERLAY}/1-operator | oc create -f -
```

### Install MultiClusterHub

```bash
GITOPS_REPO="${HOME}/workspace/containers/platform-gitops"
#OVERLAY="preprod"
#OVERLAY="prod"
kustomize build ${GITOPS_REPO}/rhacm/overlays/${OVERLAY}/2-hub | oc create -f -
```

### Install Observability

```bash
export AVP_TYPE=vault
export AVP_AUTH_TYPE=token
export VAULT_NAMESPACE=caas
export VAULT_ADDR=https://vault.app.ford.com
export VAULT_TOKEN=$(vault login -method=ldap -token-only username=${USER})

GITOPS_REPO="${HOME}/workspace/containers/platform-gitops"
OVERLAY="preprod"
kustomize build ${GITOPS_REPO}/rhacm/overlays/${OVERLAY}/3-observability | \
  argocd-vault-plugin generate - | \
  kubectl create -f -
```

### Install ACS operator and central

```bash
GITOPS_REPO="${HOME}/workspace/containers/platform-gitops"
kubectl apply -f ${GITOPS_REPO}/rhacm/policies/policy-acs-operator-central.yaml
```

### Generate cert bundle for secured cluster

Note: Create integration with StackRox API.  Create token at respective cluster url:
https://central-stackrox.apps.pd111.caas.gcp.ford.com/main/integrations/authProviders/apitoken/create

TokenName: Admin
Role: Admin


```bash
export ROX_API_TOKEN=""
#OVERLAY="preprod"
#OVERLAY="prod"
GITOPS_REPO="${HOME}/workspace/containers/platform-gitops"
PATH=$PATH:${GITOPS_REPO}/rhacm/bin
which roxctl
which yq
mkdir ${GITOPS_REPO}/rhacm/refs
${GITOPS_REPO}/rhacm/bin/deploy-bundle.sh -i ${GITOPS_REPO}/rhacm/refs/${OVERLAY}-cert-bundle.yaml | kubectl apply -f -
```

> Ref: https://github.com/stolostron/advanced-cluster-security
> TODO: Stash this in vault for DR

### Apply ACS "agent" policies

```bash
GITOPS_REPO="${HOME}/workspace/containers/platform-gitops"
kubectl apply -f ${GITOPS_REPO}/rhacm/policies/policy-acs-operator.yaml
```

### Install Certificate Policy

```bash

GITOPS_REPO="${HOME}/workspace/containers/platform-gitops"
kubectl apply -f ${GITOPS_REPO}/rhacm/policies/policy-cert.yaml

```

### Add local-cluster as an ACS agent

```bash
oc label ManagedCluster local-cluster is-sensor=true --overwrite
```

<!--
### vault managed SA (not currently used)
```bash
gcloud auth login ${USER}@ford.com --no-launch-browser --force

unset VAULT_TOKEN
export VAULT_ADDR=https://vaultdev.app.ford.com
export VAULT_NAMESPACE=gcp

vault login -method=ldap username=${USER}

GCP_PROJECT='prj-caas-rhacm-pp-p-ae3a'
GCP_PROJECT_NUMBER='816516246174'
GCP_SERVICE_ACCOUNT_NAME='acm-pp-gcs-sa'
VAULT_MANANGED_SA='vault816516246174-a-1645123663@prj-c-vault-831c.iam.gserviceaccount.com'

mkdir -p ${GITOPS_REPO}/rhacm/secrets/gcp-sa-keys

## https://github.ford.com/Secrets-Management/ford_habitat_vault_policies/blob/dev/GCP-ServiceAccounts.md
vault read ${GCP_PROJECT}/key/${GCP_PROJECT_NUMBER}-${GCP_SERVICE_ACCOUNT_NAME} -format=json | \
  jq -r '.data.private_key_data' | \
  base64 --decode > ${GITOPS_REPO}/rhacm/secrets/gcp-sa-keys/${GCP_PROJECT}-${GCP_SERVICE_ACCOUNT_NAME}.json

## List vault leases
vault list -format=json /sys/leases/lookup/${GCP_PROJECT}/key/${GCP_PROJECT_NUMBER}-${GCP_SERVICE_ACCOUNT_NAME}/ | \
  jq -r '.[]' | \
  xargs -I {} vault lease lookup ${GCP_PROJECT}/key/${GCP_PROJECT_NUMBER}-${GCP_SERVICE_ACCOUNT_NAME}/{}
#vault lease lookup ${GCP_PROJECT}/key/${GCP_PROJECT_NUMBER}-${GCP_SERVICE_ACCOUNT_NAME}/zSQKgrmzVHnpM6Hab69mSGWI.byCmw
#vault list -format=json /sys/leases/lookup/prj-caas-quay-p-bf1e/key/994081582269-quay-psql-db/ | jq -r '.[]'
#vault lease revoke ${GCP_PROJECT}/key/${GCP_PROJECT_NUMBER}-${GCP_SERVICE_ACCOUNT_NAME}/zSQKgrmzVHnpM6Hab69mSGWI.byCmw
#vault lease revoke -prefix ${GCP_PROJECT}/key/${GCP_PROJECT_NUMBER}-${GCP_SERVICE_ACCOUNT_NAME}

#export GOOGLE_APPLICATION_CREDENTIALS= ${GITOPS_REPO}/rhacm/secrets/gcp-sa-keys/db-sa-key.json
```

!-->

## Import managed cluster

> TODO: Add a task to gcp-ocp-pipeline to create this service account on newly paved cluster, then apply these objects to hub cluster.
> TODO: policy to remove above service account on new cluster after bootstrapped. RH confirms that this SA no longer needed.

### Run this on managed cluster (the cluster you want to import) (NOT ON PP111/PD111)

This will create a service account and token in the rhacm-agent-test namespace to give to the hub cluster in the next step.

```bash
kubectl create -f ${GITOPS_REPO}/rhacm/misc/Service-account-role-binding-managed-cluster.yaml

SA_SECRET_NAME=$(kubectl -n rhacm-bootstrap-delete-me get sa rhacm-bootstrap-sa -o json | jq -r '.secrets[] | select(.name | startswith("rhacm-bootstrap-sa-token")) | .name')

TOKEN=$(kubectl -n rhacm-bootstrap-delete-me get secret $SA_SECRET_NAME -o jsonpath='{.data.token}'| base64 --decode)

kubectl auth can-i delete deployments --as=system:serviceaccount:rhacm-bootstrap-delete-me:rhacm-bootstrap-sa
```

### Run this on hub cluster (i.e. PP111/PD111)

```bash
IMPORTED_CLUSTER_NAME=sb06
IMPORTED_CLUSTER_FQDN=sb06.caasdev.ford.com
sed "s~__TOKEN__~${TOKEN}~g" < ${GITOPS_REPO}/rhacm/misc/ManagedCluster.yaml | \
    sed "s~__CLUSTER_NAME__~${IMPORTED_CLUSTER_NAME}~g" | \
    sed "s~__CLUSTER_NAME_FQDN__~${IMPORTED_CLUSTER_FQDN}~g" | \
    oc create -f -

# Add StackRox / ACS to managed cluster
oc label ManagedCluster ${IMPORTED_CLUSTER_NAME} is-sensor=true --overwrite

oc logout
```

Now you should see the cluster enrolled in the multicluster hub

<!--
### Run this in hub cluster - This step only needed on pre-4.8 managed clusters - needs verification
```bash
MANAGED_CLUSTER_CRD=$(kubectl -n ${CLUSTER_NAME} get secret \
    ${CLUSTER_NAME}-import \
    -o jsonpath='{.data.crds\.yaml}' | \
    base64 --decode)

MANAGED_CLUSTER_IMPORT_COMMAND=$(kubectl -n ${CLUSTER_NAME} get secret \
    ${CLUSTER_NAME}-import \
    -o jsonpath='{.data.import\.yaml}' | \
    base64 --decode)
```

### Run this on the managed cluster - This step only needed on pre-4.8 managed clusters - needs verification
This creates the klusterlet CRD on the managed cluster and then the objects for creating a klusterlet itself..  basically everything in these 2 yaml examples [here](https://github.ford.com/TDEICHEL/platform-gitops/tree/import-managed-cluster-test/rhacm/demo/import-managed-cluster-sample)
```bash
echo $MANAGED_CLUSTER_CRD | kubectl create -f -
echo $MANAGED_CLUSTER_IMPORT_COMMAND | kubectl create -f -
```
!-->

### Delete service account from managed cluster

```bash
oc login https://api.${IMPORTED_CLUSTER_FQDN}:6443 -u ${USER}

oc delete serviceaccount -n rhacm-agent-test rhacm-admin-sa
oc delete namespace rhacm-agent-test
oc delete clusterrolebinding rhacm-admin
```

## REF

### Un-enroll a managed cluster

From the hub cluster

```bash
kubectl delete managedcluster ${CLUSTER_NAME}
kubectl delete namespace rhacm-managed-cluster-${CLUSTER_NAME}   # This won't be needed if I changed the namespace back to ${CLUSTER_NAME} in ManagedCluster.yaml
```

- Verify that the namespace ${CLUSTER_NAME} no longer exists on the hub and managed clusters.

### Cleanup script (to be run on managed cluster if needed)

> REF: https://github.com/stolostron/deploy/blob/master/hack/cleanup-managed-cluster.sh

```bash
#!/bin/bash
###############################################################################
# Copyright (c) 2020 Red Hat, Inc.
###############################################################################

if [ -z "${OPERATOR_NAMESPACE}" ]; then
	OPERATOR_NAMESPACE="open-cluster-management-agent-addon"
fi

if [ -z "${KLUSTERLET_NAMESPACE}" ]; then
	KLUSTERLET_NAMESPACE="open-cluster-management-agent"
fi

KUBECTL=oc

# Force delete klusterlet
echo "attempt to delete klusterlet"
${KUBECTL} delete klusterlet klusterlet --timeout=60s
${KUBECTL} delete namespace ${KLUSTERLET_NAMESPACE} --wait=false
echo "force removing klusterlet"
${KUBECTL} patch klusterlet klusterlet --type="json" -p '[{"op": "remove", "path":"/metadata/finalizers"}]'
echo "removing klusterlet crd"
${KUBECTL} delete crd klusterlets.operator.open-cluster-management.io --timeout=30s

# Force delete all component CRDs if they still exist
component_crds=(
	applicationmanagers.agent.open-cluster-management.io
	certpolicycontrollers.agent.open-cluster-management.io
	iampolicycontrollers.agent.open-cluster-management.io
	policycontrollers.agent.open-cluster-management.io
	searchcollectors.agent.open-cluster-management.io
	workmanagers.agent.open-cluster-management.io
	appliedmanifestworks.work.open-cluster-management.io
)

for crd in "${component_crds[@]}"; do
	echo "force delete all CustomResourceDefinition ${crd} resources..."
	for resource in `${KUBECTL} get ${crd} -o name -n ${OPERATOR_NAMESPACE}`; do
		echo "attempt to delete ${crd} resource ${resource}..."
		${KUBECTL} delete ${resource} -n ${OPERATOR_NAMESPACE} --timeout=30s
		echo "force remove ${crd} resource ${resource}..."
		${KUBECTL} patch ${resource} -n ${OPERATOR_NAMESPACE} --type="json" -p '[{"op": "remove", "path":"/metadata/finalizers"}]'
	done
	echo "force delete all CustomResourceDefinition ${crd} resources..."
	${KUBECTL} delete crd ${crd}
done

${KUBECTL} delete namespace ${OPERATOR_NAMESPACE}
```

## Additional cleanup on managed cluster (if needed)

```
kubectl get apiservice|grep ServiceNotFound

# v1.admission.cluster.open-cluster-management.io       open-cluster-management-hub/cluster-manager-registration-webhook   False (ServiceNotFound)   15d
# v1.admission.work.open-cluster-management.io          open-cluster-management-hub/cluster-manager-work-webhook           False (ServiceNotFound)   15d
# v1.clusterview.open-cluster-management.io             open-cluster-management/ocm-proxyserver                            False (ServiceNotFound)   15d
# v1alpha1.clusterview.open-cluster-management.io       open-cluster-management/ocm-proxyserver                            False (ServiceNotFound)   15d
# v1alpha1.upload.cdi.kubevirt.io                       openshift-cnv/cdi-api                                              False (ServiceNotFound)   11d
# v1beta1.proxy.open-cluster-management.io              open-cluster-management/ocm-proxyserver                            False (ServiceNotFound)   15d
# v1beta1.upload.cdi.kubevirt.io                        openshift-cnv/cdi-api                                              False (ServiceNotFound)   11d


kubectl delete apiservice v1.admission.cluster.open-cluster-management.io
kubectl delete apiservice v1.admission.work.open-cluster-management.io
kubectl delete apiservice v1.clusterview.open-cluster-management.io
kubectl delete apiservice v1alpha1.clusterview.open-cluster-management.io
kubectl delete apiservice v1beta1.proxy.open-cluster-management.io

kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n 1 kubectl get --show-kind --ignore-not-found -n open-cluster-management

# NAME                                                                   AGE
# helmrelease.apps.open-cluster-management.io/application-chart-4ff14    15d
# helmrelease.apps.open-cluster-management.io/assisted-service-e8827     15d
# helmrelease.apps.open-cluster-management.io/cluster-lifecycle-096ef    15d
# helmrelease.apps.open-cluster-management.io/console-chart-4c9ce        15d
# helmrelease.apps.open-cluster-management.io/discovery-operator-bc79c   15d
# helmrelease.apps.open-cluster-management.io/grc-6a682                  15d
# helmrelease.apps.open-cluster-management.io/management-ingress-64073   15d
# helmrelease.apps.open-cluster-management.io/policyreport-cca55         15d
# helmrelease.apps.open-cluster-management.io/search-prod-ec65d          15d

for resource in $(kubectl get helmrelease.apps.open-cluster-management.io -n open-cluster-management -o name); do
  kubectl patch -n open-cluster-management ${resource} -p '{"metadata":{"finalizers":null}}' --type=merge
done

kubectl delete crd helmrelease.apps.open-cluster-management.io
kubectl delete ValidatingWebhookConfiguration multiclusterhub-operator-validating-webhook
kubectl patch -n open-cluster-management multiclusterhubs.operator.open-cluster-management.io multiclusterhub -p '{"metadata":{"finalizers":null}}' --type=merge
kubectl delete namespace open-cluster-management
```

<!--
## Install GitOps Cluster  (this is broke, don't run)
#```bash
#GITOPS_REPO="${HOME}/workspace/containers/platform-gitops"
#RELEASE="2.4" # or 2.3
#kustomize build ${GITOPS_REPO}/rhacm/base/3-gitops | oc create -f -
#```

## Install smoke-test app using a subscription

```bash
GITOPS_REPO="${HOME}/workspace/containers/platform-gitops"
RELEASE="2.4" # or 2.3
oc apply -f ${GITOPS_REPO}/rhacm/misc/sub.yaml
```






## OLD STUFF BELOW


## Notes from Planning Spike

 - "Hub" cluster will be in GCP to start, if we find that there's something that breaks we will pivot back to EDC Sandbox (documentation does not call out GCP as supported, but the product works as both a hub environment or for adoption) 
 - Targeting OCP 4.7 for paving these minimum sized sandbox clusters 

 - Will need to pave...
   - 1 GCP (or EDC as fallback) OpenShift Sandbox cluster for Hub 
   - 1 EDC OpenShift Sandbox cluster (or use existing)
   - 1 GCP OpenShift Sandbox cluster 
   - 1 Azure OpenShift Sandbox cluster 
   - 1 GCP GKE Sandbox cluster 
 - Cluster Sizing 
 - Hub Cluster Sizing
 - TBD - Will need to reference guidance in the docs and make/document decision for what we opt to go with

## Current Status
### RHACM Hub
SB07 - https://multicloud-console.apps.sb07.caasdev.ford.com/multicloud/

### RHACM Spokes
Location | Type | Cluster
---------|------|--------
OnPrem | OCP | SB07
OnPrem | OCP | SB08
GCP | OCP | SB103
GCP | GKE | TBD
Azure | OCP | TBD


### Documentation -
 - https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.3
 - https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.3/html/install/index
 - https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.4/html/install/installing


### Misc
```bash
oc get packagemanifests -n openshift-marketplace advanced-cluster-management -o yaml
```
- Cluster proxy addon
  - https://open-cluster-management.io/getting-started/integration/cluster-proxy/
  - https://github.com/noseka1/rhacm-kustomization/blob/master/rhacm-instance/base/multiclusterhub-multiclusterhub.yaml#L8

!-->
