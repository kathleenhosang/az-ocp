# Installing OpenShift on Azure (IPI)

Example Architecture
<img width="996" alt="Screenshot 2023-09-08 at 11 14 58 AM" src="https://github.com/kathleenhosang/az-ocp/assets/40863347/8911562c-c65a-45db-a2ff-f77294375c42">

## Introduction

The Installer Provisioned Infrastructure (IPI) approach is appropriate for most client requirements, and simplifies OpenShift installations with automation. However, it is critical to understand what the installer is provisioning so you can (1) help clients ensure the cluster infrastructure is in policy, and (2) explain the solution with confidence. 

The IPI approach uses the ```openshift-install``` installation program. It is a wrapper for terraform, and the terraform scripts are not publicly available. The only way to verify installation details that are not documented is to open a support ticket, or deploy the OpenShift installer in a test environment.

This write up will summarize some of the Azure cloud topics pertinent to OpenShift, and containerized products like MAS and Cloud Pak for Data.

See Red Hat documentation here: https://docs.openshift.com/container-platform/4.10/installing/installing_azure/installing-azure-private.html

## OpenShift IPI Approach Options on Azure

1. Azure Marketplace Templates

IBM has deployed Azure templates that install OpenShift via IPI, then Maximo Application Suite and/ or Cloud Pak for Data. This option has limited flexibility, since if an option is not part of the template, it cannot be configured. For example, airgapped clusters are not supported at this time. This approach is best suited for POCs or test environments with straightfoward requirements.

![Screenshot 2023-09-06 at 8 46 44 AM](https://github.com/kathleenhosang/az-ocp/assets/40863347/b2b38180-79d0-4340-b908-de2d27b052de)


2. OpenShift Installer

The OpenShift installer uses the ```openshift-install``` package to deploy an OpenShift cluster via IPI. It will prompt for information about the platform (in this case Azure) and will use Azure specific deployment scripts. This is the approach this page focuses on. The UPI (User Provioned Infrastructure) approach has more flexibility if the IPI approach does not meet client requirements. The most common client concern with IPI is related to security, since in principal, it allows the deployment scripts to control the cloud environment infrastructure. See the documentation here: https://docs.openshift.com/container-platform/4.10/installing/installing_azure/installing-azure-private.html

![Screenshot 2023-09-06 at 5 54 16 PM](https://github.com/kathleenhosang/az-ocp/assets/40863347/ef92d86e-55d4-4574-9582-fc76819b3b2d)



## Resource Groups

In Azure, resources are logically organzied into resource groups. Resource groups are simply ways to manage Azure resources as a group, applying the same policies, access control, and lifecycle management to a set of resources. See the below image and link for more information on the different scopes of how Azure resources are managed.

![image](https://github.com/kathleenhosang/az-ocp/assets/40863347/a0209bb8-2e8b-43fd-b445-6005c88d48c4)

https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/overview#understand-scope

This is an important concept when deploying OpenShift on Azure via IPI. When the installer deploys OpenShift, it places all of the OpenShift infrastructure into a new resource group. No other resources should be placed in this resource group. If you are defining a new resource group in the ```install-config.yaml``` for the OpenShift infrastructure, it must be empty. The OpenShift program uses the resource group to manage OpenShift infrastructure. For example, when using the ```destroy cluster``` command, the installer will delete the defined OpenShift resource group.

When assigning permissions, most clients prefer to assign at the resource group level, rather than subscription level for more precise control. This can apply to the service principal that is used to deploy OpenShift. It needs permissions to the OpenShift resource group (which would need to be created before the time of installation), and the resource group that contains the DNS service and target network.

## Default Routing vs. User Defined Routing

When deploying OpenShift, the IPI installer consumes an ```install-config.yaml``` with a list of parameters. One of the optional parameters is ```OutboundType```, which can have the value ```LoadBalancer``` or ```UserDefinedRouting```. If User Defined Routing is employed, the external load balancer will not be assigned a public IP, as seen in the below diagram.


<img width="753" alt="Screenshot 2023-09-08 at 10 36 36 AM" src="https://github.com/kathleenhosang/az-ocp/assets/40863347/fe19c0a5-1e13-4285-8ef5-97571e044bcd">


See more details on using User Defined Routing in an OpenShift cluster: https://docs.openshift.com/container-platform/4.10/installing/installing_azure/installing-azure-private.html#installation-azure-user-defined-routing_installing-azure-private

The default parameter value is ```LoadBalancer``` (default routing). Azure has a set of default behaviors that it will follow. When default routing is employed, OpenShift will create a load balancer with a public IP for outbound connectivity only. This is the default mechanism the load balancer uses to connect itself and its backend pool to the outside world.
Read more here: https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview#system-routes.

User defined routing means that routing tables are being used to manually control every hop traffic takes. Azure will not employ default routes; the assumption and requirement is that the user needs to set up outbound connectivity. If User Defined Routing is employed, the load balancer will not be assigned a public IP. The load balancer requires explicit outbound connectivity. Ways to establish explicit connectivity include: employing a firewall, NAT gateway, or public IP.
Read more on explicit connectivity here: https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access.

In summary, when User Defined Routing is employed, the cluster network must use a NAT gateway or firewall for outbound connectivity. This is not an OpenShift requirement, this is an Azure load balancer requirement.

## Airgapped and IPI

The IPI approach requires connectivity to Azure APIs, which require outbound internet connectivity. These APIs are only required during the time of installation, and are on the allowlist URLs in the OpenShift documentation:

<img width="738" alt="Screenshot 2023-09-08 at 11 00 22 AM" src="https://github.com/kathleenhosang/az-ocp/assets/40863347/48a46ebc-028c-44f6-b49a-bf0ce0677a3c">

See full allowlist here: https://docs.openshift.com/container-platform/4.10/installing/install_config/configuring-firewall.html#configuring-firewall

This means that the IPI approach on Azure does not support fully airgapped solutions. Azure APIs do use an internal Azure backbone, they require outbound connectivity. The OpenShift cluster subnets require access to Azure APIs, per the below diagram showing the OpenShift cluster at the time of installation:

<img width="750" alt="Screenshot 2023-09-08 at 10 59 04 AM" src="https://github.com/kathleenhosang/az-ocp/assets/40863347/844ddee7-de6d-4ae8-9a64-ede3921c72bb">


## Using Enterprise DNS

The IPI approach requires Azure DNS, specifically Azure Private DNS for private clusters.

The private VNET must be linked to a Private Azure DNS resource. For example:

<img width="950" alt="Screenshot 2023-09-08 at 11 23 18 AM" src="https://github.com/kathleenhosang/az-ocp/assets/40863347/77ebc3b2-961a-41bc-8824-035e78ec9d83">



Azure private DNS may use DNS forwarding to forward records to an enterprise DNS server. This is the recommended approach when Enterpise DNS is a requirement for an IPI install on Azure. See documentation to set up DNS fowarding here: https://learn.microsoft.com/en-us/azure/dns/private-resolver-hybrid-dns

However, it is possible to forgo DNS forwarding and use the Enterprise DNS directly, though this should not be recommended unless there is a legitimate reason why DNS forwarding cannot be used. It is possible to install using Azure Private DNS, manually migrate all records to the enterprise DNS, then at the VNET level, set the DNS IP to the enterprise DNS IP (rather than using link to Azure private DNS). This is not the recommended approach, but if a client would like to do this, they may do so with their networking team taking lead, with the understanding that this is an approach IBM, Red Hat, and Microsoft is unable to support since the client will be managing their own Enterprise DNS set up. 


<img width="1263" alt="Screenshot 2023-09-08 at 11 32 23 AM" src="https://github.com/kathleenhosang/az-ocp/assets/40863347/7fcefe89-2d23-4fc6-9bb6-88680f664fc1">



## Azure Container Registry

Azure Container Registry (ACR) is a docker manifest v2 schema 2 compliant registry that may be used to mirror Cloud Pak for Data or MAS images.

Its port is 8080 and is implicit; the port does not need to be included in any ```oc mirror``` commands.

For push and pull credentials, the admin user should be enabled, and its credentials should be used.

<img width="1033" alt="Screenshot 2023-09-08 at 11 46 13 AM" src="https://github.com/kathleenhosang/az-ocp/assets/40863347/52bc2bf2-1ded-4c08-817f-92e1ca97726d">

When using Azure Cloud, ACR is a good option for a private registry for its ease of use and management. See further documentation here: https://learn.microsoft.com/en-us/azure/container-registry/

## Azure Storage

Azure Storage Accounts are used to logically group together Azure storage resources, including: block, file, container, disk, or any storage object. The IPI installer will create a storage account under the OpenShift resource group. It will be immediately used to host boot diagnostic logs, and the ignition files used for installation.

If a client has Azure policies around Storage Accounts, it's possible they will block the storage account from being created, and thus this approach cannot be used. Most the policies can be changed as a post-install task, which may be acceptable to the client.

Some storage classes are created by default when installing via IPI, depending on account settings and OCP version. Please see this guide to create Azure File Storage Account for MAS: https://github.com/Azure/maximo#azure-files-csi-drivers 

If using Azure storage classes for IBM applications, first test that the storage classes work using ```Step 6: Finally, test your environment```: https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/

Be aware that Azure File and Block storage are not supported options for any Cloud Pak solution. Development found both, especially Azure File, to be non-performant. They found that using the storage classes can result in locked files, due to lagging start up times. See supported storage options for Cloud Pak for Data here: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.7.x?topic=planning-storage-considerations

## Azure Key Vault

Azure Key Vault (AKV) is a solution used to store secrets, and is commonly used for certificate management. IBM cert-manager does not currently support AKV. Enhancement request is in place.


![image](https://github.com/kathleenhosang/az-ocp/assets/40863347/f5d9fe2c-7e29-43f9-b354-5d2bee33b3ab)
Diagram from: https://github.ibm.com/CloudPakForDataSWATAssets/Sustainability-Starter/blob/main/docs/maximo/day-2-operations/cerrificate-management-externalkeystore.md

