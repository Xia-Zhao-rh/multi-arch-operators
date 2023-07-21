# init operator
```
zhaoxia@xzha-mac nginx-operator-24664 % operator-sdk init --plugins=ansible --domain example.com

Writing kustomize manifests for you to edit...
Next: define a resource with:
$ operator-sdk create api
```
# create api
```
zhaoxia@xzha-mac nginx-operator-24664 % operator-sdk create api --group cache --version v1alpha1 --kind Nginx24664 --generate-role
Writing kustomize manifests for you to edit..
```
# modify main.yml
```
zhaoxia@xzha-mac nginx-operator-24664 % cat roles/nginx24664/defaults/main.yml
---
# defaults file for Memcached
size: 1

zhaoxia@xzha-mac nginx-operator-24664 % cat roles/nginx24664/tasks/main.yml 
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

# build nginx operator image
```
zhaoxia@xzha-mac nginx-operator-24664 % docker buildx build . --push --platform linux/amd64,linux/arm64,linux/ppc64le,linux/s390x --tag quay.io/olmqe/nginx-operator-base:v24664
```
# create bundle
```
zhaoxia@xzha-mac nginx-operator-24664 % make bundle IMG=quay.io/olmqe/nginx-operator-base:v24664
operator-sdk generate kustomize manifests -q

Display name for the operator (required): 
> nginx-operator-24664

Description for the operator (required): 
> nginx-operator-24664

Provider's name for the operator (required): 
> nginx-operator-24664

Any relevant URL for the provider name (optional): 
> 

Comma-separated list of keywords for your operator (required): 
> nginx-operator-24664

Comma-separated list of maintainers and their emails (e.g. 'name1:email1, name2:email2') (required): 
> xzha@redhat.com
cd config/manager && /Users/zhaoxia/go/src/github.com/multi-arch-operators/nginx-operator-24664/bin/kustomize edit set image controller=quay.io/olmqe/nginx-operator-base:24664
/Users/zhaoxia/go/src/github.com/multi-arch-operators/nginx-operator-24664/bin/kustomize build config/manifests | operator-sdk generate bundle -q --overwrite --version 0.0.1  
INFO[0000] Creating bundle.Dockerfile                   
INFO[0000] Creating bundle/metadata/annotations.yaml    
INFO[0000] Bundle metadata generated suceessfully       
operator-sdk bundle validate ./bundle
INFO[0000] All validation tests have completed successfully 
```
# modify csv yaml file
```
zhaoxia@xzha-mac nginx-operator-24664 % vi bundle/manifests/nginx-operator-24664.clusterserviceversion.yaml
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
# create bundle image
```
zhaoxia@xzha-mac nginx-operator-24664 % docker buildx build . --push --platform linux/amd64,linux/arm64,linux/ppc64le,linux/s390x -f bundle.Dockerfile -t quay.io/olmqe/nginx-operator-bundle-24664:v0.0.1
```
# create index image
```
mkdir catalog
opm generate dockerfile catalog
mkdir catalog/nginx-operator-24664
opm init nginx-operator-24664 -c alpha  -o yaml > catalog/nginx-operator-24664/index.yaml
opm render quay.io/olmqe/nginx-operator-bundle-24664:v1.0 -o yaml >> catalog/nginx-operator-24664/index.yaml
vi catalog/nginx-operator-24664/index.yaml
---
defaultChannel: alpha
name: nginx-operator-24664
schema: olm.package
---
entries:
- name: nginx-operator-24664.v0.0.1
name: alpha
package: nginx-operator-24664
schema: olm.channel
---
image: quay.io/olmqe/nginx-operator-bundle-24664:v1.0
name: nginx-operator-24664.v0.0.1

zhaoxia@xzha-mac multi-arch-operators % docker buildx build . --push --platform linux/amd64,linux/arm64,linux/ppc64le,linux/s390x -f catalog.Dockerfile -t quay.io/olmqe/nginx-operator-index-24664:v1
```
