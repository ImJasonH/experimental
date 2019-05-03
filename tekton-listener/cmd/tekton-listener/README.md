# Pipeline Listene

The pipeliner listener defines a process by which Tekton PipelineRuns can be directly triggered by events, which it consumes and handles to kickoff the pipeline.

To do this, an optional CRD `TektonListener` is provided. Once defined, the listener provides support for consuming CloudEvent and producing a predefined PipelineRun. It is intentionally designed to allow for other sources beyond CloudEvents.

The only event-type supported is `com.github.checksuite`.

# Minikube instructions

To dev/test locally with minikube:

* Get the `ko` command: `go get -u github.com/google/ko/cmd/ko`
* Load your docker environment vars: `eval $(minikube docker-env)`
* Start a registry: `docker run -it -d -p 5000:5000 registry:2`
* Set `KO_DOCKER_REPO` to local registry: `export KO_DOCKER_REPO=localhost:5000/<myproject>`
* Apply tekton components: `ko apply -L -f config/`
* Create a TektonListener (such as the example below) and await cloud events.
* Create a service or expose the pods port locally to access the endpoint.
* Listener is configured for port `8082`.


```
apiVersion: tekton.dev/v1alpha1
kind: TektonListener
metadata:
  name: test-build-tekton-listener
  namespace: tekton-pipelines
spec:
  selector:
    matchLabels:
      app: test-build-tekton-listener
  serviceName: test-build-tekton-listener
  template:
    metadata:
      labels:
        role: test-build-tekton-listener
    spec:
      serviceAccountName: tekton-pipelines-controller
  listener-image: github.com/tektoncd/pipeline/cmd/tektonlistener
  event-type: com.github.checksuite
  namespace: tekton-pipelines
  port: 80
  runspec:
    pipelineRef:
      name: demo-pipeline
    trigger:
      type: manual
    serviceAccount: 'default'
    resources:
    - name: source-repo
      resourceRef:
        name: skaffold-git
    - name: web-image
      resourceRef:
        name: skaffold-image-leeroy-web
    - name: app-image
      resourceRef:
        name: skaffold-image-leeroy-app
```

And finally, you will need a service. Since the Service fullfills the [Addressable](https://github.com/knative/eventing/blob/master/docs/spec/interfaces.md#addressable) contract, the listener service can be used as a sink for [github source](https://knative.dev/docs/reference/eventing/eventing-sources-api/#GitHubSource), for example.

```
apiVersion: v1
kind: Service
metadata:
  name: test-build-tekton-listener
  labels:
    role: test-build-tekton-listener
  namespace: tekton-pipelines
spec:
  ports:
    - name: listener
      port: 8082
      protocol: TCP
      targetPort: 8082
  selector:
    role: test-build-tekton-listener
```
Also note that the listener itself implements the Addressable interface, as well.
