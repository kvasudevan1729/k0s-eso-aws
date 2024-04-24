# Motivation

This repository demonstrates how to dynamically source secrets from an
external source (e.g. AWS SecretsManager) and inject it into the environment
of a pod using the External Secrets Operator ([ESO](https://github.com/external-secrets/external-secrets)).
This is beneficial for many reasons:
- We remove any intermediate resource (person/system) to store/populate secrets,
  thus minimising secret leaks.
- Secret value changes are propagated automatically.
- By using a secret backend in cloud, one avoids running a vault service in
  your environment (a vault service will need to be redundant, possibly backed
  by HSM and so on, so not trivial).
- ESO supports a number of backends (AWS, GCP, Vault etc.) for use in different
  types of scenarios.

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

Ensure the pods are running in the `external-secrets` namespace:
```
kubectl get pods -n external-secrets
NAME                                                READY   STATUS    ...
external-secrets-cert-controller-658ddcb44b-tl85d   1/1     Running   ...
external-secrets-d7857bc5d-2shtd                    1/1     Running   ...
external-secrets-webhook-54b7b9d4df-5vp2q           1/1     Running   ...
```

## Store AWS keys as Secret

Now create a generic secret in your namespace to store your AWS credentials.
This is what allows the eso to authenticate against AWS. This key needs
to have the necessary IAM privileges to read/write secrets to
AWS SecretsManager and possibly AWS KMS if you use a custom key for
encryption/decryption.

```
echo -n '<access key>' > ./access-key
# write '<secret key>' to ./secret-key, ensure there are *NO* newline characters.
kubectl create secret generic awssm-secret \
  --from-file=./access-key --from-file=./secret-key
rm -f ./access-key ./secret-key
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

To prove that external secrets work, we are first going to create a secret
in AWS secret manager (key-value format) and then sync that secret into
our kubernetes environment. 

```
aws secretsmanager create-secret --name redis-creds \
  --secret-string '{"username":"redis-admin","password":"redis.pass"}' \
  --region eu-west-1
```

Now create an ExternalSecret object pointing to our above secret:

```
kubectl apply -f secrets/redis-creds.yaml
kubectl get externalsecrets
```
We should see our externalsecrets resource `redis-creds` in `SecretSynced` 
state and ready to be true.

## Setup test pod

Now setup a test pod that references our redis secret, and then inspect its
environment to validate if the credentials match with what we stored in AWS
SecretsManager.

```
kubectl apply -f apps/test-eso.yaml
kubectl logs test-eso | grep -E 'username|password'
```

## Troubleshooting

If the secrets are not being sourced, or there is a sync error, please
check the controller logs:
```
kubectl logs external-secrets-cert-controller-658ddcb44b-tl85d -n external-secrets
```

## References

1. https://www.giantswarm.io/blog/manage-kubernetes-secrets-using-aws-secrets-manager
2. https://external-secrets.io/latest/
