# AzureKubernetesServicesCookbook
## Table of contents
- [AzureKubernetesServicesCookbook](#AzureKubernetesServicesCookbook)
    * [Table of contents](#Table-of-contents)
        + [Print the logs for a container](#Print-the-logs-for-a-container)
        + [Get fully qualified domain names FQDN of the cluster](#Get-fully-qualified-domain-names-FQDN-of-the-cluster)
### Print the logs for a container
```powershell
kubectl get pods -n <namespace> # list pods in a namespace
kubectl logs <pod> -n <namespace> -c <container> # print the logs for a container
```
### Query fully qualified domain names FQDN of the cluster
```powershell
az network dns zone list -o table # query the Azure DNS zone list

az aks show `
-g <resource-group> `
-n <cluster-name> `
-o tsv `
--query addonProfiles.httpApplicationRouting.config.HTTPApplicationRoutingZoneName # get aks cluster DNS zone

az network dns record-set list -g <resource-group> -z <zone-name> --output table # query FQDNs
```