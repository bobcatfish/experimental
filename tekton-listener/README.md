# Tekton Listener and Event Bindings

This experimental directory defines two new CRDs - TektonListeners and EventBindings.

The `TektonListener` is a CRD which provides a listener component, which can listen for CloudEvents and spawn specific PipelineRuns as a result.

The `EventBinding` CRD exposes a new, high-level concept of "binding" Events with a specified Pipeline. The EventBinding takes care of creating and deleting PipeLineResources and also spawns `TektonListener`s to handle event ingress and processing.

## TektonListener
The first new CRD, `TektonListener`, provides support for consuming CloudEvent and producing a predefined PipelineRun. Although only CloudEvents are currently supported, the listener is intentionally designed to allow for extension beyond CloudEvents.

An example TektonListener:
```
apiVersion: tektonexperimental.dev/v1alpha1
kind: TektonListener
metadata:
  labels:
    app: ulmaceae
  name: ulmaceae-binding-listener
  namespace: ulmaceae
spec:
  event: cloudevent
  namespace: ulmaceae
  pipelineRef:
    name: ulmaceae-pipeline
  runspec:
    pipelineRef:
      name: ulmaceae-pipeline
    resources:
    - name: source-repo
      resourceRef:
        apiVersion: v1alpha1
        name: source-repo
    - name: image-ulmaceae
      resourceRef:
        apiVersion: v1alpha1
        name: image-ulmaceae
    serviceAccount: ulmaceae-account
```

Since the Service fullfills the [Addressable](https://github.com/knative/eventing/blob/master/docs/spec/interfaces.md#addressable) contract, the listener service can be used as a sink for [github source](https://knative.dev/docs/reference/eventing/eventing-sources-api/#GitHubSource), for example.

## EventBinding
The `EventBinding` CRD provides a new high-level means of managing all of the resources needed to allow a Pipeline to be bound to a specific Event and produce PipelineRuns as a result of those events. Individual EventBindings are scoped to a specific pipeline - Bindings also create all their own PipelineResources and Listeners (and clean them up on removal as well). This spec will likely evolve the most as we discover the most effect ways to bind events to action.

An example EventBinding:

```
apiVersion: tektonexperimental.dev/v1alpha1
kind: EventBinding
metadata:
  labels:
    app: ulmaceae
  name: ulmaceae-binding
  namespace: ulmaceae
spec:
  eventname: pushevents
  eventtype: dev.knative.source.github.push
  pipelineRef:
    name: ulmaceae-pipeline
  resourceTemplates:
  - metadata:
      name: source-repo
    name: source-repo
    spec:
      params:
      - name: url
        value: https://github.com/iancoffey/ulmaceae
      type: git
  - metadata:
      name: image-ulmaceae
      namespace: ulmaceae
    name: image-ulmaceae
    spec:
      params:
      - name: url
        value: /
      type: image
  serviceAccount: ulmaceae-account
  sourceref:
    name: ulmaceae-source
```

# Instructions

TODO: icoffey is working on this next :)

## Getting Started

Preqs:

- A Tekton Pipeline must be created prior to creating an EventBinding. This is the Pipeline that will handle requests.

## Minikube instructions

To dev/test locally with minikube:

* Get the `ko` command: `go get -u github.com/google/ko/cmd/ko`
* Load your docker environment vars: `eval $(minikube docker-env)`
* Start a registry: `docker run -it -d -p 5000:5000 registry:2`
* Set `KO_DOCKER_REPO` to local registry: `export KO_DOCKER_REPO=localhost:5000/<myproject>`
* Apply tekton components: `ko apply -L -f config/`
* Create an EventBinding (such as the example above) and await cloud events.
* The Listener that the EventBinding creates can be used as an Eventing sink.
