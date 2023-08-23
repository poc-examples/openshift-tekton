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

This YAML is for creating a `Secret` in Kubernetes, specifically for Docker authentication.  It can be used by a task in order to auth into the your registry in order to push images.  Use non base64encoded secrets when applying these or they will be base64encoded twice preventing git-clone and other credentials from working properly.

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

A ServiceAccount in Kubernetes offers identity for pod processes. When you access the cluster, you're identified by a user account, while processes inside pods are authenticated as a specific ServiceAccount. This "tekton-sa" ServiceAccount might grant permissions, like database access, to workloads within the Tekton pipeline.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-sa
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

**Note**: You may require more permissions based on what you're trying to achieve. You can tail the logs for the eventlistener which will throw errors if you are missing a permission.  This is a good place to start for fixing issues around the pipeline not starting.

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

A `ClusterRoleBinding` named "tekton-eventlistener-binding" is also established. This binding connects the "tekton-eventlistener-cluster-role" to a ServiceAccount named "tekton-sa" in the "tekton" namespace. Essentially, it grants the permissions defined in the ClusterRole to any pod running with the "tekton-sa" ServiceAccount.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tekton-eventlistener-binding
subjects:
- kind: ServiceAccount
  name: tekton-sa
  namespace: tekton
roleRef:
  kind: ClusterRole
  name: tekton-eventlistener-cluster-role
  apiGroup: rbac.authorization.k8s.io
```
### Define Tekton Tasks

The `git-clone`|`podman-build`|`podman-push` Tasks in the `tekton` namespace automates the process of cloning a github repo, building a Docker image for it using Podman, and pushing the image to a specified container registry. The Task leverages a secret named `quay-secret` mentioned above for authentication during the image push process.

```
---
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: podman-build
  namespace: tekton
spec:
  workspaces:
    - name: source
    - name: containers
      mountPath: /var/lib/containers
  params:
    - name: IMAGE_NAME
      description: The name of the image
    - name: IMAGE_TAG
      description: The image tag
    - name: DOCKERFILE
      description: The path to dockerfile
      default: /workspace/source
  steps:
    - name: build-image
      resources:
        limits:
          memory: "4Gi"
        requests:
          memory: "1Gi"
      image: quay.io/rh_ee_cengleby/podman:0.12
      securityContext:
        privileged: true
        runAsUser: 0
      script: |
        podman build -t $(params.IMAGE_NAME):$(params.IMAGE_TAG) $(params.DOCKERFILE)
---
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: podman-push
  namespace: tekton
spec:
  workspaces:
    - name: containers
      mountPath: /var/lib/containers
    - name: credentials
      mountPath: /workspace/.docker/
  params: 
    - name: IMAGE_NAME
      description: The name of the image
    - name: REGISTRY_HOSTNAME
      description: The hostname of the registry server (quay.io)
    - name: REGISTRY_NAMESPACE
      description: The repository namespace
    - name: IMAGE_TAG
      description: The image tag
      default: "latest"
  steps:
    - name: build-image
      image: quay.io/rh_ee_cengleby/podman:0.12
      securityContext:
        privileged: true
        runAsUser: 0
      script: |
        podman tag $(params.IMAGE_NAME):$(params.IMAGE_TAG) $(params.REGISTRY_HOSTNAME)/$(params.REGISTRY_NAMESPACE)/$(params.IMAGE_NAME):$(params.IMAGE_TAG) \
        && podman push $(params.REGISTRY_HOSTNAME)/$(params.REGISTRY_NAMESPACE)/$(params.IMAGE_NAME):$(params.IMAGE_TAG)
      env:
        - name: "DOCKER_CONFIG"
          value: "/workspace/.docker/"
```

### Define a Tekton Pipeline

The `example-image-build-pipeline` in the `tekton` namespace defines a pipeline that has a multiple tasks, `git-clone`|`podman-build`|`podman-push`. These tasks are responsible for building and pushing a Spark image. Essentially, this pipeline orchestrates the creation and publication of a Docker image hosted in a github repo.

```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: example-image-build-pipeline
  namespace: tekton
spec:
  params:
    - name: IMAGE_NAME
    - name: REGISTRY_HOSTNAME
    - name: REGISTRY_NAMESPACE
    - name: IMAGE_TAG
    - name: GITHUB_REPO
    - name: GITHUB_BRANCH
  workspaces:
    - name: source
    - name: containers
    - name: credentials
  tasks:
    - name: git-clone
      params:
        - name: url
          value: $(params.GITHUB_REPO)
        - name: revision
          value: $(params.GITHUB_BRANCH)
        - name: sslVerify
          value: 'false'
        - name: noProxy
          value: 'true'
      taskRef:
        kind: ClusterTask
        name: git-clone
      workspaces:
      - name: output
        workspace: source
    - name: container-build
      params:
        - name: IMAGE_NAME
          value: $(params.IMAGE_NAME)
        - name: IMAGE_TAG
          value: $(params.IMAGE_TAG)
      taskRef:
        name: podman-build
      runAfter: 
      - git-clone
      workspaces:
      - workspace: containers
        name: containers
      - workspace: source
        name: source
    - name: container-push
      params:
        - name: IMAGE_NAME
          value: $(params.IMAGE_NAME)
        - name: IMAGE_TAG
          value: $(params.IMAGE_TAG)
        - name: REGISTRY_HOSTNAME
          value: $(params.REGISTRY_HOSTNAME)
        - name: REGISTRY_NAMESPACE
          value: $(params.REGISTRY_NAMESPACE)
      taskRef:
        name: podman-push
      runAfter: 
      - container-build
      workspaces:
      - workspace: containers
        name: containers
      - workspace: credentials
        name: credentials
```

### Define a Trigger Template

The `spark-image-pipeline-trigger-template` in the `tekton` namespace is a `TriggerTemplate` designed to initiate a pipeline run upon some external trigger, like a GitHub webhook. This template takes a parameter `gitrevision` which defaults to `master`. When triggered, it creates a `PipelineRun` of the `spark-image-create-pipeline`, naming the run with a unique identifier (`$(uid)`). This pipeline run is set to use the `tekton-sa` service account and passes several container-related parameters, such as the server, tag, repository, and registry, to the pipeline. The placeholders for these values are indicated by `< >` brackets.

```
apiVersion: triggers.tekton.dev/v1alpha1 
kind: TriggerTemplate 
metadata: 
  name: image-pipeline-trigger-template
  namespace: tekton
spec: 
  params: 
    - name: gitrevision 
      description: The git revision sha
  resourcetemplates: 
    - apiVersion: tekton.dev/v1beta1 
      kind: PipelineRun 
      metadata: 
        name: github-pipeline-run-$(uid)
        namespace: tekton
      spec:
        serviceAccountName: tekton-sa
        pipelineRef: 
          name: example-image-build-pipeline
        params: 
          - name: IMAGE_NAME
            value: "your_image" 
          - name: REGISTRY_HOSTNAME
            value: "quay.io" 
          - name: REGISTRY_NAMESPACE
            value: "your_namespace" 
          - name: IMAGE_TAG
            value: $(tt.params.gitrevision)
          - name: GITHUB_REPO
            value: 'https://github.com/your_repo/bc-test-image.git'
          - name: GITHUB_BRANCH
            value: 'main'
        podTemplate:
          securityContext:
            runAsUser: 1001
            fsGroup: 65532
        workspaces:
        - name: source
          persistentVolumeClaim:
            claimName: clone-output
        - name: containers
          persistentVolumeClaim:
            claimName: varlibcontainers
        - name: credentials
          secret:
            secretName: quay-secret
```
### Define a Trigger Binding

The `TriggerBinding` named `github-binding` is designed to extract specific data from an incoming event payload, like a webhook from GitHub. In this case, it captures the latest commit sha from the GitHub webhook's payload (`$(body.after)`) and assigns it to a parameter named `gitrevision`. This binding essentially bridges external event data to be consumed by a Tekton pipeline or task.

```
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerBinding
metadata:
  name: github-binding
  namespace: tekton
spec:
  params:
    - name: gitrevision
      value: "$(body.after)"
```
### Define an Event Listener

The `EventListener` named `github-event-listener` listens for external events, like webhooks from GitHub. When an event is received, it uses the `tekton-sa` service account and processes the event with a defined trigger named `github-event-trigger`. This trigger binds the event data using `github-binding` and then invokes the `image-pipeline-trigger-template` to initiate the corresponding Tekton actions (like running a pipeline) based on the processed event data.

```
apiVersion: triggers.tekton.dev/v1alpha1
kind: EventListener
metadata:
  name: github-event-listener
  namespace: tekton
spec:
  serviceAccountName: tekton-sa
  triggers:
    - name: github-event-trigger
      bindings:
        - ref: github-binding
          kind: TriggerBinding
      template:
        ref: image-pipeline-trigger-template
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
- Customize this webhook as called for by your usecase.