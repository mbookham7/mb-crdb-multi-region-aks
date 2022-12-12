## Run a workload

Open a second terminal window. Create three variables with the region names desired.
```bash
vm_type="Standard_D4_v2"
n_nodes=3
rg="mb-crdb-aks-multi-region"
clus1="crdb-aks-uksouth"
clus2="crdb-aks-ukwest"
clus3="crdb-aks-northeurope"
clus1_vnet_address_space="10.1.0.0/16"
clus2_vnet_address_space="10.2.0.0/16"
clus3_vnet_address_space="10.3.0.0/16"
clus1_subnet="10.1.1.0/24"
clus2_subnet="10.2.1.0/24"
clus3_subnet="10.3.1.0/24"  
loc1="uksouth"
loc2="ukwest"
loc3="northeurope"
```

Exec into the secure client pod and get a shell command.
```
kubectl exec -it cockroachdb-client-secure -n $loc1 --context $clus1 -- sh
```

Now we can run the simulated workload. First initalise the database the run the workload.
```
cockroach workload init movr 'postgresql://craig:cockroach@cockroachdb-public:26257/movr?sslmode=verify-full&sslrootcert=/cockroach-certs/ca.crt'

cockroach workload run movr --tolerate-errors --duration=99999m 'postgresql://craig:cockroach@cockroachdb-public:26257/movr?sslmode=verify-full&sslrootcert=/cockroach-certs/ca.crt'
```

## Kube-doom Setup

Open another terminal window and deploy Kube-Doom using Kustomize.
Create three variables with the region names desired.
```
vm_type="Standard_D4_v2"
n_nodes=3
rg="mb-crdb-aks-multi-region"
clus1="crdb-aks-uksouth"
clus2="crdb-aks-ukwest"
clus3="crdb-aks-northeurope"
clus1_vnet_address_space="10.1.0.0/16"
clus2_vnet_address_space="10.2.0.0/16"
clus3_vnet_address_space="10.3.0.0/16"
clus1_subnet="10.1.1.0/24"
clus2_subnet="10.2.1.0/24"
clus3_subnet="10.3.1.0/24"  
loc1="uksouth"
loc2="ukwest"
loc3="northeurope"
```

Change Context to northeurope.
```
kubectx $clus3
```

Deploy kube-doom using Kustomize.
```
kubectl apply -k manifest/
```

Now port forward port `5900` so you are able to use VNC Viewer to connect to the pod running kube-doom.
```
kubedoom_pod=$(kubectl get pods -n kubedoom -o name --no-headers=true)
echo $kubedoom_pod
kubectl port-forward -n kubedoom $kubedoom_pod 5900
```

To get access the the Doom interface you will need VNCViewer. This can be downloaded free from [here](https://www.realvnc.com/en/connect/download/viewer/).

VNC Password:
```
idbehold
```

Doom Cheats: 
```
IDDQD - invulnerability
IDKFA - full health, ammo, etc
IDSPISPOPD - no clipping / walk through walls
```