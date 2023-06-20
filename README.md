# azure-policy

## Modifiable aliases & properties

get-azpolicyalias | Select-object -expandproperty Aliases | Where-Object {$_.DefaultMetaData -ne $Null} | where-object {$_.DefaultMetaData.Attributes -eq "Modifiable"}
Modifiable:
- Microsoft.Compute/virtualMachines/securityProfile.encryptionAtHost
- Microsoft.Compute/virtualMachineScaleSets/virtualmachines/securityProfile.encryptionAtHost
- Microsoft.Compute/virtualMachines/storageProfile.osDisk.managedDisk.diskEncryptionSet.id
- Microsoft.Compute/virtualMachines/storageProfile.dataDisks[*].managedDisk.diskEncryptionSet.id
- Microsoft.Compute/virtualMachineScaleSets/virtualMachines/storageProfile.osDisk.managedDisk.diskEncryptionSet.id
- Microsoft.Compute/virtualMachineScaleSets/virtualMachines/storageProfile.dataDisks[*].managedDisk.diskEncryptionSet.id
- others?

## Key Vault and DES pre-reqs (example only):
- az keyvault create -l $LOCATION -n $KEYVAULT_NAME -g $RG_GOVTEST --enabled-for-disk-encryption --enabled-for-deployment --enable-soft-delete --enabled-for-template-deployment --network-acls-ips x.x.x.x/32 [SRC IP where this command is being executed] --default-action Deny --bypass AzureServices  # --network-acls-vnets $VM_SUBNET_ID
- KV_ID=$(az keyvault list --resource-group $RG_GOVTEST --query "[?name=='$KEYVAULT_NAME'].id" -o tsv)
- az keyvault key create -n $KEYVAULT_KEY_NAME --vault-name $KEYVAULT_NAME
- KV_KEY_ID=$(az keyvault key list --vault-name $KEYVAULT_NAME --query "[?name=='$KEYVAULT_KEY_NAME'].kid" -o tsv)
- KV_KEY_ID_VERSION=$(az keyvault key list-versions --id $KV_KEY_ID --query "[?name=='$KEYVAULT_KEY_NAME'].kid" -o tsv)
- az disk-encryption-set create --key-url $KV_KEY_ID_VERSION --name $DISK_ENCRYPTION_SET_NAME --resource-group $RG_GOVTEST --source-vault $KV_ID --encryption-type EncryptionAtRestWithCustomerKey
- desIdentity=$(az disk-encryption-set show -n $DISK_ENCRYPTION_SET_NAME -g $RG_GOVTEST --query [identity.principalId] -o tsv)
- az keyvault set-policy -n $KEYVAULT_NAME -g $RG_GOVTEST --object-id $desIdentity --key-permissions wrapkey unwrapkey get
### --- Enable private link on KV and config DNS (if not already configured e.g. by policy etc) - assumes vnet and subnet already in place ---
- az network private-dns zone create --resource-group $RG_GOVTEST --name privatelink.vaultcore.azure.net
- az network private-dns link vnet create --resource-group $RG_GOVTEST --virtual-network $VNET_govtest --zone-name privatelink.vaultcore.azure.net --name plinkdnslinkkv --registration-enabled false
- az network private-endpoint create --name $KEYVAULT_NAME --resource-group $RG_GOVTEST --vnet-name $VNET_govtest --subnet $SUBNET_govtest_VM --private-connection-resource-id $KV_ID --group-id vault --connection-name conn-kv
- privateEndpointNIC=$(az network private-endpoint show --ids $KEYVAULT_PRIVATE_EP_ID --query "networkInterfaces[0].id" -o tsv)
- privateEndpointIP=$(az network nic show --ids $privateEndpointNIC --query "ipConfigurations[0].privateIpAddress" -o tsv)
- az network private-dns record-set a create --resource-group $RG_GOVTEST --zone-name privatelink.vaultcore.azure.net --name $KEYVAULT_NAME
- az network private-dns record-set a add-record --resource-group $RG_GOVTEST --zone-name privatelink.vaultcore.azure.net --record-set-name $KEYVAULT_NAME --ipv4-address $privateEndpointIP

