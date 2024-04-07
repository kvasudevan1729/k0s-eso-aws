# Motivation

xxx

## Install External Secrets Operator (ESO) chart

```
helm repo add external-secrets https://charts.external-secrets.io
helm repo update external-secrets
helm upgrade --namespace external-secrets --create-namespace --install --wait external-secrets external-secrets/external-secrets
```
The chart creates a bunch of Custom Resource Definitions (CRDs), specifically:
- secretstores
- clustersecretstores
- externalsecrets

Ensure if the pods are running in the `external-secrets` namespace:
```
kubectl get pods -n external-secrets
```

## Store AWS keys as Secret

Now create a generic secret in your namespace to store your AWS credentials.
This is what allows the eso to authenticate against AWS.

```
echo -n '<access key>' > /tmp/access-key
echo -n '<secret key>' > /tmp/secret-key
kubectl create secret generic awssm-secret --from-file=/tmp/access-key --from-file=/tmp/secret-key
```

## ESO store for AWS SecretsManager

Now create a `ClusterSecretStore` using the above secret `awssm-secret` (note
the provider in the manifest).

```
kubectl apply -f secrets/eso-secret-store.yaml
```
You can check the status by running:
```
kubectl get clustersecretstores
```

## Create AWS secret

Create AWS secret in key-value format. For e.g.: 

```
aws secretsmanager create-secret --name redis-creds --secret-string '{"username":"redis-admin","password":"redis.pass"}' --region eu-west-1
```

Now create an ExternalSecret object pointing to our above secret:

```
kubectl apply -f secrets/redis-creds.yaml
kubectl get externalsecrets
```

## Setup test pod

```
kubectl apply -f apps/test-eso.yaml
kubectl logs test-eso | grep -E 'username|password'
```

## Troubleshooting

To check the logs:
```
kubectl logs external-secrets-cert-controller-658ddcb44b-tl85d -n external-secrets
```

## References

1. https://www.giantswarm.io/blog/manage-kubernetes-secrets-using-aws-secrets-manager
2. https://external-secrets.io/latest/