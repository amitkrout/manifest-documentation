# Bootstrapping Manifests

## Prerequisites

You need access to a recent OpenShift (4.3) cluster.

You will need to have completed the installation of the following additional
components:

 * OpenShift Pipelines Operator
 * Sealed Secrets [installed](https://github.com/bitnami-labs/sealed-secrets#installation)
 * ArgoCD [installed](https://operatorhub.io/operator/argocd-operator)
   There is a [guide](./argocd.md) to installing it here. **NOTE**: it MUST be
   installed to the `argocd` namespace.

You will need to be logged in with administrator privileges.

While bootstrapping the manifest, the tool will generate secrets using the Sealed Secrets operator.

## Generating credentials for pushing images

The default CI pipeline builds an image from your source application, and pushes
it to a repository, it will require credentials for this.

See [here](./quay_io.md) for a guide to creating the required credentials, if you're
using Quay.io.

## Manifest Model

The manifest models environments, applications and services.

An environment can have many applications, which in-turn can reference many
services.

Services are "per-environment", and multiple applications _within_ an
environment can reference the same service, and override the configuration with
Kustomize.

## Bootstrapping the Manifest

```shell
$ odo pipelines bootstrap \
  --app-repo-url https://github.com/<username>/taxi.git \
  --app-webhook-secret testing \
  --gitops-repo-url https://github.com/<username>/gitops.git \
  --gitops-webhook-secret testing2 \
  --image-repo quay.io/<username>/taxi \
  --dockercfgjson ~/Downloads/<username>-auth.json --prefix tst-
```

**NOTE**: DO NOT use `testing` and `testing2` as your secrets.

Per the (GitHub documentation)[https://developer.github.com/webhooks/securing/]
you should generate a secret for each of them:

```shell
$ ruby -rsecurerandom -e 'puts SecureRandom.hex(20)'
```

| Option                  | Description |
| ----------------------- | ----------- |
| --app-repo              | This is the source code to your first application. |
| --app-webhook-secret    | Creates a secret used to validate incoming hooks. |
| --gitops-repo-url       | This is where your configuration and pipelines live. |
| --gitops-webhook-secret | This is used to validate incoming hooks. |
| --image-repo            | Where should we configure your builds to push to? |
| --dockercfgjson         | This is used to authenticate image pushes to your image-repo. |
| --prefix                | This is used to help separate user namespaces. |

## Exploring the manifest

The bootstrap process generates a fairly large number of files, including a
manifest describing your first application, and configuration for a complete CI
pipeline and deployments from ArgoCD.

```yaml
environments:
- name: tst-dev
  pipelines:
    integration:
      binding: github-pr-binding
      template: app-ci-template
  apps:
  - name: taxi
    services:
    - name: taxi-svc
      source_url: https://github.com/<username>/taxi.git
      webhook:
        secret:
          name: github-webhook-secret-taxi-svc
- name: tst-stage
- name: tst-cicd
  cicd: true
- name: tst-argocd
  argo: true
```

The bootstrap creates four environments, `dev`, `stage`, `cicd` and `argocd`
with a user-supplied prefix.

Namespaces are generated for the `dev`, `stage` and `cicd` environments.

The name of the app and service are derived from the last component of your
`app-repo-url` e.g. if your bootstrap with `--app-repo-url
https://github.com/myorg/myproject.git` this would bootstrap an app called
`myproject` and a service called `myproject-svc`.

## Environment configuration

The `tst-dev` environment created is the most visible configuration.

### configuring pipelines

```yaml
environments:
- name: tst-dev
  pipelines:
    integration:
      binding: github-pr-binding
      template: app-ci-template
  apps:
  - name: taxi
    services:
    - name: taxi-svc
      source_url: https://github.com/<username>/taxi.git
      webhook:
        secret:
          name: github-webhook-secret-taxi-svc
```

The `pipelines` key describes how to trigger a Tekton PipelineRun, the
`integration` binding and template are processed when a _Pull Request_
is opened.

This is the default pipeline specification for the `tst-dev` environment, you
can find the definitions for these in these two files:

 * `environments/<prefix>cicd/base/pipelines/07-templates/app-ci-build-pr-template.yaml`
 * `environments/<prefix>cicd/base/pipelines/06-bindings/github-pr-binding.yaml`

By default this triggers a PipelineRun of this pipeline

 * `environments/<prefix>cicd/base/pipelines/05-pipelines/app-ci-pipeline.yaml`

These files are not managed directly by the manifest, you're free to change them
for your own needs, by default they use [Buildah](https://github.com/containers/buildah)
to trigger build, assuming that the Dockerfile for your application is in the root
of your repository.

### configuring services

```yaml
apps:
- name: taxi
  services:
  - name: taxi-svc
    source_url: https://github.com/<username>/taxi.git
    webhook:
      secret:
        name: github-webhook-secret-taxi-svc
```

The YAML above defines an app called `taxi`, which has a single service called `taxi-svc`.

The configuration for these is written out to:

 * `environments/<prefix>dev/services/taxi-svc/base/config/`
 * `environments/<prefix>dev/apps/taxi/base/config/`

The `taxi` app's configuration references the services configuration.

The `source_url` references the upstream repository for the service, this
coupled with the `webhook.secret` is also used to drive the CI pipeline, with a
hook confgured, changes to this repository will trigger the template and
binding), the secret is used to authenticate incoming hooks from GitHub.

## Bringing the bootstrapped environment up

First of all, let's get started with our Git repository.

From the root of your gitops directory (with the pipelines.yaml), execute the
following commands:

```shell
$ git init .
$ git add .
$ git commit -m "Initial commit."
$ git remote add origin <insert gitops repo>
$ git push -u origin master
```

This should initialise the GitOps repository, this is the start of your journey
to deploying applications via Git.

Next, we'll bring up our deployment infrastructure, this is only necessary at the
start, the configuration should be self-hosted thereafter.

```shell
$ oc apply -k environments/tst-dev/env/base
$ oc apply -k environments/tst-argocd/config
$ oc apply -k environments/tst-cicd/base
```

You should now be able to create a route to your new service, it should be
running [nginx](https://nginx.org/) and serving a page.

## Changing the initial deployment

The bootstrap creates a `Deployment` in `environments/<prefix>-dev/services/<service name>-svc/base/config/100-deployment.yaml` this should bring up nginx, this is purely for demo purposes, you'll need to change this to deploy your built image.

```yaml
spec:
  containers:
  - image: nginxinc/nginx-unprivileged:latest
    imagePullPolicy: Always
    name: taxi-svc
```

You'll want to replace this with the image for your application, if you've built
it.

## Your first CI run

Part of the configuration bootstraps a simple TektonCD pipeline for building code when a pull-request is opened.

You will need to create a new Webhook for the CI:

![Creating a Webhook with a Secret](img/github/create-github-webhook.png)

The secret should be the secret you provided on the command-line.

Configure the endpoint to point at the route for the EventListener in the
`-cicd` project in OpenShift.

Make sure you change the Content Type to `"application/json"` and enable the
"send me everything" option for events (otherwise we'll only receive
"push" events).

Make a change to your application source, the `taxi` repo from the example, it
can be as simple as editing the `README.md` and propose a change as a
Pull Request.

This should trigger the PipelineRun:

![PipelineRun with succesful completion](img/ocp/pipelinerun-success.png)

Drilling into the PipelineRun we can see that it executed our single task:

![PipelineRun with steps](img/ocp/pipelinerun-succeeded-detail.png)

And finally, we can see the logs that the build completed and the image was
pushed:

![PipelineRun with logs](img/ocp/pipelinerun-succeeded-logs.png)

## Changing the default CI run

Before this next stage, we need to ensure that there's a Webhook configured for
the "gitops" repo.

You will need to create a new Webhook for the CI:

![Creating a Webhook with a Secret](img/github/create-github-webhook.png)

It will need to be configured with the secret that you used when creating the
manifest.

This step involves changing the CI definition for your application code.

The default CI pipeline we provide is defined in the manifest file:

```
environments:
- name: tst-dev
  pipelines:
    integration:
      binding: github-pr-binding
      template: app-ci-template
```

This template drives a Pipeline that is stored in this file:

 * `environments/<prefix>cicd/base/pipelines/05-pipelines/app-ci-pipeline.yaml`

An abridged version is shown below, it has a single task `build-image`, which
executes the `buildah` task, which basically builds the source and generates an
image and pushes it to your image-repo.

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
spec:
  resources:
  - name: source-repo
    type: git
  tasks:
  - name: build-image
      inputs:
      - name: source
        resource: source-repo
    taskRef:
      kind: ClusterTask
      name: buildah
```

Normally, you'd at least run a simple test of the code before you build the
code:

Write the following Task to this file:

 * `environments/<prefix>cicd/base/pipelines/04-tasks/go-test-task.yaml`

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: go-test
  namespace: tst-cicd
spec:
  inputs:
    resources:
    - name: source
      description: the git source to execute on
      type: git
  steps:
    - name: go-test
      image: golang:latest
      command: ["go", "test", "./..."]
```

This is a simple test task for a Go application, it just runs the tests.

Update the pipeline in this file:

 * `environments/<prefix>cicd/base/pipelines/05-pipelines/app-ci-pipeline.yaml`


```yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  creationTimestamp: null
  name: app-ci-pipeline
  namespace: tst-cicd
spec:
  params:
  - name: REPO
    type: string
  - name: COMMIT_SHA
    type: string
  resources:
  - name: source-repo
    type: git
  - name: runtime-image
    type: image
  tasks:
  - name: go-ci
    resources:
      inputs:
      - name: source
        resource: source-repo
    taskRef:
      kind: Task
      name: go-test
  - name: build-image
    runAfter:
      - go-ci
    params:
    - name: TLSVERIFY
      value: "true"
    resources:
      inputs:
      - name: source
        resource: source-repo
      outputs:
      - name: image
        resource: runtime-image
    taskRef:
      kind: ClusterTask
      name: buildah
```

Commit and push this code, and open a Pull Request, you should see a PipelineRun
being executed.

![PipelineRun doing a dry run of the configuration](img/ocp/pipelinerun-dryrun.png)

This validates that the YAML can be applied, by executing a `kubectl apply --dry-run`.

## Adding an additional application
