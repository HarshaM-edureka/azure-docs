---
title: Create and publish an Azure service catalog managed application | Microsoft Docs
description: Shows how to create an Azure managed application that is intended for members of your organization.
services: managed-applications
author: tfitzmac
manager: timlt

ms.service: managed-applications
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.date: 11/02/2017
ms.author: tomfitz
---
# Publish a managed application for internal consumption

You can create and publish Azure [managed applications](overview.md) that are intended for members of your organization. For example, an IT department can publish managed applications that ensure compliance with organizational standards. These managed applications are available through the service catalog, not the Azure marketplace.

To publish a managed application for the service catalog, you must:

* Create a template that defines the resources to deploy with the managed application.
* Define the user interface elements for the portal when deploying the managed application.
* Create a .zip package that contains the required template files.
* Decide which user, group, or application needs access to the resource group in the user's subscription.
* Create the managed application definition that points to the .zip package and requests access for the identity.

For this article, your managed application contains only a storage account. It is intended to illustrate the steps of publishing a managed application. For complete examples, see [Sample projects for Azure managed applications](sample-projects.md).

## Create the resource template

Every managed application definition contains a file named **mainTemplate.json**. In it, you define the Azure resources to provision. The template is no different than a regular Resource Manager template.

Create a file named **mainTemplate.json**. The name is case-sensitive.

Add the following JSON to your file. It defines the parameters for creating a storage account, and specifies the properties for the storage account.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "storageAccountNamePrefix": {
            "type": "string"
        },
        "storageAccountType": {
            "type": "string"
        },
        "location": {
            "type": "string",
            "defaultValue": "[resourceGroup().location]"
        }
    },
    "variables": {
        "storageAccountName": "[concat(parameters('storageAccountNamePrefix'), uniqueString('storage'))]"
    },
    "resources": [
        {
            "type": "Microsoft.Storage/storageAccounts",
            "name": "[variables('storageAccountName')]",
            "apiVersion": "2016-01-01",
            "location": "[parameters('location')]",
            "sku": {
                "name": "[parameters('storageAccountType')]"
            },
            "kind": "Storage",
            "properties": {}
        }
    ],
    "outputs": {
        "storageEndpoint": {
            "type": "string",
            "value": "[reference(resourceId('Microsoft.Storage/storageAccounts/', variables('storageAccountName')), '2016-01-01').primaryEndpoints.blob]"
        }
    }
}
```

Save the mainTemplate.json file.

## Create the user interface definition

The Azure portal uses the **createUiDefinition.json** file to generate the user interface for users who create the managed application. You define how users provide input for each parameter. You can use options like a drop-down list, text box, password box, and other input tools. To learn how to create a UI definition file for a managed application, see [Get started with CreateUiDefinition](create-uidefinition-overview.md).

Create a file named **createUiDefinition.json**. The name is case-sensitive.

Add the following JSON to the file.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/0.1.2-preview/CreateUIDefinition.MultiVm.json#",
    "handler": "Microsoft.Compute.MultiVm",
    "version": "0.1.2-preview",
    "parameters": {
        "basics": [
            {}
        ],
        "steps": [
            {
                "name": "storageConfig",
                "label": "Storage settings",
                "subLabel": {
                    "preValidation": "Configure the infrastructure settings",
                    "postValidation": "Done"
                },
                "bladeTitle": "Storage settings",
                "elements": [
                    {
                        "name": "storageAccounts",
                        "type": "Microsoft.Storage.MultiStorageAccountCombo",
                        "label": {
                            "prefix": "Storage account name prefix",
                            "type": "Storage account type"
                        },
                        "defaultValue": {
                            "type": "Standard_LRS"
                        },
                        "constraints": {
                            "allowedTypes": [
                                "Premium_LRS",
                                "Standard_LRS",
                                "Standard_GRS"
                            ]
                        }
                    }
                ]
            }
        ],
        "outputs": {
            "storageAccountNamePrefix": "[steps('storageConfig').storageAccounts.prefix]",
            "storageAccountType": "[steps('storageConfig').storageAccounts.type]",
            "location": "[location()]"
        }
    }
}
```

Save the createUIDefinition.json file.

## Package the files

Add the two files to a .zip file named app.zip. The two files must be at the root level of the .zip file. If you put them in a folder, you receive an error when creating the managed application definition that states the required files are not present. 

Upload the package to an accessible location from where it can be consumed. 

```powershell
New-AzureRmResourceGroup -Name storageGroup -Location eastus
$storageAccount = New-AzureRmStorageAccount -ResourceGroupName storageGroup `
  -Name "mystorageaccount" `
  -Location eastus `
  -SkuName Standard_LRS `
  -Kind Storage `
  -EnableEncryptionService Blob

$ctx = $storageAccount.Context

New-AzureStorageContainer -Name appcontainer -Context $ctx -Permission blob

Set-AzureStorageBlobContent -File "D:\myapplications\app.zip" `
  -Container appcontainer `
  -Blob "app.zip" `
  -Context $ctx 
```

## Create the managed application definition

### Create an Azure Active Directory user group or application

The next step is to select a user group or application for managing the resources on behalf of the customer. This user group or application has permissions on the managed resource group according to the role that is assigned. The role can be any built-in Role-Based Access Control (RBAC) role like Owner or Contributor. You also can give an individual user permission to manage the resources, but typically you assign this permission to a user group. To create a new Active Directory user group, see [Create a group and add members in Azure Active Directory](../active-directory/active-directory-groups-create-azure-portal.md).

You need the object ID of the user group to use for managing the resources. 

![Get group ID](./media/publish-service-catalog-app/get-group-id.png)

### Get the role definition ID

Next, you need the role definition ID of the RBAC built-in role you want to grant access to the user, user group, or application. Typically, you use the Owner or Contributor or Reader role. The following command shows how to get the role definition ID for the Owner role:

```powershell
$ownerID=(Get-AzureRmRoleDefinition -Name Owner).Id
```

### Create the managed application definition

If you do not already have a resource group for storing your managed application definition, create one now:

```powershell
New-AzureRmResourceGroup -Name appDefinitionGroup -Location westcentralus
```

Now, create the managed application definition resource.

```powershell
$blob = Get-AzureStorageBlob -Container appcontainer -Blob app.zip -Context $ctx

New-AzureRmManagedApplicationDefinition `
  -Name "ManagedStorage" `
  -Location "westcentralus" `
  -ResourceGroupName appDefinitionGroup `
  -LockLevel ReadOnly `
  -DisplayName "Managed Storage Account" `
  -Description "Managed Azure Storage Account" `
  -Authorization "<group-id>:$ownerID" `
  -PackageFileUri $blob.ICloudBlob.StorageUri.PrimaryUri.AbsoluteUri
```

## Create the managed application by using the portal

Now, let's use the portal to deploy the managed application. You see the user interface you created in the package.

1. Go to the Azure portal. Select **+ New** and search for **service catalog**.

   ![Search service catalog](./media/publish-service-catalog-app/select-new.png)

1. Select **Service Catalog Managed Application**.

   ![Select service catalog](./media/publish-service-catalog-app/select-service-catalog.png)

1. Select **Create**.

   ![Select create](./media/publish-service-catalog-app/select-create.png)

1. Find the managed application you want to create from the list of available solutions, and select it. Select **Create**.

   ![Find the managed application](./media/publish-service-catalog-app/find-application.png)

1. Provide basic information that is required for the managed application. Specify the subscription and a new resource group to contain the managed application. Select **West Central US** for location. When done, select **OK**.

   ![Provide managed application parameters](./media/publish-service-catalog-app/provide-basics.png)

1. Provide values that are specific to the resources in the managed application. When done, select **OK**.

   ![Provide resource parameters](./media/publish-service-catalog-app/provide-resource-values.png)

1. The template validates the values you provided. If validation succeeds, select **OK** to start the deployment.

   ![Validate managed application](./media/publish-service-catalog-app/validate.png)

After the deployment finishes, the managed application exists in a resource group named applicationGroup. The storage account exists in a resource group named applicationGroup plus a hashed string value.

## Next steps

* For an introduction to managed applications, see [Managed application overview](overview.md).
* For example projects, see [Sample projects for Azure managed applications](sample-projects.md).
* To learn how to create a UI definition file for a managed application, see [Get started with CreateUiDefinition](create-uidefinition-overview.md).
