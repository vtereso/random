# PipelineResources Direction

## Background
Tekton `Pipelines`/`Tasks` separate functionality from configuration by acting as reusable components for work.
Work in actualized/instantiated in the corresponding `PipelineRun`/`TaskRun` objects (noted as Run objects onward for the sake of brevity).
Run objects store the work configuration through both `PipelineResources` and parameters.


## Goals
- Understand the true intent/purpose behind `PipelineResources`.
- Discuss potential improvements and/or update docs


## Introduction
According to the [documentation](https://github.com/tektoncd/pipeline/blob/master/docs/resources.md#pipelineresources):
> PipelinesResources in a pipeline are the set of objects that are going to be used as inputs to a Task and can be output by a Task.

This does not not explain their purpose, but rather their current embodiment.
To the case of resuability, it makes a lot of sense to structure things in a functional way (e.g. f(x)=y) where `Pipelines`/`Tasks` are the functions that ingest configuration. 
In contrast, it seems that `PipelineResources` seem to fall somewhere in-between.


# PipelineResource Role
Although each kind of `PipelineResource` does something different, I believe it is reasonable to consider them as syntactically sweet mounts.
In general, mounts are really helpful because they allow some foreign information to be attached to a pod/container.
Without mounts, there would only be two options:
- Create wrapper images around base images that expect configuration to be injected
- Have functionality and configuration coupled, which is less than ideal

To this point, Kubernetes supports volume mounts within `PodSpec`.
`PipelineResources` act as a declarative adapter/bridge between the data we want to mount, without having to generate it ourselves.
Since there are common usecases, especially pertaining to CI/CD, there are multiple kinds of `PipelineResources` that help us to this end.
Let's look at a few different `PipelineResources`:

### Git Resource
> Git resource represents a git repository, that contains the source code to be built by the pipeline. Adding the git resource as an input to a Task will clone this repository and allow the Task to perform the required actions on the contents of the repo.

### Pull Request Resource
> Adding the Pull Request resource as an input to a Task will populate the workspace with a set of files containing generic pull request related metadata such as base/head commit, comments, and labels.
>
> ...
>
> Adding the Pull Request resource as an output of a Task will update the source control system with any changes made to the pull request resource during the pipeline.

In just these two use cases (although there are more), `PipelineResources` can help simplify mounting.
However, there is a bit of quirkiness between `PipelineResources`.
Some resources like `GitResources` seem to only be input resources, while `PullRequestResources` are both, but likely never as inputs to other `Tasks`.
It would be nice if there was some sort of contract/convention that was upheld between `PipelineResources` so that they could be better defined.
Maybe just different classifications?


# PipelineResources Problems
As mentioned in the [background section](##Background), Run objects take parameters *and* `PipelineResources`.
Since these are distinct objects, `PipelineResources` need to be created ahead of time (separately), which is an inconvenience.
As a byproduct, there is currently a [proposal](https://docs.google.com/document/d/1fF2vWMs12d3FwkqkNuzS7FzQFQSFz1ZKUe6CHqkIg0c) to allow for `PipelineResource` embedding into the `PipelineRun` (although it remains a distinct k8s resource) to address this as well as resource littering.
In any case, `PipelineResources` can be tampered with/deleted.
This introduces a cookie-cutter/templating problem, where Run objects utilizing `PipelineResources` (at least as current) cannot determine whether they have been modified during or between runs.
This could potentially be resolved by capturing the values of the `PipelineResources` in the Run status internally at start time.
This way, there would be an audit trail.
Another issue is extensability.
Since the `tektoncd/pipelines` run reconciler internally resolves the before/after steps, forking the repository is, at least currently, seemingly the only way to add new types.



# Potential Improvements

## Idea 1: Extend PipelineResources
In order to extend `PipelineResources`, in whatever manifestation, it would be sensible that users would create a controller that would reconciler _something_ to this end. 
With this assumption, why not do the same natively?
For each type of `PipelineResource`, have an informer/reconciler on CUD (Create|Update|Delete) operations.

In order to detect this new `PipelineResource`, some labels could be applied to some resource kind that would indicate it as a `PipelineResource` as well as its `PipelineResource` kind.
For example, `PipelineResources` could be modeled as `ConfigMaps`; they are supported natively and have a dictionary/map in them.
This would make allow for `Kustomize` or other such tools to create `"PipelineResources"` since they are core k8s objects.
As a slight con, `ConfigMaps` are their own type, so the aforementioned reconciler would be called for any `ConfigMap` CUD operation.
Alternatively, this could be accomplished with some new standardized `PipelineResource` object (likely with a map as well), where the informer would reconcile based on only a type label/field.

With either object, another issue is how to forward information to the Run object since their form is unknown to the Run reconciler.
One way would be to have an owner reference on `PipelineResources` where each kind of `PipelineResource` reconciler is responsible for updating some correlated status on the Run object parent; once all of these statuses are ready, the Run object would start.
Since updating the status calls the reconciler again, this is probably less than ideal (racing between reconcilers).
The other approach would be for the `PipelineResources` to update their own status and have the Run reconciler await this.
This is mostly a rewording of that within the [extensibility doc](https://docs.google.com/document/d/1rcMG1cIFhhixMSmrT734MBxvH3Ghre9-4S2wbzodPiU/edit?ts=5d544481), but with a single resource kind.


## Idea 2: Surface PipelineResources as Declarative Mounts
Currently, `PipelineResources` resolve their behavior first party by [checking against some predetermined list of `PipelineResources`](https://github.com/tektoncd/pipeline/blob/master/pkg/apis/pipeline/v1alpha1/resource_types.go#L138) to determine what steps to add to the Run object.
It seems like Run objects could provide a preMount (and likely postMount) field that would take some image that would act as the declarative mounting functionality analogous to what `PipelineResources` do currently.

In this way, the same images being used for `PipelineResources` would be used as mount images.
As a result, Runs would have nothing but parameters so that the only resources that would need to be garbage collected are the Runs objects themselves.

