# An introduction to canary deployment with Kayenta and Argo Rollouts

Kubernetes is more and more becoming a standard platform for application development. Its flexibility allow us to release more frequently, hence reducing defection when rolling up a new version of application.
Kubernetes itself support 2 types of update options:
- Rolling update: replace application instance (pod) one by one until the new version replicate number is reached. Configurable with [terminationGracePeriodSecond](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#terminationGracePeriodSeconds),    [maxUnavailble and max Surge](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)
- Recreate: Shutdown all old instance and replace with newer version instance.

![alt text](./assets/container.jpeg)

While this is sufficient in many cases, one might want meticulous way of doing version upgrade. There are two main option:
- Blue-green deployment: new instances running in parallel with old instances. After proper testing, traffic is switch to the new instances.
- Canary deployment: a small subsets of new instances will be released incrementally to user (eg: 5%, 10%, 15%). Hence canary release is tend to be less error-prone, compared to other deployment strategy.

## Introducing canary deployment

With that's the case, there are still many way to do canary deployment. For example, we can:
 - Try best effor traffic splitting by control the deployment's replicaset
 - Split traffic by [nginx ingress canary annotation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary) for north-south traffic
 - Split traffic using existing service mesh solution for east-west , most popularly [istio](https://istio.io/latest/docs/concepts/traffic-management/)

However, simply splitting traffic by percentage (or header) sometimes is not enough. We need a way to analyse how our application react to that traffic. To do that, we need our application to expose metrics to a metrics provider (prometheus, cloudwatch, etc..) and a way to evaluate if the result is good enough.

![alt text](./assets/argo.jpeg)

## Introducing Argo Rollouts and Kayenta

Argo Rollouts is a part of [Argo Project](https://argoproj.github.io/), an emerging sets of opensource tools to do workflow, deployment upgrade and configuration management of Kubernetes.

Argo Rollouts support doing canary deployment with many metrics provider. Here we will focus on Prometheus. While it is still sufficient to do canary based on absolute metrics value fetched from prometheus, eg percentage of successful request rate(http_request_total{job="my-app", code ~= "200"}[5m]) / (rate(http_requests_total{job =~ "my-app"}[5m])). In some case, we still need to compare sets of result between old version and new version of deployment. This way of doing canary deployment is called canary analysis.

Kayenta, originally a part of [Spinnaker](https://spinnaker.io/), can run as a standalone canary analysis service. Argo rollouts will poll kayenta periodically during canary period to get metrics sets analysis result. Under the hood, Kayenta use [Mann-Whitney U test](https://en.wikipedia.org/wiki/Mann%E2%80%93Whitney_U_test)

Create namespace:
```sh
kubectl create namespace kayenta
```

Deploy kayenta-redis:
```sh
helm install kayenta-redis ---namespace kayenta bitnami/redis --set auth.enabled=false --set architecture=standalone --set master.persistence.enabled=false
```

Adjust your value file and deploy kayenta, kayenta will need to connect to redis, s3 (gcs/azure blob storage/minio) and metrics provider, here we use Prometheus. Please reference to cloud provider docs for instruction on how to grant access for Kayenta pod to read and write from object storage: [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html), [GKE Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)

```sh
git clone https://github.com/phuongnd96/kayenta.git
cd kayenta
helm install kayenta --namespace kayenta -f values.yaml .
```

Deploy argo-rollouts
```sh
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

Deploy argo-cd
```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Deploy nginx-ingress
```sh
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

Deploy our app. Argo-rollouts will wrap the pod with Rollouts Custom Resource to wrap our ReplicaSet. For each new Rollout, a new ReplicaSet will be created, argo-rollouts will incrementally increase the new ReplicaSet's desired number while incrementally decrease the old ReplicaSet's desired number until the new ReplicaSet is fully promoted.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: canary-demo-app
  namespace: argocd
spec:
  destination:
    namespace: canary-rollout-demo
    server: https://kubernetes.default.svc
  project: default
  source:
    path: .
    repoURL: https://github.com/phuongnd96/canary-sample-app
    targetRevision: HEAD
```

Create kayenta canary config.
Go to http://you_kayenta_endpoint/swagger-ui.html#/canary-config-controller/storeCanaryConfigUsingPOST

```json
{
    "name": "canary-14",
    "description": "test",
    "configVersion": "1",
    "applications": [
      "my-app"
    ],
    "judge": {
      "name": "NetflixACAJudge-v1.0",
      "judgeConfigurations": {}
    },
    "metrics": [
      {
        "name": "request_error",
        "query": {
          "type": "prometheus",
          "customInlineTemplate": "PromQL:sum(rate(http_requests_total{job=\"my-app\", code=~\"[45]..\",${scope}}[5m]))",
          "serviceType": "prometheus"
        },
        "groups": [
          "error"
        ],
        "analysisConfigurations": {
          "canary": {
            "critical": true,
            "nanStrategy": "replace",
            "mustHaveData": true,
            "effectSize": {
              "allowedIncrease": 1,
              "allowedDecrease": 1
            },
            "outliers": {
              "strategy": "remove"
            },
            "direction": "increase"
          }
        },
        "scopeName": "default"
      }
    ],
    "templates": {
    },
    "classifier": {
      "groupWeights": {
        "error": 100
      },
      "scoreThresholds": {
        "pass": 95.0,
        "marginal": 75.0
      }
    } 
  }
```
Update VERSION env to create a new canary deployment. 
Argo rollouts will create 2 replicas: baseline and canary and it will get analysis judge result from kayenta every 1 minutes to decide if there are any difference in value of 2 metrics set. The upgrade is considered successful by kayenta if 2 sets of metrics are identical.