---
title: Azure Service Fabric capacity planning and scaling best practices | Microsoft Docs
description: Best practices for planning and scaling Service Fabric clusters and applications.
services: service-fabric
documentationcenter: .net
author: peterpogorski
manager: jeanpaul.connock  
editor: ''
ms.assetid: 19ca51e8-69b9-4952-b4b5-4bf04cded217
ms.service: service-fabric
ms.devlang: dotNet
ms.topic: conceptual
ms.tgt_pltfrm: NA
ms.workload: NA
ms.date: 01/23/2019
ms.author: pepogors
---

# Capacity planning and scaling

Before creating any Azure Service Fabric cluster or scaling compute resources hosting your cluster, it is important to plan for capacity. For more information about planning for capacity, see [Planning the Service Fabric cluster capacity](https://docs.microsoft.com/azure/service-fabric/service-fabric-cluster-capacity). In addition to considering Nodetype and Cluster characteristics, plan for scaling operations to take longer than an hour to complete for a production environment regardless of the number of VMs you are adding.

## Vertical scaling considerations

[Vertical scaling](https://docs.microsoft.com/azure/service-fabric/virtual-machine-scale-set-scale-node-type-scale-out#upgrade-the-size-and-operating-system-of-the-primary-node-type-vms) a Node Type in Azure Service Fabric requires a number of steps and considerations. For example:
* The cluster must be healthy before scaling. Otherwise you will only destabilize cluster further.
* **Silver durability level or greater** is required for all Service Fabric Cluster NodeTypes that host stateful services.

> [!NOTE]
> Your primary NodeType that hosts Stateful Service Fabric System Services must be Silver durability level or greater. Once you enable silver durability, cluster operations such as upgrades, adding or removing of nodes, and so on will be slower because the system optimizes for data safety over speed of operations.

Vertical scaling a Virtual Machine Scale Set is a destructive operation. Instead, horizontally scale your cluster by adding a new Scale Set with the desired SKU, and migrate your services to your desired SKU to complete a safe vertical scaling operation. Changing a Virtual Machine Scale Set resource SKU is a destructive operation because it reimages your hosts which removes all locally persisted state.

Service Fabric [node properties and placement constraints](https://docs.microsoft.com/azure/service-fabric/service-fabric-cluster-resource-manager-cluster-description#node-properties-and-placement-constraints) are used by your cluster to decide where to host your Applications services. When vertically scaling your primary node type, declare identical property values for `"nodeTypeRef"`, which is found in Virtual Machine Scale Set Service Fabric Extension. The following snippet of Resource Manager template shows the properties you will declare, with the same value for your new provisioned scale sets that you are scaling to, and is only supported as a temporary stateful for your cluster:

```json
"settings": {
   "nodeTypeRef": ["[parameters('primaryNodetypeName')]"]
}
```

> [!NOTE]
> Do not leave your cluster running with multiple scale sets that use the same `nodeTypeRef` property value longer than required to complete a successful vertical scaling operation.
> Always validate operations in test environments before attempting production environment changes. By default, Service Fabric Cluster System Services have a placement constraint to target primary nodetype only.

With the node properties and placement constraints declared, do the following steps one VM instance at a time. This allows the system services (and your stateful services) to be shut down gracefully on the VM instance you are removing as new replicas are created elsewhere.
1. From PowerShell, run `Disable-ServiceFabricNode` with intent ‘RemoveNode’ to disable the node you’re going to remove. Remove the node type that has the highest number. For example, if you have a six node cluster, remove the "MyNodeType_5" virtual machine instance.
2. Run `Get-ServiceFabricNode` to make sure that the node has transitioned to disabled. If not, wait until the node is disabled. This may take a couple hours for each node. Don't proceed until the node has transitioned to disabled.
3. Decrease the number of VMs by one in that Node type. The highest VM instance will now be removed.
4. Repeat steps 1 through 3 as needed, but never scale down the number of instances in the primary node types less than what the reliability tier warrants. See [Planning the Service Fabric cluster capacity](https://docs.microsoft.com/azure/service-fabric/service-fabric-cluster-capacity) for a list of recommended instances.

## Horizontal scaling

Horizontal Scaling in Service Fabric can be done either [manually](https://docs.microsoft.com/azure/service-fabric/service-fabric-cluster-scale-up-down) or [programmatically](https://docs.microsoft.com/azure/service-fabric/service-fabric-cluster-programmatic-scaling).

> [!NOTE]
> If you are scaling a node type that has silver or gold durability, scaling will be slow.

### Scaling out

Scale out a Service Fabric cluster by increasing the instance count for a particular Virtual Machine Scale Set. You can scale out programmatically by using the AzureClient and the ID for the desired scale set to increase the capacity.

```c#
var scaleSet = AzureClient.VirtualMachineScaleSets.GetById(ScaleSetId);
var newCapacity = (int)Math.Min(MaximumNodeCount, scaleSet.Capacity + 1);
scaleSet.Update().WithCapacity(newCapacity).Apply(); 
```

To scale out manually, update the capacity in the SKU property of the desired [Virtual Machine Scale Set](https://docs.microsoft.com/rest/api/compute/virtualmachinescalesets/createorupdate#virtualmachinescalesetosprofile) resource.
```json
"sku": {
    "name": "[parameters('vmNodeType0Size')]",
    "capacity": "[parameters('nt0InstanceCount')]",
    "tier": "Standard"
}
```

### Scaling in

Scaling in requires more consideration than scaling out. For example:
* Service Fabric system services run in the Primary node type in your cluster. Never shut down or scale down the number of instances for that node type so that you have fewer instances than what the reliability tier warrants. 
* For a stateful service you need a certain number of nodes that are always up to maintain availability and preserve the state of your service. At a minimum, you need a number of nodes equal to the target replica set count of the partition/service.

To scale in manually, follow these steps:

1. From PowerShell, run `Disable-ServiceFabricNode` with intent ‘RemoveNode’ to disable the node you’re going to remove. Remove the node type that has the highest number. For example, if you have a six node cluster, remove the "MyNodeType_5" virtual machine instance.
2. Run `Get-ServiceFabricNode` to make sure that the node has transitioned to disabled. If not, wait until the node is disabled. This may take a couple hours for each node. Don't proceed until the node has transitioned to disabled.
3. Decrease the number of VMs by one in that Node type. The highest VM instance will now be removed.
4. Repeat steps 1 through 3 as needed, but never scale down the number of instances in the primary node types less than what the reliability tier warrants. See [Planning the Service Fabric cluster capacity](https://docs.microsoft.com/azure/service-fabric/service-fabric-cluster-capacity) for a list of recommended instances.

To scale in manually, update the capacity in the SKU property of the desired [Virtual Machine Scale Set](https://docs.microsoft.com/rest/api/compute/virtualmachinescalesets/createorupdate#virtualmachinescalesetosprofile) resource.

```json
"sku": {
    "name": "[parameters('vmNodeType0Size')]",
    "capacity": "[parameters('nt0InstanceCount')]",
    "tier": "Standard"
}
```

1. Repeat steps 1 through 3 until you provision the capacity you want. Don't scale down the number of instances in the primary node types to less than what the reliability tier warrants. For details about reliability tiers and the number of instances they required, refer to the [Planning the Service Fabric cluster capacity](https://docs.microsoft.com/azure/service-fabric/service-fabric-cluster-capacity).

You must prepare the node for shutdown to scaling in programmatically. This involves finding the node to be removed which is the highest instance node and deactivating it. For example:

```c#
using (var client = new FabricClient())
{
    var mostRecentLiveNode = (await client.QueryManager.GetNodeListAsync())
        .Where(n => n.NodeType.Equals(NodeTypeToScale, StringComparison.OrdinalIgnoreCase))
        .Where(n => n.NodeStatus == System.Fabric.Query.NodeStatus.Up)
        .OrderByDescending(n =>
        {
            var instanceIdIndex = n.NodeName.LastIndexOf("_");
            var instanceIdString = n.NodeName.Substring(instanceIdIndex + 1);
            return int.Parse(instanceIdString);
        })
        .FirstOrDefault();
```

Once you identify the node to remove, deactivate and remove it using the same `FabricClient` instance (`client` in this case) and node instance name (`instanceIdString` in this case) you used in the code above:

```c#
var scaleSet = AzureClient.VirtualMachineScaleSets.GetById(ScaleSetId);

// Remove the node from the Service Fabric cluster
ServiceEventSource.Current.ServiceMessage(Context, $"Disabling node {mostRecentLiveNode.NodeName}");
await client.ClusterManager.DeactivateNodeAsync(mostRecentLiveNode.NodeName, NodeDeactivationIntent.RemoveNode);

// Wait (up to a timeout) for the node to gracefully shutdown
var timeout = TimeSpan.FromMinutes(5);
var waitStart = DateTime.Now;
while ((mostRecentLiveNode.NodeStatus == System.Fabric.Query.NodeStatus.Up || mostRecentLiveNode.NodeStatus == System.Fabric.Query.NodeStatus.Disabling) &&
        DateTime.Now - waitStart < timeout)
{
    mostRecentLiveNode = (await client.QueryManager.GetNodeListAsync()).FirstOrDefault(n => n.NodeName == mostRecentLiveNode.NodeName);
    await Task.Delay(10 * 1000);
}

// Decrement VMSS capacity
var newCapacity = (int)Math.Max(MinimumNodeCount, scaleSet.Capacity - 1); // Check min count 

scaleSet.Update().WithCapacity(newCapacity).Apply();
```

## Reliability levels

The [reliability level](https://docs.microsoft.com/azure/service-fabric/service-fabric-cluster-capacity) is a property of your Service Fabric Cluster resource, and can't be configured differently for individual nodeTypes. It controls the replication factor of the system services for the cluster, and is a setting at the cluster resource level. The reliability level will determine the minimum number of nodes that your primary node type must have. The reliability tier can take the following values:
* Platinum - runs the System services with a target replica set count of seven
* Gold - runs the System services with a target replica set count of seven
* Silver - runs the System services with a target replica set count of five
* Bronze - runs the System services with a target replica set count of three

The minimum recommended reliability level is Silver.

The reliability level is set in the properties section of the [Microsoft.ServiceFabric/clusters resource](https://docs.microsoft.com/azure/templates/microsoft.servicefabric/2018-02-01/clusters), like this:

```json
"properties":{
    "reliabilityLevel": "Silver"
}
```

## Durability levels

> [!WARNING]
> Node types running with Bronze durability obtain _no privileges_. This means that infrastructure jobs that impact your stateless workloads will not be stopped or delayed, which might impact your workloads. Use Bronze durability only for node types that run stateless workloads. For production workloads, run Silver or above to ensure state consistency. Choose the right reliability based on the guidance in the [capacity planning documentation](https://docs.microsoft.com/azure/service-fabric/service-fabric-cluster-capacity).

The durability level must be set in two resources. The extension profile of the [Virtual Machine Scale Set resource](https://docs.microsoft.com/rest/api/compute/virtualmachinescalesets/createorupdate#virtualmachinescalesetosprofile):

```json
"extensionProfile": {
    "extensions":          {
        "name": "[concat('ServiceFabricNodeVmExt','_vmNodeType0Name')]",
        "properties": {
            "settings": {
                "durabilityLevel": "Bronze"
            }
        }
    }
}
```

And under `nodeTypes` in the [Microsoft.ServiceFabric/clusters resource](https://docs.microsoft.com/azure/templates/microsoft.servicefabric/2018-02-01/clusters) 

```json
"nodeTypes": [
    {
        "name": "[variables('vmNodeType0Name')]",
        "durabilityLevel": "Bronze"
    }
]
```

## Next steps

* Create a cluster on VMs or computers running Windows Server: [Service Fabric cluster creation for Windows Server](service-fabric-cluster-creation-for-windows-server.md)
* Create a cluster on VMs or computers running Linux: [Create a Linux cluster](service-fabric-cluster-creation-via-portal.md)
* Learn about [Service Fabric support options](service-fabric-support.md)

[Image1]: ./media/service-fabric-best-practices/generate-common-name-cert-portal.png