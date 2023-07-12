# build operator
## init operator
```
zhaoxia@xzha-mac nginx-operator % operator-sdk init --plugins=ansible --domain example.com

Writing kustomize manifests for you to edit...
Next: define a resource with:

$ operator-sdk create api
```

## create api
```
zhaoxia@xzha-mac nginx-operator % operator-sdk create api --group cache --version v1alpha1 --kind nginxolm --generate-role
Writing kustomize manifests for you to edit..
```

## modify main.yml
```
zhaoxia@xzha-mac nginx-operator % cat roles/nginxolm/defaults/main.yml 
---
# defaults file for NginxOLM
size: 1

zhaoxia@xzha-mac nginx-operator % cat roles/nginxolm/tasks/main.yml 
---
- name: start nginx
  kubernetes.core.k8s:
    definition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: '{{ ansible_operator_meta.name }}-nginx'
        namespace: '{{ ansible_operator_meta.namespace }}'
      spec:
        replicas: "{{size}}"
        selector:
          matchLabels:
            app: nginx
        template:
          metadata:
            labels:
              app: nginx
          spec:
            containers:
            - name: nginx
              image: "quay.io/olmqe/nginx-alpine:multi-arch"


```

## build nginx operator image
```
zhaoxia@xzha-mac nginx-operator % docker buildx build . --push --platform linux/amd64,linux/arm64,linux/ppc64le,linux/s390x --tag quay.io/olmqe/nginxolm-operator-base:multi-arch
```

# build bundle image
## create bundle
```
zhaoxia@xzha-mac nginx-operator % make bundle IMG=quay.io/olmqe/nginxolm-operator-base:multi-arch
...
INFO[0000] Creating bundle.Dockerfile                   
INFO[0000] Creating bundle/metadata/annotations.yaml    
INFO[0000] Bundle metadata generated suceessfully       
operator-sdk bundle validate ./bundle
INFO[0000] All validation tests have completed successfully

```

## modify csv file
```
zhaoxia@xzha-mac nginx-operator % vi bundle/manifests/nginx-operator.clusterserviceversion.yaml 
 installModes:
  - supported: true
    type: OwnNamespace
  - supported: true
    type: SingleNamespace
  - supported: true
    type: MultiNamespace
  - supported: true
    type: AllNamespaces
```

## build bundle image
```
zhaoxia@xzha-mac nginx-operator % docker buildx build . --push --platform linux/amd64,linux/arm64,linux/ppc64le,linux/s390x -f bundle.Dockerfile -t quay.io/olmqe/nginxolm-operator-bundle:v0.0.1-multi
```

## create another bundle image
```
zhaoxia@xzha-mac nginx-operator % vi bundle/manifests/nginx-operator.clusterserviceversion.yaml 
zhaoxia@xzha-mac nginx-operator % docker buildx build . --push --platform linux/amd64,linux/arm64,linux/ppc64le,linux/s390x -f bundle.Dockerfile -t quay.io/olmqe/nginxolm-operator-bundle:v1.0.1-multi
```

# build index image
```
mkdir catalog
opm generate dockerfile catalog
opm init nginx-operator -c alpha -o yaml > catalog/index.yaml
vi catalog/index.yaml
---
entries:
- name: nginx-operator.v0.0.1
- name: nginx-operator.v1.0.1
  replaces: nginx-operator.v0.0.1
name: alpha
package: nginx-operator
schema: olm.channel

opm render quay.io/olmqe/nginxolm-operator-bundle:v1.0.1-multi -o yaml >> catalog/index.yaml
opm render quay.io/olmqe/nginxolm-operator-bundle:v0.0.1-multi -o yaml >> catalog/index.yaml
opm validate catalog

docker buildx build . --push --platform linux/amd64,linux/arm64,linux/ppc64le,linux/s390x -f catalog.Dockerfile -t quay.io/olmqe/nginxolm-operator-index:v1
```


