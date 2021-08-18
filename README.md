# How to query billing/cost management
This demonstrates how to use a logic app to query billing in both the current subscription and in different subscription hosted in different Azure AD tenant. Much of this centres around how to do authentication of the REST requests to the Azure billing APIs.

## Microsoft documentation
These query examples centre around the cost management part of the APIs, though the process is similar for the other billing-related APIs.

[Cost Management Query Usage](https://docs.microsoft.com/en-us/rest/api/cost-management/query/usage "Azure Cost management")

## REST query
A query is an HTTP REST API call:

```
POST  https://management.azure.com/subscriptions/{{targetSubscriptionId}}/providers/Microsoft.CostManagement/query?api-version=2019-11-01
Content-Type: application/json
Authorization: Bearer {{accessToken}}

{
  "dataset": {
    "aggregation": {
      "totalCost": {
        "function": "Sum",
        "name": "PreTaxCost"
      }
    },
    "granularity": "None",
    "grouping": [
      {
        "name": "ResourceGroup",
        "type": "Dimension"
      }
    ]
  },
  "timeframe": "TheLastMonth",
  "type": "Usage"
}
```
In the code above, a POST is made to cost management API to get a resource group by resource group summary of costs (this is defined in the body of the POST request). The target subscription ID forms part of the URL path. But in order to do this we need an access token.

## Getting the access token
In order to get an access token, there must be an Azure AD app registration in the Azure AD tenant of the target subscription. This app registration must also have the role of "Billing Reader". If you want to query multiple subscriptions under the same Azure AD tenant, then you need to make sure that the billing reader role is set for each of these subscriptions for that app registration.

![alt text](app-registration-billing-reader.png "Billing reader role for AD app registration")

In the above diagram, the subscription has been selected in the portal and, via the access control (IAM) menu, the role for the AD application is added to that subscription.

This access token, then will have a clientId and secret and these must be used in the request to get an access token. So the 4 parameters needed for this call are:
1. ClientId
2. Secret
3. The tenantId of the Azure AD tenant
4. The audience - in this case, this is *https://managment.azure.com* - which is the domain of the cost management request.

```
POST https://login.microsoftonline.com/{{tentantid}}/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id={{clientid}}&resource={{audience}}&client_secret={{secret}}

```

## Trying out the REST requests
The pair of REST queries may be executed in a REST client. Firstly get the access token, and then to the cost management API - which uses the access token on its *Authorization* header.

Visual Studio Code has a third-party extension called *REST Client* https://github.com/Huachao/vscode-restclient which is invaluable in debugging REST requests to services. It's really simple to use, but also powerful. You can paramaterise requests and then use the response from one request as part of the following one. In this case the call to get the access token provides this value for the second call to cost managment API.

[REST requests for cost management](billing-blank.http "Azure Cost management")


## Logic Apps
It can often be useful to automate the process of doing cost management or billing queries and logic apps are one such vehicle. Logic apps have an HTTP action that can easily make HTTP requests.

![alt text](http-action.png "Logic app HTTP Action")

As can be seen from the above screenshot, this is exactly the same type of billing query from the REST requests, above.

The next most important thing is how to get the access token. You could, of course, have another HTTP request to get the access token, but the HTTP Action provides some more automated approaches to getting the access token. 

![alt text](authentication-methods.png "HTTP Action authentication methods")

Two of the more interesting of these are:
1. Managed identity
2. Active Directory OAuth

The former is most useful and easiest if the HTTP needs to authenticate against the current AD tenant. For this to work, you need to:
1. Enable managed identity for the logic app
2. Find the app registration that is created as part of step one and add the "Billing Reader" role to it

For *Active Directory OAuth*, there needs to be more configuration - but exactly the same set of values that the REST requests need.

![alt text](active-directory-oauth.png "HTTP Action OAuth authentication")

In the above, I have prefilled in the authority and audience with values that are correct for cost management and billing requests. All that is needed is the AD tenant ID, the clientId and secret.

Here is an example logic app that peforms both a managed identity call and an OAuth one to the cost management API.
![alt text](logic-app-overview.png "Logic app example")

The code for this logic app is [here](logic-app-redacted.json "logic App code"). 

You will need to update:
1. tenantId
2. ClientId
3. Secret
4. SubscriptionId (in the REST request path marked *subscriptionid*)

## Securing secrets in logic apps
Logic apps can make use of key vault for keys and secrets. Storing these in key vault will mean that the logic app underlying code will not contain the Azure AD app registration secret - which is good practice.

To do this, you first need to create a key vault. This can be any name, but it should be in the subscription and region of the logic app itself.
![alt text](logic-app-resources.png "Resource group with logic app and key vault")

In this key vault, you then need to create a secret for the app registration secret. In the example, I have chosen to also create the clientId as a secret. This is not necessary, but may also help keep the logic app code clean. Other values could also be added to the key vault, such as the subscription and tenent IDs.
![alt text](key-vault-secrets.png "key vault secrets created").

The names of the key vault secrets should be chosen to make their later identification easier. 

Next, the key vault reference needs to be added to the logic app itself. This also creates a logic app "API Connection" - which is used to manage the authentication of the logic app to the key vault instance created for this purpose.

![alt text](logic-app-kv-action.png "Add key vault to logic app").

The API Connection used:

![alt text](logic-app-with-kv-connections.png "logic app API connections").

Once the logic app is authenticated against the key vault, a step to get a secret can be added. If you have chosen to also store the clientId, then there will need to be on step for the clientId and one for the secret. In order to make this clearer for later, a step can be renamed (by choosing the "..." and "Rename". See below:

![alt text](logic-app-with-kv.png "key vault secrets in logic app").

In order to use these secrets in the HTTP step, these can be chosen from the *Dynamic Content* dialog:

![alt text](logic-app-http-dynamic.png "use a secret in logic app").

Note that the secret value is the property *value* in a section with the name of the step - in the above, the section is "Get secret" - which is what the step was renamed to earlier.

This need to be done for each clientId and secret needed for any steps that need these.

![alt text](logic-app-http-secrets.png "clientId and secret in logic app").

The underlying code for the logic app will then no longer have the clientId and secret in the code view of the logic app:

![alt text](logic-app-with-kv-code.png "logic app code view").

Once complete, retesting should show the logic app works as normal.

# Summary
Cost management and billing queries can easily be done through REST requests. Logic apps using the HTTP action have a more streamlined way to authenticate these requests. You can choose to use either managed identities for queries under the same AD tenant or to use OAuth for remote AD tenants. For both of these an AD app registration is needed and this app registration needs to have the *billing reader* role for all of the subscriptions you want to query.
