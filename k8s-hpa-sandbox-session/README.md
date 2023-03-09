<h1>Patrick's HPA sandbox session</h1>

Many customers come to us with HPA tickets. Public docs on this can be quite confusing (trust me I know) which is the reason for this session :D 

Following the public docs on HPA: https://docs.datadoghq.com/containers/cluster_agent/external_metrics/?tab=helm#overview

Note that in this session, this is in the assumption that you have Helm, K8s, and Minikube installed already.

# Preliminary steps

Please start up docker and run:
```
minikube start
```
Run kubectl get pods to make sure the minikube cluster is working properly.

Make sure you have a Helm chart values.yaml to work with, if not you can grab it from: https://github.com/DataDog/helm-charts/blob/main/charts/datadog/values.yaml

Ensure in your helm chart that this is set to enable HPA (which should be by default so you should not need to make any changes):
```
clusterAgent:
  enabled: true
  metricsProvider:
    enabled: true
    # clusterAgent.metricsProvider.useDatadogMetrics
    # Enable usage of DatadogMetric CRD to autoscale on arbitrary Datadog queries
    useDatadogMetrics: true
```

After that, if you haven't deployed the cluster agent with your helm chart at all please run:
```
helm install <RELEASE_NAME> -f values.yaml --set datadog.site='datadoghq.com' --set datadog.apiKey=<API_KEY> --set datadog.appKey=<APP_KEY> datadog/datadog
```
Run kubectl get pods again to ensure that you have the necessary cluster and node agent pod in your cluster.

Your app key is required for this to work, this is either set in the values.yaml file itself or passed in from the helm install command above.

If you haven't configured your app key but you have done a helm install, you can run a helm upgrade as such:
```
helm upgrade <RELEASE_NAME> -f values.yaml --set datadog.appKey=<APP_KEY> datadog/datadog
```

# Step 1 - Verify and use a sample metric coming from your k8s minikube cluster
Verify that you are getting metrics from your cluster in your sandbox account. Must use an existing metric in your sandbox. In my case I am using kubernetes.pods.running

An HPA manifest and DatadogMetric manifest needs to be created.

# Step 2 - Creating the HPA manifest
Following this template below according to the docs: https://docs.datadoghq.com/containers/cluster_agent/external_metrics/?tab=helm#example-datadogmetric-object
```
  metrics:
    - type: External
      external:
        metricName: "datadogmetric@<namespace>:<datadogmetric_name>"
        targetAverageValue: #required
```

The targetAverageValue is required but for the purposes of the session, this doesn't need to be a specific value.

In metricName set the namespace to default since we are testing this in the default namespace.

In metricName set the datadogmetric_name to match the name set in DatadogMetric manifest as shown below.

Create the HPA manifest by naming it HPA.yaml:
```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kubetest
spec:
  minReplicas: 1
  maxReplicas: 3
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  metrics:
  - type: External
    external:
      metric:
        name: datadogmetric@default:test-metric
      target:
        type: AverageValue
        averageValue: 9
```

# Step 3 - Creating the DatadogMetric manifest
Create the DatadogMetric manifest by naming it datadogmetric.yaml:
```
apiVersion: datadoghq.com/v1alpha1
kind: DatadogMetric
metadata:
  name: test-metric
spec:
  query: avg:kubernetes.pods.running{*}
```

Note that the query above was obtained from the metric explorer. Go to the metrics explorer in your sandbox account, type in kubernetes.pods.running and click on </>. Then, copypasta the query to the manifest.

# Step 4 - Deploying the two manifests
Now deploy the two manifests by running:
```
kubectl apply -f HPA.yaml
```
```
kubectl apply -f datadogmetric.yaml
```

After deploying these manifests, run the following to confirm that they are deployed successfully:

Run:
```
kubectl get hpa
```
Output you should see:
```
NAME       REFERENCE        TARGETS             MINPODS   MAXPODS   REPLICAS   AGE
kubetest   Deployment/hpa   <unknown>/9 (avg)   1         3         0          30m
```

Run:
```
kubectl get datadogmetric
```
Output you should see:
```
NAME                                                ACTIVE   VALID   VALUE                REFERENCES             UPDATE TIME
test-metric                                         True     True    1.4615384615384615   hpa:default/kubetest   63s
```

# Step 5 - Verify that it works!
Then run a describe on the DatadogMetric:
```
kubectl describe DatadogMetric test-metric
```

The status section of the output should be as follows:
```
Status:
  Autoscaler References:  hpa:default/kubetest
  Conditions:
    Last Transition Time:  2022-11-29T21:03:53Z
    Last Update Time:      2022-11-29T21:25:32Z
    Status:                True
    Type:                  Active
    Last Transition Time:  2022-11-29T21:04:12Z
    Last Update Time:      2022-11-29T21:25:32Z
    Status:                True
    Type:                  Valid
    Last Transition Time:  2022-11-29T21:03:53Z
    Last Update Time:      2022-11-29T21:25:32Z
    Status:                True
    Type:                  Updated
    Last Transition Time:  2022-11-29T21:03:53Z
    Last Update Time:      2022-11-29T21:25:32Z
    Status:                False
    Type:                  Error
  Current Value:           1.4615384615384615
Events:                    <none>
```
The error status should be false and the rest should be true.

Also Current Value should not be 0 (unless the metric query itself is only reporting 0).

If the above is satisfied, you have successfully enabled the Custom Metrics Server for HPA!

# Troubleshooting

Common errors:
```
Status:
  Autoscaler References:  hpa:default/kubetest
  Conditions:
    Last Transition Time:  2022-11-29T21:03:53Z
    Last Update Time:      2022-11-30T00:46:13Z
    Status:                True
    Type:                  Active
    Last Transition Time:  2022-11-30T00:46:13Z
    Last Update Time:      2022-11-30T00:46:13Z
    Status:                False
    Type:                  Valid
    Last Transition Time:  2022-11-29T21:03:53Z
    Last Update Time:      2022-11-30T00:46:13Z
    Status:                True
    Type:                  Updated
    Last Transition Time:  2022-11-30T00:46:13Z
    Last Update Time:      2022-11-30T00:46:13Z
    Message:               Invalid metric (from backend), query: avg:kubernetes.pods.running{*}
    Reason:                Unable to fetch data from Datadog
    Status:                True
    Type:                  Error
  Current Value:           0
```

If you see that the cluster agent is throwing an error with multiple queries specified where there is an error parsing the query:
```
2022-12-02 00:25:09 UTC | CLUSTER | ERROR | (pkg/util/kubernetes/autoscalers/datadogexternal.go:79 in queryDatadogExternal) | Error while executing metric query avg:custom.googleapis.com|rabbitmq_queue_messages|gauge{metric.queue:preprocessor_tasks}.rollup(30),avg:monitoring.googleapis.com|rabbitmq_queue{*}.rollup(30),sum:gcp.prometheus.rabbitmq_queue_messages.gauge{queue:preprocessor_tasks}: API returned error: Error parsing query: 
unable to parse avg:custom.googleapis.com|rabbitmq_queue_messages|gauge{metric.queue:preprocessor_tasks}.rollup(30),avg:monitoring.googleapis.com|rabbitmq_queue{*}.rollup(30),sum:gcp.prometheus.rabbitmq_queue_messages.gauge{queue:preprocessor_tasks}: Rule 'scope_expr' didn't match at '|rabbitmq_queue_mess' (line 1, column 26).
```
...It's also good to check if the customer might have multiple DatadogMetric manifests setup in the same cluster that may have invalid queries. You can have them run:
```
kubectl get datadogmetric --all-namespace
```

Useful commands for troubleshooting:
```
kubectl get pods
```

Then run:
```
kubectl logs <CLUSTER_AGENT_POD_NAME>
```
You should be able to see some agent logs error messages. The command above is very helpful to check if you might have configured something incorrectly.