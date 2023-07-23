# 1.	Set environment variables.
```
export RESOURCE_GROUP="agic-helm-rg"
export VNET_NAME="agic-helm-vnet"
export AKS_SUBNET_NAME="aks-snet"
export APPLICATION_GATEWAY_SUBNET_NAME="appgw-snet"
export AKS_NAME="aks-helm"
export APPLICATION_GATEWAY_PIP_NAME="appgw-pip"
export APPLICATION_GATEWAY_NAME="appgw-helm"
export USER_ASSIGNED_IDENTITY_NAME="agic-identity-helm"
export FEDERATED_IDENTITY_CREDENTIAL_NAME="fed-identity-helm"
```

# 2.	Create resource group, VNET, subnets, AKS cluster, Application Gateway Public IP, Application Gateway, and Identity.
```
az group create --name "${RESOURCE_GROUP}" --location eastus

az network vnet create --resource-group "${RESOURCE_GROUP}" --name "${VNET_NAME}" --address-prefixes 10.224.0.0/12

az network vnet subnet create --resource-group "${RESOURCE_GROUP}" --vnet-name "${VNET_NAME}" --name "${AKS_SUBNET_NAME}" --address-prefixes 10.224.0.0/16

az network vnet subnet create --resource-group "${RESOURCE_GROUP}" --vnet-name "${VNET_NAME}" --name "${APPLICATION_GATEWAY_SUBNET_NAME}" --address-prefixes 10.225.0.0/16

export AKS_SUBNET_ID="$(az network vnet subnet show --resource-group "${RESOURCE_GROUP}" --vnet-name "${VNET_NAME}" --name "${AKS_SUBNET_NAME}" --query id --output tsv)"

az aks create -g "${RESOURCE_GROUP}" -n "${AKS_NAME}" -s "Standard_B2ms" --node-count 2 --enable-oidc-issuer --enable-workload-identity --network-plugin azure --vnet-subnet-id "${AKS_SUBNET_ID}"

az network public-ip create -g "${RESOURCE_GROUP}" -n "${APPLICATION_GATEWAY_PIP_NAME}" --allocation-method Static --sku Standard --tier Regional

az network application-gateway create -g "${RESOURCE_GROUP}" -n "${APPLICATION_GATEWAY_NAME}" --sku Standard_v2 --public-ip-address "${APPLICATION_GATEWAY_PIP_NAME}" --vnet-name "${VNET_NAME}" --subnet "${APPLICATION_GATEWAY_SUBNET_NAME}" --priority 100

az identity create --name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}"
```

# 3.	Export the oidcIssuerProfile.issuerUrl.
```
export AKS_OIDC_ISSUER="$(az aks show -n "${AKS_NAME}" -g "${RESOURCE_GROUP}" --query "oidcIssuerProfile.issuerUrl" -otsv)"
```

# 4.	Created federated identity credential. 
Note the name of the service account that gets created after the helm installation is “ingress-azure” and the following command assumes it will be deployed in “default” namespace. Please change the namespace name (we are using appgw namespace to intstall AGIC) if you deploy the AGIC related Kubernetes resources in other namespaces.
```
az identity federated-credential create --resource-group ${RESOURCE_GROUP} --name ${FEDERATED_IDENTITY_CREDENTIAL_NAME} --identity-name ${USER_ASSIGNED_IDENTITY_NAME} --issuer ${AKS_OIDC_ISSUER} --subject system:serviceaccount:appgw:ingress-azure
```

# 5.	Obtain the ClientID of the identity created before that is needed for the next step.
```
export IDENTITY_CLIENT_ID="$(az identity show --resource-group "${RESOURCE_GROUP}" --name "${USER_ASSIGNED_IDENTITY_NAME}" --query 'clientId' -otsv)"
```

# 6.	Export the Application Gateway resource ID.
```
export APP_GW_ID="$(az network application-gateway show --name "${APPLICATION_GATEWAY_NAME}" --resource-group "${RESOURCE_GROUP}" --query 'id' --output tsv)"
export APP_VNET_ID="$(az network vnet show --name "${VNET_NAME}" --resource-group "${RESOURCE_GROUP}" --query 'id' --output tsv)"
```

# 7.	Add Contributor role for the identity over the Application Gateway.
```
az role assignment create --assignee "${IDENTITY_CLIENT_ID}" --scope "${APP_GW_ID}" --role Contributor
```
Note: Sometime AGIC pod throws error while applying or deleting ingress. In that case, assign Network Contributor role for the identity over the VNet. Since the application gateway resource is deployed inside a virtual network, azure also perform a check to verify the permission on the provided virtual network resource.
```
az role assignment create --assignee "${IDENTITY_CLIENT_ID}" --scope "${APP_VNET_ID}" --role "Contributor"
```

# 8. Download helm-config.yaml, which will configure AGIC.
```
wget https://raw.githubusercontent.com/Azure/application-gateway-kubernetes-ingress/master/docs/examples/sample-helm-config.yaml -O helm-config.yaml
```

# 9.	In helm-config.yaml specify the Application Gateway details including the armAuth type.
```
armAuth:
    type: workloadIdentity
    identityClientID: b50d268c-78bc-4ab4-a819-50d8bf9fbffd
```

# 10. Get the AKS cluster credentials.
```
az aks get-credentials -g "${RESOURCE_GROUP}" -n "${AKS_NAME}"
```

# 11. Add the AGIC Helm repository.
```
helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/

helm repo update

helm search repo -l application-gateway-kubernetes-ingress
```

# 12. Install the helm chart.
```
kubectl create ns appgw

helm install ingress-azure \
  --namespace appgw \
  -f helm-config.yaml \
  application-gateway-kubernetes-ingress/ingress-azure \
  --version 1.7.1
```