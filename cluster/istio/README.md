## istio

Typical installation is with istioctl or Helm.

Using a pre-generated manifest (using itioctl) in source control here.

```bash
# generating manifest for istio installation
export ISTIO_RELEASE=1.15.0 &&\
istioctl manifest generate --set profile=default \
  --manifests https://github.com/istio/istio/releases/download/$ISTIO_RELEASE/istio-$ISTIO_RELEASE-linux-amd64.tar.gz > ./k8s/apps-manifests/istio/istio-$ISTIO_RELEASE.yaml
```