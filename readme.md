 

# Getting started
Welcome to the Riverbed Operator installation guide. This guide will quickly show you how to install the Alluvio Operator and instrument Java and .Net applications running in Kubernetes.

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

## Installation using riverbed public registry.

```
kubectl apply -f https://raw.githubusercontent.com/rvbd-operator/beta/1.0.0/riverbed-operator.yaml
```

## Installation using Zeus private registry.


```
kubectl apply -f https://raw.githubusercontent.com/rvbd-operator/beta/1.0.0/riverbed-operator.yaml --edit
```

***Locate and edit the image identifier***
For example
```
        env:
        - name: RVBD_JAVA_INSTRUMENTATION_IMAGE
          value: riverbed/riverbed-java-instrumentation:12.25.0.512
        - name: RVBD_DOTNET_INSTRUMENTATION_IMAGE
          value: riverbed/riverbed-dotnet-instrumentation:12.25.0.512
        - name: RVBD_APM_AGENT_IMAGE
          value: riverbed/riverbed-apm-agent:12.25.0.512
        image: zeus.run/rmurray/riverbed-operator:1.0.0

```
Change the image identifiers as appropriate for example:

```
        - name: RVBD_JAVA_INSTRUMENTATION_IMAGE
          value: zeus.run/agent/riverbed-java:12.25.0.512
        - name: RVBD_DOTNET_INSTRUMENTATION_IMAGE
          value: zeus.run/agent/riverbed-dotnet:12.25.0.512
        - name: RVBD_APM_AGENT_IMAGE
          value: zeus.run/agent/sidecar:12.25.0.512
        image: zeus.run/rmurray/riverbed-operator:1.0.0

```
Change the image identifiers


***Create an imagePullSecret***

Create a secret to access Docker image is from a Zeus private repository, a secret needs to be created for pulling the image . To create the secret, enter the following command, where <your-registry-server> is zeus-run, <your-name> is your Docker username, <your-pword> is your Docker password, <your-email> is your Docker email.   and <your-namespace> is the namespace where you intend to install the agent (ie riverbed)  You can obtain docker credentials from https://zeus.run/#account:

```
kubectl create secret docker-registry <your-secret-name> -n riverbed-operator --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```
for example:
```
kubectl create secret docker-registry regcred -n riverbed-operator --docker-server=zeus.run --docker-username=rmurray --docker-password=xxxxxxxxx --docker-email=rmurray@riverbed.com

```

***Add image pull secret to service account***
```
kubectl patch serviceaccount riverbed-operator-controller-manager -n riverbed-operator -p '{"imagePullSecrets": [{"name": "<secret-name>"}]}'
```

# Configure the Riverbed Operator


The Customer ID and Analysis Server Host will need to be configured for the APM Agent.


```
kubectl create -f https://raw.githubusercontent.com/rvbd-operator/beta/1.0.0/riverbed_configuration_betav1.yaml --namespace=riverbed-operator --edit
```

Under the ‘spec’ section of the file:
- update the customerId to your Customer ID
- update the analysisServerHost to your Analysis Server Host identifier
- update the configName to specify the default process configuration name used by your instrumented applications

```
spec:
  customerId: "MyCustId"
  analysisServerHost: "MyHostId"
  configName: "default config"
```

***Additional configuration when using a private image registry***
Add image pull secret to riverbed-apm-agent service account
```
kubectl patch serviceaccount riverbed-apm-agent -n riverbed-operator -p '{"imagePullSecrets": [{"name": "<secret-name>"}]}'``
```

**Verify that the Riverbed APM Agent is running**
```
kubectl get pods -n riverbed-operator
```

A Riverbed APM Agent pod will be running for each node:

```
NAME                                                  READY   STATUS    RESTARTS   AGE
riverbed-apm-agent-2hpcs                               1/1     Running   0          6m36s
riverbed-apm-agent-54v54                               1/1     Running   0          6m36s
riverbed-operator-controller-manager-d44c57448-8jdth   2/2     Running   0          19m
```

# Enable auto-instrumentation for Java and .NET apps
Auto-instrumentation will take effect the next time the application is deployed.  If the application is deployed, it will automatically restart.

**Update Alpine (linux-musl64) application-deployment-name**
```
kubectl patch deployment <application-deployment-name> -p '{"spec": {"template":{"metadata":{"annotations":{"instrument.apm.riverbed/inject-runtime":"linux-musl-x64"}}}} }'
```

**Update java application-deployment-name**
```
kubectl patch deployment <application-deployment-name> -p '{"spec": {"template":{"metadata":{"annotations":{"instrument.apm.riverbed/inject-java":"true"}}}} }'
```

**Update dotnet application-deployment-name**
```
kubectl patch deployment <application-deployment-name> -p '{"spec": {"template":{"metadata":{"annotations":{"instrument.apm.riverbed/inject-dotnet":"true"}}}} }'
```
# Using non-default configurations
If you are using a configuration that is different from the configName specified in the APM Agent,  You will need to annotate you deployment.
```
kubectl patch deployment <application-deployment-name> -p '{"spec": {"template":{"metadata":{"annotations":{"instrument.apm.riverbed/configName":"myConfig"}}}} }'
```

# Deploying sample applications TBD ignore for now:
**Deploy tomcat**
```
kubectl apply -f tomcat.yaml`
```
**Deploy ims-tier1**
```
kubectl apply -f  ims-tier-autoint.yaml
```
**Deploy freshbrew (dotnet)**
```
kubectl apply -f  freshbrew-autoint.yaml

```