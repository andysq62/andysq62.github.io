---
layout: post
title: Working With the Secrets Management Powershell Module
categories: [Powershell]
---

## Working With Secrets

Ever Since Microsoft's Secret Management Powershell module has arrived I've wanted to learn how to use it as a better way to deal with secrets and automation. I run a lot of Powershell scheduled jobs doing application administration and they need credentials in order to run.



## How I Do It Now

The process I use now is to store the credentials as a PSCredential object, e.g. 



```powershell
Get-Credential | Export-CliXML -Path D:\Credentials\User01.xml
```



Then in my automation script from a jump box I retrieve the credentials and run a process:



```powershell
$Cred = Import-CliXML -Path D:\Credentials\User01.xml
$Session = New-PSSession -Name svr01 -ComputerName Svr01 -Credential $Cred
Invoke-Command -PSSession $Session {
    -- Do Stuff ---
}
```



## Install Secret Management

To begin using secret Management you first need to install the Powershell module from the Powershell Gallery:



```powershell
Install-Module Microsoft.Powershell.SecretManagement
```



This module has several extension modules for various types of vaults.  I was hoping for an extension to Delinea Secret Server, but it doesn't look like there is one available yet in the gallery.  So, my next option was to use Microsoft's own Secret Store and one can install it like so:



```powershell
Install-Module Microsoft.Powershell.SecretStore
```



Here is a look at the Cmdlets the SecretManagement module provides:



```powershell
Get-Command -Module Microsoft.Powershell.SecretManagement

CommandType     Name                                               Version    Source
-----------     ----                                               -------    ------
Cmdlet          Get-Secret                                         1.1.2      Microsoft.Powershell.SecretManagement
Cmdlet          Get-SecretInfo                                     1.1.2      Microsoft.Powershell.SecretManagement
Cmdlet          Get-SecretVault                                    1.1.2      Microsoft.Powershell.SecretManagement
Cmdlet          Register-SecretVault                               1.1.2      Microsoft.Powershell.SecretManagement
Cmdlet          Remove-Secret                                      1.1.2      Microsoft.Powershell.SecretManagement
Cmdlet          Set-Secret                                         1.1.2      Microsoft.Powershell.SecretManagement
Cmdlet          Set-SecretInfo                                     1.1.2      Microsoft.Powershell.SecretManagement
Cmdlet          Set-SecretVaultDefault                             1.1.2      Microsoft.Powershell.SecretManagement
Cmdlet          Test-SecretVault                                   1.1.2      Microsoft.Powershell.SecretManagement
Cmdlet          Unlock-SecretVault                                 1.1.2      Microsoft.Powershell.SecretManagement
Cmdlet          Unregister-SecretVault                             1.1.2      Microsoft.Powershell.SecretManagement
```



## Setting Up a Secrets Vault

The first step we need to do is register a secret vault in the secret store.  If you don't include a name, the cmdlet will use the module name, and as one can see the -DefaultVault parameter will make it the default.  You can have multiple vaults, and each is tied to a single user context.



```powershell
Register-SecretVault -Name MySecretVault -ModuleName Microsoft.PowerShell.SecretStore -DefaultVault
Get-SecretVault

Name       ModuleName                       IsDefaultVault
----       ----------                       --------------
MySecretVault Microsoft.Powershell.SecretStore True
```



## Configuring the Vault for Automation

To configure the vault up for use in automation scripts you can employ the original method for storing the vault credentials, i.e. store them in an xml file.  I just use a dummy user, 'MySecretVault'.  You could just store the password itself as a secure string in an xml file as well.



```powershell
$VaultPasswordPath = 'D:\Credentials\MySecretVault.xml'
Get-Credential | Export-CliXML -PATH $VaultPasswordPath

Windows PowerShell credential request  dialog
User name:  edit combo
To set the value use the Arrow keys or type the value.
Alt+u
Password:  password edit
```



Then retrieve the password and configure the vault for no interaction.  The timeout value is in seconds and refers to the time the vault can be accessed before it must be unlocked again.



```powershell
$password = (Import-CliXml -Path $VaultPasswordPath).Password
$VaultConfig = @{
    Authentication = 'Password'
    PasswordTimeout = 60
    Interaction = 'None'
    Password = $password
    Confirm = $false
}
Set-SecretStoreConfiguration @VaultConfig
```



Now our vault is configured to be used in an automation script, or scheduled job.  Next we need to enter some secrets into the vault.


## Setting a Secret

Setting a secret is as simple as the following:



```powershell
Set-Secret -Name 'User01' -Secret (Get-Credential)
Get-Secret -Name 'User01'

UserName                        Password
--------                        --------
User01 System.Security.SecureString
```



## Testing Automation

I set up the following scriptblock to test the automation.  This also works from a scheduled job.



```powershell
$ScriptBlock = {
    $VaultPasswordPath = 'D:\Credentials\MySecretVault.xml'
    $password = (Import-CliXml -Path $VaultPasswordPath).Password
    Unlock-SecretStore -Password $password
    $Result = Invoke-Command -ComputerName svr01 -Credential (Get-Secret -Name User01) {
        $Env:ComputerName
    }
    $Result
}

svr01
```



## References

[Microsoft.Powershell.SecretManagement Module](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.secretmanagement/?view=ps-modules)
[Using the Secret Store in Automation - Powershell](https://learn.microsoft.com/en-us/powershell/utility-modules/secretmanagement/how-to/using-secrets-in-automation?view=ps-modules)
