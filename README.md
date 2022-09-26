# gitops-apps
ArgoCD applications for a Kubernetes cluster - Managed by CultClassik/iac-github-mgmt (Github)

## Pre-bootstrapping, until further automation & secrets mgmt are in place:
* Ensure DNS delegation for "aks" subdomain is in place.
* Ensure node MSI has DNS Contributor role assigned (should add to terraform)
* Ensure settings are correct in k8s/apps-manifests/argocd, external-dns and cert-manager overlay files
* 
    ```bash
    export KUBECONFIG=$(pwd)/secret/kubeconfig

    # external-dns
    kubectl create ns external-dns &&\
    kubectl create secret generic azure-config-file --namespace external-dns --from-file ./secret/azure.json

    # cert-manager
    kubectl create ns cert-manager &&\
    kubectl apply -f secret/azwi-sa-dns-contrib.yaml -n cert-manager &&\
    kubectl create secret generic azuredns-config \
        -n cert-manager \
        --from-literal=client-secret=$ARM_CLIENT_SECRET
    ```

## Bootstrapping:
```bash
# install istio
kubectl apply -k cluster/istio

# this will error but that's ok, it will manage itself later
kubectl apply -k cluster/argocd/overlays/production

# get the initial password for "admin" in argocd
kubectl get secret argocd-initial-admin-secret \
    -n argocd \
    -o jsonpath="{.data.password}" | base64 -d; echo
```

## Initialize Vault
```bash
kubectl get pods -n vault

# initialize
kubectl exec hashicorp-vault-0 -- vault operator init \
    -key-shares=5 \
    -key-threshold=3 \
    -format=json > cluster-keys.json

# get the keys
VAULT_UNSEAL_KEY=$(jq -r ".keys_base64[0]" cluster-keys.json)

# unseal #0
kubectl exec -n vault hashicorp-vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY

# join the other nodes
kubectl exec -n vault -ti hashicorp-vault-1 -- vault operator raft join http://hashicorp-vault-0.vault-internal:8200

kubectl exec -n vault -ti hashicorp-vault-2 -- vault operator raft join http://hashicorp-vault-0.vault-internal:8200

# unseal #1
kubectl exec -n vault hashicorp-vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY

# unseal #2
kubectl exec -n vault hashicorp-vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY


# need to configure akv so that vault can auto-unseal..
```

## Start/stop cluster to keep costs down
```bash
source secret/env

az aks start -n $CLUSTER_NAME -g $RG_NAME

az aks stop -n $CLUSTER_NAME -g $RG_NAME
```

## Get the Istio gateway info
```bash
NS_INGRESS=istio-system &&\
export INGRESS_HOST=$(kubectl -n "$NS_INGRESS" get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}') &&\
export INGRESS_PORT=$(kubectl -n "$NS_INGRESS" get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}') &&\
export SECURE_INGRESS_PORT=$(kubectl -n "$NS_INGRESS" get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}') &&\
export TCP_INGRESS_PORT=$(kubectl -n "$NS_INGRESS" get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].port}') &&\
echo "INGRESS_HOST: $INGRESS_HOST" &&\
echo "INGRESS_PORT: $INGRESS_PORT" &&\
echo "SECURE_INGRESS_PORT: $SECURE_INGRESS_PORT" &&\
echo "TCP_INGRESS_PORT: $TCP_INGRESS_PORT"
```