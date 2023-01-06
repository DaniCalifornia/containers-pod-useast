# Operator-sandbox
Operator Sandbox

### Installation
This page provides instructions on installing the Datadog Agent in a Kubernetes environment through Datadog Operator

### Minimum Agent and Cluster Agent versions
Some features related to later Kubernetes versions require a minimum Datadog Agent version.

- KUBERNETES VERSION 1.16.0+ | 1.21.0+	
- AGENT VERSION	7.19.0+ | 7.36.0+	
- CLUSTER AGENT VERSION	1.9.0+ | 1.20.0+
- REASON Kubelet metrics deprecation | Kubernetes resource deprecation
	            	      	                               	             
The Datadog Operator is in public beta. If you have any feedback or questions, contact Datadog support.
The Datadog Operator is a way to deploy the Datadog Agent on Kubernetes and OpenShift. It reports deployment status, health, and errors in its Custom Resource status, and it limits the risk of misconfiguration thanks to higher-level configuration options.

### Prerequisites
Using the Datadog Operator requires the following prerequisites:
-Kubernetes Cluster version >= v1.14.X: Tests were done on versions >= 1.14.0. Still, it should work on versions >= v1.11.0. For earlier versions, because of limited CRD support, the Operator may not work as expected.
-Helm for deploying the datadog-operator.
-Kubectl CLI for installing the datadog-agent.

### Deploy an Agent with the Operator
To deploy the Datadog Agent with the operator in the minimum number of steps, see the datadog-operator Helm chart. Here are the steps:

### Install the Datadog Operator: https://artifacthub.io/packages/helm/datadog/datadog-operator
`helm repo add datadog https://helm.datadoghq.com`
`helm install my-datadog-operator datadog/datadog-operator`

### Create a Kubernetes secret with your API and app keys
`kubectl create secret generic datadog-secret --from-literal api-key=<DATADOG_API_KEY> --from-literal app-key=<DATADOG_APP_KEY>`
Replace <DATADOG_API_KEY> and <DATADOG_APP_KEY> with your Datadog API and application keys

### Create a file with the spec of your Datadog Agent deployment configuration. The simplest configuration is as follows:

``` 

apiVersion: datadoghq.com/v1alpha1
kind: DatadogAgent
metadata:
  name: datadog
spec:
  credentials:
    apiSecret:
      secretName: datadog-secret
      keyName: api-key
    appSecret:
      secretName: datadog-secret
      keyName: app-key
  agent:
    image:
      name: "gcr.io/datadoghq/agent:latest"
  clusterAgent:
    image:
      name: "gcr.io/datadoghq/cluster-agent:latest" 
```

Deploy the Datadog Agent with the above configuration file:

`kubectl apply -f /path/to/your/datadog-agent.yaml`

### Cleanup
The following command deletes all the Kubernetes resources created by the above instructions:
`kubectl delete datadogagent datadog`
`helm delete my-datadog-operator`

For further details on setting up Operator, including information about using tolerations, refer to the Datadog Operator advanced setup guide:https://docs.datadoghq.com/agent/guide/operator-advanced/

### Unprivileged
(Optional) To run an unprivileged installation, add the following to the Datadog custom resource (CR):

```
agent:
  config:
    securityContext:
      runAsUser: <USER_ID>
      supplementalGroups:
        - <DOCKER_GROUP_ID>
``` 
where <USER_ID> is the UID to run the agent and <DOCKER_GROUP_ID> is the group ID owning the Docker or containerd socket.

### NOTES:
- If you see error `CrashLoopBackOff` - run `kubectl exec -it <agent-pod-name> agent status`
and check if API key is correct!
- Kubernetes 1.25 version -> 1.24 version - Minikube version has to match Kubernetes version:
For ex: You can use two different profiles for this, like "old" (with 1.15) and "new" (with 1.20)
`minikube start --profile old --kubernetes-version=v1.15.0`
`minikube start --profile new --kubernetes-version=v1.20.0`
