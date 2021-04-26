# Always Encrypted for SQL using AKV using MSFT generated keys 

## Create CMK using MSFT generated keys 
```ps
# Connect to Azure
Connect-AzAccount
# Create a non-expiring Key of RSA type with encryption size 4096
# This key will act as CMK & have all the permitted operations
$akvKey = Add-AzKeyVaultKey -VaultName kv-aesql -Name msftgenkey1 -Destination Software -Size 4096 -debug
```
## Assign access policy to the SP for Key operations
For any application to access the encryption keys, the Service Principal under the context of which it accesses AKV must be granted permissions. The permissions can be granted access policies.

## Create metadata about the CMK in SQL database
Create a SqlColumnMasterKeySettings object that contains information about the location of CMK in Azure Key Vault.
```ps
$CMKSettings = New-SqlAzureKeyVaultColumnMasterKeySettings -KeyUrl "https://kv-aesql.vault.azure.net:443/keys/msftgenkey1/28d06e04bd6d4e43895ad691d15a1c8b"
```
Connect to SQL database & create the CMK metadata in SQL DB
```ps
# Navigate to the database in the remote instance.
cd SQLSERVER:\SQL\servercomputer\DEFAULT\Databases\yourdatabase
# example
cd SQLSERVER:\SQL\Win10EntNew\DEFAULT\Databases\abs
# create a CMK reference object in Database
New-SqlColumnMasterKey -Name "msftgencmk1" -ColumnMasterKeySettings $CMKSettings
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
New-SqlColumnEncryptionKey -Name "msftgencek1" -ColumnMasterKeyName "msftgencmk1"
# List column encryption keys in the above database.
Get-SqlColumnEncryptionKey
```

## Encrypt a column
```ps
# Create an encryption setting object 
$EncryptionSettings = New-SqlColumnEncryptionSettings Purchasing.ShipMethod.Name "Deterministic" msftgencek1

# Apply encryption settings to columns using the offline approach
Set-SqlColumnEncryption -ColumnEncryptionSettings $EncryptionSettings -LogFileDirectory . -debug
```
Validate encryption
```sql
Select * from [abs].[Purchasing].[ShipMethod]
```

## Decrypt the encrypted column
```ps
$DecryptionSetting =  New-SqlColumnEncryptionSettings -ColumnName Purchasing.ShipMethod.Name -EncryptionType "Plaintext"
# Decrypt all columns.
Set-SqlColumnEncryption -ColumnEncryptionSettings $DecryptionSetting -LogFileDirectory .
```

Validate decryption
```sql
Select * from [abs].[Purchasing].[ShipMethod]
```