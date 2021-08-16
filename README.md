# Kubernetes External Secrets

[Kubernetes External Secrets](https://github.com/external-secrets/kubernetes-external-secrets)

If using AWS Secrets Manager as the backing service for your secrets, you will need to create an IAM user that has read/write access to this service.  You can then store these credentials as a Secret in the namespace where you will install External Secrets:

```
apiVersion: v1
kind: Secret
metadata:
  name: aws-secret-manager-credentials
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: someAwsAccessKeyId
  AWS_SECRET_ACCESS_KEY: someAwsSecretAcessKey
```

Next, create a new project for external secrets, create the IAM user credentials secret in this namespace, then add and install the Helm chart:

```
oc new-project external-secrets
oc apply -f aws-secret-manager-credentials-secret.yaml

helm repo add external-secrets https://external-secrets.github.io/kubernetes-external-secrets/
helm install external-secrets external-secrets/kubernetes-external-secrets -f subchart/values.yaml
```

See `subchart/values.yaml` for an example values file.

You should be all set!  Time to create some secrets.

To create a new secret in AWS Secrets Manager:

```
echo '{"username": "pittar", "password": "notarealpassword", "email": "pittar@example.com"}' | base64
eyJ1c2VybmFtZSI6ICJwaXR0YXIiLCAicGFzc3dvcmQiOiAibm90YXJlYWxwYXNzd29yZCIsICJlbWFpbCI6ICJwaXR0YXJAZXhhbXBsZS5jb20ifQo=

aws secretsmanager create-secret --name externalsecrets/test1 --description "First test." --secret-binary eyJ1c2VybmFtZSI6ICJwaXR0YXIiLCAicGFzc3dvcmQiOiAibm90YXJlYWxwYXNzd29yZCIsICJlbWFpbCI6ICJwaXR0YXJAZXhhbXBsZS5jb20ifQo= --profile secrets --region ca-central-1
```

An example "External Secrets" CRD that will create a k8s `Secret` based on the values stored in Secrets Manager:

```
apiVersion: kubernetes-client.io/v1
kind: ExternalSecret
metadata:
  name: mytestsecret
spec:
  backendType: secretsManager
  controllerId: external-secrets
  data:
    - key: externalsecrets/test1
      name: password
      property: password
    - key: externalsecrets/test1
      name: username
      property: username
    - key: externalsecrets/test1
      name: email
      property: email
  template:
    stringData:
      spring.datasource.url: 'username:<%= data.username %>,password:<%= data.password %>'
```

Then to have this created as an actual k8s `Secret` in your namespace:

```
oc apply -f externalsecrets-secret.yaml -n mynamespace
```

After this is complete, check for a new `Secret` in your namespace.  You should have a secret by the same name as the ExternalSecret with the values/labels/annotations specified:

```
oc get secret mytestsecret -o yaml
```

Next steps:

* Figure out how to deploy this with Argo CD.