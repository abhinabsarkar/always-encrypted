# Always Encrypted
## Sample AE database
Below table has the column "Name" encrypted   
```sql
SELECT [ShipMethodID]
      ,[Name]
      ,[ShipBase]
      ,[ShipRate]
      ,[rowguid]
      ,[ModifiedDate]
  FROM [abs].[Purchasing].[ShipMethod]  
```
Azure Key vault - kv-aesql  
AKV - Aexp - rg-kv  
AKV path - https://kv-aesql.vault.azure.net:443/keys/CMKAuto1/7f18b36734ac44638de9643b4a2393d9

COLUMN MASTER KEY - CMKAuto1  - stored in AKV
COLUMN ENCRYPTION KEY - CEKAuto1 - stored in SQL DB

Only CMK is rotated 
Rotation requires 2 CMK 

## Create CMK - Self-Signed certificate using Powershell
Example shows how to generate a certificate that can be used as a column master key for Always Encrypted - Refer [link](
https://docs.microsoft.com/en-us/sql/relational-databases/security/encryption/create-and-store-column-master-keys-always-encrypted?view=sql-server-ver15#create-a-self-signed-certificate-using-powershell)

```ps
# Create a self signed certificate
$cert = New-SelfSignedCertificate -Subject "AlwaysEncryptedCert1" -CertStoreLocation Cert:LocalMachine\My -KeyExportPolicy Exportable -Type DocumentEncryptionCert -KeyUsage KeyEncipherment -KeySpec KeyExchange -KeyLength 2048
# Check the cert created
$cert
   PSParentPath: Microsoft.PowerShell.Security\Certificate::LocalMachine\My
Thumbprint                                Subject
----------                                -------
4323D9ED1E74E3DAA091FBDC4FE8F367988F66E1  CN=AlwaysEncryptedCert1
```
## Export the CMK with private key
If your column master key is a certificate stored in the local machine certificate store location, you need to export the certificate with the private key and import it to all machines that host applications that are expected to encrypt or decrypt data stored in encrypted columns, or tools for configuring Always Encrypted and for managing the Always Encrypted keys. Also, each user must be granted a read permission for the certificate stored in the local machine certificate store location to be able to use the certificate as a column master key.

### Export CMK (stored as certificate) with private key manually
Open mmc.exe --> File menu --> Add/remove snap-in --> Certificates --> Add --> Computer account --> Finish --> Ok

Certificates snap-in --> Certificates --> Personal folder --> Right-click the Certificate --> All Tasks --> Export --> Next --> Export the private key --> .PFX --> Check password checkbox --> type password --> save file

### Export CMK (stored as certificate) with private key using Powershell 
Export a certificate from the local machine store to a PFX file which includes the entire chain and all external properties
```ps
# Store password as a secure string
$mypwd = ConvertTo-SecureString -String "1234" -Force -AsPlainText
# export certificate
Get-ChildItem -Path cert:\localMachine\my\4323D9ED1E74E3DAA091FBDC4FE8F367988F66E1 | Export-PfxCertificate -FilePath d:\mypfx.pfx -Password $mypwd
```
Store the certificate in Keys object store in Azure Key Vault
## Making Azure Key Vault Keys Available to Applications - Permissions required
When using an Azure Key Vault key as a column master key, your application needs to authenticate to Azure and your application's identity needs to have the following permissions on the key vault: get, unwrapKey, and verify.

To provision column encryption keys that are protected with a column master key stored in Azure Key Vault, you need the get, unwrapKey, wrapKey, sign, and verify permissions. Additionally, to create a new key in an Azure Key Vault you need the create permission; to list the key vault contents, you need the list permission.

## Create metadata about the CMK in SQL database
Refer [Provision Always Encrypted keys using PowerShell](https://docs.microsoft.com/en-us/sql/relational-databases/security/encryption/configure-always-encrypted-keys-using-powershell?view=sql-server-ver15)

Install PowerShell & connect to Azure 
```ps
# check the powershell version
Get-InstalledModule -Name Az -AllVersions
# install az module for powershell
Install-Module -Name Az -AllowClobber -Scope CurrentUser
# login to Azure interactively
Connect-AzAccount
# logout Azure
Disconnect-AzAccount
```
Install SQL Server PowerShell modules
```ps
# install sqlserver powershell module
Install-Module -Name SqlServer -Scope CurrentUser -AllowClobber
# get the sqlserver modules installed
Get-Module SqlServer -ListAvailable
```
Create a SqlColumnMasterKeySettings object that contains information about the location of CMK in Azure Key Vault.
```ps
$CMKSettings = New-SqlAzureKeyVaultColumnMasterKeySettings -KeyUrl "https://myvault.vault.contoso.net:443/keys/CMK/4c05f1a41b12488f9cba2ea964b6a700"
```
Connect to SQL database & create the CMK metadata in SQL DB
```ps
# Navigate to the database in the remote instance.
cd SQLSERVER:\SQL\servercomputer\DEFAULT\Databases\yourdatabas
# example
cd SQLSERVER:\SQL\Win10EntNew\DEFAULT\Databases\abs
# create a CMK reference object in Database
New-SqlColumnMasterKey -Name "ABSCMK1" -ColumnMasterKeySettings $CMKSettings
# List column master keys in the above database.
Get-SqlColumnMasterKey
```

## Generate Column Encryption Key
Since the CMK is stored in Azure Key Vault, first authenticate to Azure
```ps
# Azure authentication can be done interactively or automated
Add-SqlAzureAuthenticationContext -Interactive
```
Generate a CEK using CMK. This command generates a plaintext value of a column encryption key, encrypts the plaintext value with the specified master key, and then creates a column encryption key object, encapsulating the generated encrypted value in the database.
```ps
# Ensure you are connected to the SQL database first
New-SqlColumnEncryptionKey -Name "ABSCEK1" -ColumnMasterKeyName "ABSCMK1"
```
## Test the column encryption
Test column encryption using "ABSCEK1" using SSMS on a column
```sql
SELECT TOP (1000) [SalesReasonID]
      ,[Name]
      ,[ReasonType]
      ,[ModifiedDate]
  FROM [abs].[Sales].[SalesReason]
```
In order to decrypt the column, the following settings should be enabled in the SSMS client: 
* add "Column Encryption Setting = Enabled" in the Additional Connection Parameters in the SSMS Connect to Server window.
* query the table for the encrypted values

```sql
/****** Script creating table with data from another table  ******/
SELECT * FROM [abs].[Purchasing].[ShipMethod]

SELECT * 
INTO [abs].[Purchasing].[ShipMethod]
FROM [AdventureWorks2019].[Purchasing].[ShipMethod]

Select * from [abs].[Sales].[SalesReason]

SELECT * 
INTO [abs].[Sales].[SalesReason]
FROM [AdventureWorks2019].[Sales].[SalesReason]
```
