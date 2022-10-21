# AKS - Multi Region cockroachDB

Description: Setting up and configuring a multi region cockroach cluster on Azure AKS
Tags: Azure

- Create a Resource Group for the project

    ```bash
    az group create --name $rg --location $loc1
    ```

- Networking configuration

    In order to enable VPC peering between the regions, the CIDR blocks of the VPCs must not overlap. This value cannot change once the cluster has been created, so be sure that your IP ranges do not overlap.

    - Create vnets for all Regions

        ```bash
        az network vnet create -g $rg -n crdb-$loc1 --location $loc1 --address-prefix $clus1_vnet_address_space \
            --subnet-name crdb-$loc1-sub1 --subnet-prefix $clus1_subnet
        ```

        ```bash
        az network vnet create -g $rg -n crdb-$loc2 --location $loc2 --address-prefix $clus2_vnet_address_space \
            --subnet-name crdb-$loc2-sub1 --subnet-prefix $clus2_subnet
        ```

        ```bash
        az network vnet create -g $rg -n crdb-$loc3 --location $loc3 --address-prefix $clus3_vnet_address_space \
            --subnet-name crdb-$loc3-sub1 --subnet-prefix $clus3_subnet
        ```

    - Peer the Vnets

        ```bash
        az network vnet peering create -g $rg -n $loc1-$loc2-peer --vnet-name crdb-$loc1 \
            --remote-vnet crdb-$loc2 --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit
        ```

        ```bash
        az network vnet peering create -g $rg -n $loc2-$loc3-peer --vnet-name crdb-$loc2 \
            --remote-vnet crdb-$loc3 --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit
        ```

        ```bash
        az network vnet peering create -g $rg -n $loc1-$loc3-peer --vnet-name crdb-$loc1 \
            --remote-vnet crdb-$loc3 --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit
        ```

        ```bash
        az network vnet peering create -g $rg -n $loc2-$loc1-peer --vnet-name crdb-$loc2 \
            --remote-vnet crdb-$loc1 --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit
        ```

        ```bash
        az network vnet peering create -g $rg -n $loc3-$loc2-peer --vnet-name crdb-$loc3 \
            --remote-vnet crdb-$loc2 --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit
        ```

        ```bash
        az network vnet peering create -g $rg -n $loc3-$loc1-peer --vnet-name crdb-$loc3 \
            --remote-vnet crdb-$loc1 --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit
        ```

- Create the Kubernetes clusters in each region
    - To get SubnetID

        ```bash
        loc1subid=$(az network vnet subnet list --resource-group $rg --vnet-name crdb-$loc1 | jq -r '.[].id')
        loc2subid=$(az network vnet subnet list --resource-group $rg --vnet-name crdb-$loc2 | jq -r '.[].id')
        loc3subid=$(az network vnet subnet list --resource-group $rg --vnet-name crdb-$loc3 | jq -r '.[].id')
        ```

    - Create K8s Clusters in each region

        ```bash
        az aks create \
        --name $clus1 \
        --resource-group $rg \
        --network-plugin azure \
        --zones 1 2 3 \
        --vnet-subnet-id $loc1subid \
        --node-vm-size $vm_type \
        --node-count $n_nodes \
        --location $loc1
        ```

        ```bash
        az aks create \
        --name $clus2 \
        --resource-group $rg \
        --network-plugin azure \
        --vnet-subnet-id $loc2subid \
        --node-vm-size $vm_type \
        --node-count $n_nodes \
        --location $loc2
        ```

        ```bash
        az aks create \
        --name $clus3 \
        --resource-group $rg \
        --network-plugin azure \
        --zones 1 2 3 \
        --vnet-subnet-id $loc3subid \
        --node-vm-size $vm_type \
        --node-count $n_nodes \
        --location $loc3

        ```

    - To Configure Kubectl context use

        ```bash
        az aks get-credentials --name $clus1 --resource-group $rg
        ```

        ```bash
        az aks get-credentials --name $clus2 --resource-group $rg
        ```

        ```bash
        az aks get-credentials --name $clus3 --resource-group $rg
        ```

[home](/README.md)