# Installing OpenShift on Azure (IPI)

## Introduction

The infrastructure provisioned installation (IPI) approach is appropriate for most client requirements, and simplifies the installation with automation. However, it is critical to understand exactly what it is doing so you can (1) help the client ensure the cluster infrastructure is in policy, and (2) explain the solution with confidence. 

The IPI approach uses the openshift-install installation program. It is a wrapper for terraform, and the terraform scripts are not publicly available. The only way to verify installation details not documented is to open a support ticket, or deploy the OpenShift installer in a test environment.

This write up will summarize some of the Azure cloud topics pertinent to OpenShift, and containerized products like MAS and Cloud Pak for Data.

## OpenShift IPI Approach Options on Azure

1. Azure Marketplace Template for MAS

2. Azure Marketplace Template for CPD

3. OpenShift Installer

This is what this guide will cover

## Resource Groups

## Default Routing vs. User Defined Routing

When deploying OpenShift, the IPI installer consumes an install-config.yaml with a list of parameters. One of the optional parameters is OutboundRouting.

Get screenshot of explanation from docs

The two options are default routing, or user defined routing.

The default parameter value is default routing. Azure has a set of default behaviors that it will follow. Read more here. For example, private virtual machines are assigned a pseudo IP address managed by Azure. This is how outbound connectivity is secured.

User defined routing means that routing tables are being used to manually control every hop traffic takes. Azure will not employ default routes; the assumption and requirement is that the user needs to set up outbound connectivity.

When default routing is employed, OpenShift will create a load balancer with a public IP for outbound connectivity only. This is the default mechanism the load balancer uses to connect itself/ its backend pool to the outside world.

If User Defined Routing is employed, the load balancer will not be assigned a public IP. The load balancer requires explicit outbound connectivity. Read more on explicit connectivity here. Ways to establish explicit connectivity include: employing a firewall, NAT gateway, or public IP.

This means, that when User Defined Routing is being used, the cluster network must use a NAT gateway or firewall for outbound connectivity. This is not an OpenShift requirement, this is an Azure load balancer requirement.

## Airgapped and IPI

The IPI approach requires connectivity to Azure APIs, which require outbound internet connectivity. These APIs are only required during the time of installation, and are on the allowlist URLs in the OpenShift documentation:

This means that the IPI approach on Azure does not support fully airgapped solutions. Azure APIs do use an internal Azure backbone, they require outbound connectivity.

## Using Enterprise DNS

The IPI approach requires Azure DNS, specifically Azure Private DNS for private cluster.

Azure private DNS may use DNS forwarding to forward records to an enterprise DNS server. This is the recommended approach.

It is possible to install using the Azure Private DNS, manually migrate all records to the enterprise DNS, then replace the load balancer IP to point to the enterprise DNS IP. This is not the recommended approach, but if a client would like to do this, they may do with their networking team taking lead. 

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
