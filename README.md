# Installing OpenShift on Azure (IPI)

## Introduction

The Installer Provisioned Infrastructure (IPI) approach is appropriate for most client requirements, and simplifies OpenShift installations with automation. However, it is critical to understand exactly what the installer is provisioning so you can (1) help the client ensure the cluster infrastructure is in policy, and (2) explain the solution with confidence. 

The IPI approach uses the openshift-install installation program. It is a wrapper for terraform, and the terraform scripts are not publicly available. The only way to verify installation details that are not documented is to open a support ticket, or deploy the OpenShift installer in a test environment.

This write up will summarize some of the Azure cloud topics pertinent to OpenShift, and containerized products like MAS and Cloud Pak for Data.


## OpenShift IPI Approach Options on Azure

1. Azure Marketplace Templates

IBM has deployed Azure templates that install OpenShift via IPI, then Maximo Applicaition Suite and/ or Cloud Pak for Data. This option has limited flexibility, since if an option is not part of the template, it cannot be configured. For example, airgapped clusters are not supported at this time. This approach is best suited for POCs or test environments with straightfoward requirements.

![Screenshot 2023-09-06 at 8 46 44 AM](https://github.com/kathleenhosang/az-ocp/assets/40863347/b2b38180-79d0-4340-b908-de2d27b052de)


2. OpenShift Installer

The OpenShift installer uses the ```openshift-install``` package to deploy an OpenShift cluster via IPI. It will prompt for information about the platform (in this case Azure) and will use Azure specific deployment scripts. This is the approach this page focuses on. The UPI (User Provioned Infrastructure) approach has more flexibility if the IPI approach does not meet client requirements. The most common client concern with IPI is related to security, since in principal, you are allowing the deployment scripts to control the cloud environment infrastructure. See the documentation here: https://docs.openshift.com/container-platform/4.10/installing/installing_azure/installing-azure-private.html

![Screenshot 2023-09-06 at 5 54 16 PM](https://github.com/kathleenhosang/az-ocp/assets/40863347/ef92d86e-55d4-4574-9582-fc76819b3b2d)



## Resource Groups

In Azure, resources are logically organzied into resource groups. Resource groups are simply ways to manage Azure resources as a group- applying the same policies, access control, and lifecycle management. See the below image and link for more information on the different scopes of how Azure resources are managed.

![image](https://github.com/kathleenhosang/az-ocp/assets/40863347/a0209bb8-2e8b-43fd-b445-6005c88d48c4)

https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/overview#understand-scope

This is an important concept when deploying OpenShift on Azure via IPI. When the installer deploys OpenShift, it places all of the OpenShift infrastructure into a new resource group. No other resources should be placed in this resource group. If you are defining a new resource group in the ```install-config.yaml``` for the OpenShift infrastructure, it must be empty. The OpenShift program uses the resource group to manage OpenShift infrastructure. For example, when using the ```destroy cluster``` command, the installer will delete the defined OpenShift resource group.

When assigning permissions, most clients prefer to assign at the resource group level, rather than subscription level for more control. This can apply to the service principal that is used to deploy OpenShift. It needs permissions to the OpenShift resource group (which would need to be created before the time of installation), and the resource group that ccontains the DNS service and target network.

## Default Routing vs. User Defined Routing

When deploying OpenShift, the IPI installer consumes an ```install-config.yaml``` with a list of parameters. One of the optional parameters is ```OutboundType```, which can have the value ```LoadBalancer``` or ```UserDefinedRouting```. If User Defined Routing is employed, the external load balancer will not be assigned a public IP, as seen in the below diagram.


<img width="753" alt="Screenshot 2023-09-08 at 10 36 36 AM" src="https://github.com/kathleenhosang/az-ocp/assets/40863347/fe19c0a5-1e13-4285-8ef5-97571e044bcd">


See more details on using User Defined Routing in an OpenShift cluster: https://docs.openshift.com/container-platform/4.10/installing/installing_azure/installing-azure-private.html#installation-azure-user-defined-routing_installing-azure-private

The default parameter value is ```LoadBalancer``` (default routing). Azure has a set of default behaviors that it will follow. When default routing is employed, OpenShift will create a load balancer with a public IP for outbound connectivity only. This is the default mechanism the load balancer uses to connect itself/ its backend pool to the outside world.
Read more here: https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview#system-routes.

User defined routing means that routing tables are being used to manually control every hop traffic takes. Azure will not employ default routes; the assumption and requirement is that the user needs to set up outbound connectivity. If User Defined Routing is employed, the load balancer will not be assigned a public IP. The load balancer requires explicit outbound connectivity. Ways to establish explicit connectivity include: employing a firewall, NAT gateway, or public IP.
Read more on explicit connectivity here: https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access.

In summary, when User Defined Routing is employed, the cluster network must use a NAT gateway or firewall for outbound connectivity. This is not an OpenShift requirement, this is an Azure load balancer requirement.

## Airgapped and IPI

The IPI approach requires connectivity to Azure APIs, which require outbound internet connectivity. These APIs are only required during the time of installation, and are on the allowlist URLs in the OpenShift documentation:

<img width="738" alt="Screenshot 2023-09-08 at 11 00 22 AM" src="https://github.com/kathleenhosang/az-ocp/assets/40863347/48a46ebc-028c-44f6-b49a-bf0ce0677a3c">

See full allowlist here: https://docs.openshift.com/container-platform/4.10/installing/install_config/configuring-firewall.html#configuring-firewall

This means that the IPI approach on Azure does not support fully airgapped solutions. Azure APIs do use an internal Azure backbone, they require outbound connectivity. The OpenShift cluster subnets require access to Azure APIs:

<img width="750" alt="Screenshot 2023-09-08 at 10 59 04 AM" src="https://github.com/kathleenhosang/az-ocp/assets/40863347/844ddee7-de6d-4ae8-9a64-ede3921c72bb">


## Using Enterprise DNS

The IPI approach requires Azure DNS, specifically Azure Private DNS for private cluster.

Azure private DNS may use DNS forwarding to forward records to an enterprise DNS server. This is the recommended approach.

It is possible to install using the Azure Private DNS, manually migrate all records to the enterprise DNS, then replace the load balancer IP to point to the enterprise DNS IP. This is not the recommended approach, but if a client would like to do this, they may do so with their networking team taking lead, with the understanding that this is not an approach IBM or Red Hat is able to support. 

## Azure Container Registry

Azure Container Registry (ACR) is a docker 2v2 schema compliant registry that may be used to mirror Cloud Pak for Data or MAS images.

It’s port is 8080 and is implicit; as of 2023, the port for ACR cannot be included in any oc mirror commands or it will fail.

## Azure Storage Accounts

## Azure File and Block Storage

Azure File and Block storage 

Azure Block Storage is created by default when installing via IPI. Validate if additional steps are required to use

Azure File Storage must be created manually. Please see this guide to do so:
Add guide

Be aware that Azure File and Block storage are not supported options for any Cloud Pak solution. Development found both, especially Azure File, to not be performant. They found that using the storage classes can result in locked files, due to lagging start up times.

## Azure Key Vault

Azure Key Vault (AKV) is a solution used to store secrets, and is commonly used for certificate management. IBM cert-manager does not currently support AKV. Enhancement request is being tracked here: get link where we are tracking the enhancement request
