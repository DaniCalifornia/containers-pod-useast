# Kubernetes Secrets

### Prerequisites:
- minikube installed

### Documentation Referenced:
https://docs.datadoghq.com/agent/guide/secrets-management/?tab=linux#script-for-reading-from-multiple-secret-providers

### Steps
Encode your secret in base64:
``echo -n 'secretPassword' | openssl base64``

Add the base64 encoded password in the secret.yaml and apply it:
Create the redis-secret:
``kubectl apply -f secret.yaml``

Deploy agents:
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

#### Check Secrets:
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

Redeploy the redis pod with the updates
``kubectl apply -f redis.yaml``

####Check Secrets:
``kubectl exec -it AGENT_POD agent secret``
```
=== Secrets stats ===
Number of secrets decrypted: 1
Secrets handle decrypted:
- k8s_secret@default/redis-secret/redis_password: from redisdb
```