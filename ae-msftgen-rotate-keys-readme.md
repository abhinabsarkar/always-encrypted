# Rotate Microsoft generated keys
Refer [link](https://docs.microsoft.com/en-us/sql/relational-databases/security/encryption/rotate-always-encrypted-keys-using-powershell?view=sql-server-ver15#column-master-key-rotation-without-role-separation)

## Create a new CMK in Azure Key Vault
```ps
# Connect to Azure
Connect-AzAccount
# Create a non-expiring Key of RSA type with encryption size 4096
# This key will act as CMK & have all the permitted operations
$akvKey = Add-AzKeyVaultKey -VaultName kv-aesql -Name msftgenkey2 -Destination Software -Size 4096 -debug
``` 

## Create CMK metadata in database
Create a SqlColumnMasterKeySettings object that contains information about the location of CMK in Azure Key Vault.
```ps
$CMKSettings = New-SqlAzureKeyVaultColumnMasterKeySettings -KeyUrl "https://kv-aesql.vault.azure.net/keys/msftgenkey2/23bdc76cc8d44436b978a74f5c5e1d2f"
```
Connect to SQL database & create the CMK metadata in SQL DB
```ps
# Navigate to the database in the remote instance.
cd SQLSERVER:\SQL\Win10EntNew\DEFAULT\Databases\abs
# create a CMK reference object in Database
New-SqlColumnMasterKey -Name "msftgencmk2" -ColumnMasterKeySettings $CMKSettings
# List column master keys in the above database.
Get-SqlColumnMasterKey
```

## Initiate rotating the column master key
The Invoke-SqlColumnMasterKeyRotation cmdlet replaces an existing source column master key with a new target column master key for the Always Encrypted feature.

The cmdlet retrieves all column encryption key objects that contain encrypted key values that are encrypted with the specified source column master key.

Then, the cmdlet decrypts the current encrypted values, re-encrypts the resulting plaintext values with the target column master key, and then updates the impacted column encryption key objects to add the new encrypted values.

As a result, each impacted column encryption key contains two encrypted values: one produced using the current source column master key and another, produced using the target column master key.
```ps
Invoke-SqlColumnMasterKeyRotation -SourceColumnMasterKeyName "msftgencmk1" -TargetColumnMasterKeyName "msftgencmk2" -debug
```
![Alt text](/images/initiate-sql-cmk-rotation.jpg)

## Complete the CMK rotation process
The Complete-SqlColumnMasterKeyRotation cmdlet completes the process of replacing an existing column master key with a new, target, column master key for the Always Encrypted feature.

The cmdlet gets all column encryption key objects containing encrypted key values that are encrypted with the specified source column master key.

The cmdlet then updates each column encryption key object to remove the entry for an encrypted value that was produced using the specified column master key.

As a result, each impacted column encryption key object will have only one encrypted value entry, produced using the column master key that is the target of the rotation.

> Note: before executing this step, make sure all applications that query encrypted columns that are protected with the old column master key, have been configured to use the new column master key.

```ps
# pass the name of the key parameter to be rotated 
Complete-SqlColumnMasterKeyRotation -SourceColumnMasterKeyName "msftgencmk1"
```
![Alt text](/images/complete-sql-cmk-rotation.jpg)

## Remove the old CMK 
```ps
Remove-SqlColumnMasterKey -Name "msftgencmk1"
```

