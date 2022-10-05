# the CNBi flow

[![hackmd-github-sync-badge](https://hackmd.io/6BgU6fdSRvyMLZNPwGykfg/badge)](https://hackmd.io/6BgU6fdSRvyMLZNPwGykfg)


## Proposal

We will introduce a new custom resource defintition (CRD) — called `CustomNBImage` — to be:

1. the interface between UI and service and
2. contain all the configuration items required for any CNBi use case.

A first draft of the [CNBi CRD](https://github.com/thoth-station/meteor-operator/blob/main/config/samples/meteor_v1alpha1_customnbimage.yaml) is available.

We will create a custom notebook image controller, that will reconcile the state of CNBi custom resource objects.

The ODH dashboard interfaces with the `CustomNBImage` CR:
 - for creating new custom notebooks
 - to get the state of currently building custom notebooks

### Relationship Diagram

> for cardinality notation of entity relationship diagram see https://vertabelo.com/blog/crow-s-foot-notation/

```mermaid
erDiagram
    CNBiController ||--|{ PipelineRun : creates
    PipelineRun ||--|{ Image : creates
    PipelineRun ||--|{ ImageStreamTag : creates
    ImageStreamTag ||--|| Image : references

    PipelineRun {
        string git
    }
    Image {
        string spec
    }
    ImageStreamTag {
        string tag
        string ref
}
```

## Import Image from an external Registry ([`ImportImage`](https://github.com/thoth-station/meteor-operator/blob/main/api/v1alpha1/customnbimage_types.go#L52))

The ODH dashboard creates a `CustomNBImage` (shortname: `cnbi`). _Note: the yellow box denotes the ownership (by Meteor's CNBi) of resources._

```mermaid
flowchart LR
    O[ODH] ==> CR[/CustomNBImage/]
    subgraph CNBi
        direction TB
        CR -.- C[CNBi controller] ==> PR[/PipelineRun/]
        subgraph Import
            PR
        end
    end
    PR --> IS[/ImageStream/] -.-> JH[JupyterHub]
    I[(Image)] -.-> PR & JH
```




### Example

This is a proposal for an import of an Image. It carries information according to [Open Data Hub annotations](https://github.com/opendatahub-io/jupyterhub-singleuser-profiles/blob/master/jupyterhub_singleuser_profiles/images.py#L10-L19) only `created-by` is interpreted in a different way. Annotations are passed to the PipelineRun.


```
apiVersion: meteor.zone/v1alpha1
kind: CustomNBImage
metadata:
  name: s2i-minimal-py38-notebook
  annotations:
    opendatahub.io/notebook-image-name: s2i-minimal-py38-notebook
    opendatahub.io/notebook-image-desc: minimal notebook image for python 3.8
    opendatahub.io/notebook-image-creator: goern
spec:
  buildType: ImageImport
  fromImage: quay.io/thoth-station/s2i-minimal-py38-notebook:v0.2.2
```

### States and Conditions

Phase 1 BYON import state diagram from https://github.com/open-services-group/byon/issues/23#issuecomment-1055586737

```mermaid
    stateDiagram-v2
        [*] --> Importing
        Importing --> Validating
        Validating --> Success
        Validating --> Failure
        Success --> [*]
        Failure --> [*]

        note right of Importing
            Import pipeline is scheduled but
            ImageStream was not yet created
        end note

        note left of Validating
            Import pipeline is running,
            phase can be sourced from
            ImageStream annotation
        end note
  ```

### Conditions

* failure: Importing could fail with a "auth required" condition
* failure: ImageStream resource cant be created
* failure: image does not meet the minimum requirements
* success: ImageStreamTag has been created due to a new upstream tag
* pipelineTaskName: setup: ??
* pipelineTaskName: create-imagestream: Succeeded=Failed: "......"
* pipelineTaskName: update-imagestream: Succeeded=Failed: "......"
* pipelineTaskName: validate: Succeeded=TaskRunImagePullFailed: pick up message


A demo of a similar `CustomNBImage` (using an earlier iteration of the CRD) in action is available here: https://asciinema.org/a/517335

## Build based on a list of Python packages ([`PackageList`](https://github.com/thoth-station/meteor-operator/blob/main/api/v1alpha1/customnbimage_types.go#L57))

This build type is used to create a custom notebook image based on a runtime environment (OS and Python version) or a base image and a set of Python packages which should be installed in the runtime environment. The list of packages being installed could be generated by `pipenv` or Thoth Guidance Service.

The PipelineRun created by the CNBi Controller will create a new container image and the corresponding ImageStreamTag (with all the annotations required by ODH).

```mermaid
flowchart TB
    O[ODH] ==> CR[/CustomNBImage/]
    subgraph CNBi Operator
        direction LR
        CR -.- C[CNBi controller] ==> PRprepare[/PipelineRun: prepare/]
        subgraph OpenShift Pipelines
            B[(base image)] & g[(git)] -.uses.-> PRprepare
            PRprepare --> PRbuild[/PipelineRun: build/]
            PRbuild --> PRvalidate[/PipelineRun: validate/]
        end
    end
    PRvalidate --> I[(Image)] & IS[/ImageStream/] -.-> JH[JupyterHub]
    O --unsure about this?!--> g
```

### Example

This example shows how to build a notebook image using a specific runtime environment and a
list of packages.

```
apiVersion: meteor.zone/v1alpha1
kind: CustomNBImage
metadata:
  name: ubi8-py38-sample-3
[...]
spec:
  buildType: PackageList
  runtimeEnvironment:
    osName: ubi
    osVersion: "8"
    pythonVersion: "3.8"
  packageVersions:
    - pandas
    - boto3>=1.24.0
```


This next example shows how to declare a build based on an existing container image, updating or adding packages:


```
apiVersion: meteor.zone/v1alpha1
kind: CustomNBImage
metadata:
  name: ubi8-py38-sample-3
[...]
spec:
  buildType: PackageList
  builderImage: quay.io/thoth-station/s2i-minimal-py38-notebook:v0.2.2
  packageVersions:
    - pandas
    - boto3>=1.24.0
```

## Build from a GitHub Repository ([`GitRepository`](https://github.com/thoth-station/meteor-operator/blob/main/api/v1alpha1/customnbimage_types.go#L60))

This example shows how to declare a build of a GitHub repository that contains notebooks:

```
apiVersion: meteor.zone/v1alpha1
kind: CustomNBImage
metadata:
  name: ubi8-py38-sample-3
[...]
spec:
  buildType: GitRepository
  repositoryUrl: https://github.com/AICoE/elyra-aidevsecops-tutorial
  gitRef: master
```


## CustomNBImage state diagram

The CNBi resource might be in a set of states, they reflect the overall state of the resource, for
example: 'can not import a container image, becuase the registry requires authentication' leads to a failed state. On the other hand, a 'can not push to internal registry' is considered an CNBi internal error that might be reconciled.

The following diagram shows the CNBi states.

```mermaid
stateDiagram-v2
    [*] --> Pending

    Pending --> Building
    Pending --> Failed
    Building --> Ready
    Building --> Failed

    Failed --> [*]
    Ready --> [*]
```

The internal states while reconciling will be described in the use case specific chapters below. Following [Typical status properties](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties) we will add use case-specific conditions to the `.status` of a CNBi resource






## FAQ

> question: do we keep the git repo internall to the pipelinerun or do we push it to somewhere for later use? is the repo base-url a config of the controller? @codificat

If we want the possibility that the user can point to a git repo to request a build, then the git repo must be exposed at the CustomNBImage level.

Therefore, the git repo should be in the CNBi CR.

> Follow-up question: can it be, though, that for some use cases it is in the `spec` (e.g. "I want to build from that repo") while for others might only appear in the `status` (e.g. "FYI this is where your source of truth is being kept")?

> question: do we have multiple pipelinerun for prepare and build or just one? is 'prepare' specific to use case and 'build' agnostic? @FIkOzrY0QJa6x7Z2vsT1UQ @codificat

Let's confirm:
- *Prepare* involves getting the git repo up to date with the necessary information. This can involve e.g. updating `requirements.txt` (possibly with Thoth advice)

Some use cases, like *Import an image (AKA BYON)* do not need such git repo preparation.

In any case, I believe that by default the approach would be: (potentially) multiple Tasks, single Pipeline(Run).

One exception / potential reason to have multiple Pipeline(Run)s would be if the UX for a certain use case involves multiple steps where each step deserves a separate PR. In other words: if the git repo preparation has its own UX workflow, then it needs a dedicated Pipeline(Run).


## Things to discuss/review

### Using a git repository as _the_ source of truth

Use **git** as the go-to place for permanent storage of the build resources and configuration

### Deployment of the operator

#### Namespaced resources?

The current approach of `meteor-operator` follows the *operator-sdk* pattern:
- A namespace is created to host the operator
- The operator handles CRs in any other namespace

Things to consider:
- The pipelines and tasks we have are namespaced
- KfDef does not like the current `meteor-operator/config` kustomization layout

#### Deployment of the operator

- Standalone: (see the [Makefile](https://github.com/goern/meteor-operator/blob/3ea3b5a75c5333ac60a0d0dde2cc57d4fcdc1c71/Makefile#L127))
  1. `make install-pipelines`
  2. `make deploy`

- Via KfDef: [attempt on os-c stage](https://github.com/operate-first/apps/pull/2398/files)
- odh-manifests (like byon pipelines [are deployed in os-c stage](https://github.com/operate-first/odh-manifests/blob/34b43f6b2e2fe1195b37709736a89d64c3bf4411/jupyterhub/jupyterhub/overlays/byon/kustomization.yaml))

#### Deployment of the pipeline manifests

We should agree on:
- where to host the pipeline manifests. e.g. the thoth-station/helm-charts repo
- how to deploy the pipeline manifests in ODH


### DONE Single or multiple Custom Resources?

We have different use cases that require different actions, and therefore different pipelines to be run. How to handle that?

Alternatives considered:
1. continue with a single `CustomNBImage` CRD, and have a field that determines the action (e.g. *import*, *build image*, *create image*...)
2. the same but without an explicit field; deduce the action (and therefore the pipeline) from the parameters that are defined. I don't quite like that - explicit better than implicit.
3. have multiple CRDs, e.g. one per action type, with specific fields (e.g. `CustomNBImageImport` points to an image to import, `CustomNBImageBuild` points to a repo..)

We are going for option 1: a single `CustomNBImage` resource with a `buildType` field that determines the actions to carry out.



## References

This section contains pointers to other components that are relevant to this effort.

### BYON import

The [BYON pipelines](https://github.com/open-services-group/byon/blob/3b23be51f6ea49507a03263bb2f9a48c3d8e6ed0/Makefile#L50-L67) expect the following parameters:

- url: points to the image in the registry
- name: to show in JH spawner (?)
- desc:
- creator:

#### Background: Import (the BYON way)

For the phase 1 *Bring your own notebook (BYON)* functionality, the ODH dashboard creates a `PipelineRun`.

```mermaid
flowchart LR
    subgraph BYON Import
        PR[/PipelineRun/]
    end
    O[ODH] ==> PR --> IS[/ImageStream/] -.-> JH[JupyterHub]
    I[(Image)] -.-> PR & JH
```

### Meteor build resource

Here is how a [Meteor custom resource](https://github.com/AICoE/meteor-operator/blob/34731bb723ba13d8d1bb214d4626bb02438af72e/config/samples/meteor_v1alpha1_meteor.yaml) looks like:

```yaml
apiVersion: meteor.zone/v1alpha1
kind: Meteor
metadata:
  name: demo
spec:
  url: https://github.com/aicoe-aiops/meteor-demo
  ref: main
  ttl: 5000
```
