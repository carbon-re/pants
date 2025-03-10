---
title: "Deployments"
slug: "helm-deployments"
hidden: false
createdAt: "2022-07-19T13:16:00.000Z"
---
> 🚧 Helm deployment support is in alpha stage
> 
> Pants has experimental support for managing deployments via the `experimental-deploy` goal. Helm deployments provides with a basic implementation of this goal.
> 
> Please share feedback for what you need to use Pants with your Helm deployments by either [opening a GitHub issue](https://github.com/pantsbuild/pants/issues/new/choose) or [joining our Slack](doc:getting-help)!

Motivation
----------

Helm's ultimate purpose is to simplify the deployment of Kubernetes resources and help in making these reproducible. However it is quite common to deploy the same software application into different kind of environments using slightly different configuration overrides.

This hinders reproducibility since operators end up having a set of configuration files and additional shell scripts that ensure that the Helm command line usued to deploy a piece of software into a given environment is always the same.

Pants solves this problem by providing with the ability to manage the configuration files and the different parameters of a deployment as single unit such that a simple command line as `pants experimental-deploy ::` will always have the same effect on each of the deployments previously defined.

Defining Helm deployments
-------------------------

Helm deployments are defined using the `helm_deployment` target which has a series of fields that can be used to guarantee the reproducibility of the given deployment. `helm_deployment` targets need to be added by hand as there is no deterministic way of instrospecting your repository to find sources that are specific to Helm:

```python src/chart/BUILD
helm_chart()
```
```yaml src/chart/Chart.yaml
apiVersion: v2
description: Example Helm chart
name: example
version: 0.1.0
```
```python src/deployment/BUILD
helm_deployment(
  name="dev",
  chart="//src/chart",
  sources=["common-values.yaml", "dev-override.yaml"]
)

helm_deployment(
  name="stage",
  chart="//src/chart",
  sources=["common-values.yaml", "stage-override.yaml"]
)

helm_deployment(
  name="prod",
  chart="//src/chart",
  sources=["common-values.yaml", "prod-override.yaml"]
)
```
```yaml src/deployment/common-values.yaml
# Default values common to all deployments
env:
  SERVICE_NAME: my-service
```
```yaml src/deployment/dev-override.yaml
# Specific values to the DEV environment
env:
  ENV_ID: dev
```
```yaml src/deployment/stage-override.yaml
# Specific values to the STAGE environment
env:
  ENV_ID: stage
```
```yaml src/deployment/prod-override.yaml
# Specific values to the PRODUCTION environment
env:
  ENV_ID: prod
```

There are quite a few things to notice in the previous example:

* The `helm_deployment` target requires you to explicitly set the `chart` field to specify which chart to use.
* We have three different deployments using different sets of configuration files with the same chart.
* One of those value files (`common-values.yaml`) provides with default values that are common to all deployments.
* Each deployment uses an additional `xxx-override.yaml` file with values that are specific to the given deployment.

The `helm_deployment` target has many additional fields including the target kubernetes namespace, adding inline override values (similar to using helm's `--set` arg) and many others. Please run `pants help helm_deployment` to see all the posibilities.

Dependencies with `docker_image` targets
----------------------------------------

A Helm deployment will in most cases deploy one or more Docker images into Kubernetes. Furthermore, it's quite likely there is going to be at least a few first party Docker images among those. Pants is capable of analysing the Helm chart being used in a deployment to detect those required first-party Docker images using Pants' target addresses to those Docker images.

To illustrate this, let's imagine the following scenario: Let's say we have a first-party Docker image that we want to deploy into Kubernetes as a `Pod` resource kind. For achieving this we define the following workspace:

```python src/docker/BUILD
docker_image()
```
```text src/docker/Dockerfile
FROM busybox:1.28
```
```python src/chart/BUILD
helm_chart()
```
```yaml src/chart/Chart.yaml
apiVersion: v2
description: Example Helm chart
name: example
version: 0.1.0
```
```yaml src/chart/values.yaml
# Default image in case this chart is used by other tools after being published
image: example.com/registry/my-app:latest
```
```yaml src/chart/templates/pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    chart: "{{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}"
spec:
  containers:
    - name: my-app
      # Uses the `image` value entry from the deployment inputs
      image: {{ .Values.image }}
```
```python src/deployment/BUILD
# Overrides the `image` value for the chart using the target address for the first-party docker image.
helm_deployment(chart="src/chart", values={"image": "src/docker:docker"})
```

> 📘 Docker image references VS Pants' target addresses
> 
> You should use typical Docker registry addresses in your Helm charts. Because Helm charts are distributable artifacts and may be used with tools other than Pants, you should create your charts such that when that chart is being used, all Docker image addresses are valid references to images in accessible Docker registries. As shown in the example, we recommend that you make the image address value configurable, especially for charts that deploy first-party Docker images.
> Your chart resources can still use off-the-shelf images published with other means, and in those cases you will also be referencing the Docker image address. Usage of Pants' target addresses is intended for your own first-party images because the image reference of those is not known at the time we create the sources (they are computed later).

With this setup we should be able to run `pants dependencies src/deployment` and Pants should give the following output:

```text
src/chart
src/docker
```

This should work with any kind of Kubernetes resource that leads to Docker image being deployed into Kubernetes, such as `Deployment`, `StatefulSet`, `ReplicaSet`, `CronJob`, etc. Please get in touch with us in case you find Pants was not capable to infer dependencies in any of your `helm_deployment` targets by either [opening a GitHub issue](https://github.com/pantsbuild/pants/issues/new/choose) or [joining our Slack](doc:getting-help).

> 📘 How the Docker image reference is calculated during deployment?
> 
> Pants' will rely on the behaviour of the `docker_image` target when it comes down to generate the final image reference. Since a given image may have more than one valid image reference, **Pants will try to use the first one that is not tagged as `latest`**, falling back to `latest` if none could be found.
> It's good practice to publish your Docker images using tags other than `latest` and Pants preferred behaviour is to choose those as this guarantees that the _version_ of the Docker image being deployed is the expected one.

Value files
-----------

It's very common that Helm deployments use a series of files providing with values that customise the given chart. When using deployments that may have more than one YAML file as the source of configuration values, the Helm backend needs to sort the file names in a way that is consistent across different machines, as the order in which those files are passed to the Helm command is relevant. The final order depends on the same order in which those files are specified in the `sources` field of the `helm_deployment` target. For example, given the following `BUILD` file:

```python src/deployment/BUILD
helm_deployment(name="dev", chart="//src/chart", sources=["first.yaml", "second.yaml", "last.yaml"])
```

This will result in the Helm command receiving the value files as in that exact order.

If using any glob pattern in the `sources` field, the plugin will first group the files according to the order in which those glob patterns are listed. In this grouping, files that are resolved by more than one pattern will be part of the most specific group. Then we use alphanumeric ordering for the files that correspond to each of the previous groups. To illustrate this scenario, consider the following list of files:

```
src/deployment/002-config_maps.yaml
src/deployment/001-services.yaml
src/deployment/first.yaml
src/deployment/dev/daemon_sets.yaml
src/deployment/dev/services-override.yaml
src/deployment/last.yaml
```

And also the following `helm_deployment` target definition:

```python src/deployment/BUILD
helm_deployment(
  name="dev",
  chart="//src/chart",
  sources=["first.yaml", "*.yaml", "dev/*-override.yaml", "dev/*.yaml", "last.yaml"]
)
```

In this case, the final ordering of the files would be as follows:

```
src/deployment/first.yaml
src/deployment/001-services.yaml
src/deployment/002-config_maps.yaml
src/deployment/dev/services-override.yaml
src/deployment/dev/daemon_sets.yaml
src/deployment/last.yaml
```

We believe that this approach gives a very consistent and predictable ordering while at the same time total flexibility to the end user to organise their files as they best fit each particular case of a deployment.

Inline values
-------------

In addition to value files, you can also use inline values in your `helm_deployment` targets by means of the `values` field. All inlines values that are set this way will override any entry that may come from value files.

Inline values are defined as a key-value dictionary, like in the following example:

```python src/deployment/BUILD
helm_deployment(
  name="dev",
  chart="//src/chart",
  values={
    "nameOverride": "my_custom_name",
    "image.pullPolicy": "Always",
  },
)
```

### Using dynamic values

Inline values also support interpolation of environment variables. Since Pants runs all processes in a hermetic sandbox, to be able to use environment variables you must first tell Pants what variables to make available to the Helm process using the `[helm].extra_env_vars` option. Consider the following example:

```python src/deployment/BUILD
helm_deployment(
  name="dev",
  chart="//src/chart",
  values={
    "configmap.deployedAt": "{env.DEPLOY_TIME}",
  },
)
```
```toml pants.toml
[helm]
extra_env_vars = ["DEPLOY_TIME"]
```

Now you can launch a deployment using the following command:

```
DEPLOY_TIME=$(date) pants experimental-deploy src/deployment:dev
```

> 🚧 Ensuring repeatable deployments
> 
> You should always favor using static values (or value files) VS dynamic values in your deployments. Using interpolated environment variables in your deployments can render your deployments non-repetable anymore if those values can affect the behaviour of the system deployed, or what gets deployed (i.e. Docker image addresses).
> Dynamic values are supported to give the option of passing some info or metadata to the software being deployed (i.e. deploy time, commit hash, etc) or some less harmful settings of a deployment (i.e. replica count. etc). Be careful when chossing the values that are going to be calculated dynamically.

Third party chart artifacts
---------------------------

Previous examples on the usage of the `helm_deployment` target are all based on the fact that the deployment declares a dependency on a Helm chart that is also part of the same repository. Since charts support having dependencies with other charts in the same repository or with external 3rd party Helm artifacts (declared as `helm_artifact`), all that dependency resolution is handled for us.

However, `helm_deployment`s are not limited to only first party charts, as it is also possible to declare a deployment having a dependency on a 3rd party Helm artifact instead. As an example, consider the following workspace layout:

```python 3rdparty/helm/jetstack/BUILD
helm_artifact(
  name="cert-manager",
  artifact="cert-manager",
  version="v0.7.0",
  repository="https://charts.jetstack.io",
)
```
```python src/deploy/BUILD
helm_deployment(
  name="main",
  chart="//3rdparty/helm/jetstack:cert-manager",
  values={
    "installCRDs": "true"
  },
)
```

In this example, the deployment at `src/deploy:main` declares a dependency on a 3rd party Helm artifact instead of a chart in the same repository. The only difference in this case when compared to first party charts is that Pants will resolve and fetch the third party artifact automatically. Once the artifact has been resolved, there is no difference to Pants.

Post-renderers
--------------

User-defined [Helm post-renderers](https://helm.sh/docs/topics/advanced/#post-rendering) are supported by the Helm backend by means of the `post_renderers` field in the `helm_deployment` target. This field takes addresses to other runnable targets (any target that can be run using `pants run [address]`) and will build and run those targets as part of `experimental-deploy` goal. The referenced targets can be either shell commands or custom-made in any of the other languages supported by Pants.

As an example, let's show how we can use the tool [`vals`](https://github.com/variantdev/vals) as a post-renderer and replace all references to secret values stored in HashiCorp Vault by their actual values. The following example is composed of a Helm chart that creates a secret resource in Kubernetes and a Helm deployment that is configured to use `vals` as a post-renderer:

```python src/chart/BUILD
helm_chart()
```
```yaml src/chart/Chart.yaml
apiVersion: v2
description: Example Helm chart with vals
name: example
version: 0.1.0
```
```yaml src/chart/templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
  namespace: default
data:
  username: admin
  # This should be replaced by `vals` during the post-rendering
  password: ref+vault://path/to/admin#/password
type: Opaque
```
```python src/deploy/BUILD
run_shell_command(
  name="vals",
  command="vals eval -f -",
)

helm_deployment(
  chart="//src/chart",
  post_renderers=[":vals"],
)
```

In the previous example we define a `run_shell_command` target that will invoke the `vals eval` command (`vals` needs to be installed in the local machine) as part of the Helm post-rendering machinery, which will result on the `ref+vault` reference being replaced by the actual value stored in Vault at the given path.

> 📘 Using multiple post-renderers
>
> If more than one target address is given in the `post_renderers` field, then they will be invoked in the same order given piping the output of one them into the input of the next one.

Deploying
---------

Continuing with the example in the previous section, we can deploy it into Kubernetes using the command `pants experimental-deploy src/deployment`. This will trigger the following steps:

1. Analyse the dependencies of the given deployment.
2. Build and publish any first-party Docker image and Helm charts that are part of those dependencies.
3. Post-process the Kubernetes manifests generated by Helm by replacing all references to first-party Docker images by their real final registry destination.
4. Initiate the deployment of the final Kubernetes resources resulting from the post-processing.

The `experimental-deploy` goal also supports default Helm pass-through arguments that allow to change the deployment behaviour to be either atomic or a dry-run or even what is the Kubernetes config file (the `kubeconfig` file) and target context to be used in the deployment.

Please note that the list of valid pass-through arguments has been limited to those that do not alter the reproducibility of the deployment (i.e. `--create-namespace` is not a valid pass-through argument). Those arguments will have equivalent fields in the `helm_deployment` target.

For example, to make an atomic deployment into a non-default Kubernetes context you can use a command like the following one:

```
pants experimental-deploy src/deployments:prod -- --kube-context my-custom-kube-context --atomic
```

> 📘 How does Pants authenticate with the Kubernetes cluster?
>
> Short answer is: it doesn't. 
> Pants will invoke Helm under the hood with the appropriate arguments to only perform the deployment. Any authentication steps that may be needed to perform the given deployment have to be done before invoking the `experimental-deploy` goal. If you are planning to run the deployment procedure from your CI/CD pipelines, ensure that all necessary preliminary steps (including authentication with the cluster) are done before the one that triggers the deployment.
