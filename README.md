# OpenFaaS - Serverless Functions Made Simple

<img src="https://blog.alexellis.io/content/images/2017/08/faas_side.png" alt="OpenFaaS logo" width="60%">

[OpenFaaS](https://github.com/openfaas/faas) (Functions as a Service) is a framework for building serverless functions with Docker and Kubernetes which has first class support for metrics. Any process can be packaged as a function enabling you to consume a range of web events without repetitive boiler-plate coding.

## Highlights

* Ease of use through UI portal and *one-click* install
* Write functions in any language for Linux or Windows and package in Docker/OCI image format
* Portable - runs on existing hardware or public/private cloud. Native [Kubernetes](https://github.com/openfaas/faas-netes) support, Docker Swarm also available
* [Operator / CRD option available](https://github.com/openfaas/faas-netes/)
* [faas-cli](http://github.com/openfaas/faas-cli) available with stack.yml for creating and managing functions
* Auto-scales according to metrics from Prometheus
* Scales to zero and back again and can be tuned at a per-function level
* Works with service-meshes
    * Tested with [Istio](https://istio.io) including mTLS
    * Tested with [Linkerd2](https://github.com/openfaas-incubator/openfaas-linkerd2) including mTLS and traffic splitting with SMI
* (changes)
## Deploy OpenFaaS

### 1) Install Community Edition (CE) with arkade

It is recommended that you use arkade to install OpenFaaS. arkade is a CLI tool which automates the helm CLI and chart download and installation. The `openfaas` app also has a number of options available via `arkade install openfaas --help`

The installation with arkade is as simple as the following which installs OpenFaaS, sets up an Ingress record, and a TLS certificate with cert-manager.

```bash
arkade install openfaas
arkade install openfaas-ingress \
  --domain openfaas.example.com \
  --email wm@example.com
```

See a complete example here: [Get TLS for OpenFaaS the easy way with arkade](https://blog.alexellis.io/tls-the-easy-way-with-openfaas-and-k3sup/)

If you wish to continue without using arkade, read on for instructions.

### 2) Install with helm

To use the chart, you will need to use Helm 3:

Get it from arkade:

```bash
arkade get helm
```

Or use the Helm installation script:

```bash
curl -sSLf https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

We recommend creating two namespaces, one for the OpenFaaS *core services* and one for the *functions*.

You can skip this step if you're using arkade to install OpenFaaS.

```sh
kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
```

You will now have `openfaas` and `openfaas-fn`. If you want to change the names or to install into multiple installations then edit `namespaces.yml` from the `faas-netes` repo.

Add the OpenFaaS `helm` chart:

```sh
helm repo add openfaas https://openfaas.github.io/faas-netes/
```

Now decide how you want to expose the services and edit the `helm upgrade` command as required.

* To use NodePorts/ClusterIP - (the default and best for development, with port-forwarding)
* To use an IngressController add `--set ingress.enabled=true` (recommended for production, for use with TLS)
* To use a LoadBalancer add `--set serviceType=LoadBalancer` (not recommended, since it will expose plain HTTP)

#### Deploy OpenFaaS Community Edition (CE)

> OpenFaaS Community Edition is meant exploration and development.
> 
> OpenFaaS Pro has been tuned for production use including flexible auto-scaling, high-available deployments, durability, add-on features, and more.

Deploy CE from the helm chart repo directly:

```sh
helm repo update \
 && helm upgrade openfaas \
  --install openfaas/openfaas \
  --namespace openfaas
```

> The above command will also update your helm repo to pull in any new releases.

Retrieve the OpenFaaS credentials with:

```sh
PASSWORD=$(kubectl -n openfaas get secret basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode) && \
echo "OpenFaaS admin password: $PASSWORD"
```

#### Deploy OpenFaaS Pro

* Create the required secret with your [OpenFaaS Pro license](https://www.openfaas.com/pricing/):

```bash
kubectl create secret generic \
  -n openfaas \
  openfaas-license \
  --from-file license=$HOME/.openfaas/LICENSE
```

If you wish to use the OpenFaaS Pro dashboard, [you must run the steps to "Create a signing key"](https://docs.openfaas.com/openfaas-pro/dashboard/#installation) before installing the Helm chart.

Now deploy OpenFaaS from the helm chart repo:

```sh
helm repo update \
 && helm upgrade openfaas \
  --install openfaas/openfaas \
  --namespace openfaas  \
  --set openfaasPro=true
```

The main change here is to add: `--set openfaasPro=true`

For production, we recommend creating your own values.yaml file, but make sure you do not copy any more settings into it than strictly necessary. This way the file can be maintained easily over time.

Example installation with a values.yaml file instead of using `--set`:

```sh
helm repo update \
 && helm upgrade openfaas \
  --install openfaas/openfaas \
  --namespace openfaas  \
  -f values.yaml \
  -f values-pro.yaml
```

You can also review recommended Pro values in [values-pro.yaml](values-pro.yaml)

#### Installing OpenFaaS Pro without Cluster Admin access

In order to install OpenFaaS Pro, you need to create at least one namespace, a Cluster Admin role and Custom Resource Definitions (CRDs), however some DevOps teams prevent business teams from getting access to Cluster Admin.

This option is reserved for OpenFaaS Pro customers, see the installation steps here: [Split installation instructions](https://github.com/openfaas/openfaas-pro/blob/master/split-installation.md)

See also:

* Scale-down to zero (in this document)
* [OpenFaaS Pro SSO/OIDC](https://docs.openfaas.com/openfaas-pro/sso/)
* [OpenFaaS Pro Kafka Event Connector](https://docs.openfaas.com/openfaas-pro/kafka-events/)

## Test changes for the helm chart

If you are working on a patch for the helm chart, you can deploy it directly from a local folder.

You can run the following command from within the `faas-netes` folder, not the chart's folder.

```sh
helm upgrade openfaas \
  --install chart/openfaas \
  --namespace openfaas \
  -f ./chart/openfaas/values.yaml \
  -f ./chart/openfaas/values-pro.yaml
```

In the example above, I'm overlaying two additional YAML files for settings for the chart.

You can override specific images by adding `--set gateway.image=` for instance.

#### Pre-create basic-auth credentials for OpenFaaS Pro/CE

If you're using a GitOps tool like ArgoCD or Flux to install OpenFaaS, then you will need to pre-create the basic-auth credentials, so that they remain stable.

Why? The chart has a pre-install hook which can generate basic-auth credentials. It is enabled by default and can be turned off with `--set generateBasicAuth=false`.

Example command to generate a random password:

```sh
# generate a random password
kubectl -n openfaas create secret generic basic-auth \
--from-literal=basic-auth-user=admin \
--from-literal=basic-auth-password="$PASSWORD"

echo "OpenFaaS admin password: $PASSWORD"
```

#### Tuning function cold-starts

The concept of a cold-start in OpenFaaS only applies if you enable it for a specific function, using [scale to zero](https://docs.openfaas.com/openfaas-pro/scale-to-zero/). Otherwise there is not a cold-start, because at least one replica of your function remains available.

See also: [Fine-tuning the cold-start in OpenFaaS](https://www.openfaas.com/blog/fine-tuning-the-cold-start/)

There are two ways to reduce the Kubernetes cold-start for a pre-pulled image, which is around 1-2 seconds.

1) Don't set the function to scale down to zero, just set it a minimum availability i.e. 1/1 replicas
2) Use async invocations via the `/async-function/<name>` route on the gateway, so that the latency is hidden from the caller
3) Tune the readinessProbes to be aggressively low values. This will reduce the cold-start at the cost of more `kubelet` CPU usage

To achieve a coldstart of between 0.7 and 1.9s, set the following in `values.yaml`:

```yaml
functions:
  imagePullPolicy: "IfNotPresent"
...
  readinessProbe:
    initialDelaySeconds: 0
    timeoutSeconds: 1
    periodSeconds: 1
    successThreshold: 1
    failureThreshold: 3
  livenessProbe:
    initialDelaySeconds: 0
    timeoutSeconds: 1
    periodSeconds: 1
    failureThreshold: 3
```

In addition:

* Pre-pull images on each node
* Use an in-cluster registry to reduce the pull latency for images
* Set the `functions.imagePullPolicy` to `IfNotPresent` so that the `kubelet` only pulls images which are not already available
* Explore alternatives such as not scaling to absolute zero, and using async calls which do not show the cold start

For OpenFaaS CE, both liveness and readiness probes are set to, and the `PullPolicy` for functions is set to `Always`:

* `initialDelaySeconds: 2`
* `periodSeconds: 2`
* `timeoutSeconds: 1`

### Verify the installation

Once all the services are up and running, log into your gateway using the OpenFaaS CLI. This will cache your credentials into your `~/.openfaas/config.yml` file.

Fetch your public IP or NodePort via `kubectl get svc -n openfaas gateway-external -o wide` and set it as an environmental variable as below:

```sh
export OPENFAAS_URL=http://127.0.0.1:31112
```

If using a remote cluster, you can port-forward the gateway to your local machine:

```sh
export OPENFAAS_URL=http://127.0.0.1:8080
kubectl port-forward -n openfaas svc/gateway 8080:8080 &
```

Now log in with the CLI and check connectivity:

```sh
echo -n $PASSWORD | faas-cli login -g $OPENFAAS_URL -u admin --password-stdin

faas-cli version
```

## OpenFaaS Operator and Function CRD

If you would like to work with Function CRDs there is an alternative controller to faas-netes named [OpenFaaS Operator](https://github.com/openfaas-incubator/openfaas-operator) which can be swapped in at deployment time.
The OpenFaaS Operator is suitable for development and testing and may replace the faas-netes controller in the future.
The Operator is compatible with Kubernetes 1.9 or later.

To use it, add the flag: `--set operator.create=true` when installing with Helm.

### faas-netes vs OpenFaaS Operator

The faas-netes controller is the most tested, stable and supported version of the OpenFaaS integration with Kubernetes. In contrast the OpenFaaS Operator is based upon the codebase and features from `faas-netes`, but offers a tighter integration with Kubernetes through [CustomResourceDefinitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/). This means you can type in `kubectl get functions` for instance.

See also: [Introducing the OpenFaaS Operator](https://www.openfaas.com/blog/kubernetes-operator-crd/)

## Deployment with `helm template`

This option is good for those that have issues with or concerns about installing Tiller, the server/cluster component of helm. Using the `helm` CLI, we can pre-render and then apply the templates using `kubectl`.

1. Clone the faas-netes repository
    ```sh
    git clone https://github.com/openfaas/faas-netes.git
    cd faas-netes
    ```

2. Render the chart to a Kubernetes manifest called `openfaas.yaml`

    ```sh
    helm template \
      openfaas chart/openfaas/ \
      --namespace openfaas \
      --set basic_auth=true \
      --set functionNamespace=openfaas-fn > openfaas.yaml
    ```

    You can set the values and overrides just as you would in the install/upgrade commands above.

3. Install the components using `kubectl`

    ```sh
    kubectl apply -f namespaces.yml,openfaas.yaml
    ```

Now [verify your installation](#verify-the-installation).

## Exposing services

### NodePorts

By default a NodePort will be created for the API Gateway.

### Metrics

You temporarily access the Prometheus metrics by using `port-forward`

```sh
kubectl --namespace openfaas port-forward deployment/prometheus 31119:9090
```

Then open `http://localhost:31119` to directly query the OpenFaaS metrics scraped by Prometheus.

### LB

If you're running on a cloud such as AKS or GKE you will need to pass an additional flag of `--set serviceType=LoadBalancer` to tell `helm` to create LoadBalancer objects instead. An alternative to using multiple LoadBalancers is to install an Ingress controller.

### Deploy with an IngressController

In order to make use of automatic ingress settings you will need an IngressController in your cluster such as Traefik or Nginx.

Add `--set ingress.enabled` to enable ingress pass `--set ingress.enabled=true` when running the installation via `helm`.

By default services will be exposed with following hostnames (can be changed, see values.yaml for details):

* `gateway.openfaas.local`

### Endpoint load-balancing

Some configurations in combination with client-side KeepAlive settings may because load to be spread unevenly between replicas of a function. If you experience this, there are three ways to work around it:

* [Install Linkerd2](https://github.com/openfaas-incubator/openfaas-linkerd2) which takes over load-balancing from the Kubernetes L4 Service (recommended)
* Disable KeepAlive in the client-side code (not recommended)
* Configure the gateway to pass invocations through to the faas-netes provider (alternative to using Linkerd2)

    ```sh
    --set gateway.directFunctions=false
    ```

    In this mode, all invocations will pass through the gateway to faas-netes, which will look up endpoint IPs directly from Kubernetes, the additional hop may add some latency, but will do fair load-balancing, even with KeepAlive.

### SSL / TLS

If you require TLS/SSL then please make use of an IngressController. A full guide is provided to [enable TLS for the OpenFaaS Gateway using cert-manager and Let's Encrypt](https://docs.openfaas.com/reference/ssl/kubernetes-with-cert-manager/).

### Service meshes
If you use a service mesh like Linkerd or Istio in your cluster, then you should enable the `directFunctions` mode using:

```sh
--set gateway.directFunctions=true
```

### Istio mTLS

Istio requires OpenFaaS Pro to function correctly.

To install OpenFaaS Pro with Istio mTLS pass `--set istio.mtls=true` and disable the HTTP probes:

```sh
helm upgrade openfaas --install chart/openfaas \
    --namespace openfaas  \
    --set openfaasPro=true \
    --set exposeServices=false \
    --set gateway.directFunctions=true \
    --set gateway.probeFunctions=true \
    --set istio.mtls=true
```

The above command will enable mTLS for the openfaas control plane services and functions excluding NATS.

## Zero scale

### Scale-up from zero (on by default)

Scaling up from zero replicas is enabled by default, to turn it off set `scaleFromZero` to `false` in the helm chart options for the `gateway` component.

```sh
--set gateway.scaleFromZero=true/false
```

### Scale-down to zero (off by default)

Scaling to zero is managed by the OpenFaaS Standard autoscaler, which:

* can save on infrastructure costs
* helps to reduce the attack surface of your application
* mitigates against functions which use high CPU or memory at idle, or which contain leaks

OpenFaaS Pro will only scale down functions which have marked themselves as eligible for this behaviour through the use of a label: `com.openfaas.scale.zero=true`. The time to wait until scaling a specific function to zero is controlled by: `com.openfaas.scale.zero-duration` which is a Go duration set in minutes i.e. `15m`.

See also: [Scale to Zero docs](https://docs.openfaas.com/openfaas-pro/scale-to-zero/).

## Removing the OpenFaaS

All control plane components can be cleaned up with helm:

```sh
helm uninstall -n openfaas openfaas
```

Follow this by the following to remove all other associated objects:

```sh
kubectl delete namespace openfaas openfaas-fn
```

In some cases your additional functions may need to be either deleted before deleting the chart with `faas-cli` or manually deleted using `kubectl delete`.

## Kubernetes versioning

This Helm chart currently supports version 1.16+

Note that OpenFaaS itself may support a wider range of versions, [see here](../../README.md#kubernetes-versions)

## Getting help

Comprehensive documentation is offered in the [OpenFaaS docs](https://docs.openfaas.com/).

For technical support and questions, join the [Weekly Office Hours call](https://docs.openfaas.com/community)

For suspected bugs, feel free to raise an issue. If you do not fill out the whole issue template, or delete the template, it's unlikely that you'll get the help that you're looking for.

## Configuration

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`.

See [values.yaml](./values.yaml) for detailed configuration.

### General parameters

| Parameter               | Description                           | Default                                                    |
| ----------------------- | ----------------------------------    | ---------------------------------------------------------- |
| `affinity`| Global [affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) rules assigned to deployments | `{}` |
yaml) |
| `async` | Enables asynchronous function invocations. If `.nats.external.enabled` is `false`, also deploys NATS | `true` |
| `queueMode` | Set to `jetstream` to run the async system backed by NATS JetStream. By default the async system uses NATS Streaming|
| `basic_auth` | Enable basic authentication on the gateway and Prometheus. Warning: do not disable. | `true` |
| `generateBasicAuth` | Generate admin password for basic authentication | `true` |
| `basicAuthPlugin.image` | Container image used for basic-auth-plugin | See [values.yaml](./values.yaml) |
| `basicAuthPlugin.replicas` | Replicas of the basic-auth-plugin | `1` |
| `basicAuthPlugin.resources` | Resource limits and requests for basic-auth-plugin containers | See [values.yaml](./values.yaml) |
| `caBundleSecretName` | Name of the Kubernetes secret that contains the CA bundle for making HTTP requests for IAM (optional) | `""` |
| `clusterRole` | Use a `ClusterRole` for the Operator or faas-netes. Set to `true` for multiple namespace, pro scaler and CPU/RAM metrics in OpenFaaS REST API | `false` |
| `createCRDs` | Create the CRDs for OpenFaaS Functions and Profiles | `true` |
| `exposeServices` | Expose `NodePorts/LoadBalancer`  | `true` |
| `functionNamespace` | Functions namespace, preferred `openfaas-fn` | `openfaas-fn` |
| `gatewayExternal.annotations` | Annotation for getaway-external service | `{}` |
| `httpProbe` | Setting to true will use HTTP for readiness and liveness probe on the OpenFaaS system Pods (compatible with Istio >= 1.1.5) | `true` |
| `ingress.enabled` | Create ingress resources | `false` |
| `istio.mtls` | Create Istio policies and destination rules to enforce mTLS for OpenFaaS components and functions | `false` |
| `kubernetesDNSDomain` | Domain name of the Kubernetes cluster | `cluster.local` |
| `k8sVersionOverride` | Override kubeVersion for the ingress creation | `""` |
| `nodeSelector` | Global [NodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) | `{}` |
| `openfaasImagePullPolicy` | Image pull policy for openfaas components, can change to `IfNotPresent` in offline env | `Always` |
| `openfaasPro` | Deploy OpenFaaS Pro | `false` |
| `oem` | Deploy OpenFaaS oem | `false` |
| `psp` | Enable [Pod Security Policy](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) for OpenFaaS accounts | `false` |
| `rbac` | Enable RBAC | `true` |
| `securityContext` | Deploy with a `securityContext` set, this can be disabled for use with Istio sidecar injection | `true` |
| `serviceType` | Type of external service to use `NodePort/LoadBalancer` | `NodePort` |
| `tolerations` | Global [Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) | `[]` |

### Gateway

| Parameter               | Description                           | Default                                                    |
| ----------------------- | ----------------------------------    | ---------------------------------------------------------- |
| `gateway.directFunctions` | Invoke functions directly using `Service` without delegating to the provider | `false` |
| `gateway.image` | Container image used for the gateway | See [values.yaml](./values.yaml) |
| `gateway.logsProviderURL` | Set a custom logs provider url | `""` |
| `gateway.maxIdleConns` | Set max idle connections from gateway to functions | `1024` |
| `gateway.maxIdleConnsPerHost` | Set max idle connections from gateway to functions per host | `1024` |
| `gateway.nodePort` | Change the port when creating multiple releases in the same baremetal cluster | `31112` |
| `gateway.probeFunctions` | Set to true for Istio users as a workaround for: https://github.com/openfaas/faas/issues/1721 | `false` |
| `gateway.readTimeout` | Read timeout for the gateway API | `65s` |
| `gateway.replicas` | Replicas of the gateway, pick more than `1` for HA | `1` |
| `gateway.resources` | Resource limits and requests for the gateway containers | See [values.yaml](./values.yaml) |
| `gateway.scaleFromZero` | Enables an intercepting proxy which will scale any function from 0 replicas to the desired amount | `true` |
| `gateway.upstreamTimeout` | Maximum duration of upstream function call, should be lower than `readTimeout`/`writeTimeout` | `60s` |
| `gateway.writeTimeout` | Write timeout for the gateway API | `65s` |
| `gatewayPro.image` | Container image used for the gateway when `openfaasPro=true` | See [values.yaml](./values.yaml) |

### faas-netes / operator

| Parameter               | Description                           | Default                                                    |
| ----------------------- | ----------------------------------    | ---------------------------------------------------------- |
| `faasnetes.image` | Container image used for provider API | See [values.yaml](./values.yaml) |
| `faasnetes.readTimeout` | Read timeout for the faas-netes API | `""` (defaults to gateway.readTimeout)|
| `faasnetes.resources` | Resource limits and requests for faas-netes container | See [values.yaml](./values.yaml) |
| `faasnetes.writeTimeout` | Write timeout for the faas-netes API | `""` (defaults to gateway.writeTimeout) |
| `faasnetesPro.image` | Container image used for faas-netes when `openfaasPro=true` | See [values.yaml](./values.yaml) |
| `faasnetesOem.image` | Container image used for faas-netes when `oem=true` | See [values.yaml](./values.yaml) |
| `faasnetesPro.logs.format` | Set the log format, supports `console` or `json` | `console` |
| `faasnetesPro.logs.debug` | Print debug logs | `false` |
| `operator.create` | Use the OpenFaaS operator CRD controller, default uses faas-netes as the Kubernetes controller | `false` |
| `operator.image` | Container image used for the openfaas-operator | See [values.yaml](./values.yaml) |
| `operator.resources` | Resource limits and requests for openfaas-operator containers | See [values.yaml](./values.yaml) |
| `operator.image` | Container image used for the openfaas-operator | See [values.yaml](./values.yaml) |
| `operator.kubeClientQPS` | QPS rate-limit for the Kubernetes client, (OpenFaaS for Enterprises) | `""` (defaults to 100) |
| `operator.kubeClientBurst` | Burst rate-limit for the Kubernetes client (OpenFaaS for Enterprises) | `""` (defaults to 250) |
| `operator.reconcileWorkers` | Number of reconciliation workers to run to convert Function CRs into Deployments | `1` |
| `operator.logs.format` | Set the log format, supports `console` or `json` | `console` |
| `operator.logs.debug`           | Print debug logs    | `false`                        |
| `operator.leaderElection.enabled`| When set to true, only one replica of the operator within the gateway pod will perform reconciliation | `false` |

### Functions

| Parameter               | Description                           | Default                                                    |
| ----------------------- | ----------------------------------    | ---------------------------------------------------------- |
| `functions.httpProbe` | Use a httpProbe instead of exec | `true` |
| `functions.imagePullPolicy` | Image pull policy for deployed functions (OpenFaaS Pro) | `Always` |
| `functions.livenessProbe.initialDelaySeconds` | Number of seconds after the container has started before [probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) is initiated  | `2` |
| `functions.livenessProbe.periodSeconds` | How often (in seconds) to perform the [probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `2` |
| `functions.livenessProbe.timeoutSeconds` | Number of seconds after which the [probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) times out | `1` |
| `functions.livenessProbe.failureThreshold` | After a [probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) fails failureThreshold times in a row, Kubernetes considers that the overall check has failed. | `3 `|
| `functions.readinessProbe.initialDelaySeconds` | Number of seconds after the container has started before [probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) is initiated | `2` |
| `functions.readinessProbe.periodSeconds` | How often (in seconds) to perform the [probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `2` |
| `functions.readinessProbe.timeoutSeconds` | Number of seconds after which the [probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) times out | `1` |
| `functions.readinessProbe.successThreshold` | Minimum consecutive successes for the [probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) to be considered successful after having failed. | `1` |
| `functions.readinessProbe.failureThreshold` | After a [probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) fails failureThreshold times in a row, Kubernetes considers that the overall check has failed. | `3 `|
| `functions.setNonRootUser` | Force all function containers to run with user id `12000` | `false` |

### Autoscaler (OpenFaaS Pro)

| Parameter               | Description                           | Default                                                    |
| ----------------------- | ----------------------------------    | ---------------------------------------------------------- |
| `autoscaler.disableHorizontalScaling` | Set to true, to only scale to zero, without scaling replicas between the defined Min and Max count for the function | `false` |
| `autoscaler.enabled ` | Enable the autoscaler - if openfaasPro is set to true | `true` |
| `autoscaler.image` | Container image used for the autoscaler | See [values.yaml](./values.yaml) |
| `autoscaler.replicas` | Replicas of the autoscaler | `1` |
| `autoscaler.resources` | Resource limits and requests for the autoscaler pods | See [values.yaml](./values.yaml) |

### Async and NATS

| Parameter               | Description                           | Default                                                    |
| ----------------------- | ----------------------------------    | ---------------------------------------------------------- |
| `jetstreamQueueWorker.durableName` | Durable name used by JetStream consumers | `faas-workers` |
| `jetstreamQueueWorker.image` | Container image used for the queue-worker when the `queueMode` is `jetstream` | See [values.yaml](./values.yaml) |
| `jetstreamQueueWorker.maxWaiting` | Configure the max waiting pulls for the queue-worker JetStream consumer. The value should be at least max_inflight * queue_worker.replicas. Note that this value can not be updated once the consumer is created. | `512` |
| `jetstreamQueueWorker.logs.debug` | Log debug messages | `false` |
| `jetstreamQueueWorker.logs.format` | Set the log format, supports `console` or `json` | `console` |
| `nats.channel` | The name of the NATS Streaming channel or NATS JetStream stream to use for asynchronous function invocations | `faas-request` |
| `nats.external.clusterName` | The name of the externally-managed NATS Streaming server | `""` |
| `nats.external.enabled` | Whether to use an externally-managed NATS Streaming server | `false` |
| `nats.external.host` | The host at which the externally-managed NATS Streaming server can be reached | `""` |
| `nats.external.port` | The port at which the externally-managed NATS Streaming server can be reached | `""` |
| `nats.image` | Container image used for NATS | See [values.yaml](./values.yaml) |
| `nats.resources` | Resource limits and requests for the nats pods | See [values.yaml](./values.yaml) |
| `nats.streamReplication` | JetStream stream replication factor. For production a value of at least 3 is recommended. | `1` |
| `queueWorker.ackWait` | Max duration of any async task/request | `60s` |
| `queueWorker.image` | Container image used for the CE edition of the queue-worker| See [values.yaml](./values.yaml) |
| `queueWorker.maxInflight` | Control the concurrent invocations | `1` |
| `queueWorker.replicas` | Replicas of the queue-worker, pick more than `1` for HA | `1` |
| `queueWorker.resources` | Resource limits and requests for the queue-worker pods | See [values.yaml](./values.yaml) |
| `queueWorker.queueGroup` | The name of the queue group used to process asynchronous function invocations | `faas` |
| `queueWorkerPro.backoff` | The backoff algorithm used for retries. Must be one off `exponential`, `full` or `equal`| `exponential` |
| `queueWorkerPro.httpRetryCodes` | Comma-separated list of HTTP status codes the queue-worker should retry | `408,429,500,502,503,504` |
| `queueWorkerPro.image` | Container image used for the Pro version of the queue-worker | See [values.yaml](./values.yaml) |
| `queueWorkerPro.initialRetryWait` | Time to wait for the first retry | `10s` |
| `queueWorkerPro.insecureTLS` | Enable insecure TLS for callback invocations | `false` |
| `queueWorkerPro.maxRetryAttempts` | Amount of times to try sending a message to a function before discarding it | `10` |
| `queueWorkerPro.maxRetryWait` | Maximum amount of time to wait between retries | `120s` |
| `queueWorkerPro.printResponseBody` | Print the function response body | `false` |
| `queueWorkerPro.printRequestBody` | Print the request body| `false` |
| `stan.image` | Container image used for NATS streaming server | See [values.yaml](./values.yaml) |

### Access and Identity Management (IAM)

| Parameter               | Description                           | Default                                                    |
| ----------------------- | ----------------------------------    | ---------------------------------------------------------- |
| `iam.enabled` | Enable Access and Identity Management for OpenFaaS | `false` |
| `iam.systemIssuer.url` | URL for the OpenFaaS OpenID connect provider for system components. This is usually the public url of the gateway | `https://gateway.example.com` |
| `iam.dashboardIssuer.url` | URL of the OpenID connect provider to used by the dashboard | `https://example.eu.auth0.com` |
| `iam.dashboardIssuer.clientId` | OAuth Client Id for the dashboard | `""` |
| `iam.dashboardIssuer.clientSecret` | Name of the Kubernetes secret that contains the OAuth client secret for the dashboard | `""` |
| `iam.dashboardIssuer.scopes` | OpenID Connect (OIDC) scopes for the dashboard | `[openid, email, profile]` |
| `iam.kubernetesIssuer.create` | Create a JwtIssuer object for the kubernetes service account issuer | `true` |
| `iam.kubernetesIssuer.tokenExpiry` | Expiry time of OpenFaaS access tokens exchanged for tokens issued by the Kubernetes issuer. | `2h` | 
| `iam.kubernetesIssuer.url` | URL for the Kubernetes service account issuer. | `https://kubernetes.default.svc.cluster.local` |

### Dashboard (OpenFaaS Pro)

| Parameter               | Description                           | Default                                                    |
| ----------------------- | ----------------------------------    | ---------------------------------------------------------- |
| `dashboard.enabled` | Enable the dashboard | `false` |
| `dashboard.image` | Container image used for the dashboard | See [values.yaml](./values.yaml) |
| `dashboard.publicURL` | URL used to expose the dashboard. Needs to be a fully qualified domain name (FQDN) | `https://dashboard.example.com` |
| `dashboard.logs.debug` | Log debug messages | `false` |
| `dashboard.logs.format` | Set the log format, supports `console` or `json` | `console` |
| `dashboard.replicas` | Replicas of the dashboard | `1` |
| `dashboard.resources` | Resource limits and requests for the dashboard pods | See [values.yaml](./values.yaml) |
| `dashboard.signingKeySecret` | Name of signing key secret for sessions. Can be left blank for development, see https://docs.openfaas.com/openfaas-pro/dashboard/ for production and staging. | `""` |

### OIDC/SSO (OpenFaaS Pro)

| Parameter               | Description                           | Default                                                    |
| ----------------------- | ----------------------------------    | ---------------------------------------------------------- |
| `oidcAuthPlugin.image` | Container image used for the oidc-auth-plugin | See [values.yaml](./values.yaml) |
| `oidcAuthPlugin.insecureTLS` | Enable insecure TLS | `false` |
| `oidcAuthPlugin.replicas` | Replicas of the oidc-auth-plugin | `1` |
| `oidcAuthPlugin.resources` | Resource limits and requests for the oidc-auth-plugin containers | See [values.yaml](./values.yaml) |
| `oidcAuthPlugin.verbose` | Enable verbose logging | `false` |

### Event subscriptions

| Parameter | Description | Default |
|-----------|-------------|---------|
| `eventSubscription.endpoint` | The webhook endpoint to receive events. | `""`|
| `eventSubscription.endpointSecret` | Name of the Kubernetes secret that contains the secret key for signing webhook requests. | `""` |
| `eventSubscription.insecureTLS` | Enable insecure TLS for webhook invocations | `false` |
| `eventSubscription.metering.enabled` | Enable metering events. | `false` |
| `eventSubscription.metering.defaultRAM` | Default memory value used in function_usage events for metering when no memory limit is set on the function.  | `512Mi` |
| `eventSubscription.metering.excludedNamespaces` | Comma-separated list of namespaces to exclude from metering for when functions are used to handle the metering webhook events | `""` |
| `eventSubscription.auditing.enabled` | Enable auditing events | `false` |
| `eventSubscription.auditing.httpVerbs` | Comma-separated list of HTTP methods to audit | `"PUT,POST,DELETE"`|
| `eventWorker.image` | Container image used for the events-worker | See [values.yaml](./values.yaml) |
| `eventWorker.resources` |  Resource limits and requests for the event-worker container | See [values.yaml](./values.yaml) |
| `eventWorker.logs.format` | Set the log format, supports `console` or `json` | `console` |
| `eventWorker.logs.debug` | Print debug logs    | `false` |

### faas-idler (OpenFaaS Pro)

Deprecated and replaced by the new autoscaler, which supports scale to zero.

### ingressOperator

| Parameter               | Description                           | Default                                                    |
| ----------------------- | ----------------------------------    | ---------------------------------------------------------- |
| `ingressOperator.create` | Create the ingress-operator component | `false` |
| `ingressOperator.image` | Container image used in ingress-operator| `openfaas/ingress-operator:0.6.2` |
| `ingressOperator.replicas` | Replicas of the ingress-operator| `1` |
| `ingressOperator.resources` | Limits and requests for memory and CPU usage | Memory Requests: 25Mi |

### alertmanager

For legacy scaling in OpenFaaS Community Edition.

| Parameter               | Description                           | Default                                                    |
| ----------------------- | ----------------------------------    | ---------------------------------------------------------- |
| `alertmanager.create` | Create the AlertManager component | `true` |
| `alertmanager.image` | Container image used for alertmanager | See [values.yaml](./values.yaml) |
| `alertmanager.resources` | Resource limits and requests for alertmanager pods | See [values.yaml](./values.


### Prometheus (built-in, for autoscaling and metrics)

| Parameter               | Description                           | Default                                                    |
| ----------------------- | ----------------------------------    | ---------------------------------------------------------- |
| `prometheus.create` | Create the Prometheus component | `true` |
| `prometheus.image` | Container image used for prometheus | See [values.yaml](./values.yaml) |
| `prometheus.resources` | Resource limits and requests for prometheus containers | See [values.yaml](./values.yaml) |
# charts
