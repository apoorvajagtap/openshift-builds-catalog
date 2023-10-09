# Source-To-Image (S2I) ClusterBuildStrategy
The `source-to-image` ClusterBuildStrategy is composed of [source-to-image](https://github.com/openshift/source-to-image/) and [buildah](https://github.com/containers/buildah) in order to generate a `Dockerfile` and prepare the application to be built later with a builder-image.

`s2i` requires a specially crafted image, which can be informed as `builder-image` parameter on the `Build` resource.

## Getting Started with Buildah ClusterBuildStrategy

### Pre-requisites:
- The `Builds for Red Hat OpenShift` Operator is available in the OpenShift cluster.
- The `source-to-image-redhat` clusterBuildStrategy is already installed. This could be checked by following command:
```
$ oc get cbs | grep source-to-image
```
If the buildStrategy is not available, [Install ClusterBuildStrategies](../../README.md) first.

### Steps:
1. Create a builds object. 
- `source` refers to the location where the source code is.
- `strategy` references the BuildStrategy to use to build the container.
- `paramValues` is a list of key/value that could be used to set strategy parameters.
    - Here, we pass the `builder-image` parameter to specify the builder image to be used to build the output image.
- `output` refers to the location where the built image would be pushed.
    - Here, it is expected to be pushed to a custom quay repo.
    - `pushSecret` is the secret name that stores the credentials for pushing container images. Follow [Creating registry secret](section), if the secret doesn't exist already.
```
$ cat <<EOF | oc create -f -     
apiVersion: shipwright.io/v1beta1
kind: Build
metadata:
  name: s2i-nodejs-build
spec:
  source:
    git:
      url: https://github.com/shipwright-io/sample-nodejs
    contextDir: source-build/
  strategy:
    name: source-to-image-redhat
    kind: ClusterBuildStrategy
  paramValues:
  - name: builder-image
    value: "quay.io/centos7/nodejs-12-centos7"
  output:
    image: quay.io/repo/s2i-nodejs-example
    pushSecret: registry-credential
EOF
```

2. Ensure that the `pipeline` ServiceAccount is available in the current namespace, and has `privileged` SCC assigned. If not, create one & add the SCC.
- If missed, the buildRun will report failure with `PodAdmissionFailed` Reason.
```
$  oc adm policy add-scc-to-user privileged -z pipeline
```

3. Create a buildRun object. 
- `spec.build.name` refers to the respective build, which is expected to be available in the same namespace.
```
apiVersion: shipwright.io/v1beta1
kind: BuildRun
metadata:
  name: s2i-nodejs-buildrun
spec:
  build:
    name: s2i-nodejs-build
EOF
```

4. The BuildRun object creates a TaskRun which spins the pods to perform the buildStrategy steps. 
```
$ oc get br                       
NAME                  SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
s2i-nodejs-buildrun   Unknown     Pending   2s

$ oc get tr
NAME                        SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
s2i-nodejs-buildrun-phxxm   Unknown     Pending   4s

$ oc get tr s2i-nodejs-buildrun-phxxm -ojson | jq .metadata.ownerReferences
[
  {
    "apiVersion": "shipwright.io/v1alpha1",
    "blockOwnerDeletion": true,
    "controller": true,
    "kind": "BuildRun",
    "name": "s2i-nodejs-buildrun",
    "uid": "e09a3c39-8880-46d2-81a8-73c59b77d772"
  }
]

$ oc get pods -w
NAME                            READY   STATUS            RESTARTS   AGE
s2i-nodejs-buildrun-phxxm-pod   0/4     PodInitializing   0          6s
s2i-nodejs-buildrun-phxxm-pod   4/4     Running           0          7s
```

5. Once all the containers completes the tasks, and pod reports Completed status, the respective taskRun & buildRun marks Succeeded as true.
- In case of any failures encountered by the containers, the `status.FailureDetails` section of the buildRun is populated with relevant logs. 
```
$ oc get pods
NAME                            READY   STATUS      RESTARTS   AGE
s2i-nodejs-buildrun-phxxm-pod   0/4     Completed   0          6m35s

$ oc get tr
NAME                        SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
s2i-nodejs-buildrun-phxxm   True        Succeeded   6m39s       13s

$ oc get br
NAME                  SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
s2i-nodejs-buildrun   True        Succeeded   6m41s       15s
```

6. Validate whether the image has been pushed to the quay repo (specified in build.Spec.Output.Image) successfully or not.

### Creating registry secret
The following step will create a secret of type `docker-registry` with the required credentials for authentication.

```
$ oc create secret docker-registry registry-credential --docker-server=quay.io --docker-username=<username> --docker-password=<password>
```