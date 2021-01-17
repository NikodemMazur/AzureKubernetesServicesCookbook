# AzureKubernetesServicesCookbook
## Table of contents
## Recipes
### Print the logs for a container
```powershell
kubectl get pods -n <namespace> # list pods in a namespace
kubectl logs <pod> -n <namespace> -c <container> # print the logs for a container
```