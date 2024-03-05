 

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
The Riverbed Operator requires that a certificate manager is installed in your cluster and that your cluster uses Kubernetes version 1.22 or greater.

**Check if a certificate manager is installed on your cluster**
```
kubectl get pods --namespace cert-manager -l app=cert-manager
```
**Install a certificate manager in your cluster**
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```

# Install the Riverbed Operator


```
kubectl apply -f https://raw.githubusercontent.com/rvbd-operator/beta/1.0.0/riverbed-operator.yaml
```

## Post install steps if using Zeus private registry.


***Create an imagePullSecret***

Create a secret named **regcred** to access Docker image is from a Zeus private repository, a secret needs to be created for pulling the image . To create the secret, enter the following command, where <your-registry-server> is zeus-run, <your-name> is your Docker username, <your-pword> is your Docker password, <your-email> is your Docker email.   and <your-namespace> is the namespace where you intend to install the agent (ie riverbed)  You can obtain docker credentials from https://zeus.run/#account:

```
kubectl create secret docker-registry regcred -n riverbed-operator --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```
for example:
```
kubectl create secret docker-registry regcred -n riverbed-operator --docker-server=zeus.run --docker-username=rmurray --docker-password=xxxxxxxxx --docker-email=rmurray@riverbed.com

```

***Add image pull secret to service account***
```
kubectl patch serviceaccount riverbed-operator-controller-manager -n riverbed-operator -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```

***Locate and edit the image identifier in riverbed-operator-controller-manager deployment***

```
kubectl edit deployment -n riverbed-operator riverbed-operator-controller-manager
```

For example if this is the original image identifiers:
```
        env:
        - name: RVBD_JAVA_INSTRUMENTATION_IMAGE
          value: riverbed/riverbed-java-instrumentation:12.25.0.512
        - name: RVBD_DOTNET_INSTRUMENTATION_IMAGE
          value: riverbed/riverbed-dotnet-instrumentation:12.25.0.512
        - name: RVBD_APM_AGENT_IMAGE
          value: riverbed/riverbed-apm-agent:12.25.0.512
        image: riverbed/riverbed-operator:1.0.0-15


```
Change the image identifiers as appropriate to the following:

```
        - name: RVBD_JAVA_INSTRUMENTATION_IMAGE
          value: zeus.run/agent/riverbed-java:12.25.0.512
        - name: RVBD_DOTNET_INSTRUMENTATION_IMAGE
          value: zeus.run/agent/riverbed-dotnet:12.25.0.512
        - name: RVBD_APM_AGENT_IMAGE
          value: zeus.run/agent/sidecar:12.25.0.512
        image: zeus.run/riverbed/riverbed-operator:1.0.0-15

```

# Configure the Riverbed Operator


The Customer ID and Analysis Server Host will need to be configured for the APM Agent. Additional configuration may be required (outlined below)


```
kubectl create -f https://raw.githubusercontent.com/rvbd-operator/beta/1.0.0/riverbed_configuration_betav1.yaml --namespace=riverbed-operator --edit
```

Under the ‘spec’ section of the file:
- update the customerId to your Customer ID
- update the analysisServerHost to your Analysis Server Host identifier
- update the configName to specify the default process configuration name used by your instrumented applications

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

# Configuring instrumentation for Java and .NET apps
Instrumentation is configured by adding annotations to the application pod or namespace. add an annotation to a pod to enable injection. The annotation can be added to a namespace, so that all pods within that namespace will get instrumentation, or by adding the annotation to individual PodSpec objects, available as part of Deployment, Statefulset, and other resources.

Annotations applied to the application deployment override annotations to the namepace.  
If the application is annotated and the application is running, it will automatically restart.  If the annotation is applied to the namespace, 
you will need restart the application for the instrumentation configuration to take effect.


| Annotation                      | Values                        | Defaults            | Description                |
|---------------------------------|-------------------------------|---------------------|----------------------------|
| instrument.apm.riverbed/runtime | "linux-musl64" or "linux-x64" | "linux-x64"         | Runtime environment        |
| instrument.apm.riverbed/inject-java | "true" or "false"             | "false"             | For Java instrumentation   |
| instrument.apm.riverbed/inject-dotnet | "true" or "false"             | "false"             | For Dotnet instrumentation |
| instrument.apm.riverbed/configName | "configuration Name           | operator configName | Process Configuration Name |


# Examples instrumentation patching for Java and .NET apps
**Configuring Alpine (linux-musl64) applications**
If auto-instrumenting alpine applications(dotnet or java), you must annotate the application deployment with the `linux-musl-x64` runtime information:
```
kubectl patch deployment <application-deployment-name> -p '{"spec": {"template":{"metadata":{"annotations":{"instrument.apm.riverbed/runtime":"linux-musl-x64"}}}} }'
```

**Update java application-deployment-name**
To auto-instrument java applications,  you need to add the following annotation to either namespace or the application deployment
``
instrument.apm.riverbed/inject-java":"true"
``
To annotate the application deployment:
```
kubectl patch deployment <application-deployment-name> -p '{"spec": {"template":{"metadata":{"annotations":{"instrument.apm.riverbed/inject-java":"true"}}}} }'
```
To annotate all applications in a namespace:

```
kubectl annotate namespace <application-namespace> instrument.apm.riverbed/inject-java=true
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

# Deploying sample applications :

***Create an imagePullSecret***

Create a secret named regcred to access Docker image is from a Zeus private repository, a secret needs to be created for pulling the image . To create the secret, enter the following command, where <your-registry-server> is zeus-run, <your-name> is your Docker username, <your-pword> is your Docker password, <your-email> is your Docker email.   and <your-namespace> is the namespace where you intend to install the agent (ie riverbed)  You can obtain docker credentials from https://zeus.run/#account:

```
kubectl create secret docker-registry regcred -n riverbed-operator --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```
for example:
```
kubectl create secret docker-registry regcred -n riverbed-operator --docker-server=zeus.run --docker-username=rmurray --docker-password=xxxxxxxxx --docker-email=rmurray@riverbed.com

```

***Additional configuration when using a private image registry***

Add image pull secret to riverbed-apm-agent service account
```
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "regcred"}]}'``
```

**Deploy tomcat**
```
kubectl apply -f https://raw.githubusercontent.com/rvbd-operator/beta/1.0.0/tomcat.yaml
```

**Deploy freshbrew (dotnet)**
```
kubectl apply -f https://raw.githubusercontent.com/rvbd-operator/beta/1.0.0/freshbrew-autoinst.yaml

```