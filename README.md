# AzureKubernetesServicesCookbook
## Table of contents
- [AzureKubernetesServicesCookbook](#AzureKubernetesServicesCookbook)
    * [Table of contents](#Table-of-contents)
        + [Print the logs for a container](#Print-the-logs-for-a-container)
        + [Query fully qualified domain names FQDN of the cluster](#Query-fully-qualified-domain-names-FQDN-of-the-cluster)
        + [Print addon-http-application-routing ingress activity](#Print-addon-http-application-routing-ingress-activity)
        + [Start stop cluster](#Start-stop-cluster)
        + [Force to rollout new image](#Force-to-rollout-new-image)
        + [Apply k8s manifest](#Apply-k8s-manifest)
        + [Create AKS cluster](#Create-AKS-cluster)
            - [Define variables for the configuration values](#Define-variables-for-the-configuration-values)
            - [Create resources](#Create-resources)
                - [Resource group](#Resource-group)
                - [Azure Container Registry](#Azure-Container-Registry)
                - [Virtual network (required to use k8s network policies)](#Virtual-network-(required-to-use-k8s-network-policies))
                - [AKS Cluster](#AKS-Cluster)
            - [Link AKS with kubectl](#Link-AKS-with-kubectl)
### Print the logs for a container
```powershell
kubectl get pods -n <namespace> # list pods in a namespace
kubectl logs <pod> -n <namespace> -c <container> # print the logs for a container
```
### Query fully qualified domain names FQDN of the cluster
DNS zone name is needed to setup k8s ingresses
```powershell
az network dns zone list -o table # query the Azure DNS zone list

az aks show `
-g <resource-group> `
-n <cluster-name> `
-o tsv `
--query addonProfiles.httpApplicationRouting.config.HTTPApplicationRoutingZoneName # get aks cluster DNS zone

az network dns record-set list -g <resource-group> -z <zone-name> --output table # query FQDNs
```
### Print addon-http-application-routing ingress activity
```powershell
kubectl logs -f deploy/addon-http-application-routing-nginx-ingress-controller -n kube-system
```
### Start stop cluster
```powershell
az aks stop --name <cluster-name> --resource-group <resource-group>
az aks start --name <cluster-name> --resource-group <resource-group>
az aks show -n <cluster-name> -g <resource-group> --query powerState # verify if the cluster is stopped/started
```
### Force to rollout new image
```powershell
kubectl rollout restart deployment/<deployment-name> -n <namespace> # restart single deployment
kubectl rollout restart deployments -n <namespace> # restart all deployments within namespace
```

### Apply k8s manifest
```powershell
kubectl apply -f deployment.yml
```
### Create AKS cluster
#### Define variables for the configuration values
```powershell
$rgName = ...
$aksName = ...
$registryName = ...
$vnName = ...
$aksSubnetName = ...

# list available locations
az account list-locations `
  --query "[].{Name: name, DisplayName: displayName}" `
  --output table
az configure --defaults location=westeurope # set the default location

# query the latest AKS version available in your default region
$aksVersion=$(az aks get-versions `
  --query 'orchestrators[-1].orchestratorVersion' `
  --output tsv)
```
#### Create resources
##### Resource group
```powershell
az group create --name $rgName --location westeurope # create RG
```
##### Azure Container Registry
```powershell
# create ACR
az acr create `
  --name $registryName `
  --resource-group $rgName `
  --sku Basic
az acr list ` # show loginServer (needed when configuring pipeline in Azure DevOps)
 --resource-group $rgName `
 --query "[].{loginServer: loginServer}" `
 --output table
 ```
 ##### Virtual network (required to use k8s network policies)
```powershell
# create a virtual network and subnet
az network vnet create `
    --resource-group $rgName `
    --name $vnName `
    --address-prefixes 10.0.0.0/8 `
    --subnet-name $aksSubnetName `
    --subnet-prefix 10.240.0.0/16
# create a service principal and read in the application ID
$sp = $(az ad sp create-for-rbac --role Contributor --output json)
$spId = $($sp | ConvertFrom-Json).appId
$spPassword = $($sp | ConvertFrom-Json).password
# get the virtual network resource ID
$vnetId = $(az network vnet show --resource-group $rgName --name $vnName --query id -o tsv)
# assign the service principal Contributor permissions to the virtual network resource
az role assignment create --assignee $spId --scope $vnetId --role Contributor
# get the virtual network subnet resource ID
$subnetId = $(az network vnet subnet show --resource-group $rgName --vnet-name $vnName --name $aksSubnetName --query id -o tsv)
```
 ##### AKS Cluster
```powershell
az aks create `
    --resource-group $rgName `
    --name $aksName `
    --node-count 1 `
    --enable-addons http_application_routing,monitoring `
    --enable-managed-identity `
    --generate-ssh-keys `
    --node-vm-size Standard_B2s `
    --attach-acr $registryName `
    --kubernetes-version $aksVersion `
    --network-plugin azure `
    --network-policy azure `
    --service-cidr 10.0.0.0/16 `
    --dns-service-ip 10.0.0.10 `
    --docker-bridge-address 172.17.0.1/16 `
    --vnet-subnet-id $subnetId `
    --service-principal $spId `
    --client-secret $spPassword
```
- `--node-count 1` - virtual machines count
- `--enable-addons http_application_routing,monitoring` - http ingress (not production ready) and container health monitoring add-ons
- `--generate-ssh-keys` - create SSH key pair to access AKS nodes
- `--attach-acr $registryName` - attach ACR (you can also do it after the cluster creation with az aks update -n <cluster-name> -g <rg-name> --attach-acr <acr-name>)
- `--kubernetes-version $aksVersion` - latest k8s version
- `--network-plugin azure` - required to use k8s network policies
- `--network-policy azure` - same as above
- `--service-cidr 10.0.0.0/16` - IP range from which to assign service cluster IPs
- `--dns-service-ip 10.0.0.10` - Kubernetes DNS service
- `--docker-bridge-address 172.17.0.1/16` - IP address and netmask for the Docker bridge
- `--vnet-subnet-id $subnetId` - external vn is required to use k8s network policies
#### Link AKS with kubectl
```powershell
az aks get-credentials ` # this command will add an entry to your ~/.kube/config file, which holds all the information to access your clusters
    --name $aksName `
    --resource-group $rgName
```