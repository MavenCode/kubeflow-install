# Kubeflow Manifests

Installation manifest for Kubeflow.

## kfctl

1. Run the following commands to pass the OIDC parameters to the `kfctl` template

```sh
# Note will need to populate the .env file
source .env
./params.sh
```

2. Run `kfctl` to generate the kustomize folder

```sh
kfctl build -V -f kfctl_k8s_istio.v1.0.1.yaml
```

3. In the generated kustomize folder make the following corrections.

In `kustomize/oidc-authservice/base/statefulset.yaml`

```yaml
spec:
  containers:
  - name: authservice
    image: mavencode/kubeflow-oidc-authservice:6ac9400
    imagePullPolicy: Always
    ports:
    - name: http-api
      containerPort: 8080
    env:
      - name: USERID_HEADER
        value: $(userid-header)
      - name: USERID_PREFIX
        value: $(userid-prefix)
      - name: USERID_CLAIM
        value: preferred_username
      - name: OIDC_PROVIDER
        value: $(oidc_provider)
      - name: OIDC_AUTH_URL
        value: $(oidc_auth_url)
      - name: OIDC_SCOPES
        value: "profile email"
    ...
```

In `kustomize/oidc-authservice/base/envoy-filter.yaml`

```yaml
kind: EnvoyFilter
metadata:
  name: authn-filter
spec:
  workloadLabels:
    istio: ingressgateway-kubeflow
```

To make kubeflow run on ONLY https ingressgateway modify `kustomize/istio/base/kf-istio-resources.yaml`
```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: Gateway
  metadata:
    name: kubeflow-gateway
  ...
  - hosts:
    - '*'
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      mode: SIMPLE
      privateKey: /etc/istio/ingressgateway-certs/tls.key
      serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
```

make sure `istio-ingressgateway-certs` is created and attached to the kubeflow ns

In `kustomize/api-service/base/kustomization.yaml`

```yaml
images:
- name: gcr.io/ml-pipeline/api-server
  newTag: 0.2.5
  newName: gcr.io/ml-pipeline/api-server
```

In `kustomize/metadata/base/kustomization.yaml`

```yaml
images:
- name: gcr.io/ml-pipeline/frontend
  newTag: 0.2.5
  newName: mavencode/ml_metadata_store_server
```


In `kustomize/pipelines-ui/base/kustomization.yaml`

```yaml
images:
- name: gcr.io/ml-pipeline/frontend
  newTag: 0.2.5
  newName: gcr.io/ml-pipeline/frontend
```


In `kustomize/notebook-controller/base/deployment.yaml`

```yaml
env:
  ...
  - name: ENABLE_CULLING
    value: "true"
  - name: IDLE_TIME
    value: "1440"
  - name: CULLING_CHECK_PERIOD
    value: "1"
```

In `kustomize/jupyter-web-app/base/config-map.yaml`

```yaml
spawnerFormDefaults:
  image:
    # The container Image for the user's Jupyter Notebook
    # If readonly, this value must be a member of the list below
    value: mavencode/minimal-notebook-cpu:<git-sha>
    # The list of available standard container Images
    options:
      - mavencode/minimal-notebook-cpu:latest
      - mavencode/minimal-notebook-gpu:latest
      - mavencode/machine-learning-notebook-cpu:latest
      - mavencode/machine-learning-notebook-gpu:latest
      - mavencode/r-studio-cpu:latest
    # By default, custom container Images are allowed
    # Uncomment the following line to only enable standard container Images
    readOnly: true
```

In `kustomize/webhook/overlays/cert-manager/kustomizations.yaml`,
```yaml
- name: cert_name
  objref:
      kind: Certificate
      group: certmanager.k8s.io
      version: v1alpha1
      name: admission-webhook-cert
```

In `kustomize/webhook/kustomization.yaml`:

```yaml
- fieldref:
    fieldPath: metadata.name
  name: cert_name
  objref:
    group: certmanager.k8s.io
    kind: Certificate
    name: admission-webhook-cert
    version: v1alpha1
```


In `kustomize/argo/base/configmap.yaml`

```yaml
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
  namespace: kubeflow
data:
  config: |
    {
    executorImage: $(executorImage),
    containerRuntimeExecutor: $(containerRuntimeExecutor),
    archive: true
    artifactRepository:
```

In `kustomize/argo/base/cluster-role.yaml`

```yaml
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - pods/exec
  verbs:
  - update
  - patch
  - delete
```

In `kustomize/pipelines-ui/base/deployment.yaml`

```yaml
    spec:
      containers:
      - name: ml-pipeline-ui
        image: gcr.io/ml-pipeline/frontend
        imagePullPolicy: IfNotPresent
        env:
        - name: ARGO_ARCHIVE_LOGS
          value: "true"
        - name: ARGO_ARCHIVE_ARTIFACTORY
          value: minio
        - name: ARGO_ARCHIVE_BUCKETNAME
          value: mlpipeline
        - name: ARGO_ARCHIVE_PREFIX
          value: artifacts
```


4. Run the following commands to deploy and configure Kubeflow.

```sh
kfctl apply -f kfctl_istio_dex.v.1.0.0.yaml
kubectl -n kubeflow patch --type=json gateway kubeflow-gateway -p '[{"op":"replace","path":"/spec/selector/istio","value":"ingressgateway-kubeflow"}]'
```

## Argo

When working with Pipelines and Kubeflow you should have the `argo` client.

```sh
kubectl get workflow -o name | grep ${prefix} | cut -d'/' -f2 | xargs argo -n kubeflow delete

argo delete -n kubeflow --older 1d
```
