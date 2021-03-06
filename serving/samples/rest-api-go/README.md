# Creating a RESTful service

A simple RESTful service for testing purposes. It exposes an endpoint, which takes
a stock ticker (stock symbol), then outputs the stock price. It uses the the REST resource
name from environment defined in configuration.

## Prerequisites

1. [Install Knative Serving](https://github.com/knative/docs/blob/master/install/README.md)
1. Install [Docker](https://www.docker.com/)

## Setup

Build the app container and publish it to your registry of choice:

```shell
REPO="gcr.io/<your-project-here>"

# Build and publish the container, run from the root directory.
docker build \
  --tag "${REPO}/serving/samples/rest-api-go" \
  --file=serving/samples/rest-api-go/Dockerfile .
docker push "${REPO}/serving/samples/rest-api-go"

# Replace the image reference with our published image.
perl -pi -e "s@github.com/knative/docs/serving/samples/rest-api-go@${REPO}/serving/samples/rest-api-go@g" serving/samples/rest-api-go/*.yaml

# Deploy the Knative Serving sample
kubectl apply -f serving/samples/rest-api-go/sample.yaml
```

## Exploring

Once deployed, you can inspect the create resources with the `kubectl` commands:

```shell
# This will show the route that we created:
kubectl get route -o yaml
```

```shell
# This will show the configuration that we created:
kubectl get configurations -o yaml
```

```shell
# This will show the Revision that was created by our configuration:
kubectl get revisions -o yaml

```

To access this service via `curl`, you need to determine its ingress address:

```shell
watch get svc knative-ingressgateway -n istio-system
```

When the service is ready, you'll see an IP address in the EXTERNAL-IP field:

```
NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                                      AGE
knative-ingressgateway   LoadBalancer   10.23.247.74   35.203.155.229   80:32380/TCP,443:32390/TCP,32400:32400/TCP   2d
```

Once the `ADDRESS` gets assigned to the cluster, you can run:

```shell
# Put the host name into an environment variable.
export SERVICE_HOST=`kubectl get route stock-route-example -o jsonpath="{.status.domain}"`

# Put the ingress IP into an environment variable.
export SERVICE_IP=`kubectl get svc knative-ingressgateway -n istio-system -o jsonpath="{.status.loadBalancer.ingress[*].ip}"`
```

If your cluster is running outside a cloud provider (for example on Minikube),
your services will never get an external IP address. In that case, use the istio `hostIP` and `nodePort` as the service IP:

```shell
export SERVICE_IP=$(kubectl get po -l knative=ingressgateway -n istio-system -o 'jsonpath={.items[0].status.hostIP}'):$(kubectl get svc knative-ingressgateway -n istio-system -o 'jsonpath={.spec.ports[?(@.port==80)].nodePort}')
```

Now use curl with the service IP as if DNS were properly configured:

```shell
curl --header "Host:$SERVICE_HOST" http://${SERVICE_IP}
# Welcome to the stock app!
```

```shell
curl --header "Host:$SERVICE_HOST" http://${SERVICE_IP}/stock
# stock ticker not found!, require /stock/{ticker}
```

```shell
curl --header "Host:$SERVICE_HOST" http://${SERVICE_IP}/stock/<ticker>
# stock price for ticker <ticker>  is  <price>
```

## Updating

You can update this app to a new version. For example, update it with a new `configuration.yaml` via:

```shell
kubectl apply -f serving/samples/rest-api-go/updated_configuration.yaml
```

Once deployed, traffic will shift to the new revision automatically. You can verify the new version
by checking the route status:

```shell
# This will show the route that we created:
kubectl get route -o yaml
```

Or, you run the curl command with the service host and IP:

```shell
curl --header "Host:$SERVICE_HOST" http://${SERVICE_IP}
# Welcome to the share app!
```

```shell
curl --header "Host:$SERVICE_HOST" http://${SERVICE_IP}/share
# share ticker not found!, require /share/{ticker}
```

```shell
curl --header "Host:$SERVICE_HOST" http://${SERVICE_IP}/share/<ticker>
# share price for ticker <ticker>  is  <price>
```

## Manual traffic splitting

You can manually split traffic to specific revisions. Get your revisions names via:

```shell

kubectl get revisions
```

```
NAME                                AGE
stock-configuration-example-00001   11m
stock-configuration-example-00002   4m
```

Update `traffic` part in [serving/samples/rest-api-go/sample.yaml](./sample.yaml) as:

```yaml
traffic:
  - revisionName: <YOUR_FIRST_REVISION_NAME>
    percent: 50
  - revisionName: <YOUR_SECOND_REVISION_NAME>
    percent: 50
```

Then, update your change via:

```shell
kubectl apply -f serving/samples/rest-api-go/sample.yaml
```

Once updated, you can verify the traffic splitting by looking at the route status or running 
the curl command as before.

## Cleaning up

To clean up the sample service:

```shell
kubectl delete -f serving/samples/rest-api-go/sample.yaml
```
