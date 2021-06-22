# Deployment details

The following resources are deployed as part of the demo:
- [API Management service](https://azure.microsoft.com/en-gb/services/api-management/) - setting up API Management resources required to support the demo project (APIs, Products, Policies, Named Values)
- [App Service Plan](https://docs.microsoft.com/en-us/azure/app-service/overview)
- [Web App for Containers](https://azure.microsoft.com/en-gb/services/app-service/containers/), using the billing portal app container image.
- [Service Principal RBAC assignment](https://docs.microsoft.com/en-us/azure/role-based-access-control/overview)

## Technology

This project is currently using [bicep](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/bicep-overview) for local development and deployment. Bicep is a templating language for declaratively deploying Azure resources. However, at the time of development the `Deploy to Azure` button does not support Bicep, so it must be decompiled into an ARM template prior to deployment. This happens when the solution is built (which is done by running the [build.ps1](../build.ps1) script.

The ARM template which is generated on build can be found in the [/output](../output/) folder.

The deployment can then be run using the [deploy.ps1](../deploy.ps1) script (see steps in the [README](../README.md)).

## API Management Service

Deploying the demo deploys an instance of Azure API Management service. As part of this deployment we define a set of products, APIs (with accompanying policies) and named values.

### Products 

The products on the APIM instance are defined in [apiConfiguration/config.yaml](../apiConfiguration/config.yaml).

An example product definition is as follows:

```yaml
name: basic
displayName: Basic
description: Basic tier with a monthly quota of 50,000 calls.
terms: Terms here
subscriptionRequired: true
approvalRequired: true
state: published
policy: .\apiConfiguration\policies\products\basic.xml
```

Here we are defining the basic product. We define a name, display name, and description. We make it so that subscription is required in order to use this product, and approval is required before that subscription is activated. We also link to a policy file for that product. The policy file for the basic product is:

```xml
<policies>
  <inbound>
    <base />
    <quota calls="50000" renewal-period="2592000" />
    <rate-limit calls="100" renewal-period="60"/>
  </inbound>
  <backend>
    <base />
  </backend>
  <outbound>
    <base />
  </outbound>
  <on-error>
    <base />
  </on-error>
</policies>
```

Here we are defining an inbound policy that limits a single subscription to 50,000 calls per month. It also adds a rate limiting policy to ensure that a single subscription is limited to 100 calls every minute. 

The ARM templates for deploying these policies are generated from the [/templates/apimmonetization-productGroups bicep file](../templates/apimmonetization-productGroups.bicep). As part of deployment, we define who can access each product. For the following products:

- Free
- Developer
- PAYG
- Basic 
- Standard
- Pro
- Enterprise

It is set up so that developers and guests are able to sign up to the products. 

The only product for which this is not true is the `Admin` product. This is purely used internally for communication between services, so we don't want to expose this as a product which consumers are able to sign up for and use.

### APIs

There are two APIs which are defined as part of the deployment of the APIM instance.

- The address API
- The billing API

The overall configuration of these APIs is also defined in [apiConfiguration/config.yaml](../apiConfiguration/config.yaml).

We can see that each API is linked to specific products.

#### Address API

The address API is the example API which we are giving consumers access to via APIM and Stripe. In your solution, you would replace this with the APIs which consumers will be subscribing to. In our example, a subscriber is able to post an address to the API, which will then be validated.

See [apiConfiguration/openApi/address.yaml](../apiConfiguration/openApi/address.yaml) for the definition for this API.

In [apiConfiguration/config.yaml](../apiConfiguration/config.yaml) we can also see that this API can be accessed using a range of policies: free, developer, payg, basic, standard, pro, enterprise. I.e. this is a consumer-facing API that can be accessed using any of the tiers.

This means that anyone that signs up to any of the above products will be able to access this API using their API key. However, depending on the product they may be limited to a specific number of calls per month, or a specific rate of calls per minute.

#### Billing API

This API is an internal API. 

There are two endpoints defined on this API.

The first in the `monetizationModels` endpoint. It is used to retrieve the monetization models defined in the [payment/monetizationModels.json](../payment/monetizationModels.json) file. 

The second is the `products` endpoint. Incoming requests are redirected to the APIM management API and this endpoint is used to retrieve the products which are defined on the APIM instance. 

See [apiConfiguration/openApi/billing.yaml](../apiConfiguration/openApi/billing.yaml) for the definition for this API.

This API is only exposed under the `Admin` product and will not be exposed to consumers.

#### Overall

In summary, we have defined one internal (billing) and one external (address) API. The only API that consumers will be able to subscribe to (using the set of differently priced products) will be the address API.

Each of these APIs and products is defined in [apiConfiguration/config.yaml](../apiConfiguration/config.yaml), and each has a set of policies defined, and an Open API definition attached.

### Named Values

APIM also has a concept of named values. Named values allow you to define specific terms which will be replaced within your policies. For example, if we have the following policy definition:

```xml
<inbound>
  <base />
  <set-backend-service base-url="https://management.azure.com/subscriptions/{{subscriptionId}}/resourceGroups/{{resourceGroupName}}/providers/Microsoft.ApiManagement/service/{{apimServiceName}}" />
</inbound>
```
We can see that there are certain values within this definition that will need to be replaced. We can defined named values on the APIM instance which will automatically override these values. 

These values are defined in the [templates/apimmonetization-namedValues](../templates/apimmonetization-namedValues.bicep) file. Here we set up a list of values, and what they should be replaced by in the deployed instance. These named values are also set as part of deployment.

## Billing portal

Aside from APIM, the other resource that is deployed by the deployment script is the billing portal. The billing portal is a node.js app which consumers are directed to in order to activate their subscriptions. The integration between APIM and the billing portal is handled using the [user registration and product subscription delegation features in APIM](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-setup-delegation).

As part of the main deployment, the billing portal app is deployed to [Azure Web App for Containers](https://azure.microsoft.com/en-gb/services/app-service/containers/). The container image is pulled from the [GitHub Container Registry](https://docs.github.com/en/packages/guides/about-github-container-registry) associated with this repo.

Some additional configuration is added to the app as part of the payment provider initialisation, which is explained in more detail for both [Stripe](./stripe-details.md) and [Adyen](./adyen-details.md) separately.