# Rotate Always Encrypted keys
Always Encrypted uses two types of keys, so there are two high-level key rotation workflows; rotating column master keys, and rotating column encryption keys.
* Column encryption key rotation - involves decrypting data that is encrypted with the current key, and re-encrypting the data using the new column encryption key. Because rotating a column encryption key requires access to both the keys and the database, column encryption key rotation can only be performed without role separation.
* Column master key rotation - involves decrypting column encryption keys that are protected with the current column master key, re-encrypting them using the new column master key, and updating the metadata for both types of keys. Column master key rotation can be completed with or without role separation (when using the SqlServer PowerShell module).
## Column Master Key Rotation
Create a new column master key
```ps
# Create a self signed certificate
$cert = New-SelfSignedCertificate -Subject "AlwaysEncryptedCert2" -CertStoreLocation Cert:LocalMachine\My -KeyExportPolicy Exportable -Type DocumentEncryptionCert -KeyUsage KeyEncipherment -KeySpec KeyExchange -KeyLength 2048
# Check the cert created
$cert
   PSParentPath: Microsoft.PowerShell.Security\Certificate::LocalMachine\My
Thumbprint                                Subject
----------                                -------
D7DB940C57E48D98082AFE9EF5D69BD3B1157686  CN=AlwaysEncryptedCert2
```
Export CMK (stored as certificate) with private key from the local machine store to a PFX file which includes the entire chain and all external properties
```ps
# Store password as a secure string
$mypwd = ConvertTo-SecureString -String "password" -Force -AsPlainText
# export certificate
Get-ChildItem -Path cert:\localMachine\my\D7DB940C57E48D98082AFE9EF5D69BD3B1157686 | Export-PfxCertificate -FilePath d:\AECert2.pfx -Password $mypwd
```