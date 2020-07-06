---
title: "Get change notifications delivered different ways (preview)"
description: "Change notifications can be delivered via different technologies including webhooks and Azure Event Hubs."
author: "baywet"
localization_priority: Priority
ms.custom: graphiamtop20
---

# Get change notifications delivered different ways (preview)

Change notifications can be delivered different ways to subscribers. If change notifications' main delivery mode is through webhooks, it can be challenging to leverage webhooks for high throughput scenarios or when the receiver cannot expose a publicly available notificiation URL.  

Good examples of high throughput scenarios include applications subscribing to a large set of resources, applications subscribing to resources that change with a high frequency and multi-tenant applications that subscribe to resources accross a large set of organizations.

## Using Azure Event Hubs to receive change notifications

[Azure Event Hubs](https://azure.microsoft.com/services/event-hubs) is a popular real-time events ingestion and distribution service built for scale. You can leverage Azure Events Hubs to receive change notifications instead of traditional webhooks. This feature is currently in preview.  
Using Azure Event Hubs to receive change notifications differs in a few ways including:

- You don't rely on publicly exposed notification URLs : the Event Hubs SDK will relay the notifications to your application
- You don't need to implement the [notification URL validation](webhooks.md#notification-endpoint-validation)
- You'll need to provision an Azure Event Hub
- You'll need to provision an Azure Key Vault

### Setup the Azure KeyVault and Azure Event Hubs

This section will walk you through the setup of required Azure Services.

#### Option 1: Using the Azure CLI

The [Azure CLI](/cli/azure/what-is-azure-cli?view=azure-cli-latest) allows you to script and automate adminstrative tasks in Azure. The CLI can be [installed on your local machine](/cli/azure/install-azure-cli?view=azure-cli-latest) or run directly from the [Azure Cloud Shell](/azure/cloud-shell/quickstart).

```shell
# --------------
# TODO: update the following values
#sets the name of the resource group
resourcegroup=rg-graphevents-dev
#sets the location of the resources
location='uk south'
#sets the name of the Azure Event Hubs namespace
evhamespacename=evh-graphevents-dev
#sets the name of the hub under the namespace
evhhubname=graphevents
#sets the name of the access policy to the hub
evhpolicyname=grapheventspolicy
#sets the name of the Azure KeyVault
keyvaultname=kv-graphevents
#sets the name of the secret in Azure KeyVault that will contain the connection string to the hub
keyvaultsecretname=grapheventsconnectionstring
# --------------
az group create --location $location --name $resourcegroup
az eventhubs namespace create --name $evhamespacename --resource-group $resourcegroup --sku Basic --location $location
az eventhubs eventhub create --name $evhhubname --namespace-name $evhamespacename --resource-group $resourcegroup --partition-count 2 --message-retention 1
az eventhubs eventhub authorization-rule create --name $evhpolicyname --eventhub-name $evhhubname --namespace-name $evhamespacename --resource-group $resourcegroup --rights Send
evhprimaryconnectionstring=`az eventhubs eventhub authorization-rule keys list --name $evhpolicyname --eventhub-name $evhhubname --namespace-name $evhamespacename --resource-group $resourcegroup --query "primaryConnectionString" --output tsv`
az keyvault create --name $keyvaultname --resource-group $resourcegroup --location $location --enable-soft-delete true --sku standard --retention-days 90
az keyvault secret set --name $keyvaultsecretname --value $evhprimaryconnectionstring --vault-name $keyvaultname --output none
graphspn=`az ad sp list --display-name 'Microsoft Graph Change Tracking' --query "[].appId" --output tsv`
az keyvault set-policy --name $keyvaultname --resource-group $resourcegroup --secret-permissions get --spn $graphspn --output none
keyvaulturi=`az keyvault show --name $keyvaultname --resource-group $resourcegroup --query "properties.vaultUri" --output tsv`
domainname=`az ad signed-in-user show --query 'userPrincipalName' | cut -d '@' -f 2 | sed 's/\"//'`
notificationUrl="EventHub:${keyvaulturi}secrets/${keyvaultsecretname}?tenantId=${domainname}"
echo "Notification Url:\n${notificationUrl}"
```

> **Note:** the script provided above is compatible with Linux based shells, Windows WSL, and Azure Cloud Shell. It will require some updates to run in Windows shells.

#### Option 2: Using the Azure Portal

##### Configuring the Azure Event Hub

In this section you will:

- Create an Azure Event Hub namespace
- Add a hub to that namespace that will relay and deliver notifications
- Add a shared access policy that will allow you to get a connection string to the newly created hub.

Steps:

1. Open a browser to the [Azure Portal](https://portal.azure.com).
1. Select "Create a resource".
1. Type "Event Hubs" in the search bar.
1. Select the "Event Hubs" suggestion. The Event Hubs creation page will load.  
    ![event hubs creation page](images/change-notifications/eventhubs.png)
1. On the Event Hubs creation page click "Create".
1. Fill-in the Event Hubs namespace creation details and click "Create".  
    ![event hubs creation](images/change-notifications/eventhubscreation.png)
1. Once the Event Hub namespace is provisioned, navigate to it's page.  
    ![event hubs created](images/change-notifications/eventhubscreated.png)
1. Click on "Event Hubs" and "+ Event Hub".  
    ![event hubs addition](images/change-notifications/eventhubsaddition.png)
1. Give a name to the new Event Hub and click "Create".  
    ![event hub addition panel](images/change-notifications/eventhubadditionpanel.png)
1. Once the Event Hub has been created click on it's name, on "Shared access policies" and "+ Add" to add a new policy.  
    ![policy panel](images/change-notifications/policypanel.png)
1. Give a name to the policy, check "Send" and click "Create".  
    ![policy creation](images/change-notifications/policyaddition.png)
1. Once the policy has been created, click on it's name to open the details panel, then copy the "Connection string-primary key" value. Write it down, you'll need it as the next step.  
    ![policy string](images/change-notifications/policycs.png)

##### Configuring the Azure Key Vault

In order to access the Event Hub securely and to allow for key rotations, the Microsoft Graph gets the connection string to the Event Hub through Azure Key Vault.  
In this section you will:

- Create an Azure Key Vault to store secret.
- Add the connection string to the Event Hub as a secret.
- Add an access policy for the Microsoft Graph to access the secret.

Steps:

1. Open a browser to the [Azure Portal](https://portal.azure.com).
1. Select "Create a resource".
1. Type "Key Vault" in the search bar.
1. Select the "Key Vault" suggestion. The Key Vault creation page will load.
1. On the Key Vault creation page, click "Create".  
    ![key vault creation page](images/change-notifications/keyvault.png)
1. Fill-in the Key Vault creation details, click "Review + Create" and "Create".  
    ![key vault creation](images/change-notifications/keyvaultcreation.png)
1. Navigate to the newly crated key vault using the "Go to resource" from the notification.  
    ![key vault created](images/change-notifications/keyvaultcreated.png)
1. Copy the "DNS name", you will need it for the next step.  
    ![key vault dns name](images/change-notifications/dnsname.png)
1. Navigate to "Secrets" and click on "+ Generate/Import".  
    ![secrets page](images/change-notifications/secretslistpage.png)
1. Give a name to the secret, and keep the name for later, you will need it for the next step. For the value, paste in the connection string you generated at the Event Hubs step. Click "Create".  
    ![secrets creation](images/change-notifications/secretscreation.png)
1. Click on "Access Policies" and "+ Add Access Policy".  
    ![access policies list](images/change-notifications/accesspolicieslist.png)
1. For "Secret permissions" select "Get" and for "Select Principal" select "Microsoft Graph Change Tracking". Click "Add".  
    ![add policy](images/change-notifications/accesspolicyadd.png)

### Creating the subscription and receiving notifications

Once you have created the required Azure KeyVault and Azure Event Hubs services, you will be able to create your subscription and start receiving change notifications via Azure Event Hubs.

#### Creating the Subcription

Subscriptions to change notifications with Event Hubs are almost identical to change notifications with webhooks, the key difference being they rely on Event Hubs to deliver notifications. All other operations are similar, including [subscription creation](/graph/api/subscription-post-subscriptions?view=graph-rest-beta).  

The main difference during subscription creation will be the **notificationUrl**, you must set it to `EventHub:https://<azurekeyvaultname>.vault.azure.net/secrets/<secretname>?tenantId=<domainname>`:

- azure key vault name: the name you gave to the key vault when you created it. Can be found in the DNS name.
- secret name: the name you gave to the secret when you created it. Can be found on the Azure Key Vault "Secrets" page.
- domain name: name of your tenant e.g. consto.onmicrosoft.com or contoso.com. This domain will be used to access the Azure Key Vault so it is important that it matches the domain used by the Azure subscription which holds the Azure Key Vault. To get this information, you can navigate to the overview page of the Azure Key Vault you created, click on the subscription and the domain name will be displayed under the **Directory** field.

#### Receiving notifications

Events will be now delivered to your application by Event Hubs, please refer to [receiving events](https://docs.microsoft.com/azure/event-hubs/get-started-dotnet-standard-send-v2#receive-events) from the Event Hubs documentation.

### Frequent questions

#### Microsoft Graph Change Tracking application missing

It is possible the **Microsoft Graph Change Tracking** service principal is missing from your tenant depending on when the tenant was created and administrative operations. To resolve this issue, run [the following query](https://developer.microsoft.com/en-us/graph/graph-explorer?request=servicePrincipals&method=POST&version=v1.0&GraphUrl=https://graph.microsoft.com&requestBody=eyJhcHBJZCI6IjBiZjMwZjNiLTRhNTItNDhkZi05YTgyLTIzNDkxMGM0YTA4NiJ9) in [Microsoft Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer).

Query details:

```http
POST https://graph.microsoft.com/v1.0/servicePrincipals
{
    "appId": "0bf30f3b-4a52-48df-9a82-234910c4a086"
}
```

> **Note:**  it is possible you get an access denied running this query. In this case select **Modify Permissions**, then select **Consent** for the **Application.ReadWrite.All** row. After consenting to this new permission, run the request again.