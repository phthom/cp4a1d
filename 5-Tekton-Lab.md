

# TEKTON Pipeline LAB

![image-20200720135840579](images/image-20200720135840579-5246320.png)

Duration : 30 minutes



Container-based software development is growing. Since it’s easy to replicate the environment, developers generally create applications on their desktop, and debug and test them locally. Later, they build and deploy the application to a Kubernetes cluster.

In this tutorial, I show you how to deploy an application to a OpenShift cluster using a Tekton Pipeline (which is a Kubernetes-style continuous integration and continuous delivery (CI/CD) pipeline).

## Prerequisites

To complete this tutorial, you need to:

- You should have access to a cluster (credentials are given by the instructor) 

- Access a Kubernetes cluster through the `kubectl` CLI. 

- Use your  namespace labproj<xx>

- Configure the [Git CLI](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)



## Estimated time

After your prerequisites are configured, this tutorial takes about **40 minutes**.



## Task A - Build and deploy an application using Tekton Pipeline

[Tekton](https://github.com/tektoncd/pipeline) is a powerful and flexible Kubernetes-native open source framework for creating CI/CD systems. It allows you to build, test, and deploy across multiple cloud providers or on-premises systems by abstracting away the underlying implementation details.

Before I show you how to use Tekton Pipelines, here are some of the high-level concepts you need to understand:

The Tekton Pipeline project extends the Kubernetes API by five additional custom resource definitions (CRDs) to define pipelines:

- A *task* is an individual job and defines a set of build steps such as compiling code, running tests, and building and deploying images.
- A *Taskrun* runs the task you defined. With taskrun, it’s possible to execute a single task, which binds the inputs and outputs of the task.
- *Pipeline* describes a list of tasks that compose a pipeline.
- *Pipelinerun* defines the execution of a pipeline. It references the pipeline to run and which PipelineResource(s) to use as input and output.
- The *PipelineResource* defines an object that is an input (such as a Git repository) or an output (such as a Docker image) of the pipeline.

To automate the application’s build and deploy workflow using Tekton Pipelines, follow these steps:

Create and clone a github repo on your laptop:

```
cd
mkdir deploy
cd deploy
git clone https://github.com/IBM/deploy-app-using-tekton-on-kubernetes deploy-app
cd deploy-app/src
```

Check that you are still logged to your cluster:

```bash
oc status
oc login --token=<token> --server=https://c107-e.us-south.containers.cloud.ibm.com:30322
```

Check or re-assign your project:

```bash
oc project labproj<xx>
```

Results:

```bash
oc project
Using project "labproj99" on server "https://c107-e.us-south.containers.cloud.ibm.com:30322".
```



### 1. Check the Tekton Pipelines component to your Openshift cluster

Be sure your are still connected to your cluster:

```bash
# oc get nodes
NAME              STATUS    ROLES     AGE       VERSION
niceaha0.ibm.ws   Ready     compute   28d       v1.11.0+d4cacc0
niceaha1.ibm.ws   Ready     compute   28d       v1.11.0+d4cacc0
niceahi0.ibm.ws   Ready     infra     28d       v1.11.0+d4cacc0
niceahm0.ibm.ws   Ready     master    28d       v1.11.0+d4cacc0
```

As a first step, **check  the Tekton Pipelines to your OpenShift cluster** using the following command:

```
  oc get pods --namespace kabanero
```

Results

```bash
# oc get pods --namespace kabanero
NAME                                            READY     STATUS    RESTARTS   AGE
appsody-operator-549fd759c8-zp77v               1/1       Running   0          28d
controller-manager-0                            1/1       Running   3          28d
icpa-landing-b856b4d48-ndxnb                    1/1       Running   0          28d
kabanero-cli-654564cb49-cdxrt                   1/1       Running   0          28d
kabanero-landing-5d58578bf4-54vk9               1/1       Running   0          28d
kabanero-operator-8667c666bc-jk2tn              1/1       Running   0          28d
knative-eventing-operator-67cdf5dc9f-gf55k      1/1       Running   0          28d
knative-serving-operator-b64558bbc-78zhl        1/1       Running   0          28d
openshift-pipelines-operator-66c4d787cf-m8fjh   1/1       Running   0          28d
tekton-dashboard-55fd66fbff-hwzdt               2/2       Running   0          28d
webhooks-extension-65d44777-4drx5               1/1       Running   0          28d
```

You should see the last free lines concerning **Tekton**. Tekton is part of Kabanero package. 

```
openshift-pipelines-operator-66c4d787cf-m8fjh   1/1       Running   0          28d
tekton-dashboard-55fd66fbff-hwzdt               2/2       Running   0          28d
webhooks-extension-65d44777-4drx5               1/1       Running   0          28d
```

For more information on this, refer to the [Tekton documentation](https://github.com/tektoncd/pipeline/blob/master/docs/install.md#adding-the-tekton-pipelines). After completing these steps, your Kubernetes cluster is ready to run Tekton Pipelines. Let’s start by creating the definition of custom resources.



### 2. Create a PipelineResource

In this tutorial’s example, the source code of the application, Dockerfile, and deployment configuration is available in the [GitHub repository](https://github.com/IBM/deploy-app-using-tekton-on-kubernetes) that you cloned earlier.

To create the input PipelineResource to access the Git repository, do the following:

In the `git.yaml` file, define the `PipelineResource` for the Git repository by doing the following:

- Specify the resource `type` as Git.

- Provide the Git repository URL as `url`.

- Provivde `revision` as the name of the branch of the Git repository to be used.

The complete YAML file is available at `cat ../tekton-pipeline/resources/git.yaml`. 

```bash
# cat ../tekton-pipeline/resources/git.yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: git
spec:
  type: git
  params:
    - name: revision
      value: master
    - name: url
      value: https://github.com/IBM/deploy-app-using-tekton-on-kubernetes
```

As Tekton is using Customer Resource Definitions (CRDs), you will notice that Kind is set to PipelineResource.

Apply the file to the cluster as shown.

```bash
oc apply -f ../tekton-pipeline/resources/git.yaml
```

Results

```bash
# oc apply -f ../tekton-pipeline/resources/git.yaml
pipelineresource.tekton.dev/git created
```

You can check your new pipeline resource (which name is git) :

```bash
# oc get pipelineresource
NAME      AGE
git       5m

```



### 3. Create tasks

A *task* defines the steps of the pipeline. To deploy an application to a cluster using source code in the Git repository, we define two tasks — `build-image-from-source` and `deploy-to-cluster`. In the task definition, the parameters used as arguments (`args`) are referred to as `$(inputs.params.<var_name>)`.

**Define build-image-from-source**

This task includes two <u>steps</u>:

1. The `list-src` step lists the source code from the cloned repository. This is done to verify whether source code is cloned properly.
2. The `build-and-push` step builds the container image using Dockerfile and pushes the built image to the container registry. In this example, we use Kaniko to build and push the image. You can use Kaniko, builday, podman, etc. Kaniko uses the Dockerfile name, its location, and destination to upload the container image as arguments.

You can have a look to the task definition (notice kind: Task)

```bash
# cat ../tekton-pipeline/task/build-src-code.yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: build-image-from-source
spec:
  inputs:
    resources:
      - name: git-source
        type: git
    params:
      - name: pathToContext
        description: The path to the build context, used by Kaniko - within the workspace
        default: .
      - name: pathToDockerfile
        description: The path to the dockerfile to build
        default: Dockerfile
      - name: imageUrl
        description: value should be like - us.icr.io/test_namespace/builtImageApp
      - name: imageTag
        description: Tag to apply to the built image
  steps:
    - name: list-src
      image: alpine
      command:
        - "ls"
      args:
        - "$(inputs.resources.git-source.path)"
    - name: build-and-push
      image: gcr.io/kaniko-project/executor
      command:
        - /kaniko/executor
      args:
        - "--dockerfile=$(inputs.params.pathToDockerfile)"
        - "--destination=$(inputs.params.imageUrl):$(inputs.params.imageTag)"
        - "--context=$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/"
```



All required parameters are passed through parameters. Apply the file to the cluster using the following command:

```bash
  oc apply -f ../tekton-pipeline/task/build-src-code.yaml
```

Results

```bash
# oc apply -f ../tekton-pipeline/task/build-src-code.yaml
task.tekton.dev/deploy-application created
```



**Define deploy-to-cluster**

Now let’s deploy the application in a pod using the built container image, and make it available as a service to access from anywhere. This task uses the deployment configuration located as `~/src/deploy.yaml`.

This task includes two <u>steps</u>:

1. The `update-yaml` step updates the container image URL in place of `IMAGE` in the deploy.yaml.
2. The `deploy-app` step deploys the application in a Kubernetes pod and exposes it as a service using `~/src/deploy.yaml`. This step uses `kubectl` to create a deployment configuration on a Kubernetes cluster.

You can have a look to the task definition (notice kind: Task).

```bash
# cat ../tekton-pipeline/task/deploy-to-cluster.yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: deploy-application
spec:
  inputs:
    resources:
      - name: git-source
        type: git
    params:
      - name: pathToContext
        description: The path to the build context, used by Kaniko - within the workspace
        default: .
      - name: pathToYamlFile
        description: The path to the yaml file to deploy within the git source
        default: deploy.yaml
      - name: imageUrl
        description: Url of image repository
        default: url
      - name: imageTag
        description: Tag of the images to be used.
        default: "latest"
  steps:
    - name: update-yaml
      image: alpine
      command: ["sed"]
      args:
        - "-i"
        - "-e"
        - "s;IMAGE;$(inputs.params.imageUrl):$(inputs.params.imageTag);g"
        - "$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)"
    - name: deploy-app
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)"

(END)      - name: pathToContext
        description: The path to the build context, used by Kaniko - within the workspace
        default: .
      - name: pathToYamlFile
        description: The path to the yaml file to deploy within the git source
        default: deploy.yaml
      - name: imageUrl
        description: Url of image repository
        default: url
      - name: imageTag
        description: Tag of the images to be used.
        default: "latest"
  steps:
    - name: update-yaml
      image: alpine
      command: ["sed"]
      args:
        - "-i"
        - "-e"
        - "s;IMAGE;$(inputs.params.imageUrl):$(inputs.params.imageTag);g"
        - "$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)"
    - name: deploy-app
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)"

~
(END)      - name: pathToContext
        description: The path to the build context, used by Kaniko - within the workspace
        default: .
      - name: pathToYamlFile
        description: The path to the yaml file to deploy within the git source
        default: deploy.yaml
      - name: imageUrl
        description: Url of image repository
        default: url
      - name: imageTag
        description: Tag of the images to be used.
        default: "latest"
  steps:
    - name: update-yaml
      image: alpine
      command: ["sed"]
      args:
        - "-i"
        - "-e"
        - "s;IMAGE;$(inputs.params.imageUrl):$(inputs.params.imageTag);g"
        - "$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)"
    - name: deploy-app
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)"
```

All required parameters are passed through parameters.

Apply the file to the cluster as:

```bash
  oc apply -f ../tekton-pipeline/task/deploy-to-cluster.yaml
```

Results

```bash
# oc apply -f ../tekton-pipeline/task/deploy-to-cluster.yaml
task.tekton.dev/deploy-application created
```

You can check your 2 tasks:

```bash
# oc get tasks
NAME                      AGE
build-image-from-source   6m
deploy-application        11s

```

You can also have a description of a specific task:

```bash
# oc describe task deploy-application
Name:         deploy-application
Namespace:    labproj03
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"tekton.dev/v1alpha1","kind":"Task","metadata":{"annotations":{},"name":"deploy-application","namespace":"labproj03"},"spec":{"inputs":{"...
API Version:  tekton.dev/v1alpha1
Kind:         Task
Metadata:
  Creation Timestamp:  2020-03-17T10:33:24Z
  Generation:          1
  Resource Version:    7129987
  Self Link:           /apis/tekton.dev/v1alpha1/namespaces/labproj03/tasks/deploy-application
  UID:                 bb06a9ea-683a-11ea-a7d7-065675a7ed1c
Spec:
  Inputs:
    Params:
      Default:      .
      Description:  The path to the build context, used by Kaniko - within the workspace
      Name:         pathToContext
      Default:      deploy.yaml
      Description:  The path to the yaml file to deploy within the git source
      Name:         pathToYamlFile
      Default:      url
      Description:  Url of image repository
      Name:         imageUrl
      Default:      latest
      Description:  Tag of the images to be used.
      Name:         imageTag
    Resources:
      Name:  git-source
      Type:  git
  Steps:
    Args:
      -i
      -e
      s;IMAGE;$(inputs.params.imageUrl):$(inputs.params.imageTag);g
      $(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)
    Command:
      sed
    Image:  alpine
    Name:   update-yaml
    Args:
      apply
      -f
      $(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)
    Command:
      kubectl
    Image:  lachlanevenson/k8s-kubectl
    Name:   deploy-app
Events:     <none>


```





### 4. Create a pipeline

A pipeline lists the tasks to be executed. It provides the input, output resources, and input parameters required by each task. If there is any dependency between the tasks, that is also addressed.

In the `tekton-pipeline/resources/pipeline.yaml`:

- A pipeline uses the above mentioned tasks `build-image-from-source` and `deploy-to-cluster`.
- The `runAfter` key is used here because the tasks need to be executed one after the other.
- The PipelineResource (Git repository) is provided through the `resources` key.

You can have a look to the pipeline definition (notice kind: pipeline).

```bash
# cat ../tekton-pipeline/pipeline/pipeline.yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: application-pipeline
spec:
  resources:
    - name: git-source
      type: git
  params:
    - name: pathToContext
      description: The path to the build context, used by Kaniko - within the workspace
      default: .
    - name: pathToYamlFile
      description: The path to the yaml file to deploy within the git source
      default: config.yaml
    - name: imageUrl
      description: Url of image repository
      default: deploy_target
    - name: imageTag
      description: Tag to apply to the built image
      default: latest
  tasks:
  - name: build-image-from-source
    taskRef:
      name: build-image-from-source
    params:
      - name: pathToContext
        value: "$(params.pathToContext)"
      - name: imageUrl
        value: "$(params.imageUrl)"
      - name: imageTag
        value: "$(params.imageTag)"
    resources:
      inputs:
        - name: git-source
          resource: git-source
  - name: deploy-application
    taskRef:
      name: deploy-application
    runAfter:
      - build-image-from-source
    params:
      - name: pathToContext
        value: "$(params.pathToContext)"
      - name: pathToYamlFile
        value: "$(params.pathToYamlFile)"
      - name: imageUrl
        value: "$(params.imageUrl)"
      - name: imageTag
        value: "$(params.imageTag)"
    resources:
      inputs:
        - name: git-source
          resource: git-source
```



All required parameters are passed through parameters. The parameters value is defined in the pipeline as `$(params.imageUrl)` which is different than the `args` in the task definition. Apply this configuration as:

```bash
  oc apply -f ../tekton-pipeline/pipeline/pipeline.yaml
```

Results

```bash
# oc apply -f ../tekton-pipeline/pipeline/pipeline.yaml
pipeline.tekton.dev/application-pipeline created
```

At this point, you only have set up several definitions. Nothing has been yet executed.



### 5. Create PipelineRun

To execute the pipeline, you need a `PipelineRun` resource definition, which passes all required parameters. `PipelineRun` triggers the pipeline, and the pipeline, in turn, creates `TaskRuns` and so on. In a similar manner, all parameters get substituted down to the tasks.

If a parameter is not defined in the `PipelineRun`, then the default value gets picked up from the `params` under `spec` from the resource definition itself. For example, the `pathToDockerfile` parameter is used in the task `build-image-from-source`, but its value is not provided in `pipeline-run.yaml`. Because of this, its default value defined in `~/tekton-pipeline/build-src-code.yaml` is used during the task execution.

In the `PipelineRun` definition, `tekton-pipeline/pipeline/pipeline-run.yaml`:

- References the pipeline `application-pipeline` created through `pipeline.yaml`.
- References the PipelineResource `git` to use as input.
- Provides the value of parameters under `params` which are required during the execution of the pipeline and the tasks.
- Specifies a service account.

Note that through the pipeline, you can push images to the registry and deploy it to a cluster. You need to ensure that it has the sufficient privileges to access the container registry and the cluster. The credentials for the registry are provided by a service account. You need to define a service account before executing `PipelineRun`.

*Note: Do not apply the PipelineRun file yet because you still need to define the service account for it.*

### 6. Execute the pipeline

Before executing the `PipelineRun`, modify the `imageUrl` and the `imageTag` in `tekton-pipeline/pipeline/pipelinerun.yaml`. Refer to the Set up deploy target section above to decide on an image URL and tag. If the image URL is *us.icr.io/test_namespace/builtApp* and the image tag is *1.0*, then update the configuration file to:

Edit the following file:

```
../tekton-pipeline/pipeline/pipeline-run.yaml
```

Replace the following parameters :

- IMAGE_URL  with 			us.icr.io/labproj99/hello
- IMAGE_TAG  with                  1.0
- Also change your project<xx> at the end of the file

  You can look at the file before executing the pipelinerun:

```bash
# cat ../tekton-pipeline/pipeline/pipeline-run.yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  name: application-pipeline-run
spec:
  pipelineRef:
    name: application-pipeline
  resources:
    - name: git-source
      resourceRef:
        name: git
  params:
    - name: pathToContext
      value: "src"
    - name: pathToYamlFile
      value: "deploy.yaml"
    - name: "imageUrl"
      value: "docker-registry.default.svc:5000/labproj99/hello"
    - name: "imageTag"
      value: "1.0"
  serviceAccountName: pipeline
```



Now, create the `PipelineRun` configuration, like so:

```
  oc create -f ../tekton-pipeline/pipeline/pipeline-run.yaml
```

This creates a pipeline with the below message on your terminal:

```
  # oc create -f ../tekton-pipeline/pipeline/pipeline-run.yaml
  pipelinerun.tekton.dev/application-pipeline-run created
```

Check the status of your new pipeline:

```
  oc describe pipelinerun application-pipeline-run
```

You may need to rerun this command based on the status. It shows the interim status as:

```
Status:
  Conditions:
    Last Transition Time:  2019-11-11T06:51:06Z
    Message:               Not all Tasks in the Pipeline have finished executing
    Reason:                Running
    Status:                Unknown
    Type:                  Succeeded

   ...
   ...
   Events:              <none>
```

Note the message where it tells you that “**Not all Tasks in the Pipeline have finished executing**.”

Once the execution of your pipeline is complete, you should see the following as an output of the `describe` command. The Message now reads: “All Tasks have completed executing.”

```
Status:
  Completion Time:  2019-11-07T09:41:59Z
  Conditions:
    Last Transition Time:  2019-11-07T09:41:59Z
    Message:               All Tasks have completed executing
    Reason:                Succeeded
    Status:                True
    Type:                  Succeeded
..
..
Events:
  Type    Reason     Age   From                 Message
  ----    ------     ----  ----                 -------
  Normal  Succeeded  7s    pipeline-controller  All Tasks have completed executing
```

In case of **failure**, it shows which task has failed. It also gives you the additional details to check logs. For details about a resource (for instance, the pipeline) use the `oc describe` command to get more information. You must **delete the pipelinerun** before a new execution.

```
  oc delete pipelinerun application-pipeline-run
```

### 7. Verify your results

To verify whether the pod and service is running as expected, check the output of the following commands:

```
  # kubectl get pods
  NAME                                                              READY   STATUS      RESTARTS   AGE
app-d78b485cb-567ql                                               1/1     Running     0          69m
application-pipeline-run-build-image-from-source-4ct9x-po-t6lxj   0/3     Completed   0          2m21s
application-pipeline-run-deploy-application-km86w-pod-kdj9j       0/3     Completed   0          100s

# kubectl get service
    NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
    app          NodePort    xxx.xx.xx.xxx   <none>        3300:32426/TCP   4m51s
```

After successful execution of `PipelineRun`, the application is accessible at `http://<public-ip-of-kubernetes-cluster>:32426/`, where you can retrieve the public IP of your Kubernetes cluster from your IBM Cloud dashboard, and the port 32426 is defined as `nodePort` in `deploy.yaml`.

Congratulations! You successfully deployed your application using a Tekton Pipeline. You should now understand the basics of Tekton Pipelines and how to get started on building your own. There are more features available, including webhooks and web-based dashboards. I suggest trying it out with IBM Cloud Kubernetes Service.

## End of Lab

Hopefully this tutorial showed you how and why you should use Tekton Pipelines for deploying an application to Kubernetes. Tekton is included in [IBM Cloud Pak for Applications](https://cloud.ibm.com/catalog/content/ibm-cp-applications?cm_sp=ibmdev-_-developer-tutorials-_-cloudreg), which offer a faster, more secure way to move your business applications to the cloud, in a container-enabled environment. Cloud Pak for Applications is built and supported on Red Hat OpenShift. Explore and try out more with the help of this [developer guide](https://developer.ibm.com/series/developers-guide-to-ibm-cloud-pak-for-applications/).




