
# CockroachDB Multi Region on Azure Kubernetes Service

In this demo we will install CockroachDB in a multi regional configuration running in Kubernetes on Azure Kubernetes Service (AKS)

- Create a set of variables

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

1. [Setup Azure networking and Infrastructure.](/markdown/infra-setup.md)

2. [CockroachDB installation and configuration.](/markdown/crdb-setup.md)
