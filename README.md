# openshift-tekton

### References

- [Creating Applications w/ CICD Pipelines](https://docs.openshift.com/container-platform/4.12/cicd/pipelines/creating-applications-with-cicd-pipelines.html)

### Install Tekton & Tekton Triggers

First, you need to ensure Tekton & Tekton Triggers are installed in your OpenShift cluster.

This configuration is a request to OpenShift to automatically install and keep updated the latest stable version of the OpenShift Pipelines Operator from Red Hat's official catalog.

```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator-rh
  namespace: openshift-operators
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: openshift-pipelines-operator-rh.v1.10.4
```

### Prepare Feature Flags

This configuration contains a set of on/off switches and settings (flags) for various features and behaviors of Tekton Pipelines in the OpenShift environment. By changing these, you can control how Tekton behaves and which features are active.

Here we added
```
  disable-home-env-overwrite: "false"
  disable-working-directory-overwrite: "false"
```

This was for preventing.

```
warning: unsuccessful cred copy: ".docker" from "/tekton/creds" to "/": unable to create destination directory: mkdir /.docker: permission denied
Skipping step because a previous step failed
```

Here is the final feature flag settings in this example.

```
apiVersion: v1
data:
  await-sidecar-readiness: "true"
  custom-task-version: v1beta1
  disable-affinity-assistant: "true"
  disable-creds-init: "true"
  disable-home-env-overwrite: "false"
  disable-working-directory-overwrite: "false"
  embedded-status: both
  enable-api-fields: stable
  enable-custom-tasks: "true"
  enable-provenance-in-status: "false"
  enable-tekton-oci-bundles: "false"
  performance: <v1alpha1.PipelinePerformanceProperties Value>
  require-git-ssh-secret-known-hosts: "false"
  resource-verification-mode: skip
  running-in-environment-with-injected-sidecars: "true"
  send-cloudevents-for-runs: "false"
  verification-mode: skip
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/instance: default
    app.kubernetes.io/part-of: tekton-pipelines
    operator.tekton.dev/operand-name: tektoncd-pipelines
  name: feature-flags
  namespace: openshift-pipelines
```
### Create Namespace

Think of a `Project` in OpenShift as a workspace for applications and services. It's a way to group together related resources, much like a folder on your computer.  Here we are creating a project specifically for usage by `tekton`.  It's possible that multiple `tekton pipeline|task runs` could require isolation with limited permissions. 

```
apiVersion: project.openshift.io/v1
kind: Project
metadata:
  name: tekton
  labels:
    app: tekton
```
### Create Registry Secret

This YAML is for creating a `Secret` in Kubernetes, specifically for Docker authentication.  It can be used by a task in order to auth into the your registry in order to push images.

```
apiVersion: v1
data:
  config.json: Base64EncodedString
kind: Secret
metadata:
  name: quay-secret
  namespace: tekton
type: Opaque
```

### Create Github Secret

```
apiVersion: v1
kind: Secret
metadata:
  name: github-basic-auth
  namespace: tekton
  annotations:
    tekton.dev/git-0: github.com/org/repository.git
type: kubernetes.io/basic-auth
stringData:
  username: Base64EncodedString
  password: Base64EncodedString
```

## Create Service Account

A ServiceAccount in Kubernetes offers identity for pod processes. When you access the cluster, you're identified by a user account, while processes inside pods are authenticated as a specific ServiceAccount. This "spark-tekton-sa" ServiceAccount might grant permissions, like database access, to workloads within the Tekton pipeline.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spark-tekton-sa
  namespace: tekton
```

### Create Permissions for Service Account

A `ClusterRole` named "tekton-eventlistener-cluster-role" is created. It specifies permissions across various resources:
	- Use of the privileged security context.
		- This is optional - if you prepare your images and installation requirements in advance.  This option can be removed and risk is removed from the configuration.
	- Create pipeline runs.
	- List cluster interceptors and interceptors.
	- Retrieve event listeners, trigger bindings, and trigger templates.
	- Create or modify events.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tekton-eventlistener-cluster-role
rules:
- apiGroups:
  - "security.openshift.io"
  resources:
  - securitycontextconstraints
  resourceNames:
  - privileged
  verbs:
  - use
- apiGroups:
  - tekton.dev
  resources:
  - pipelineruns
  verbs:
  - create
- apiGroups:
  - triggers.tekton.dev
  resources:
  - clusterinterceptors
  - interceptors
  verbs:
  - list
- apiGroups:
  - triggers.tekton.dev
  resources:
  - eventlisteners
  - triggerbindings
  - triggertemplates
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
```

### Bind Permissions to Service Account

A `ClusterRoleBinding` named "tekton-eventlistener-binding" is also established. This binding connects the "tekton-eventlistener-cluster-role" to a ServiceAccount named "spark-tekton-sa" in the "tekton" namespace. Essentially, it grants the permissions defined in the ClusterRole to any pod running with the "spark-tekton-sa" ServiceAccount.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tekton-eventlistener-binding
subjects:
- kind: ServiceAccount
  name: spark-tekton-sa
  namespace: tekton
roleRef:
  kind: ClusterRole
  name: tekton-eventlistener-cluster-role
  apiGroup: rbac.authorization.k8s.io
```
### Define Tekton Tasks

The `build-push-spark-image` Task in the `tekton` namespace automates the process of downloading Apache Spark version 3.4.1, building a Docker image for it using Podman, and pushing the image to a specified container registry. The Task leverages a secret named "quay-secret" mentioned above for authentication during the image push process.

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-push-spark-image
  namespace: tekton
spec:
  params:
    - name: registry
      description: The target container registry where the image should be pushed
      default: "<container_registry>"
    - name: server
      description: The target registry server
      default: "<container_server>"
    - name: repository
      description: The repository name
      default: "<container_repository>"
    - name: tag
      description: The image tag
      default: "<container_tag>"
  steps:
    - name: wget-spark
      image: busybox
      script: |
        wget -O /workspace/spark-3.4.1.tgz \
          https://dlcdn.apache.org/spark/spark-3.4.1/spark-3.4.1-bin-hadoop3.tgz \
        && tar -xvf /workspace/spark-3.4.1.tgz -C /workspace/ \
        && rm -f /workspace/spark-3.4.1.tgz
      volumeMounts:
      - name: docker-config
        mountPath: /workspace/.docker/
    - name: build-push-image
      image: ubuntu:22.04
      securityContext:
        privileged: true
      script: |
        apt-get update \
        && apt-get -y upgrade \
        && apt install -y podman \
        && ln -s /usr/bin/podman /usr/bin/docker \
        && docker --version \
        && cp -r /workspace/.docker /workspace/.docker-rwx \
        && cat /workspace/.docker-rwx/.dockerconfigjson > /workspace/.docker-rwx/config.json \
        && echo "[registries.search]\nregistries = ['docker.io']" > /etc/containers/registries.conf \
        && /workspace/spark-3.4.1-bin-hadoop3/bin/docker-image-tool.sh -r spark-py -t v3.4.1 -p /workspace/spark-3.4.1-bin-hadoop3/kubernetes/dockerfiles/spark/bindings/python/Dockerfile build \
        && docker tag localhost/spark-py/spark-py:v3.4.1 $(params.server)/$(params.registry)/$(params.repository):$(params.tag) \
        && docker push $(params.server)/$(params.registry)/$(params.repository):$(params.tag)
      env:
      - name: "DOCKER_CONFIG"
        value: "/workspace/.docker-rwx/"
      volumeMounts:
      - name: docker-config
        mountPath: /workspace/.docker/
  volumes:
  - name: docker-config
    secret:
      secretName: quay-secret
```

### Define a Tekton Pipeline

The `spark-image-create-pipeline` in the `tekton` namespace defines a pipeline that has a single task, `build-push-spark`. This task references another pre-defined task named `build-push-spark-image`, which is responsible for building and pushing a Spark image. Essentially, this pipeline orchestrates the creation and publication of a Spark Docker image.

```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: spark-image-create-pipeline
  namespace: tekton
spec:
  tasks:
    - name: build-push-spark
      taskRef:
        name: build-push-spark-image
```

### Define a Trigger Template

The `spark-image-pipeline-trigger-template` in the `tekton` namespace is a `TriggerTemplate` designed to initiate a pipeline run upon some external trigger, like a GitHub webhook. This template takes a parameter `gitrevision` which defaults to `master`. When triggered, it creates a `PipelineRun` of the `spark-image-create-pipeline`, naming the run with a unique identifier (`$(uid)`). This pipeline run is set to use the `spark-tekton-sa` service account and passes several container-related parameters, such as the server, tag, repository, and registry, to the pipeline. The placeholders for these values are indicated by `< >` brackets.

```
apiVersion: triggers.tekton.dev/v1alpha1 
kind: TriggerTemplate 
metadata: 
  name: spark-image-pipeline-trigger-template
  namespace: tekton
spec: 
  params: 
    - name: gitrevision 
      description: The git revision 
      default: master 
  resourcetemplates: 
    - apiVersion: tekton.dev/v1beta1 
	  kind: PipelineRun 
	  metadata: 
	    name: github-pipeline-run-$(uid) 
	    namespace: tekton
	  spec:
		serviceAccountName: spark-tekton-sa
	    pipelineRef: 
		  name: spark-image-create-pipeline
		params: 
		  - name: server
			value: "<container_server>"
		  - name: tag
			value: "<container_tag>"
		  - name: repository
			value: "<container_repository>"
		  - name: registry
			value: "<container_registry>"
```
### Define a Trigger Binding

The `TriggerBinding` named `github-binding` is designed to extract specific data from an incoming event payload, like a webhook from GitHub. In this case, it captures the latest commit ID from the GitHub webhook's payload (`$(body.head_commit.id)`) and assigns it to a parameter named `gitrevision`. This binding essentially bridges external event data to be consumed by a Tekton pipeline or task.

```
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerBinding
metadata:
  name: github-binding
spec:
  params:
    - name: gitrevision
      value: $(body.head_commit.id)
```
### Define an Event Listener

The `EventListener` named `github-event-listener` listens for external events, like webhooks from GitHub. When an event is received, it uses the `spark-tekton-sa` service account and processes the event with a defined trigger named `github-event-trigger`. This trigger binds the event data using `github-binding` and then invokes the `spark-image-pipeline-trigger-template` to initiate the corresponding Tekton actions (like running a pipeline) based on the processed event data.

```
apiVersion: triggers.tekton.dev/v1alpha1
kind: EventListener
metadata:
  name: github-event-listener
spec:
  serviceAccountName: spark-tekton-sa
  triggers:
    - name: github-event-trigger
      bindings:
        - ref: github-binding
      template:
        ref: spark-image-pipeline-trigger-template
```
### Expose the Event Listener

The `Route` in OpenShift, named `el-github-https`, directs external traffic to the `el-github-event-listener` service within the `tekton` namespace. This routing is done over the `http-listener` port. For security, it employs `edge` termination for TLS, ensuring encrypted communication. This route doesn't support wildcard subdomains (`wildcardPolicy: None`) and prioritizes the specified service with a weight of 100, ensuring all traffic is directed to it.

```
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    eventlistener: github-event-listener
  name: el-github-https
  namespace: tekton
spec:
  port:
    targetPort: http-listener
  tls:
    termination: edge
  to:
    kind: Service
    name: el-github-event-listener
    weight: 100
  wildcardPolicy: None
```

### Set Up the GitHub Webhook

- Go to your GitHub repository -> Settings -> Webhooks.
- Click "Add webhook".
- Set the "Payload URL" to the URL where your `EventListener` is accessible.
- Choose the events you wish the webhook to trigger on (e.g., "push").
- Set the content type to `application/json`.
- Save the webhook.
