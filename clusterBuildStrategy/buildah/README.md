# Buildah ClusterBuildStrategy
The `buildah` ClusterBuildStrategy uses [buildah](https://github.com/containers/buildah) to build and push a container image, out of a `Dockerfile`. The `Dockerfile` should be specified using a parameter in the `Build` resource.

## Getting Started with Buildah ClusterBuildStrategy

### Pre-requisites:
- The `Builds for Red Hat OpenShift` Operator is available in the OpenShift cluster.
- The `buildah` clusterBuildStrategy is already installed. This could be checked by following command:
```
$ oc get cbs | grep buildah
```
If the buildStrategy is not available, [Install ClusterBuildStrategies](../../README.md) first.

### Steps:

1. Create a builds object. 
- `source` refers to the location where the source code is.
- `strategy` references the BuildStrategy to use to build the container.
- `paramValues` is a list of key/value that could be used to set strategy parameters.
    - Here, we pass the `dockerfile` parameter to specify the Dockerfile's location required to build the output image.
- `output` refers to the location where the built image would be pushed.
    - Here, it is expected to be pushed to the OpenShift cluster’s internal registry.
    - `buildah-example` is the name of the current project.
```
$ cat <<EOF | oc create -f -
apiVersion: shipwright.io/v1beta1
kind: Build
metadata:
  name: buildah-golang-build
spec:
  source:
    git: 
      url: https://github.com/shipwright-io/sample-go
    contextDir: docker-build
  strategy:
    name: buildah-strategy-managed-push
    kind: ClusterBuildStrategy
  paramValues:
  - name: dockerfile
    value: Dockerfile
  output:
    image: image-registry.openshift-image-registry.svc:5000/buildah-example/sample-go-app
EOF
```

2. Create a buildRun object. 
- `spec.build.name` refers to the respective build, which is expected to be available in the same namespace.
```
$ cat <<EOF | oc create -f -
apiVersion: shipwright.io/v1beta1
kind: BuildRun
metadata:
  name: buildah-golang-buildrun
spec:
  build:
    name: buildah-golang-build
EOF
```

3. The BuildRun object creates a TaskRun which spins the pods to perform the buildStrategy steps. 
```
$ oc get br
NAME                      SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
buildah-golang-buildrun   Unknown     Pending   1s

$ oc get tr
NAME                      	     SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
buildah-golang-buildrun-dtrg2   Unknown     Pending   3s

$ oc get tr buildah-golang-buildrun-dtrg2 -ojson | jq .metadata.ownerReferences
[
  {
    "apiVersion": "shipwright.io/v1alpha1",
    "blockOwnerDeletion": true,
    "controller": true,
    "kind": "BuildRun",
    "name": "buildah-golang-buildrun",
    "uid": "676dd1da-6ec2-4c04-86b2-cfe5be265260"
  }
]

$ oc get pods
NAME                                READY     STATUS        RESTARTS    AGE
buildah-golang-buildrun-dtrg2-pod   0/2       Init:0/2      0           9s
```

4. Once all the containers completes the tasks, and pod reports Completed status, the respective taskRun & buildRun marks Succeeded as true.
- In case of any failures encountered by the containers, the `status.FailureDetails` section of the buildRun is populated with relevant logs. 
```
$ oc get pods
NAME                                READY     STATUS        RESTARTS     AGE
buildah-golang-buildrun-dtrg2-pod   0/2       Completed     0            3m18s

$ oc get tr
NAME                            SUCCEEDED     REASON        STARTTIME     COMPLETIONTIME
buildah-golang-buildrun-dtrg2   True          Succeeded     11m           8m51s

$ oc get br
NAME                      SUCCEEDED      REASON         STARTTIME       COMPLETIONTIME
buildah-golang-buildrun   True           Succeeded      13m             11m
```

5. Validate whether the image has been pushed to the registry (specified in build.Spec.Output.Image) successfully or not.