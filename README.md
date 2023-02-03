# A quick demo of Cloud Native Buildpacks – <https://buildpacks.io/>

Including [`pack`](https://buildpacks.io/docs/tools/pack/) and [`kpack`](https://github.com/pivotal/kpack).

## What you'll need

* `pack`, `docker`, and `dive` CLI tools
  * `brew install buildpacks/tap/pack docker dive`
* `kubectl` and `kp` CLI tools
  * `brew install kubernetes-cli vmware-tanzu/kpack-cli/kp`
* Docker
* A Kubernetes cluster with at least 4GB memory
  * If using Colima, start it with `colima start -k -m4 -ax86_64`
  * You may also need to export the location of the Docker daemon: `export DOCKER_HOST=unix://$HOME/.colima/default/docker.sock`

## 0. Prepare the environment

### Initial setup

1. Clone the Paketo Buildpacks sample apps, [paketo-buildpacks/samples](https://github.com/paketo-buildpacks/samples), to a local directory
    * `git clone https://github.com/paketo-buildpacks/samples.git`
1. Install `kpack` into the Kubernetes cluster as described [here](https://github.com/pivotal/kpack/blob/main/docs/install.md)
    * `kubectl apply -f https://github.com/pivotal/kpack/releases/download/v0.9.2/release-0.9.2.yaml`
1. In your GitHub and Docker Hub account settings, create personal access tokens for each and set them in [creds.yml](creds.yml)
1. Create the secrets for GitHub and Docker Hub
    * `kubectl apply -f creds.yml`
1. Create the stack, store and builder
    * `kubectl apply -f builder.yml`

### Resetting after a previous run

1. Delete any images previously created by `pack`
    * `docker rmi my-nodejs-app`
1. Delete the `kpack` resources for the stack, store, builder and image
    * `kubectl delete -f builder.yml -f image.yml`
1. Delete builder and app image from Docker Hub (wait for deletion to finish)
1. Re-create the stack, store and builder
    * `kubectl apply -f builder.yml`

## 1. Demo `pack`

1. Get into one of the subdirectories containing a sample app
    * `cd samples/nodejs/npm/`
1. Examine the contents (note lack of Dockerfile or any other build config)
    * `ls -la`
1. Make sure `pack` is installed
    * `pack --help`
1. See what builders `pack` suggests
    * `pack builder suggest`
1. Build the app into an image using the Paketo base builder
    * `pack build my-nodejs-app --builder paketobuildpacks/builder:base`
1. Check that the image was built and is available locally
    * `docker images`
1. Examine how the layers are composed in the image
    * `dive my-nodejs-app`
1. Download the SBOM from the image
    * `pack sbom download my-nodejs-app`
    * `docker inspect my-nodejs-app`

## 2. Demo `kpack`

### Check the setup

1. If you ran `pack` in a sample app above, get back to the directory containing the kpack resources
    * `cd ../../..`
1. Check that `kp` is installed
    * `kp --help`
1. Make sure the `kpack` CRDs are present in the cluster
    * `kubectl api-resources | grep kpack`
1. Check the readiness of the stack, store and builder
    * `kubectl get clusterstack,clusterstore,builder`
1. Check the builder image on Docker Hub

### Create the `Image` and watch a build

1. Apply the `Image` manifest
    * `kubectl apply -f image.yml`
1. Check that the `Image` spawned a `Build` and a build pod
    * `kubectl get image,build,po`
1. Look into the composition of the build pod, in particular the init containers
    * `kubectl describe pod/my-nodejs-app-build-1-build-pod`
1. Stream the build logs using `kp`
    * `kp builds logs my-nodejs-app`
1. Check that a new app image was published to Docker Hub

### Simulate a buildpack/store update

1. Edit the store configuration, changing the `nodejs` image version from `1.0.0` to `1.1.0`
    * `kubectl edit clusterstore nodejs`
1. After a short pause, check that the builder has been updated (spot "Observed Store Generation" and buildpack version)
    * `kubectl describe builder my-nodejs-builder`
1. Check that a new builder image was published to Docker Hub
1. Find the new `Build` and the new build pod
    * `kubectl get image,build,po`
1. Stream the build logs using `kp`
    * `kp builds logs my-nodejs-app`
1. Check that a new app image was published to Docker Hub
