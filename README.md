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
1. istio
    * `kubectl apply -k cluster/istio`
2. argocd
    * 
        ```bash
        # this will error but that's ok, it will manage itself later
        kubectl apply -k cluster/argocd/overlays/production

        # get the initial password for "admin" in argocd
        kubectl get secret argocd-initial-admin-secret \
            -n argocd \
            -o jsonpath="{.data.password}" | base64 -d; echo
        ```

## Start/stop cluster to keep costs down
* 
    ```bash
    source secret/env

    az aks start -n $CLUSTER_NAME -g $RG_NAME

    az aks stop -n $CLUSTER_NAME -g $RG_NAME
    ```



2. Install External-DNS
    ```bash
    # https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md
    # https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/istio.md
    cat <<-EOF > ./secret/azure.json
    {
    "tenantId": "${ARM_TENANT_ID}",
    "subscriptionId": "${ARM_SUBSCRIPTION_ID}",
    "resourceGroup": "rg-aks-cluster-001",
    "aadClientId": "${ARM_CLIENT_ID}",
    "aadClientSecret": "${ARM_CLIENT_SECRET}"
    }
    EOF

    # note the azure RG name that contains the dns one is located in some of these files
    kubectl create ns external-dns

    kubectl create secret generic azure-config-file --namespace external-dns --from-file ./secret/azure.json

    kubectl apply -k ./k8s/apps-manifests/external-dns
    ```
2. Install cluster-apps
    * Installs everything else and will manage argocd itself as well
    * Configure k8s secret for cert manager, see below
        ```bash
        # namespace would normally be created by argocd
        kubectl create ns cert-manager

        # the secret should normally be stored in vault and created by argocd
        kubectl create secret generic azuredns-config \
            -n cert-manager \
            --from-literal=client-secret=$ARM_CLIENT_SECRET
        ```
    * `kubectl apply -k ./k8s/cluster-apps`
3. Monitoring bootstrap and troubleshooting
    * Can't access UI thru istio yet!
    ```bash
    # get the initial password for "admin" in argocd
    kubectl get secret argocd-initial-admin-secret \
        -n argocd \
        -o jsonpath="{.data.password}" | base64 -d; echo

    # port forward to access the argocd ui at http://localhost:8080/argocd/
    kubectl port-forward svc/argocd-server \
        -n argocd 8080:443
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