# Step: Deploy CockroachDB

Description: 

```
kubectl apply -f ./manifest/dns-lb.yaml --context $clus1
kubectl apply -f ./manifest/dns-lb.yaml --context $clus2
kubectl apply -f ./manifest/dns-lb.yaml --context $clus3
```

Retrieve the IP addresses of the LB in each region and add these to the config maps.

```
kubectl create namespace $loc1 --context $clus1
kubectl create namespace $loc2 --context $clus2
kubectl create namespace $loc3 --context $clus3
```

Backup existing Config maps

```
kubectl -n kube-system get configmap coredns --context $clus1 -o yaml > $clus1-configmap-back.yaml
kubectl -n kube-system get configmap coredns --context $clus2 -o yaml > $clus2-configmap-back.yaml
kubectl -n kube-system get configmap coredns --context $clus3 -o yaml > $clus3-configmap-back.yaml
```

Create three new config maps for the template. You will need to update these with the IP of the DNS Load Balancer we created earlier.

```
cp ./manifest/custom_configmap.yaml manifest/$clus1-configmap.yaml
cp ./manifest/custom_configmap.yaml manifest/$clus2-configmap.yaml  
cp ./manifest/custom_configmap.yaml manifest/$clus3-configmap.yaml 
```

```
kubectl -n kube-system create -f manifest/$clus1-configmap.yaml --context $clus1
kubectl -n kube-system create -f manifest/$clus2-configmap.yaml --context $clus2
kubectl -n kube-system create -f manifest/$clus3-configmap.yaml --context $clus3
```

```
kubectl delete pod --namespace kube-system --selector k8s-app=kube-dns --context $clus1
kubectl delete pod --namespace kube-system --selector k8s-app=kube-dns --context $clus2
kubectl delete pod --namespace kube-system --selector k8s-app=kube-dns --context $clus3

```

```
kubectl -n kube-system describe configmap coredns-custom --context $clus1
kubectl -n kube-system describe configmap coredns-custom --context $clus2
kubectl -n kube-system describe configmap coredns-custom --context $clus3
```

```
mkdir certs my-safe-directory
```

```
cockroach cert create-ca \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

```
cockroach cert create-client \
root \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs \
--context $clus1 \
--namespace $loc1
```

```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs \
--context $clus2 \
--namespace $loc2
```

```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs \
--context $clus3 \
--namespace $loc3
```

```
cockroach cert create-node \
localhost 127.0.0.1 \
cockroachdb-public \
cockroachdb-public.$loc1 \
cockroachdb-public.$loc1.svc.cluster.local \
"*.cockroachdb" \
"*.cockroachdb.$loc1" \
"*.cockroachdb.$loc1.svc.cluster.local" \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs \
--context $clus1 \
--namespace $loc1
```

```
rm certs/node.crt
rm certs/node.key
```

```
cockroach cert create-node \
localhost 127.0.0.1 \
cockroachdb-public \
cockroachdb-public.$loc2 \
cockroachdb-public.$loc2.svc.cluster.local \
"*.cockroachdb" \
"*.cockroachdb.$loc2" \
"*.cockroachdb.$loc2.svc.cluster.local" \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs \
--context $clus2 \
--namespace $loc2
```

```
rm certs/node.crt
rm certs/node.key
```

```
cockroach cert create-node \
localhost 127.0.0.1 \
cockroachdb-public \
cockroachdb-public.$loc3 \
cockroachdb-public.$loc3.svc.cluster.local \
"*.cockroachdb" \
"*.cockroachdb.$loc3" \
"*.cockroachdb.$loc3.svc.cluster.local" \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs \
--context $clus3 \
--namespace $loc3
```

```
rm certs/node.crt
rm certs/node.key
```

```
kubectl -n $loc1 apply -f ./manifest/uksouth-cockroachdb-statefulset-secure.yaml --context $clus1
kubectl -n $loc2 apply -f ./manifest/ukwest-cockroachdb-statefulset-secure.yaml --context $clus2
kubectl -n $loc3 apply -f ./manifest/northeurope-cockroachdb-statefulset-secure.yaml --context $clus3
```

```
kubectl -n $loc1 delete -f aws-cockroachdb-statefulset-secure.yaml --context $clus1
kubectl -n $loc2 delete -f gke-cockroachdb-statefulset-secure.yaml --context $clus2
kubectl -n $loc3 delete -f azure-cockroachdb-statefulset-secure.yaml --context $clus3
```

```
kubectl exec \
--context $clus1 \
--namespace $loc1 \
-it cockroachdb-0 \
-- /cockroach/cockroach init \
--certs-dir=/cockroach/cockroach-certs
```

```
kubectl get pods --context $clus1 --namespace $loc1
kubectl get pods --context $clus2 --namespace $loc2
kubectl get pods --context $clus3 --namespace $loc3
```

```
kubectl config use-context $clus1
kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/client-secure.yaml --namespace $loc1
```

```
kubectl exec -it cockroachdb-client-secure -n $loc1 -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
```
```
CREATE USER craig WITH PASSWORD 'cockroach';
GRANT admin TO craig;
```

```
SET CLUSTER SETTING cluster.organization = '';
SET CLUSTER SETTING enterprise.license = '';
```

```
INSERT INTO system.locations VALUES
  ('region', 'uksouth', 50.941, -0.799),
  ('region', 'ukwest', 53.427, -3.084),
  ('region', 'northeurope', 53.3478, -6.2597);
```
kubectl port-forward cockroachdb-0 8080 -n $loc1
```

```
kubectl apply -f ./manifest/admin-ui.yaml --context $clus1 --namespace $loc1
kubectl apply -f ./manifest/admin-ui.yaml --context $clus2 --namespace $loc2
kubectl apply -f ./manifest/admin-ui.yaml --context $clus3 --namespace $loc3
```

kubectl get svc --context $clus1 --namespace $loc1
kubectl get svc  --context $clus2 --namespace $loc2
kubectl get svc --context $clus3 --namespace $loc3
