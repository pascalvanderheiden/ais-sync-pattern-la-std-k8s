# ais-sync-pattern-la-std-k8s
Running Logic Apps Standard in Kubernetes on-premise managed by Azure Arc.

## Prerequisites
* Install Minikube to run Kubernetes locally [Minikube](https://minikube.sigs.k8s.io/docs/start/)
* Install Visual Studio Code [Visual Studio Code](https://code.visualstudio.com/download)
* Install Azure CLI [Azure CLI](https://docs.microsoft.com/nl-nl/cli/azure/install-azure-cli-windows?tabs=azure-cli)
* Install Azure Function Core Tools [Azure Functions Core Tools](https://github.com/Azure/azure-functions-core-tools)
* Install Azure Logic Apps (Standard) Extension for VSCode [Azure Logic Apps (Standard)](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurelogicapps)
* Install Azurite Extension for VSCode [Azurite](https://marketplace.visualstudio.com/items?itemName=Azurite.azurite)

## Setup Kubernetes Cluster locally
* Start Minikube (initialize Kubernetes cluster locally)
```
minikube start
```
* Log into Azure CLI and follow the instructions
```
az login
```
* Install Azure CLI extention
```
az extension add --name connectedk8s
az extension add --name k8s-extension
az extension add --name customlocation
az extension add --upgrade -n appservice-kube
```
* Register providers
```
az provider register --namespace Microsoft.Kubernetes
az provider register --namespace Microsoft.KubernetesConfiguration
az provider register --namespace Microsoft.ExtendedLocation
az provider register --namespace Microsoft.Web
az provider register --namespace Microsoft.OperationalInsights
```

## Setup Azure Arc & connect to Kubernetes cluster 
* Create a resource group Cluster
```ps1
$resourceGroup="ais-sync-pattern-la-std-k8s-rg"
az group create --name $resourceGroup --location WestEurope --output table
```
* Create a resource group for Monitor
```ps1
$resourceGroupMonitor="ais-sync-pattern-la-std-mon-rg"
az group create --name $resourceGroupMonitor --location WestEurope --output table
```
* Create a Log Analytics Workspace for Arc logging
```ps1
$logAnalytics="surface-laptop-ws"
az monitor log-analytics cluster create -g $resourceGroupMonitor -n $logAnalytics --sku-capacity 1000
```
* Create Application Insights Service for Logic Apps logging
```ps1
$appInsights="surface-laptop-ai"
az monitor app-insights component create -g $resourceGroupMonitor \
   --app appInsights --location westeurope --kind web --application-type web \
   --retention-time 120
```
* Connect an existing Kubernetes Cluster
```ps1
$clusterName="SURFACE_LAPTOP"
az connectedk8s connect --name $clusterName --resource-group $resourceGroup
```
* Verify cluster connection
```ps1
az connectedk8s list --resource-group $resourceGroup --output table
```
* View Arc agents on Kubernetes
```
kubectl get deployments,pods -n azure-arc
```
* Enable feature Custom Locations on Cluster
```ps1
az connectedk8s enable-features -n $clusterName -g $resourceGroup --features cluster-connect custom-locations
```
* Install App Service extention
```ps1
$extensionName="appservice-ext" # Name of the App Service extension
$namespace="default" # Namespace in your cluster to install the extension and provision resources
$kubeEnvironmentName="SURFACE_LAPTOP" # Name of the App Service Kubernetes environment resource
$logAnalyticsWorkspaceId=$(az monitor log-analytics workspace show `
    --resource-group $resourceGroupMonitor `
    --workspace-name $logAnalytics `
    --query customerId `
    --output tsv)
$logAnalyticsWorkspaceIdEnc=[Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($logAnalyticsWorkspaceId))# Needed for the next step
$logAnalyticsKey=$(az monitor log-analytics workspace get-shared-keys `
    --resource-group $resourceGroupMonitor `
    --workspace-name $logAnalytics `
    --query primarySharedKey `
    --output tsv)
$logAnalyticsKeyEnc=[Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($logAnalyticsKey))

az k8s-extension create `
    --resource-group $resourceGroup `
    --name $extensionName `
    --cluster-type connectedClusters `
    --cluster-name $clusterName `
    --extension-type 'Microsoft.Web.Appservice' `
    --release-train stable `
    --auto-upgrade-minor-version true `
    --scope cluster `
    --release-namespace $namespace `
    --configuration-settings "Microsoft.CustomLocation.ServiceAccount=default" `
    --configuration-settings "appsNamespace=${namespace}" `
    --configuration-settings "clusterName=${kubeEnvironmentName}" `
    --configuration-settings "keda.enabled=true" `
    --configuration-settings "buildService.storageClassName=default" `
    --configuration-settings "buildService.storageAccessMode=ReadWriteOnce" `
    --configuration-settings "customConfigMap=${namespace}/kube-environment-config" `
    --configuration-settings "envoy.annotations.service.beta.kubernetes.io/azure-load-balancer-resource-group=${aksClusterGroupName}" `
    --configuration-settings "logProcessor.appLogs.destination=log-analytics" `
    --configuration-protected-settings "logProcessor.appLogs.logAnalyticsConfig.customerId=${logAnalyticsWorkspaceIdEnc}" `
    --configuration-protected-settings "logProcessor.appLogs.logAnalyticsConfig.sharedKey=${logAnalyticsKeyEnc}"
```
* Save the App Service extention property id
```ps1
$extensionId=$(az k8s-extension show `
    --cluster-type connectedClusters `
    --cluster-name $clusterName `
    --resource-group $resourceGroup `
    --name $extensionName `
    --query id `
    --output tsv)
```
* Create a Custom Location
```ps1
$customLocationName="surface-laptop" # Name of the custom location
$connectedClusterId=$(az connectedk8s show --resource-group $resourceGroup --name $clusterName --query id --output tsv)
az customlocation create `
    --resource-group $resourceGroup `
    --name $customLocationName `
    --host-resource-id $connectedClusterId `
    --namespace $namespace `
    --cluster-extension-ids $extensionId
```
