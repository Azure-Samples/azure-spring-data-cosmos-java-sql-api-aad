# Azure Cosmos DB Spring Data RBAC with AAD Sample

Azure Cosmos DB Spring Data RBAC with AAD Sample code

## Features

This repo provides basic Spring Data sample code for Java SQL API to connect to Cosmos DB using built-in role-based access control (RBAC), and authenticating using Azure Active Directory (AAD).

## Getting Started

### Prerequisites

- `Java Development Kit 8` or `JDK 11` if you run the `azure-spring-data-cosmos-java-11-getting-started`. 
- An active Azure account. If you don't have one, you can sign up for a [free account](https://azure.microsoft.com/free/). Alternatively, you can use the [Azure Cosmos DB Emulator](https://docs.microsoft.com/en-us/azure/cosmos-db/local-emulator) for development and testing. As emulator https certificate is self signed, you need to import its certificate to java trusted cert store, [explained here](https://docs.microsoft.com/en-us/azure/cosmos-db/local-emulator-export-ssl-certificates)
- (Optional) SLF4J is a logging facade.
- (Optional) [SLF4J binding](http://www.slf4j.org/manual.html) is used to associate a specific logging framework with SLF4J.
- (Optional) Maven.

SLF4J is only needed if you plan to use logging, please also download an SLF4J binding which will link the SLF4J API with the logging implementation of your choice. See the [SLF4J user manual](http://www.slf4j.org/manual.html) for more information.

### Clone this repo

1. git clone https://github.com/Azure-Samples/azure-spring-data-cosmos-java-sql-api-aad.git
2. cd azure-spring-data-cosmos-java-sql-api-aad

### Create an Azure AD application and service principal

1. Following the instructions [here](https://learn.microsoft.com/azure/active-directory/develop/howto-create-service-principal-portal) for creating an Azure AD application and service principal.

1. In the [authentication](https://learn.microsoft.com/azure/active-directory/develop/howto-create-service-principal-portal#authentication-two-options) section, be sure to select **option 2** to create a new application secret, and make sure you store the secret value somewhere in a text editor. 

1. Search for your app in Azure Portal --> Azure Active Directory --> App Registrations. You should see information like the below
    ![app](/media/aad-app.png?raw=true "aad app")

1. Review `resources/application.properties` in the repo you have cloned. 
    - Replace `<Cosmos URI>` with the URI of your Cosmos DB account
    - Replace `<tenantId>` with `Directory (tenant) ID` from the portal. 
    - Replace `<clientId>` with `Application (client) ID` from the portal
    - Replace `<clientSecret>` with the application secret value you created earlier
    - For the value of `cosmos.defaultScope` replace the `<cosmos account>` part with the name of your Cosmos DB account (note this is used to test connection to AAD)
    
### Configure RBAC for your Cosmos DB account

Next we need to create a role that can access your Cosmos DB account appropriately. You should refer to the full instructions [here](https://learn.microsoft.com/azure/cosmos-db/how-to-setup-rbac) for the various options, we'll just keep things simple here by creating a custom role using Azure CLI that has full permissions.

1. First, create a JSON file called `role-definition-rw.json` that contains the following:

    ```json
    {
        "RoleName": "MyReadWriteRole",
        "Type": "CustomRole",
        "AssignableScopes": ["/"],
        "Permissions": [{
            "DataActions": [
                "Microsoft.DocumentDB/databaseAccounts/readMetadata",
                "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/*",
                "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/*"
            ]
        }]
    }
    ```
 
1. Upload the JSON file to Azure CLI, then create the role, replacing `<myResourceGroup>` and `<myCosmosAccount>` with the resource group name and Cosmos DB account name respectively:

    ```azurecli-interactive
    resourceGroupName='<myResourceGroup>'
    accountName='<myCosmosAccount>'
    az cosmosdb sql role definition create --account-name $accountName --resource-group $resourceGroupName --body @role-definition-rw.json
    ```
1. Now list the role definition you created to fetch it's ID: 

    ```azurecli-interactive
    az cosmosdb sql role definition list --account-name $accountName --resource-group $resourceGroupName
    ```

1. This should bring back a response like the below:

    ```json
    [
      {
        "assignableScopes": [
          "/subscriptions/<mySubscriptionId>/resourceGroups/<myResourceGroup>/providers/Microsoft.DocumentDB/databaseAccounts/<myCosmosAccount>"
        ],
        "id": "/subscriptions/<mySubscriptionId>/resourceGroups/<myResourceGroup>/providers/Microsoft.DocumentDB/databaseAccounts/<myCosmosAccount>/sqlRoleDefinitions/<roleDefinitionId>",
        "name": "<roleDefinitionId>",
        "permissions": [
          {
            "dataActions": [
              "Microsoft.DocumentDB/databaseAccounts/readMetadata",
              "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/*",
              "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/*"
            ],
            "notDataActions": []
          }
        ],
        "resourceGroup": "<myResourceGroup>",
        "roleName": "MyReadWriteRole",
        "sqlRoleDefinitionGetResultsType": "CustomRole",
        "type": "Microsoft.DocumentDB/databaseAccounts/sqlRoleDefinitions"
      }
    ]
    ``` 

1. Now go to Azure Portal --> Azure Active Directory --> **Enterprise Applications** and search for the application you created earlier. Record the `Object ID` found here.

1. Now create a role assignment. Replace the `<aadPrincipalId>` with `Object ID` you recorded above (note this is **NOT** the same as Object ID visible from the app registrations view you saw earlier). Also replace `<myResourceGroup>` and `<myCosmosAccount>` accordingly in the below. Replace `roleDefinitionId>` with the value fetched from running the above command. Then run in Azure CLI:

    ```azurecli-interactive
    resourceGroupName='<myResourceGroup>'
    accountName='<myCosmosAccount>'
    readOnlyRoleDefinitionId = '<roleDefinitionId>' # as fetched above
    # For Service Principals make sure to use the Object ID as found in the Enterprise applications section of the Azure Active Directory portal blade.
    principalId = '<aadPrincipalId>'
    az cosmosdb sql role assignment create --account-name $accountName --resource-group $resourceGroupName --scope "/" --principal-id $principalId --role-definition-id $readOnlyRoleDefinitionId
    ```


### Run the application

1. Now that you have created an AAD application and service principle, created a custom role, and assigned that role permissions to your Cosmos DB account, you should be able to start your application.
1. run `mvn clean spring-boot:run`
1. Note the following line of code in `SampleAppConfiguration.java`
    ```java
    checkAADSetup(servicePrincipal);
    ```
    This will check access to your Cosmos DB account via AAD. If this check fails, there is an issue with AAD setup and/or connectivity to AAD. If setup is correct and there are no errors, in a production application you can remove this code.

## Resources

Please refer to azure spring data cosmos for sql api [source code](https://github.com/Azure/azure-sdk-for-java/tree/main/sdk/cosmos/azure-spring-data-cosmos) for more information.
