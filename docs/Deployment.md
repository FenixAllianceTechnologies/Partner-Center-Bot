# Deployment 
Deploying and configure the Partner Center Bot requires numerous configurations. This document will guide you through each of the
configurations. If you need help please log an issue using the [issue tracker](https://github.com/Microsoft/Partner-Center-Bot/issues).

## Prerequisites 
The following are _required_ prerequisites in order to perform the steps outlined in this guide. 

| Prerequisite                            | Purpose                                                                                 |
|-----------------------------------------|-----------------------------------------------------------------------------------------|
|  Azure AD global admin privileges       | Required to create the required Azure AD application utilized to obtain access tokens.  |
|  Partner Center admin agent privileges  | Required to perform various Partner Center operations through the Partner Center API.   |

If you do not have the privileges then you will not able to perform the remaining operations in this guide.

## Azure Key Vault 
Azure Key Vault is used to protect application secrets and connection strings. An instance of Key Vault will be deployed, 
the following sections will walk you through the configurations that are not automated yet.  

### Create Certificate
A certificate is utilized to obtain the required access token in order to interact with the vault. Perform 
the following to create the certificate

Modify the following PowerShell and then invoke it 

```powershell
$cert = New-SelfSignedCertificate -Subject "PartnerCenterBot" -CertStoreLocation Cert:CurrentUser -KeyExportPolicy Exportable -Type DocumentEncryptionCert -KeyUsage KeyEncipherment -KeySpec KeyExchange -KeyLength 2048 

Export-Certificate -Cert $cert -FilePath C:\cert\bot.cer
```

### Create Azure AD Application 
An Azure AD application is utilized to access the instance of Key Vault. This application is confiugred to 
utilize the certificate that was created in the above step. Perform the following to create the and configure 
the application

1. Open PowerShell and install the [Azure PowerShell cmdlets](https://docs.microsoft.com/en-us/powershell/azureps-cmdlets-docs/)
if you necessary
2. Update the following cmdlets and then invoke them

    ```powershell
    Login-AzureRmAccount

    ## Update these variable before invoking the rest of the cmdlets
    $certFile = "C:\cert\bot.cer"
    $identifierUri = "https://{0}/{1}" -f "tenant.onmicrosoft.com", [System.Guid]::NewGuid()
    $resourceGroupName = "ResourceGroupName"

    $cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2
    $cert.Import($certFile)
    $value = [System.Convert]::ToBase64String($cert.GetRawCertData())

    $app = New-AzureRmADApplication -DisplayName "Partern Center Bot Vault App" -HomePage "https://localhost" -IdentifierUris $identifierUri -CertValue $value -EndDate $cert.NotAfter -StartDate $cert.NotBefore
    $spn = New-AzureRmADServicePrincipal -ApplicationId $app.ApplicationId

    # Get the certificate thumbprint value for the VaultApplicationCertThumbprint parameter
    $cert.Thumbprint

    # Get the application identifier that will be used to access the instance of Key Vault
    $app.ApplicationId
    ```

## Partner Center Azure AD Application
The Partner Center API is utilized to verify the authenticated user belongs to a customer that 
has a relationship with the configured partner. Perform the following to create the application 

1. Login into the [Partner Center](https://partnercenter.microsoft.com) portal using credentials that have _AdminAgents_ and _Global Admin_ privileges
2. Click _Dashboard_ -> _Account Settings_ -> _App Management_ 
3. Click on _Register existing_ app if you want to use an existing Azure AD application, or click _Add new web app_ to create a new one

    ![Partner Center App](Images/appmgmt01.png)

4. Document the _App ID_ and _Account ID_ values. Also, if necessary create a key and document that value. 

    ![Partner Center App](Images/appmgmt02.png)

## Creating the Bot Azure AD Application
The bot requires an Azure AD application that grants privileges to Azure AD and the Microsoft Graph. Perform the following tasks to create and configure the application 

1. Login into the [Azure Management portal](https://portal.azure.com) using credentials that have _Global Admin_ privileges
2. Open the _Azure Active Directory_ user experince and then click _App registration_

	![Azure AD application creation](Images/aad01.png)

3. Click _+ Add_ to start the new application wizard
4. Specify an appropriate name for the bot, select _Web app / API_ for the application, an appropriate value for the sign-on URL, and then click _Create_
5. Click _Required permissions_ found on the settings blade for the the application and then click _+ Add_ 
6. Add the _Microsoft Graph_ API and grant it the _Read directory data_ application permission
7. Add the _Partner Center API_  and grant it the _Access Partner Center PPE_ delegated permission

	![Azure AD application permissions](Images/aad02.png)

8. Click _Grant Permissions_, found on the _Required Permissions_ blade, to consent to the application for the reseller tenant 

    ![Azure AD application consent](Images/aad03.png)

9. Enable pre-consent for this application by completing the steps outlined in the [Pre-consent](Preconsent.md) documentation.

## Create a New LUIS Application
Perform the following to create and configure the Language Understanding Intelligent Service (LUIS) application

1. Browse to https://www.luis.ai/ and login using an appropriate account 
2. Click the *Import App* button to import the [Partner-Center-Bot.json](../Partner-Center-Bot.json) file
3. Train and publish the application. Be sure to document the application identifier and subscription key

Please checkout [Publish your app](https://docs.microsoft.com/en-us/azure/cognitive-services/LUIS/publishapp) for more information.

## Create a New Instance of the QnA Service 
Perform the following in order to create a new instnace of the QnA service

1. Browse to https://qnamaker.ai/ and login using an appropriate account
2. Click the _Create new service_ link found at the top of the page
3. Complete the form as necessary

    ![New QnA Service](Images/qnaservice01.png)

4. Click the _Create_ button at the bottom of the page to create the service
5. Make any necessary modifications to the knowledge question and answer pairs
6. Click the _Save and retrain_ button to apply any changes you made and then click the _Publish_ button
7. Click the _Publish_ button and then document the knowledge base identifier and subscription key

    ![New QnA Service](Images/qnaservice02.png)

## Register With the Bot Framework
Registering the bot with the framework is how the connector service knows how to interact with the bot's web service. Perform the 
following to register the bot and update the required configurations

1. Go to the Microsoft Bot Framework portal at https://dev.botframework.com and sign in.
2. Click the "Register a Bot" button and fill out the form. Be sure to document the application identifier and password that you generate as part of this registration. 

## Deploy the ARM Template 
An Azure Resource Manager (ARM) template has been developed in order to simplify the deployment of the solution. When you click the 
*Deploy to Azure* below it will take you to a website where you can populate the parameters for the template. 

[![Deploy to Azure](http://azuredeploy.net/deploybutton.png)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FMicrosoft%2FPartner-Center-Bot%2Fmaster%2Fazuredeploy.json)
[![Visualize](http://armviz.io/visualizebutton.png)](http://armviz.io/#/?load=https%3A%2F%2Fraw.githubusercontent.com%2FMicrosoft%2FPartner-Center-Bot%2Fmaster%2Fazuredeploy.json)

The following table provides details for the appropriate value for each of the parameters.

| Parameter                             | Value                                                                                                    |
|---------------------------------------|----------------------------------------------------------------------------------------------------------|
| Application Id                        | Identifier for the application created in the Azure AD Application section                               |
| Application Secret                    | Secret key create in the step 9 of the Azure AD application section                                      |
| Application Tenant Id                 | Identifier for the tenant where the application from the Azure AD Application was created                |
| Key Vault App Cert Thumbprint         | Thumbprint of the certificated created in the Azure Key Vault section                                    |
| Key Vault App Id                      | Identifier for the Azure AD application created in the Azure Key Vault section                           |
| Key Vault Name                        | Name for the Key Vault. This name should not contain any special characters or spaces                    |
| Key Vault Tenant Id                   | Identifier for the tenant where the instance of Azure Key Vault is being created                         |
| LUIS App Id                           | Identifier for the LUIS application created in the *Create a New LUIS Application* section               |
| LUIS App Key                          | Key for the LUIS application created in the *Create a New LUIS Application* section                      |
| Microsoft App Id                      | Identifier of the application created in the *Register With the Bot Framework* section                   |
| Micorosoft App Password               | Password of the application created in the *Register With the Bot Framework* section                     |
| Partner Center Application Id         | App ID value obtained from the Partner Center Azure AD Application section                               |
| Partner Center Application Secret     | Key created in the Partner Center Azure AD Application section                                           | 
| Partner Center Application Tenat Id   | Account ID value obtained from the Partner Center Azure AD Application section                           |
| QnA Knowledgebase Id                  | Identifier for the knowledgebase created in the *Create a New Instance of the QnA Service* section       |
| QnA Subscription Key                  | Subscription key for the knowledgebase created in the *Create a New Instance of the QnA Service* section |

## Configure Azure Key Vault Access Policy
Now that the instance of Azure Key Vault has been provisioned you can add the access policy that will 
enable the Azure AD application the ability to create, delete, and read secrets. Perform the following 
to create the access policy

1. Open PowerShell and install the [Azure AD PowerShell cmdlets](https://docs.microsoft.com/en-us/powershell/azure/install-adv2?view=azureadps-2.0)
if you necessary
2. Update the following cmdlets and then invoke them

    ```powershell
    Connect-AzureAD
    Login-AzureRmAccount

    ## Update these variable before invoking the rest of the cmdlets

    # The value for the AppId should be the application identifier for the Azure AD application created for Key Vault
    $spn = Get-AzureADServicePrincipal | ? {$_.AppId -eq 'b6b84568-6c01-4981-a80f-09da9a20bbed'}
    $resourceGroupName = "ResourceGroupName"
    $vaultName = "VaultName"

    Set-AzureRmKeyVaultAccessPolicy -VaultName $vaultName -ObjectId $spn.Id -PermissionsToSecrets delete,get,set -ResourceGroupName $resourceGroupName
    ```