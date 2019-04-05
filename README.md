# AKS-ARM
# edit your DNS
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters":            {
    "developer":           { "type": "string", "defaultValue": "", "metadata": {"description": "Indended for testing ARM deployment on a personal account, leave blank in ALL other cases"}},    
    "environment":         { "type": "string", "defaultValue": "test", "allowedValues": [ "dev", "test", "prod" ] },
    "servicePrincipalId": { "type": "string", "defaultValue": "", "metadata": { "description": "Specifies the object ID of a user, service principal or security group in the Azure Active Directory tenant for the vault. The object ID must be unique for the list of access policies. Get it by using Get-AzADUser or Get-AzADServicePrincipal cmdlets." }}
  },
  "variables": {
    "subscriptionId"        : "[subscription().id]",
    "tenantId"              : "[subscription().tenantId]",
    "location"              : "[resourceGroup().location]",
    "environment"           : "[concat(parameters('environment'), parameters('developer'))]",
    "storageName"           : "[concat('formsystorage', variables('environment'))]",
    "keyVaultName"          : "[concat('FormsyKeyVault', '-', variables('environment'))]",
    "workspaceName"         : "[concat('FormsyWorkspace),'-',variables('environment'))]",
    "clusterName"           : "[concat('formsyAKS', '-', variables('environment'))]",
    "workSpaceName"         : "[concat('formsyWorkspace', '-', variables('environment'))]",
    "kubernetesVersion"     : "1.7.7",
    "keyVaultId"            : "[resourceId('Microsoft.KeyVault/vaults', variables('keyVaultName'))]",
    "storageId"             : "[resourceId('Microsoft.Storage/storageAccounts', variables('storageName'))]",
    "searchServiceName"     : "[concat('formsysearch', '-', variables('environment'))]",
    "searchServiceId"       : "[resourceId('Microsoft.Search/searchServices', variables('SearchServiceName'))]",
    "containerRegistryName" : "[concat('formsyContainerService', variables('environment'))]",
    "vNetName"              : "[concat('formsyVNet',variables('environment'))]",
    "omsWorkspaceId"        : "[resourceId('Microsoft.OperationalInsights/workspaces',variables('workspaceName'))]",
    "vNetId"                : "[resourceId('Microsoft.Network/virtualNetworks',variables('vNetName'))]"
  },
  "functions": [],
  "resources": [
      {
        "apiVersion": "2018-03-31",
        "dependsOn": [
            "[concat('Microsoft.Resources/deployments/', 'WorkspaceDeployment-xxx')]",
            "[variables('vNetId')]",
            "[concat('Microsoft.Resources/deployments/', 'ClusterSubnetRoleAssignmentDeployment-xx')]"
        ],
        "type": "Microsoft.ContainerService/managedClusters",
        "location": "[variables('location')]",
        "name": "[variables('clusterName')]",
        "properties": {
            "kubernetesVersion": "[variables('kubernetesVersion')]",
            "enableRBAC": false,
            "dnsPrefix": "formsy-[variables('envrionment')]",
            "agentPoolProfiles": [
                {
                    "name": "agentpool",
                    "osDiskSizeGB": 1024,
                    "count": 2,
                    "vmSize": "Standard_D2_v2",
                    "osType": "linux",
                    "storageProfile": "ManagedDisks",
                    "vnetSubnetID": "[variables('vNetId')]"
                }
            ],
            "servicePrincipalProfile": {
                "ClientId": {
                  "reference":{
                      "keyVault":{"id":"[variables('keyVaultId')]"},
                      "secretName":"containerAdminUsername"
                  }
                },
                "Secret": {
                  "reference":{
                      "keyVault":{"id":"[variables('keyVaultId')]"},
                      "secretName":"containerAdminPassword"
                  }
                }
            },
            "networkProfile": {
                "networkPlugin": "azure",
                "serviceCidr": "10.0.0.0/16",
                "dnsServiceIP": "x.x.x.x",
                "dockerBridgeCidr": "172.17.0.1/16"
            },
            "addonProfiles": {
                "httpApplicationRouting": {
                    "enabled": true
                },
                "omsagent": {
                    "enabled": true,
                    "config": {
                        "logAnalyticsWorkspaceResourceID": "[variables('omsWorkspaceId')]"
                    }
                }
            }
        },
        "tags": {}
    },
    {
        "type": "Microsoft.Resources/deployments",
        "name": "SolutionDeployment-20190315130545",
        "apiVersion": "2017-05-10",
        "properties": {
            "mode": "Incremental",
            "template": {
                "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
                "contentVersion": "1.0.0.0",
                "parameters": {},
                "variables": {},
                "resources": [
                    {
                        "apiVersion": "2015-11-01-preview",
                        "type": "Microsoft.OperationsManagement/solutions",
                        "location": "[variables('location')]",
                        "name": "[concat('ContainerInsights', '(', split(variables('omsWorkspaceId'),'/')[8], ')')]",
                        "properties": {
                            "workspaceResourceId": "[variables('omsWorkspaceId')]"
                        },
                        "plan": {
                            "name": "[concat('ContainerInsights', '(', split(variables('omsWorkspaceId'),'/')[8], ')')]",
                            "product": "[concat('OMSGallery/', 'ContainerInsights')]",
                            "promotionCode": "",
                            "publisher": "Microsoft"
                        }
                    }
                ]
            }
        },
        "dependsOn": [
            "[concat('Microsoft.Resources/deployments/', 'WorkspaceDeployment-20190315130545')]"
        ]
    },
    {
        "type": "Microsoft.Resources/deployments",
        "name": "WorkspaceDeployment-20190315130545",
        "apiVersion": "2017-05-10",
        "properties": {
            "mode": "Incremental",
            "template": {
                "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
                "contentVersion": "1.0.0.0",
                "parameters": {},
                "variables": {},
                "resources": [
                    {
                        "apiVersion": "2015-11-01-preview",
                        "type": "Microsoft.OperationalInsights/workspaces",
                        "location": "[variables('location')]",
                        "name": "[variables('workSpaceName')]",
                        "properties": {
                            "sku": {
                                "name": "Standalone"
                            }
                        }
                    }
                ]
            }
        }
    },
      {
        "apiVersion": "2018-08-01",
        "name": "[variables('vNetName')]",
        "type": "Microsoft.Network/virtualNetworks",
        "location": "[variables('location')]",
        "properties": {
            "subnets": [
                {
                    "name": "default",
                    "id": "[variables('vNetId')]",
                    "properties": {
                        "addressPrefix": "10.240.0.0/16"
                    }
                }
            ],
            "addressSpace": {
                "addressPrefixes": [
                    "10.0.0.0/8"
                ]
            }
        },
        "tags": {}
    },
    {
        "type": "Microsoft.Resources/deployments",
        "name": "ClusterSubnetRoleAssignmentDeployment-xxxx",
        "apiVersion": "2017-05-10",
        "properties": {
            "mode": "Incremental",
            "template": {
                "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
                "contentVersion": "1.0.0.0",
                "parameters": {},
                "variables": {},
                "resources": [
                    {
                        "type": "Microsoft.Network/virtualNetworks/subnets/providers/roleAssignments",
                        "apiVersion": "2017-05-01",
                        "name": "[concat(variables('vNetName'),'/default/Microsoft.Authorization/xxxxx')]",
                        "properties": {
                            "roleDefinitionId": "[concat('/subscriptions/', subscription().subscriptionId, '/providers/Microsoft.Authorization/roleDefinitions/', 'xxxxx)]",
                            "principalId": "[parameters('securityPrincipalId')]",
                            "scope": "[concat('/subscriptions/',subscription().subscriptionId,'/resourceGroups/',resourceGroup().name,'/providers/Microsoft.Network/virtualNetworks/',variables('vNetName'),'/subnets/default')]"
                        }
                    }
                ]
            }
        },
        "dependsOn": [
            "[variables('vNetId')]"
        ]
    },
    {
      "kind": "StorageV2",
      "apiVersion": "2018-07-01",
      "name": "[variables('storageName')]",
      "type": "Microsoft.Storage/storageAccounts",
      "location": "[variables('location')]",
      "sku": { "name": "Standard_LRS", "tier" : "Standard" },
      "properties" : { "accessTier" : "Hot"},
      "resources" : [
        {
          "name": "default/inbox",
          "type": "blobServices/containers",
          "apiVersion": "2018-07-01",
          "dependsOn" : ["[variables('storageName')]"],
          "properties": {
            "publicAccess": "None",
            "metadata": {"description" : "This is the upload folder for the blob storage"}
          }
        },
        {
          "name": "default/completed",
          "type": "blobServices/containers",
          "apiVersion": "2018-07-01",
          "dependsOn" : ["[variables('storageName')]"],
          "properties": {
            "publicAccess": "None",
            "metadata": {"description" : "This is the upload folder for the blob storage"}
          }
        }
      ]
    },
    {
      "type": "Microsoft.Search/searchServices",
      "apiVersion": "2015-08-19",
      "name": "[variables('searchServiceName')]",
      "location": "[variables('location')]",
      "sku": {
        "name": "free"
      },
      "properties": {
        "replicaCount": 1,
        "partitionCount": 1,
        "hostingMode": "default"
      }
    },
    {
      "apiVersion": "2016-10-01",
      "name": "[variables('keyVaultName')]",
      "location": "[variables('location')]",
      "type": "Microsoft.KeyVault/vaults",
      "properties": {
        "tenantId" : "[variables('tenantId')]",
        "sku": { "name": "standard", "family": "A" },
        "enabledForDeployment": false,
        "enabledForTemplateDeployment": false,
        "enabledForDiskEncryption": false,
        "accessPolicies": [
          {
          "tenantId" : "[variables('tenantId')]",
          "objectId" : "[parameters('servicePrincipalId')]",
          "permissions" : {
            "secrets": [ "get", "list" ],
            "keys" : ["get", "list"],
            "certificates" : []
          }
        }]        
      }
    },
    {
      "type": "Microsoft.ContainerRegistry/registries",
      "apiVersion": "2017-10-01",
      "name": "[variables('containerRegistryName')]",
      "location": "[variables('location')]",
      "sku": {
        "name": "Standard"
      },
      "properties": {
        "adminUserEnabled": true
      }
    },
    {
      "type": "Microsoft.KeyVault/vaults/secrets",
      "apiVersion": "2016-10-01",
      "name": "[concat(variables('keyVaultName'), '/', 'storageConnectionString')]",
      "properties": {
        "contentType": "text/plain",
        "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageName'), 'AccountKey=', listKeys(variables('storageId'),'2018-07-01').keys[0].value)]"
      },
      "dependsOn": [
        "[resourceId('Microsoft.KeyVault/vaults', variables('keyVaultName'))]",
        "[variables('storageId')]"
      ]
    },
    {
      "type": "Microsoft.KeyVault/vaults/secrets",
      "apiVersion": "2016-10-01",
      "name": "[concat(variables('keyVaultName'), '/', 'formsySearchKey')]",
      "properties": {
        "contentType": "string",
        "value": "[listAdminKeys(variables('searchServiceId'), '2015-08-19').PrimaryKey]"
      },
      "dependsOn": [
        "[resourceId('Microsoft.KeyVault/vaults', variables('keyVaultName'))]",
        "[variables('searchServiceName')]"
      ]
    },
    {
      "type": "Microsoft.KeyVault/vaults/secrets",
      "apiVersion": "2016-10-01",
      "name": "[concat(variables('keyVaultName'), '/', 'containerAdminUsername')]",
      "properties": {
        "contentType": "string",
        "value": "[variables('containerRegistryName')]"
      },
      "dependsOn": [
        "[resourceId('Microsoft.KeyVault/vaults', variables('keyVaultName'))]",
        "[variables('containerRegistryName')]"
      ]
    },
    {
      "type": "Microsoft.KeyVault/vaults/secrets",
      "apiVersion": "2016-10-01",
      "name": "[concat(variables('keyVaultName'), '/','containerAdminPassword')]",
      "properties": {
        "contentType": "string",
        "value": "[listCredentials(resourceId('Microsoft.ContainerRegistry/registries', variables('containerRegistryName')), '2017-10-01').passwords[0].value]"
      },
      "dependsOn": [
        "[resourceId('Microsoft.KeyVault/vaults', variables('keyVaultName'))]",
        "[variables('containerRegistryName')]"
      ]
    }
  ],
  "outputs": {
    "controlPlaneFQDN": {
        "type": "string",
        "value": "[reference(concat('Microsoft.ContainerService/managedClusters/', variables('clusterName'))).fqdn]"
    }
}
}
