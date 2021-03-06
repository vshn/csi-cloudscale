# csi-cloudscale [![Build Status](https://travis-ci.org/cloudscale-ch/csi-cloudscale.svg?branch=master)](https://travis-ci.org/cloudscale-ch/csi-cloudscale)
A Container Storage Interface ([CSI](https://github.com/container-storage-interface/spec)) Driver for cloudscale.ch volumes. The CSI plugin allows you to use cloudscale.ch volumes with your preferred Container Orchestrator.

The cloudscale.ch CSI plugin is mostly tested on Kubernetes. Theoretically it
should also work on other Container Orchestrator's, such as Mesos or
Cloud Foundry. Feel free to test it on other CO's and give us a feedback.

## Releases

The cloudscale.ch CSI plugin follows [semantic versioning](https://semver.org/).
The current version is: **`v0.2.0`**. This means that the project is still
under active development and may not be production ready.

* Bug fixes will be released as a `PATCH` update.
* New features (such as CSI spec bumps) will be released as a `MINOR` update.
* Significant breaking changes makes a `MAJOR` update.


## Installing to Kubernetes

**Requirements:**

* Kubernetes v1.10.5 minimum 
* `--allow-privileged` flag must be set to true for both the API server and the kubelet
* (if you use Docker) the Docker daemon of the cluster nodes must allow shared mounts


### [Rancher](https://rancher.com/) users:

`Mount Propagation` is [disabled by
default](https://github.com/rancher/rke/issues/765) on latest `v2.0.6` version
of Rancher, which prevents the `csi-cloudscale` to function correctly. To fix
the issue temporary, make sure to add the following settings to your cluster
configuration YAML file:

```
services:
  kube-api:
    extra_args:
      feature-gates: MountPropagation=true

  kubelet:
    extra_args:
      feature-gates: MountPropagation=true
```


#### 1. Create a secret with your cloudscale.ch API Access Token:

Replace the placeholder string starting with `a05...` with your own secret and
save it as `secret.yml`: 

```
apiVersion: v1
kind: Secret
metadata:
  name: cloudscale
  namespace: kube-system
stringData:
  access-token: "a05dd2f26b9b9ac2asdas__REPLACE_ME____123cb5d1ec17513e06da"
```

and create the secret using kubectl:

```
$ kubectl create -f ./secret.yml
secret "cloudscale" created
```

You should now see the cloudscale secret in the `kube-system` namespace along with other secrets

```
$ kubectl -n kube-system get secrets
NAME                  TYPE                                  DATA      AGE
default-token-jskxx   kubernetes.io/service-account-token   3         18h
cloudscale            Opaque                                1         18h
```

#### 2. Deploy the CSI plugin and sidecars:

Before you continue, be sure to checkout to a [tagged
release](https://github.com/cloudscale-ch/csi-cloudscale/releases). Always use the [latest stable version](https://github.com/cloudscale-ch/csi-cloudscale/releases/latest) 
For example, to use the latest stable version (`v0.2.0`) you can execute the following command:

```
$ kubectl apply -f https://raw.githubusercontent.com/cloudscale/csi-cloudscale/master/deploy/kubernetes/releases/csi-cloudscale-v0.2.0.yaml
```

This file will be always updated to point to the latest stable release.

A new storage class will be created with the name `cloudscale-volume-ssd` which is
responsible for dynamic provisioning. This is set to **"default"** for dynamic
provisioning. If you're using multiple storage classes you might want to remove
the annotation from the `csi-storageclass.yaml` and re-deploy it. This is
based on the [recommended mechanism](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md#recommended-mechanism-for-deploying-csi-drivers-on-kubernetes) of deploying CSI drivers on Kubernetes

*Note that the deployment proposal to Kubernetes is still a work in progress and not all of the written
features are implemented. When in doubt, open an issue or ask #sig-storage in [Kubernetes Slack](http://slack.k8s.io)*

#### 3. Test and verify:

Create a PersistentVolumeClaim. This makes sure a volume is created and provisioned on your behalf:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: cloudscale-volume-ssd
```

Check that a new `PersistentVolume` is created based on your claim:

```
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM             STORAGECLASS            REASON    AGE
pvc-0879b207-9558-11e8-b6b4-5218f75c62b9   5Gi        RWO            Delete           Bound     default/csi-pvc   cloudscale-volume-ssd             3m
```

The above output means that the CSI plugin successfully created (provisioned) a
new Volume on behalf of you. You should be able to see this newly created
volume under the [Volumes tab in the cloudscale.ch UI](https://control.cloudscale.ch/volumes)

The volume is not attached to any node yet. It'll only attached to a node if a
workload (i.e: pod) is scheduled to a specific node. Now let us create a Pod
that refers to the above volume. When the Pod is created, the volume will be
attached, formatted and mounted to the specified Container:

```
kind: Pod
apiVersion: v1
metadata:
  name: my-csi-app
spec:
  containers:
    - name: my-frontend
      image: busybox
      volumeMounts:
      - mountPath: "/data"
        name: my-cloudscale-volume
      command: [ "sleep", "1000000" ]
  volumes:
    - name: my-cloudscale-volume
      persistentVolumeClaim:
        claimName: csi-pvc 
```

Check if the pod is running successfully:


```
$ kubectl describe pods/my-csi-app
```

Write inside the app container:

```
$ kubectl exec -ti my-csi-app /bin/sh
/ # touch /data/hello-world
/ # exit
$ kubectl exec -ti my-csi-app /bin/sh
/ # ls /data
hello-world
```

## Development

Requirements:

* Go: min `v1.10.x`

After making your changes, run the unit tests: 

```
$ make test
```

If you want to test your changes, create a new image with the version set to `dev`:

```
apt install docker.io
# At this point you probably need to add your user to the docker group
docker login --username=cloudscalech --email=hello@cloudscale.ch
$ VERSION=dev make publish
```

This will create a binary with version `dev` and docker image pushed to
`cloudscalech/cloudscale-csi-plugin:dev`


To run the integration tests run the following:

```
$ KUBECONFIG=$(pwd)/kubeconfig make test-integration
```


### Release a new version

To release a new version bump first the version:

```
$ make bump-version
```

Make sure everything looks good. Create a new branch with all changes:

```
$ git checkout -b new-release
$ git add .
$ git push origin
```

After it's merged to master, [create a new Github
release](https://github.com/cloudscale-ch/csi-cloudscale/releases/new) from
master with the version `v0.2.0` and then publish a new docker build:

```
$ git checkout master
$ make publish
```

This will create a binary with version `v0.2.0` and docker image pushed to
`cloudscalech/cloudscale-csi-plugin:v0.2.0`

## Contributing

At cloudscale.ch we value and love our community! If you have any issues or
would like to contribute, feel free to open an issue/PR
