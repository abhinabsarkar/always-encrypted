# Rotate Microsoft generated keys

## Create a new CMK in Azure Key Vault
```ps
# Create a non-expiring Key of RSA type with encryption size 4096
# This key will act as CMK & have all the permitted operations
$akvKey = Add-AzKeyVaultKey -VaultName kv-aesql -Name msftgenkey2 -Destination Software -Size 4096
``` 

## Create CMK metadata in database
Create a SqlColumnMasterKeySettings object that contains information about the location of CMK in Azure Key Vault.
```ps
$CMKSettings = New-SqlAzureKeyVaultColumnMasterKeySettings -KeyUrl "https://aesql.vault.azure.net/keys/msftgenkey2/xhdfdgfhfh"
```
Connect to SQL database & create the CMK metadata in SQL DB
```ps
# create a CMK reference object in Database
New-SqlColumnMasterKey -Name "msftgencmk2" -ColumnMasterKeySettings $CMKSettings
```

## Initiate rotating the column master key
Replace an existing source column master key (CMK-current) with a new target column master key (CMK-new) for the column encryption key (CEK).

```ps
Invoke-SqlColumnMasterKeyRotation -SourceColumnMasterKeyName "msftgencmk1" -TargetColumnMasterKeyName "msftgencmk2"
```
![Alt text](/images/initiate-sql-cmk-rotation.jpg)

## Complete the CMK rotation process
Completes the process of replacing an existing column master key with a new, target, column master key for the column encryption key (CEK).

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

