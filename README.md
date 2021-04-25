# flux-demo-branch-release

A manifest repo acting as a tenant within [flux-demo-branch-admin](https://github.com/janakerman/flux-demo-branch-admin).

Downsides:
* All services in this repo are release together. It's not possible for a 
* Promoting a service to test

## Adding a new service

1. Create a service in the development environment (a collection of manifests) to `environments/development`.

Create the base manifests.

```
SERVICE=foo
   
mkdir -p environments/base/$SERVICE/

cat << EOF > environments/base/$SERVICE/release.yaml 
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: $SERVICE
  namespace: branch
spec:
  releaseName: $SERVICE
  chart:
    spec:
      chart: podinfo
      version: "5.0.0"
      sourceRef:
        kind: HelmRepository
        name: $SERVICE
        namespace: branch
  interval: 5m
  install:
    remediation:
      retries: 3
  # Default values
  # https://github.com/stefanprodan/podinfo/blob/master/charts/podinfo/values.yaml
  values:
    cache: redis-master.redis:6379
    ingress:
      enabled: true
      annotations:
        kubernetes.io/ingress.class: nginx
      path: "/"

EOF

cat << EOF > environments/base/$SERVICE/repository.yaml 
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: $SERVICE
  namespace: branch
spec:
  interval: 5m
  url: https://stefanprodan.github.io/podinfo
EOF

cat << EOF > environments/base/$SERVICE/kustomization.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: branch
resources:
  - release.yaml
  - repository.yaml
EOF
```

Create the manifests for development environments.

```
SERVICE=foo
ENV=development

mkdir -p environments/$ENV/$SERVICE/

cat << EOF > environments/$ENV/$SERVICE/release.yaml 
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: $SERVICE
  namespace: branch
spec:
  values:
    ingress:
      hosts:
        - podinfo.$ENV
EOF

cat << EOF > environments/$ENV/$SERVICE/repository.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: branch
resources:
  - ../../base/$SERVICE
patchesStrategicMerge:
  - values.yaml
EOF
```

At this point, if we were to merge to test, we wouldn't see any deployment to test. First, we need to create the manifests for test environments.

```
SERVICE=foo
ENV=test

mkdir -p environments/$ENV/$SERVICE/

cat << EOF > environments/$ENV/$SERVICE/release.yaml 
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: $SERVICE
  namespace: branch
spec:
  values:
    ingress:
      hosts:
        - podinfo.$ENV
EOF

cat << EOF > environments/$ENV/$SERVICE/repository.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: branch
resources:
  - ../../base/$SERVICE
patchesStrategicMerge:
  - values.yaml
EOF
```

Now, you'd need to merge the manifests to test to see a deployment in the test environment.
