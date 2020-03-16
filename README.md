
# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

# Legal Notices

Microsoft and any contributors grant you a license to the Microsoft documentation and other content
in this repository under the [Creative Commons Attribution 4.0 International Public License](https://creativecommons.org/licenses/by/4.0/legalcode),
see the [LICENSE](LICENSE) file, and grant you a license to any code in the repository under the [MIT License](https://opensource.org/licenses/MIT), see the
[LICENSE-CODE](LICENSE-CODE) file.

Microsoft, Windows, Microsoft Azure and/or other Microsoft products and services referenced in the documentation
may be either trademarks or registered trademarks of Microsoft in the United States and/or other countries.
The licenses for this project do not grant you rights to use any Microsoft names, logos, or trademarks.
Microsoft's general trademark guidelines can be found at http://go.microsoft.com/fwlink/?LinkID=254653.

Privacy information can be found at https://privacy.microsoft.com/en-us/

Microsoft and any contributors reserve all other rights, whether under their respective copyrights, patents,
or trademarks, whether by implication, estoppel or otherwise.

## Setup in Azure 

```
az account set --subscription <your subscription>

export rgName=io-d-rg-testca
export location=westeurope

az group create --name $rgName --location $location

bash setup-infra.sh

echo https://"$(az vm show \
  --name webservervm1 \
  --resource-group $rgName \
  --show-details \
  --query [publicIps] \
  --output tsv)"
```

### Configure the backend pool for encryption

```
$ privateip="$(az vm list-ip-addresses \
  --resource-group $rgName \
  --name webservervm1 \
  --query "[0].virtualMachine.network.privateIpAddresses[0]" \
  --output tsv)"

$ echo $privateip 

$ export appGatewayName=gw-shipping 

$ az network application-gateway address-pool create   \
  --resource-group $rgName   \
  --gateway-name gw-shipping   \
  --name ap-backend

$ az network application-gateway address-pool create \   --resource-group $rgName \
--gateway-name $appGatewayName \
--name ap-backend \
--servers $privateip

$ az network application-gateway root-cert create   \
--resource-group $rgName    \
--gateway-name $appGatewayName   \
--name shipping-root-cert   \
--cert-file server-config/shipping-ssl.crt

$ az network application-gateway http-settings create \
--resource-group $rgName   \
--gateway-name $appGatewayName   \
--name https-settings   \
--port 443   \
--protocol Https   \
--host-name $privateip

$ export rgID="$(az group show --name $rgName --query id --output tsv)"

$ az network application-gateway http-settings update     --resource-group $rgName     \
--gateway-name $appGatewayName     \
--name https-settings     \
--set trustedRootCertificates='[{"id": "'$rgID'/providers/Microsoft.Network/applicationGateways/'$appGatewayName'/trustedRootCertificates/shipping-root-cert"}]'

$ az network application-gateway frontend-port create \
--resource-group $rgName   \
--gateway-name $appGatewayName   \
--name https-port   \
--port 443

$ export certPassword=somepassword

$ az network application-gateway ssl-cert create    \
--resource-group $rgName    \
--gateway-name $appGatewayName \
--name appgateway-cert    \
--cert-file server-config/appgateway.pfx    \
--cert-password $certPassword

$ az network application-gateway http-listener create \
--resource-group $rgName   \
--gateway-name $appGatewayName   \
--name https-listener   \
--frontend-port https-port   \
--ssl-cert appgateway-cert

$ az network application-gateway rule create     \
--resource-group $rgName     \
--gateway-name $appGatewayName     \
--name https-rule     \
--address-pool ap-backend     \
--http-listener https-listener     \
--http-settings https-settings     \
--rule-type Basic

$ echo https://$(az network public-ip show \
  --resource-group $rgName \
  --name appgwipaddr \
  --query ipAddress \
  --output tsv)
 

## Deploy Application Gateway with a template

### Generate certificate 

With Power Shell
```
$pfx_cert = get-content './appgateway.pfx' -AsByteStream
[System.Convert]::ToBase64String($pfx_cert) | Out-File ‘appgateway-pfx-encoded-bytes.txt’

```

bash 
```
$ export certData=$(cat server-config/appgateway-pfx-encoded-bytes.txt)
```

### Root CA certificate

```
$ openssl req -x509 -new -nodes -key appgateway-privatekey.key -sha256 -days 1825 -out appgateway.pem
```

Encode the pem file in a base64 string with this online tool 
https://www.browserling.com/tools/file-to-base64
and save it to the file appgateway-pem-encoded-bytes.txt



$ export clientCertData=$(cat server-config/appgateway-pem-encoded-bytes.txt)
```

$ az group deployment validate \
  --resource-group $rgName \
  --template-file templates/appgateway.json \
  --parameters certPassword=$certPassword \
  --parameters certData=$certData \
  --parameters clientCertData=$clientCertData
 ```

```
$ az group deployment create \
  --name UpdateAppGateWay \
  --resource-group $rgName \
  --template-file templates/appgateway.json \
  --parameters certPassword=$certPassword \
  --parameters certData=$certData \
  --parameters clientCertData=$clientCertData
```