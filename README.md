# KCD Spain 2024 - Examples
Examples used for KCD Spain 2024 - Serverless Platforms on K8s


## Openfunction.dev

Install [Openfunction on a cluster](https://openfunction.dev/docs/getting-started/installation/)

```
helm repo add openfunction https://openfunction.github.io/charts/
helm repo update
kubectl create namespace openfunction
helm install openfunction openfunction/openfunction -n openfunction
```

Verify: 

```
kubectl get po -n openfunction
```

### Create your first function

Before you start, to be able to use Openfunction in its full glory, you need to [set up a bunch of stuff](https://openfunction.dev/docs/getting-started/quickstarts/prerequisites/).

Create a secret so Openfunction can push images to your Docker Hub account, or any other registry:

```
REGISTRY_SERVER=https://index.docker.io/v1/
REGISTRY_USER=<your_registry_user>
REGISTRY_PASSWORD=<your_registry_password>
kubectl create secret docker-registry push-secret \
 --docker-server=$REGISTRY_SERVER \
 --docker-username=$REGISTRY_USER \
 --docker-password=$REGISTRY_PASSWORD
```

I will be using this repo, that is public, to host my functions code. If you have private repos, you need to set the credentials to access those too. 

Then we can start by creating our [first Sync function](https://openfunction.dev/docs/getting-started/quickstarts/sync-functions/).


```
apiVersion: core.openfunction.io/v1beta2
kind: Function
metadata:
  name: function-sample
spec:
  version: "v2.0.0"
  image: "salaboy/go-func:v1.0.0"
  imageCredentials:
    name: push-secret
  build:
    builder: openfunction/builder-go:latest
    env:
      FUNC_NAME: "HelloWorld"
      FUNC_CLEAR_SOURCE: "true"
      # # Use FUNC_GOPROXY to set the goproxy if failed to fetch go modules
      # FUNC_GOPROXY: "https://goproxy.cn"
    srcRepo:
      url: "https://github.com/salaboy/kcd-spain-2024.git"
      sourceSubPath: "functions/go"
      revision: "main"
  serving:
    template:
      containers:
        - name: function # DO NOT change this
          imagePullPolicy: IfNotPresent 
    triggers:
      http:
        port: 8080
```

Let's apply this function to the cluster: 

```
kubectl apply -f functions/go/function.yaml
```

Check the function state: 

```
kubectl get function
```

Send a request to the function: 

```
kubectl get function function-sample -o=jsonpath='{.status.addresses}'
```

But.. none of those addresses are accessible with further configurations :( 

Because we know that Openfunction is using Knative Serving we can get the address:

```
kubectl get ksvc
```

This will show us the Knative Service created for the function: 

```
NAME                       URL                                                              LATESTCREATED                   LATESTREADY                     READY   REASON
serving-4skvc-ksvc-k5gzk   http://serving-4skvc-ksvc-k5gzk.default.34.91.134.132.sslip.io   serving-4skvc-ksvc-k5gzk-v200   serving-4skvc-ksvc-k5gzk-v200   True    

```

Then we can curl the function to the public address: 

```
curl  http://serving-4skvc-ksvc-k5gzk.default.34.91.134.132.sslip.io/KCDSpain
```

## Challenges

