# QRA Cloud VNet Azure Marketplace Deployment Guide

The QRA Cloud platform is delivered as a marketplace application that provisions and configures the cloud infrastructure required to host the solution in your environment.

The deployment creates and configures all necessary cloud resources, including:

- Kubernetes service
- Networking components
- Monitoring and supporting infrastructure

During deployment, customers can choose between:

1. **Deploy a New AKS Cluster (Recommended)**
2. **Bring Your Own AKS Cluster (BYO AKS)**

This document explains both options and their requirements.

## Deployment Options (Azure)

The QRA Cloud platform is deployed as an Azure Managed Application. The sections below describe the available deployment options for the Azure platform.

## Option 1 - Deploy a New Managed AKS Cluster (Recommended)

The marketplace deployment provisions and configures a new AKS cluster, including:

- Provision a new Kubernetes cluster
- Configure it according to platform requirements
- Apply required networking and security settings
- Install required Kubernetes components
- Configure ingress, scaling, and platform dependencies

### Advantages

- Fully automated deployment
- Pre-configured to meet QRA recommended platform standards
- No manual Kubernetes configuration required
- Simpler support model
- Reduced risk of misconfiguration

### Recommended For

- Customers without an existing AKS cluster
- Customers wanting a turnkey deployment
- Customers prioritizing simplicity and supportability

## Option 2 - Bring Your Own AKS Cluster(BYO AKS)


> **Important:** Because Azure Managed Applications deploy resources into a managed resource group, the deployment context does not have sufficient permissions to write resources outside of the managed resource group. Due to this limitation, additional configuration steps are required.

### BYO AKS Requirements

If you choose to bring your own AKS cluster, you must ensure the following prerequisites are met.

#### AKS Cluster Requirements

- Kubernetes version 1.32.9 or later
- Node pool
  - Mode: User
  - OS: Linux
  - Autoscaling: enabled
  - VM size: We recommend `F series v6` such as `Standard_F4als_v6`
  - Node labels
    - `qracloud.io/workload: qra`
- KEDA scaler enabled

#### Networking Requirements

- Virtual network
  - Address space of at least `/16`
  - The following subnets must be configured:

    | Subnet    | CIDR        |
    |-----------|-------------|
    | qraSystem | x.x.1.0/24 |
    | qraApp    | x.x.2.0/24 |
    | qraDb     | x.x.3.0/26 |
    | qraSql    | x.x.4.0/26 |
    | qraRedis  | x.x.5.0/27 |

- Load-balanced ingress is enabled. See [Required Kubernetes Components](#required-kubernetes-components) and take note of the ingress IP.

#### Required Kubernetes Components

You must pre-install the following components on your cluster before initiating the marketplace deployment.

**Ingress Controller** — used for ingress routing:

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace \
    --set controller.service.type=LoadBalancer \
    --set controller.service.externalTrafficPolicy=Local \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal"=true \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal-subnet"=<your system node pool subnet name> \
    --wait \
    --timeout 300s
```

**cert-manager** — used for TLS certificate management:

```bash
helm upgrade cert-manager oci://quay.io/jetstack/charts/cert-manager \
    --install \
    --version v1.19.2 \
    --namespace cert-manager \
    --create-namespace \
    --set crds.enabled=true \
    --set "extraArgs={--dns01-recursive-nameservers=ns1-06.azure-dns.com:53\,ns2-06.azure-dns.net:53\,ns3-35.azure-dns.org:53\,ns4-35.azure-dns.info:53\,8.8.8.8:53}" \
    --server-side --force-conflicts
```

**KEDA Scaler** — used for pod autoscaling:

```bash
helm upgrade --install keda kedacore/keda \
    --namespace keda \
    --create-namespace \
    --wait \
    --timeout 5m
```

**QRA Cloud Helm Chart** — installs the QRA Cloud platform:

```bash
helm upgrade qracloud oci://$REGISTRY_NAME/helm/qracloud/platform/qra \
    --install \
    --namespace qra --create-namespace \
    --set global.registry.name=$REGISTRY_NAME \
    --set global.registry.username=$REGISTRY_USERNAME \
    --set global.registry.password=$REGISTRY_PASSWORD \
    --set global.database.username=$DB_USERNAME \
    --set global.database.password=$DB_PASSWORD \
    --set keycloak.adminUser=$KC_ADMIN_USERNAME \
    --set keycloak.adminPassword=$KC_ADMIN_PASSWORD \
    --set global.tenantName="${TENANT_NAME}" \
    --set global.tenantAdmin=$TENANT_ADMINISTRATORS \
    --set infrastructure.telemetry.appInsights.connectionString="${APPINSIGHTS_CONNECTION_STRING}" \
    --set global.azureDNS.clientSecret=$TLS_SECRET \
    --set global.database.connectionString="${QRA_DB_CONNECTIONSTRING}" \
    --set global.keycloak.connectionString="${QRA_KC_DB_CONNECTIONSTRING}" \
    --set global.redis.connectionString="${QRA_REDIS_CONNECTIONSTRING}" \
    --set global.nodePool=$QRA_NODE_POOL \
    --set global.qwl.replicas=$QWL_REPLICAS \
    --wait
```

> A list of credentials and values will be provided separately to configure your deployment during marketplace installation.

## Identity & Permissions

The following permissions must be configured:

- AKS must allow workload identity or managed identity (if required)
- The platform must have sufficient RBAC permissions to:
  - Create namespaces
  - Deploy workloads
  - Create services and ingress resources
  - Create secrets and config maps

## Namespace & Isolation

The platform is deployed into the `qra` namespace. Ensure this namespace does not already exist or conflict with your cluster policies prior to deployment.

## Post-Installation Steps

After the marketplace deployment completes, the following steps are required to fully configure the deployment.

### VNet Connection

The solution uses its own VNet to configure its infrastructure. To allow your organization to access it, you must establish a connection to the provided VNet via VNet peering, VPN gateway, or a similar mechanism. **VNet peering** is recommended for its simplicity and low latency.

### DNS Zone Virtual Network Link

The solution provides a private DNS zone that makes domains such as `qracloud.io` and other PaaS service domains resolvable. A VNet must be linked to the DNS zone before it can resolve these domains.

#### Managed AKS

For managed AKS deployments, link your VNet to the `qracloud.io` private DNS zone.

#### BYO AKS

For BYO AKS deployments, link your VNet to the following private DNS zones created by the deployment:

![Private DNS zones](./assets/private-dns-zones.png)

## Recommendation

We strongly recommend deploying a **new AKS cluster through the Managed Application** unless:

- You accept the increased operational ownership that comes with a custom AKS configuration outside QRA's standard support model — overall operational overhead is higher due to the additional maintenance and complexity on your end
- You require consolidation into an existing AKS cluster
- You have existing Kubernetes operational expertise

Using a managed AKS deployment ensures:

- Faster onboarding
- Reduced configuration risk
- A fully supported deployment posture

> If you need assistance evaluating which option is best for your environment, please contact our [technical support team](mailto:support@qracorp.com) before initiating deployment.
