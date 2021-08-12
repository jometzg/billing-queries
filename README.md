# How to query billing/cost management
This demonstrates how to use a logic app to query billing in both the current subscription and in another subscription in another Azure AD tenant. Much of this centres around how to do authentication of the REST requests to the Azure billing APIs.

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
In order to get an access token, there must be an Azure AD app registration in the Azure AD tenant of the target subscription. This app registration must also have the role of "Billing Reader".

![alt text](app-registration-billing-reader.png "Billing reader role for AD app registration")

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
