# Private Locations

### Prerequisites:
- [minikube installed](https://datadoghq.atlassian.net/wiki/spaces/TS/pages/1248530082/How+to+test+Kubernetes+yourself)

### Objective
Understand why private locations are used and how to deploy a private location.

### Steps
The public documentation on private locations currently only contains instructions for a docker. However, the datadog site contains instructions for other environments as well.

1. ##### Navigate to the Private Locations Page
   UX Monitoring -> Settings -> [Private Locations](https://app.datadoghq.com/synthetics/settings/private-locations)

2. ##### Creating a New Private Location
- Click the **Add Private Location** button
- Provide a ``name`` and ``API key`` for the private location. 
  *Optional*: Add tags and description
- Set Permissions: limit users who have access to this private location
- Configurations: set proxy URL and any blocked IPs
- A configuration file will be generated in two formats (``bash script`` and ``JSON``)
  - Depending on your environment, you'll need one over the other. In Kubernetes, we will be using the ``JSON`` file.
  - Save the JSON file on your machine. I have chosen to name it ``config.json``

3. ##### Installing the Private Location
There are instructions for Docker, Kubernetes, Amazon ECS, AWS Fargate, and Amazon EKS. Click on the instructinos for Kubernetes.

The first step is to create a secret containing your configurations for the private location. Apply the following command:
``kubectl create secret generic private-location-worker-config --from-file=<FILE_NAME>``
Replace FILE_NAME with the file containing the config. A secret named private-location-worker-config will be created.

Now, the private location can be deployed. A deployment file will be provided and it looks like the following:
```
apiVersion: apps/v1
kind: Deployment
metadata:
    name: datadog-private-location-worker
    namespace: default
spec:
    selector:
        matchLabels:
            app: private-location
    template:
        metadata:
            name: datadog-private-location-worker
            labels:
                app: private-location
        spec:
            containers:
                - name: datadog-private-location-worker
                  image: gcr.io/datadoghq/synthetics-private-location-worker
                  volumeMounts:
                      - mountPath: /etc/datadog/synthetics-check-runner.json
                        name: worker-config
                        subPath: <SECRET_DATA>.json #This should match the name of the json file containing the config
            volumes:
                - name: worker-config
                  secret:
                      secretName: private-location-worker-config #This should match your secret name
```
**Note**: Make sure that the ``volumeMoun.subPath`` is the same as the .json file name in the secret. You can confirm this by describing the secret that was created earlier:
``kubectl describe secret generic private-location-worker-config``

Create the deployment:
``kubectl apply -f private-location-worker-deployment.yaml``

After the deploying, you should see in the UI that the Private Location Status is now reporting.

4. ##### Testing an Internal Endpoint
Let's test an internal endpoint. To test this, I used a basic nginx pod which exposes port 80:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```
Deploy this pod using:
``kubectl apply -f nginx.yaml``
Get the pod ip and curl port 80 to confirm that it is working.
``kubectl exec <AGENT_POD> -- curl <NGINX_POD_IP>:80``
You should see the following request:
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### References:
https://docs.datadoghq.com/getting_started/synthetics/private_location/