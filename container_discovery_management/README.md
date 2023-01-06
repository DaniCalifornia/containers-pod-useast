# Container Log Exclusion  With Datadog

## Objective

To understand and replicate the process of excluding logs from certain containers in a Kubernetes environment, as described in the Container Discovery Management documentation below:

- https://docs.datadoghq.com/agent/guide/autodiscovery-management/?tab=containerizedagent#pagetitle

![Container Discovery Management 2022-10-28 at 2 31 18 PM](https://user-images.githubusercontent.com/60328238/198711855-f43d7ab1-cc91-40c2-a59d-d09b5e0c0bc1.jpg)


This is distinct from the practice of excluding/including matching logs from certain containers, discussed in our Advanced Logging documentation:

- https://docs.datadoghq.com/agent/logs/advanced_log_collection/?tab=configurationfile#pagetitle

![Advanced Log Collection 2022-10-28 at 2 34 59 PM](https://user-images.githubusercontent.com/60328238/198711744-ad91a93b-d2fe-4c20-bf1e-1b859ccc9ad9.jpg)


## Prerequisites
As a pre-requisite you should install Docker Desktop for Mac, as well as install Minikube and Kubernetes tools needed.

- [Install Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-install/)
- Install command line tools with brew
    ```bash
    brew install minikube
    brew install kubernetes-cli
    brew install helm
    ```
- Add the Datadog Helm Repository
    ```bash
    helm repo add datadog https://helm.datadoghq.com
    helm repo update
    ```

Once this is installed run `minikube start` to initialize your Minikube Kubernetes Cluster. The default parameters of this should be sufficient. Once this is ready you can run the following to validate the setup is ready.

```shell 
$ minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

### Setting up API Key
The following sessions provide a Datadog Agent configuration via Helm. These samples reference a pre-existing Secret storing your Datadog API Key. To create this you can run the following with respect to one of your API Keys:

```
kubectl create secret generic datadog-agent --from-literal='api-key=<API_KEY>'
```

Example:

```
kubectl create secret generic datadog-agent --from-literal='api-key=abcdabcdabcdabcdabcdabcdabcdabcd'
```

When running this make sure you swap in your API Key correctly. You can validate that your API Key exists correctly by running:

```
$ kubectl describe secret datadog-agent
Name:         datadog-agent
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
api-key:  32 bytes
```

This resulting Data should be an `api-key` containing 32 bytes representing the 32 characters of your API Key. If this does not line up, run `kubectl delete secret datadog-agent` and double check the command you ran.

# Exercise

## 1 - Install Your Agent with the Helm Chart
For this exercise, we will be using the Helm chart deployment method.  

Use the `values.yaml` file found in this directory for your Helm configuration. This file uses the `Secret` `datadog-agent` created in the [Prerequisites](#prerequisites) for the API Key. As well as enables the log collection for all containers:

```yaml
datadog:
  apiKeyExistingSecret: datadog-agent
  #(...)
  logs:
    enabled: true
    containerCollectAll: true
``` 

***In the same path where of the `values.yaml`***, run the following command:

```
helm install <RELEASE_NAME> -f values.yaml datadog/datadog
```

*Note:* Replace `<RELEASE_NAME>` with your desired Helm name, for example `datadog` should work.

You will know the Helm chart has been successfully installed and the agent deployed by waiting a moment and running the command `kubectl get pods`, the command to see the number of pods running in your environment and their statuses, similar to below: 

```shell
% kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
datadog-cluster-agent-5dccc74546-2ll6r   1/1     Running   0          88s
datadog-st6k6                            2/2     Running   0          16m
```

*Note:* Two pods beginning with "datadog" should be deployed, with one being the Cluster Agent. BOTH should have a `Running` status and should have matching paired numbers under the "Ready" column. 

***GREAT! You have deployed the Node and Cluster Agents to your environment!***

## 2 - Deploy the Logs Application
This directory also has a `loggers.yaml` file containing the `Deployment` for our desired logging application. In this same directory, run the following command to deploy the application to your minikube cluster: 

```shell
kubectl apply -f loggers.yaml
```

Next, run the `kubectl get pods` command and you should see a new `Pod` appear for `loggers`:

```shell
% kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
datadog-cluster-agent-5dccc74546-2ll6r   0/1     Running   0          2s
datadog-st6k6                            1/2     Running   0          14m
loggers-6746df8595-w4cp9                 3/3     Running   0          22h
```

*Note:* Do not proceed until the new `Pod` has a `Running` status and has matching paired numbers under the "Ready" column similar to the Agent pods. 

## 3 - Confirm Container Logs are Appearing 
Check the Log Explorer in your Datadog sandbox account to confirm that the logs from this `Deployment` arrived. A great "tell" is looking for the logs containing the word "cookies," as one of the containers has logs that will be emitting logs with this word present. Alternatively, you can filter on the `kube_deployment:loggers` tag: 

![image](https://user-images.githubusercontent.com/60328238/206784546-f5220e91-ac92-4e00-8070-bfd61846efff.png)

The `Deployment` we are creating from the [`loggers.yaml`](loggers.yaml) file contains the following 3 containers. The first two containers are using a barebones "busybox" container image and are emitting some sample logs every 5 seconds through the script provided. The third container is using a different image, `chentex/random-logger`, that likewise emits different logs every few seconds.

***loggers.yaml:***
```yaml
    #(...)
    spec:
      containers:
      # Container named "example-logger" outputting date-related logs every 5 seconds
      - name: example-logger
        image: busybox
        imagePullPolicy: Always
        command: [ "/bin/sh", "-c", "--" ]
        args: [ "while true; do sleep 5; 
          echo `date '+%FT%T'` example stdout log; 
          echo `date '+%FT%T'` example stderr log 1>&2;
        done;"
        ]
      # Container named "yummy-logger" outputting hunger-inducing logs every 5 seconds
      - name: yummy-logger
        image: busybox
        imagePullPolicy: Always
        command: [ "/bin/sh", "-c", "--" ]
        args: [ "while true; do sleep 5; 
          echo `date '+%FT%T'` I like cookies and milk on Thursdays; 
          echo `date '+%FT%T'` But only if I have chocolate and bonbons on Fridays;
        done;"
        ]
      # Container named "random-logger" outputting logs every 5 seconds
      - name: random-logger
        image: chentex/random-logger
        imagePullPolicy: Always
```

Note: You are also free to change the log phrases, but ensure that those logs appear on your Logs page. 

## 4 - Implement Container Exclusion 

Reviewing our documentation, you may exclude containers from the Agent Autodiscovery perimeter with an exclude rule based on their `name`, `image`, or `kube_namespace` to collect ***NO DATA*** from these containers. If a container matches an exclude rule, it is not included unless it first matches an include rule. To continue, we can also exclude certain *behaviors* from a container, and not the entire container itself. We will illustrate this now by excluding **logs from certain containers**

![image](https://user-images.githubusercontent.com/60328238/201407644-103284be-7508-4ac5-8a70-7f2ec2544c10.png)

Following the steps from above, we will update our `values.yaml` file to exclude one of the containers from having their logs collected. With respect to Helm we can use `datadog.containerExclude`:

```yaml
datadog:
  #(...)
  containerExclude: image:chentex/random-logger
```

Through Helm this will add the environment variable for us:
```yaml
  env:
    - name: DD_CONTAINER_EXCLUDE
      value: "image:chentex/random-logger"
```

The full `values.yaml` would look like: 
```yaml
targetSystem: "linux"
datadog:
  apiKeyExistingSecret: datadog-agent
  logLevel: DEBUG
  processAgent:
    enabled: false
  kubelet:
    tlsVerify: false
  kubeStateMetricsEnabled: false
  logs:
    enabled: true
    containerCollectAll: true
  containerExclude: image:chentex/random-logger
```

After adding the above in your Helm chart, save your `values.yaml` file and apply the changes by upgrading your Helm chart:

```shell
helm upgrade <RELEASE-NAME> -f values.yaml datadog/datadog
```

## 5 - Confirmation of Exclusion

Run `kubectl get pods` to ensure that your busybox deployment, Node, and Cluster Agents are running in your environment:

![image](https://user-images.githubusercontent.com/60328238/206787591-3f07b177-6acf-431e-b9bb-a1608d84bcaa.png)

Now, notice that in our example we are trying to exclude logs from the `chentex/random-logger` container. Let's go to our log page and filter by facet to only see `image_name:"chentex/random-logger"` logs: 

![image](https://user-images.githubusercontent.com/60328238/206786356-03ee33c2-4b0f-40ce-b5b2-7af20425ec7f.png)

We see that these particular logs stopped after a certain point. Good sign! Let's filter by another container in the `loggers` deployment file (like `service:busybox`):

![image](https://user-images.githubusercontent.com/60328238/206786793-fa4620a9-0037-4f02-abe1-c967d575e5a3.png)

We see that *these* logs are still coming, meaning we've sucessfully excluded logs from a certain container :)

To revert to receiving **all** logs again, simply comment out the exclusion configuration in your values.yaml file and redeploy the Agent. Similar steps can be taken if one would like to exclude other things from containers, such as their metrics. 

## 6 - Removing / Uninstalling the Agent

To remove the Datadog Cluster and Node Agents, you can run the following command: 

```shell
helm uninstall datadog
```

To remove the logger container deployment:

```shell
kubectl delete deployments loggers  
```

# Reference Links

- K8s Installation Steps (Helm), ***Ensure Logs are enabled*** 
  - https://docs.datadoghq.com/containers/kubernetes/installation/?tab=helm
- Container Discovery Setup
  - https://docs.datadoghq.com/agent/guide/autodiscovery-management/?tab=containerizedagent#inclusion-and-exclusion-behavior
- Environment Variable list (Where/How to input container images to exclude logs from)
  - https://docs.datadoghq.com/containers/docker/?tab=standard#environment-variables
