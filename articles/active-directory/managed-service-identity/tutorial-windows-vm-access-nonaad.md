---
title: Use a Windows VM Managed Service Identity to access Azure Key Vault
description: A tutorial that walks you through the process of using a Windows VM Managed Service Identity to access Azure Key Vault. 
services: active-directory
documentationcenter: ''
author: daveba
manager: mtillman
editor: daveba

ms.service: active-directory
ms.component: msi
ms.devlang: na
ms.topic: tutorial
ms.tgt_pltfrm: na
ms.workload: identity
ms.date: 11/20/2017
ms.author: daveba
---

# Tutorial: Use a Windows VM Managed Service Identity to access Azure Key Vault 

[!INCLUDE[preview-notice](../../../includes/active-directory-msi-preview-notice.md)]

This tutorial shows you how to use a system assigned identity for a Windows virtual machine (VM) to access Azure Key Vault. Serving as a bootstrap, Key Vault makes it possible for your client application to then use the secret to access resources not secured by Azure Active Directory (AD). Managed Service Identities are automatically managed by Azure and enable you to authenticate to services that support Azure AD authentication, without needing to insert credentials into your code. 

You learn how to:


> [!div class="checklist"]
> * Grant your VM access to a secret stored in a Key Vault 
> * Get an access token using the VM identity and use it to retrieve the secret from Key Vault 

## Prerequisites

[!INCLUDE [msi-qs-configure-prereqs](../../../includes/active-directory-msi-qs-configure-prereqs.md)]

[!INCLUDE [msi-tut-prereqs](../../../includes/active-directory-msi-tut-prereqs.md)]

- [Sign in to Azure portal](https://portal.azure.com)

- [Create a Windows virtual machine](/azure/virtual-machines/windows/quick-create-portal)

- [Enable system assigned identity on your virtual machine](/azure/active-directory/managed-service-identity/qs-configure-portal-windows-vm#enable-system-assigned-identity-on-an-existing-vm)

## Grant your VM access to a Secret stored in a Key Vault 
 
Using Managed Service Identity your code can get access tokens to authenticate to resources that support Azure AD authentication.  However, not all Azure services support Azure AD authentication. To use Managed Service Identity with those services, store the service credentials in Azure Key Vault, and use Managed Service Identity to access Key Vault to retrieve the credentials. 

First, we need to create a Key Vault and grant our VM’s identity access to the Key Vault.   

1. At the top of the left navigation bar, select **Create a resource** > **Security + Identity** > **Key Vault**.  
2. Provide a **Name** for the new Key Vault. 
3. Locate the Key Vault in the same subscription and resource group as the VM you created earlier. 
4. Select **Access policies** and click **Add new**. 
5. In Configure from template, select **Secret Management**. 
6. Choose **Select Principal**, and in the search field enter the name of the VM you created earlier.  Select the VM in the result list and click **Select**. 
7. Click **OK** to finishing adding the new access policy, and **OK** to finish access policy selection. 
8. Click **Create** to finish creating the Key Vault. 

    ![Alt image text](../managed-service-identity/media/msi-tutorial-windows-vm-access-nonaad/msi-blade.png)


Next, add a secret to the Key Vault, so that later you can retrieve the secret using code running in your VM: 

1. Select **All Resources**, and find and select the Key Vault you created. 
2. Select **Secrets**, and click **Add**. 
3. Select **Manual**, from **Upload options**. 
4. Enter a name and value for the secret.  The value can be anything you want. 
5. Leave the activation date and expiration date clear, and leave **Enabled** as **Yes**. 
6. Click **Create** to create the secret. 
 
## Get an access token using the VM identity and use it to retrieve the secret from the Key Vault  

If you don’t have PowerShell 4.3.1 or greater installed, you'll need to [download and install the latest version](https://docs.microsoft.com/powershell/azure/overview).

First, we use the VM’s Managed Service Identity to get an access token to authenticate to Key Vault:
 
1. In the portal, navigate to **Virtual Machines** and go to your Windows virtual machine and in the **Overview**, click **Connect**.
2. Enter in your **Username** and **Password** for which you added when you created the **Windows VM**.  
3. Now that you have created a **Remote Desktop Connection** with the virtual machine, open PowerShell in the remote session.  
4. In PowerShell, invoke the web request on the tenant to get the token for the local host in the specific port for the VM.  

    The PowerShell request:
    
    ```powershell
    $response = Invoke-WebRequest -Uri 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fvault.azure.net' -Method GET -Headers @{Metadata="true"} 
    ```
    
    Next, extract the full response which is stored as a JavaScript Object Notation (JSON) formatted string in the $response object.  
    
    ```powershell
    $content = $response.Content | ConvertFrom-Json 
    ```
    
    Next, extract the access token from the response.  
    
    ```powershell
    $KeyVaultToken = $content.access_token 
    ```
    
    Finally, use PowerShell’s Invoke-WebRequest command to retrieve the secret you created earlier in the Key Vault, passing the access token in the Authorization header.  You’ll need the URL of your Key Vault, which is in the **Essentials** section of the **Overview** page of the Key Vault.  
    
    ```powershell
    (Invoke-WebRequest -Uri https://<your-key-vault-URL>/secrets/<secret-name>?api-version=2016-10-01 -Method GET -Headers @{Authorization="Bearer $KeyVaultToken"}).content 
    ```
    
    The response will look like this: 
    
    ```powershell
    {"value":"p@ssw0rd!","id":"https://mytestkeyvault.vault.azure.net/secrets/MyTestSecret/7c2204c6093c4d859bc5b9eff8f29050","attributes":{"enabled":true,"created":1505088747,"updated":1505088747,"recoveryLevel":"Purgeable"}} 
    ```
    
Once you’ve retrieved the secret from the Key Vault, you can use it to authenticate to a service that requires a name and password. 

## Next steps

In this tutorial, you learned how to create a Managed Service Identity to access Azure Key Vault.  To learn more about Azure Key Vault see:

> [!div class="nextstepaction"]
>[Azure Key Vault](/azure/key-vault/key-vault-whatis)
