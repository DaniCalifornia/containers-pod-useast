# Kubernetes Secrets

### Prerequisites:
- [minikube installed](https://datadoghq.atlassian.net/wiki/spaces/TS/pages/1248530082/How+to+test+Kubernetes+yourself)

### Objective
Understand how to set up Kubernetes secretes and use these secrets in the datadog agent. This sandbox covers two methods of providing secrets to the Datadog agent:
1. File
2. Existing Secrets

### Steps
For this sandbox, we will create a secret and use this secret in a redis integration. This integration is configured through annotations in the ``redis.yaml`` file. Let's create the secret first:

Encode your secret in base64:
``echo -n 'secretPassword' | openssl base64``

Add the base64 encoded password in the secret.yaml and apply it:
``kubectl apply -f secret.yaml``
Confirm that this secret has been created by running the following commands:
```
kubectl get secret
redis-secret                    Opaque               1      21d

kubectl describe secret redis-secret
Name:         redis-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
redis_password:  5 bytes
```

Now that the secret has been created, deploy the agents with the helm chart:
``helm install datadog -f values.yaml datadog/datadog``


## Reading Secret From File
Uncomment the following annotations from redis.yaml:
```
    ad.datadoghq.com/redis.instances: |
      [
        {
          "host": "%%host%%",
          "port":"6379",
          "password":"ENC[file@/etc/secret-volume/redis_password]"
        }
      ]      
```

Deploy redis pod with these annotations applied:
``kubectl apply -f redis.yaml``

This creates a redis pod which the agent will discover and attempt to run the redis integration using the secret password. The syntax for reading from a file is the following:
``ENC[file@/path/to/file]``
In the ``values.yaml``, we provided this file containing this secret as a volume so it is available for the agent:
```
agents:
  volumes:
    - name: secret-volume
      secret:
        secretName: redis-secret
  volumeMounts:
  - name: secret-volume
    mountPath: /etc/secret-volume
```


#### Confirm that Secret is Decrypted:
To check the secrets that the agent has decrypted, you can run the following command:
``kubectl exec -it AGENT_POD agent secret``
```
=== Secrets stats ===
Number of secrets decrypted: 1
Secrets handle decrypted:
- file@/etc/secret-volume/redis_password: from redisdb
```

Looking at the configcheck, you can also see the hidden secret:
``kubectl exec -it AGENT_POD agent configcheck``

```
=== redisdb check ===
Configuration provider: kubernetes-container-allinone
Configuration source: container:docker://778f508b40bcf3993c47895c11f3202e8db4c9b57943d8088032323d73ad9122
Instance ID: redisdb:1f73270ba8f91102
host: 172.17.0.6
password: "********"
```


## Reading Secret From Existing Secret
Uncomment the following annotations from redis.yaml:
```
    ad.datadoghq.com/redis.instances: |
      [
        {
          "host": "%%host%%",
          "port":"6379",
          "password":"ENC[k8s_secret@default/redis-secret/redis_password]"
        }
      ]      
```
And comment out the following annotations:
```
    ad.datadoghq.com/redis.instances: |
      [
        {
          "host": "%%host%%",
          "port":"6379",
          "password":"ENC[file@/etc/secret-volume/redis_password]"
        }
      ]      
```

Redeploy the redis pod with the updates
``kubectl apply -f redis.yaml``
This will redeploy the redis pod with updated annotations to read a secret from an existing Kubernetes secret instead of a secret file. This existing secret should have been created earlier with this command:
``kubectl apply -f secret.yaml``

The agent will discover this updated pod and decrypt the secret.

#### Confirm that Secret is Decrypted:
``kubectl exec -it AGENT_POD agent secret``
```
=== Secrets stats ===
Number of secrets decrypted: 1
Secrets handle decrypted:
- k8s_secret@default/redis-secret/redis_password: from redisdb
```

### Documentation Referenced:
https://docs.datadoghq.com/agent/guide/secrets-management/?tab=linux#script-for-reading-from-multiple-secret-providers