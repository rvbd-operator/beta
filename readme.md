 

# Getting started
Welcome to the Alluvio Operator installation guide. This guide will quickly show you how to install the Alluvio Operator and instrument Java and .Net applications running in Kubernetes.

# Attach to your cluster
Ensure that kubectl points to your Kubernetes cluster where the Alluvio Operator and Alluvio APM Agent will run.

**Attach to your Azure cluster**
```
az aks get-credentials --resource-group <your-resource-group> --name <your-aks-cluster-name>
```

**Attach to your AWS cluster**
```
aws eks --region <region-name> update-kubeconfig --name <cluster-name>
```

# Install a Certificate Manager
The Alluvio Operator requires that a certificate manager is installed in your cluster and that your cluster uses Kubernetes version 1.22 or greater.

**Check if a certificate manager is installed on your cluster**
```
kubectl get pods --namespace cert-manager -l app=cert-manager
```
**Install a certificate manager in your cluster**
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```

# Install the Alluvio Operator
```
kubectl apply -f https://raw.githubusercontent.com/rvbd-operator/beta/main/alluvio-operator.yaml
```

# Configure the Alluvio APM Agent
The Customer ID and Analysis Server Host will need to be configured for the APM Agent.

```
kubectl create -f https://raw.githubusercontent.com/rvbd-operator/beta/main/alluvio_configuration.yaml --namespace=alluvio-operator --edit
```

Under the ‘spec’ section of the file:
- update the customerId to your Customer ID
- update the analysisServerHost to your Analysis Server Host identifier

```
spec:
  customerId: "MyCustId"
  analysisServerHost: "MyHostId"
```

**Verify that the Alluvio APM Agent is running**
```
kubectl get pods -n alluvio-operator
```

An Alluvio APM Agent pod will be running for each node:

```
NAME                                                  READY   STATUS    RESTARTS   AGE
alluvio-apm-agent-2hpcs                               1/1     Running   0          6m36s
alluvio-apm-agent-54v54                               1/1     Running   0          6m36s
alluvio-operator-controller-manager-d44c57448-8jdth   2/2     Running   0          19m
alluvio-reporter-8699f9bbb7-nxp4n                     1/1     Running   0          6m36s
```

# Enable auto-instrumentation for Java and .NET apps
Auto-instrumentation will take effect the next time the application is deployed.

**Update application-deployment-name and run**
```
kubectl patch deployment <application-deployment-name> -p '{"spec": {"template":{"metadata":{"annotations":{"instrument.apm.alluvio/inject":"true"}}}} }'
```
 
