# Generate multi-arch bundle image
## create bundle.Dockerfile

```
$ opm alpha bundle generate -d manifests -c stable -p kubeturbo -e stable 
INFO[0000] Building annotations.yaml                    
INFO[0000] Writing annotations.yaml in /Users/zhaoxia/go/src/github.com/redhat-openshift-ecosystem/community-operators-prod/operators/kubeturbo/8.4.0/metadata 
INFO[0000] Building Dockerfile                          
INFO[0000] Writing bundle.Dockerfile in /Users/zhaoxia/go/src/github.com/redhat-openshift-ecosystem/community-operators-prod/operators/kubeturbo/8.4.0 
```

## Generate bundle image

```
$ docker buildx build . --push --platform linux/amd64,linux/arm64 -f bundle.Dockerfile -t quay.io/olmqe/kubeturbo-bundle:v8.4.0
[+] Building 5.2s (6/8)                                                                                                                                                                                   
 => [internal] load build definition from bundle.Dockerfile                                                                                                                                          0.0s
 => => transferring dockerfile: 514B                                                                                                                                                                 0.0s
 => [internal] load .dockerignore                                                                                                                                                                    0.0s
 => => transferring context: 2B                                                                                                                                                                      0.0s
 => [internal] load build context                                                                                                                                                                    0.0s
 => => transferring context: 53.56kB                                                                                                                                                                 0.0s
 => [linux/amd64 1/2] COPY manifests /manifests/                                                                                                                                                     0.0s
 => [linux/amd64 2/2] COPY metadata /metadata/                                                                                                                                                       0.0s
 => exporting to image                                                                                                                                                                               4.8s
 => => exporting layers                                                                                                                                                                              0.1s
 => => exporting manifest sha256:ae47f0a73b83525a5c7be712ec4cf219f264a8e7714fd59be26ce9c4b532f41c                                                                                                    0.0s
 => => exporting config sha256:0f451e0f692f3528c1b1ce250d170c66585f58fbe3925ce98fe62502a8ee0a44                                                                                                      0.0s
 => => exporting manifest sha256:2ef433ef826da5ca3811738bc20a64a7cc59850f8c1c2385a33516143f1f2905                                                                                                    0.0s
 => => exporting config sha256:1639091eaf4d3ed39d9fcc839ba02e3f6a46a3434abe5ec0d6eeaf4ad9bf1bd9                                                                                                      0.0s
 => => exporting manifest list sha256:a2747cc67cc2a58b0810a1079654f34182deda2f804496fbc142ec9f87ce4264                                                                                               0.0s
 => => pushing layers                                                                                                                                                                                2.5s
 => => pushing manifest for quay.io/olmqe/kubeturbo-bundle:v8.4.0@sha256:a2747cc67cc2a58b0810a1079654f34182deda2f804496fbc142ec9f87ce4264 
 ```   

## Generate index image

mkdir catalog
opm generate dockerfile catalog
mkdir catalog/kubeturbo-operator
opm init kubeturbo-operator -c stable  -o yaml > catalog/kubeturbo-operator/index.yaml
opm render quay.io/olmqe/kubeturbo-bundle:v8.4.0 -o yaml >> catalog/kubeturbo-operator/index.yaml

vi catalog/nginx-operator-24644/index.yaml
```
---
defaultChannel: stable
name: kubeturbo-operator
schema: olm.package
---
entries:
- name: kubeturbo-operator.v8.5.0
name: stable
package: kubeturbo-operator
schema: olm.channel
---
image: quay.io/olmqe/kubeturbo-bundle:21824-v8.5.0-wrong
name: kubeturbo-operator.v8.5.0
package: kubeturbo
```
docker buildx build . --push --platform linux/amd64,linux/arm64 -f catalog.Dockerfile -t quay.io/olmqe/quay.io/olmqe/kubeturbo-index:v1

