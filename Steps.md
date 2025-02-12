I have setup 2 contexts: cluster1 and cluster2

# install to all clusters

```bash
for context in cluster1 cluster2; do
    linkerd --context=$context install --crds | kubectl --context=$context apply -f -
    linkerd --context=$context install | kubectl --context=$context apply -f - 
    linkerd check       
done

linkerd --context=cluster1 check --pre   
linkerd multicluster install | \
    kubectl --context=cluster2 apply -f -
linkerd multicluster install | \
    kubectl --context=cluster2 apply -f -

```

# remove linkerd from all cluster so that we can build Trust anchor certificate from scratch

https://linkerd.io/2.15/tasks/uninstall/

```bash
for context in cluster1 cluster2; do
    linkerd --context=$context viz uninstall | kubectl --context=$context delete -f -
    linkerd --context=$context multicluster uninstall | kubectl --context=$context delete -f -
    linkerd --context=$context uninstall | kubectl --context=$context delete -f -
done
```

# build mTLS
https://linkerd.io/2.15/tasks/generate-certificates/#trust-anchor-certificate

```bash
step certificate create root.linkerd.cluster.local ca.crt ca.key \
--profile root-ca --no-password --insecure
step certificate create identity.linkerd.cluster.local issuer.crt issuer.key \
--profile intermediate-ca --not-after 8760h --no-password --insecure \
--ca ca.crt --ca-key ca.key

for context in cluster1 cluster2; do
    linkerd --context=$context install --crds | kubectl --context=$context apply -f -
    # install the Linkerd control plane, with the certificates we just generated.
    linkerd --context=$context install \
      --identity-trust-anchors-file ca.crt \
      --identity-issuer-certificate-file issuer.crt \
      --identity-issuer-key-file issuer.key \
      | kubectl --context=$context apply -f -
done

```

# setup multicluster
https://linkerd.io/2.15/tasks/installing-multicluster/

## install

```bash
for context in cluster1 cluster2; do
    linkerd --context=$context multicluster install | kubectl --context=$context apply -f -
    linkerd --context=$context multicluster check
done
```

## link cluster2 to cluster1

```bash
linkerd --context=cluster1 multicluster link --cluster-name cluster1 | kubectl --context=cluster2 apply -f -
```

Here error appear:
```
Error: Gateway linkerd-gateway.linkerd-multicluster has no ingress addresses
```
that's because local k8s doesn't support LoadBalancer

# install MetalLB

```bash
# on both
for context in cluster1 cluster2; do
    kubectl --context=$context apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
done
```

### setup MetalLB

we can apply it to vagrant script

https://metallb.universe.tf/configuration/

cluster1:
```yml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.50.214 - 192.168.50.219
```

```yml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
```

cluster2: 
cluster2-ippool.yml

# setup multicluster 2
## link cluster2 to cluster1 again
## check result
https://linkerd.io/2.15/tasks/installing-multicluster/#step-2-link-the-clusters

```bash
linkerd --context=cluster2 multicluster check
linkerd --context=cluster2 multicluster gateways
```

# https://linkerd.io/2.15/tasks/multicluster/

## install viz
```bash
for ctx in cluster1 cluster2; do
  linkerd --context=${ctx} viz install | \
    kubectl --context=${ctx} apply -f - || break
done
```

## check gateway status

```bash
for ctx in cluster1 cluster2; do
  echo "Checking gateway on cluster: ${ctx} ........."
  kubectl --context=${ctx} -n linkerd-multicluster \
    rollout status deploy/linkerd-gateway || break
  echo "-------------"
done
```

## check load balancer status

```bash
for ctx in cluster1 cluster2; do
  printf "Checking cluster: ${ctx} ........."
  while [ "$(kubectl --context=${ctx} -n linkerd-multicluster get service -o 'custom-columns=:.status.loadBalancer.ingress[0].ip' --no-headers)" = "<none>" ]; do
      printf '.'
      sleep 1
  done
  printf "\n"
done
```

## install test services
```bash
for ctx in cluster2 cluster1; do
  kubectl --context=${ctx} create namespace test
  echo "Adding test services on cluster: ${ctx} ........."
  kubectl --context=${ctx} apply \
    -n test -k "multicluster/${ctx}/"
  kubectl --context=${ctx} -n test \
    rollout status deploy/podinfo || break
  echo "-------------"
done
```

got this error:
```
error: invalid Kustomization: json: cannot unmarshal string into Go struct field Kustomization.patches of type types.Patch
```

I installed `brew install kustomize` and run `kustomize edit fix` under every dir `multicluster/base&cluster1&cluster2` but still not work.

# reinit
I removed test namespace and prepare to install redis & redisinsight into cluster1 & cluster2

## deploy redis
```bash
kubectl --context=cluster1 create namespace test
kubectl --context=cluster1 apply -n test -f "resources/1-redis.yml"
kubectl --context=cluster1 get svc redis -n test
```

## deploy redisinsight
```bash
kubectl --context=cluster2 create namespace test
kubectl --context=cluster2 apply -n test -f "resources/2-redisinsight.yml"
kubectl --context=cluster2 get svc redisinsight-service -n test
```
Here we use LoadBalancer to expose redisinsight to the internet since we want to visit it
Now we can visit redisinsight with the ip/port given by the last result.

## export redis
```
kubectl --context=cluster1 label svc redis mirror.linkerd.io/exported=true -n test
```
check mirrored service:
```
kubectl --context=cluster2 get svc  -n test
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
redis-cluster1         ClusterIP      10.107.7.197   <none>           6379/TCP       75m
redisinsight-service   LoadBalancer   10.98.80.117   192.168.50.215   80:31583/TCP   125m
```

## install antother redis instance into cluster2, to use its cli
```
kubectl --context=cluster2 apply -n test -f "resources/1-redis.yml"
```

When I connect mirrored redis with redis cli, the connection will be closed automatically

let me create a simpler service

# try hello-world

```bash
kubectl --context=cluster1 delete namespace test
kubectl --context=cluster2 delete namespace test
kubectl --context=cluster1 create namespace test
kubectl --context=cluster2 create namespace test
```

## install hello-world
```bash
kubectl --context=cluster1 apply -n test -f "resources/1-hello.yml"
```

## export it to cluster2
```
kubectl --context=cluster1 label svc hello mirror.linkerd.io/exported=true -n test
```

## install busybox to cluster2 to test
```bash
kubectl --context=cluster2 apply -n test -f "resources/2-busybox.yml"
```

busybox still not work. 
try a curl one
