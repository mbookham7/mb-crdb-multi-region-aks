# Step: Deploy CockroachDB

Description:

## Prepare Kubernetes

For multi region CockroachDB to work DNS needs to be shared across the three clusters. To do this we expose the coredns deployment outside of the cluster via a LoadBalancer. Use the commands below to create a service that exposes the core dns pods on the VNet.

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

You will need to edit each of these config maps adding the name of the remaining two namespaces and the IP's of the DNS Load Balancer we setup in a previous step.
Below is an example output for reference.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  cockroach.server: | # you may select any name here, but it must end with the .server file extension
    uksouth.svc.cluster.local:53 {
        errors
        cache 30
        forward . 10.1.1.91 {
        }
    }
    ukwest.svc.cluster.local:53 {
        errors
        cache 30
        forward . 10.2.1.91
    }
```

Once you have updated the config maps for all three regions you can apply them to each of your AKS clusters.

```
kubectl -n kube-system replace -f manifest/$clus1-configmap.yaml --context $clus1 --force
kubectl -n kube-system replace -f manifest/$clus2-configmap.yaml --context $clus2 --force
kubectl -n kube-system replace -f manifest/$clus3-configmap.yaml --context $clus3 --force
```

To check the contents of your config map you can use the describe command to output this to the screen.

```
kubectl -n kube-system describe configmap coredns-custom --context $clus1
kubectl -n kube-system describe configmap coredns-custom --context $clus2
kubectl -n kube-system describe configmap coredns-custom --context $clus3
```

To ensure that to config map is has taken effect in coredns restart the pods.

```
kubectl delete pod --namespace kube-system --selector k8s-app=kube-dns --context $clus1
kubectl delete pod --namespace kube-system --selector k8s-app=kube-dns --context $clus2
kubectl delete pod --namespace kube-system --selector k8s-app=kube-dns --context $clus3

```

## CockroachDB Deployment

Kubernetes is all prepared now for our deployment of CockroachDB. Now we must complete all the steps required to setup CockroachDB.
First we create two new folders for our certificates.

```
mkdir certs my-safe-directory
```

All communication between the node and clients is secure. The simplest way to achieve this is with the built in certificate authority. We can use any machine with the cockroach binary installed to create all the required certificates. Create the CA certificate and key pair:

```
cockroach cert create-ca \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

Create a client certificate and key pair for the root user:

```
cockroach cert create-client \
root \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

For all 3 regions, upload the client certificate and key to the Kubernetes cluster as a secret.

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

Create the certificate and key pair for your CockroachDB nodes in one region:

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

Upload the node certificate and key to the Kubernetes cluster as a secret, specifying the appropriate context and namespace:

```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs \
--context $clus1 \
--namespace $loc1
```

Remove or store in a safe place the node certificate for region one. As mine is a demo environment I am going to delete it.

```
rm certs/node.crt
rm certs/node.key
```

Now repeat the process for region two, replacing the namespace name for namespace name for region two. In this example this is done via the use of environment variables.

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

Upload the node certificate and key to the Kubernetes cluster as a secret, specifying the appropriate context and namespace:

```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs \
--context $clus2 \
--namespace $loc2
```

Remove or store in a safe place the node certificate for nodes in region two.

```
rm certs/node.crt
rm certs/node.key
```

Now repeat the process for region three, replacing the namespace name for namespace name for region three. In this example this is done via the use of environment variables.

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

Upload the node certificate and key to the Kubernetes cluster as a secret, specifying the appropriate context and namespace:

```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs \
--context $clus3 \
--namespace $loc3
```

Remove or store in a safe place the node certificate for nodes in region three.

```
rm certs/node.crt
rm certs/node.key
```

Apply the provided Kubernetes manifests into each AKS cluster. These contain several different resources including the statefulSet. In this file you will need to make a couple of edits. The first of these is to add the correct namespace name for each region to the StatefulSet config for each region. See the example below:

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cockroachdb
  # TODO: Use this field to specify a namespace other than "default" in which to deploy CockroachDB (e.g., us-east-1).
  namespace: northeurope
spec:
```

Next is to update the join command that is ran when the CockroachDB binary is started. This join command must contain the DNS name of all of the pods in the cluster. This is used when new pods join the CockroachDB cluster.
See an example below:

```
--join cockroachdb-0.cockroachdb.uksouth,cockroachdb-1.cockroachdb.uksouth,cockroachdb-2.cockroachdb.uksouth,cockroachdb-0.cockroachdb.ukwest,cockroachdb-1.cockroachdb.ukwest,cockroachdb-2.cockroachdb.ukwest,cockroachdb-0.cockroachdb.northeurpoe,cockroachdb-1.cockroachdb.northeurope,cockroachdb-2.cockroachdb.northeurope
```

Once you have updated the three files as required you can apply these to the three AKS clusters.

```
kubectl -n $loc1 apply -f ./manifest/uksouth-cockroachdb-statefulset-secure.yaml --context $clus1
kubectl -n $loc2 apply -f ./manifest/ukwest-cockroachdb-statefulset-secure.yaml --context $clus2
kubectl -n $loc3 apply -f ./manifest/northeurope-cockroachdb-statefulset-secure.yaml --context $clus3
```

Check to see if you pods are running. They should all be running but not ready.

```
kubectl get pods --context $clus1 --namespace $loc1
kubectl get pods --context $clus2 --namespace $loc2
kubectl get pods --context $clus3 --namespace $loc3
```

If they are all running we then need to initialize the cluster. This is done by accessing one of the CockroachDB pods and running `cockroach init`.

```
kubectl exec \
--context $clus1 \
--namespace $loc1 \
-it cockroachdb-0 \
-- /cockroach/cockroach init \
--certs-dir=/cockroach/cockroach-certs
```

Check the pods again, they should all now be ready.

```
kubectl get pods --context $clus1 --namespace $loc1
kubectl get pods --context $clus2 --namespace $loc2
kubectl get pods --context $clus3 --namespace $loc3
```

Now create a pod with a secure connection to the cluster and we can configure the cluster settings. Create the pod:

```
kubectl config use-context $clus1
kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/client-secure.yaml --namespace $loc1
```

Connect to the pod.

```
kubectl exec -it cockroachdb-client-secure -n $loc1 -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
```

Create and user and make it an admin.

```
CREATE USER craig WITH PASSWORD 'cockroach';
GRANT admin TO craig;
```

Apply your Enterprise Key if you have one...

```
SET CLUSTER SETTING cluster.organization = 'Cockroach Labs';
SET CLUSTER SETTING enterprise.license = 'crl-0-EMCU/cYGGAIiDkNvY2tyb2FjaCBMYWJz';
```

Configure the map view

```
INSERT INTO system.locations VALUES
  ('region', 'uksouth', 50.941, -0.799),
  ('region', 'ukwest', 53.427, -3.084),
  ('region', 'northeurope', 53.3478, -6.2597);
```

Expose the Admin UI externally.

```
kubectl apply -f ./manifest/admin-ui.yaml --context $clus1 --namespace $loc1
kubectl apply -f ./manifest/admin-ui.yaml --context $clus2 --namespace $loc2
kubectl apply -f ./manifest/admin-ui.yaml --context $clus3 --namespace $loc3
```

Check the Service has an IP address and test access to the UI.

```
kubectl get svc --context $clus1 --namespace $loc1
kubectl get svc  --context $clus2 --namespace $loc2
kubectl get svc --context $clus3 --namespace $loc3
```

Congratulations! You should now have a working cluster.....


## CleanUp

Delete you your certs and Azure resources.

```
rm -R certs my-safe-directory
az group delete --name $rg
```

Fix a broken cluster
```
kubectl -n $loc1 delete -f ./manifest/uksouth-cockroachdb-statefulset-secure.yaml --context $clus1
kubectl -n $loc2 delete -f ./manifest/ukwest-cockroachdb-statefulset-secure.yaml --context $clus2
kubectl -n $loc3 delete -f ./manifest/northeurope-cockroachdb-statefulset-secure.yaml --context $clus3
```

Delete the PVC for each of the CockroachDB pods.
```
kubectl delete pvc -l app=cockroachdb -n $loc1 --context $clus1
kubectl delete pvc -l app=cockroachdb -n $loc2 --context $clus2
kubectl delete pvc -l app=cockroachdb -n $loc3 --context $clus3
```

Redeploy the statefulSet manifests.
```
kubectl -n $loc1 apply -f ./manifest/uksouth-cockroachdb-statefulset-secure.yaml --context $clus1
kubectl -n $loc2 apply -f ./manifest/ukwest-cockroachdb-statefulset-secure.yaml --context $clus2
kubectl -n $loc3 apply -f ./manifest/northeurope-cockroachdb-statefulset-secure.yaml --context $clus3
```

[home](/README.md)