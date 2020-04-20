# .NET Core OpenShift Workshop

# Prerequistes
This lab assumes the following are already configured on your machine
* OpenShift CLI
* git (2.23.0 or later)
* .NET Core SDK (3.1 or later)
* Docker 
* Helm

## Containerization POC

### Install S2I
1. Download the latest release from [https://github.com/openshift/source-to-image/releases](https://github.com/openshift/source-to-image/releases)
1. Extract and copy s2i binary to a directory on your PATH

### Clone the Example Repo
1. Clone the repo https://github.com/redhat-developer/s2i-dotnetcore-ex.git
    ```
    cd ~/
    mkdir git
    cd git
    git clone https://github.com/redhat-developer/s2i-dotnetcore-ex.git
    cd s2i-dotnetcore-ex
    git checkout dotnetcore-3.1
    ```

### Local Run
1. Change to the app directory
    ```
    cd ~/git/s2i-dotnetcore-ex/app
    ```
1. Run the app
    ```
    dotnet run
    ```
1. Note the output of what ports the app is running on it should be listening on https://localhost:5001
1. Navigate your web browser to https://localhost:5001 and confirm the application is running.  You may have to accept the security warning.

### Build S2I Locally
This will build the container image from the local file system, this is useful when you are actively working on code and are not ready to commit to a repository
1. Run the build to create the .NET app and stream it into an image
    | Command Parts | Description |
    | ------------- | ----------- |
    | s2i           | The actual command |
    | build         | Tells the command to build an image from source |
    | .             | The location of the source files (current directory) |
    | registry.redhat.io/ubi8/dotnet-31:3.1 | The builder image to use |
    | dotnetcore-ex | The Image name to create in the local registry |
    ```
    cd ~/git/s2i-dotnetcore-ex/app
    docker pull registry.redhat.io/ubi8/dotnet-31:3.1
    s2i build . registry.redhat.io/ubi8/dotnet-31:3.1 dotnetcore-ex
    ```
1. Run the app locally as a container, while exposing the ports locally.  Notice that when running using the built image from the S2I build it defaults to listening to an un-secure port 8080 which differs from when we ran locally in the previous section.
    ```
    docker run -p 8080:8080 dotnetcore-ex
    ```
1. Open your web browser and navigate to http://localhost:8080

## Build Pipeline POC
This lab will walk through first having the S2I build handled by a BuildConfig in OpenShift and then move to using Jenkins to trigger the build

### Clone Repo
1. Clone the lab repo
    ```
    cd ~/
    git clone https://github.com/cloudfirst-dev/openshift-dotnet-labs.git dotnet-labs
    cd ~/git/dotnet-labs
    ```

### Create the ImageStreams
To be able to build, we need to use the custom builder images we used locally and expose them as ImageStreams inside your namespace.  Since we are using version 3.1 of .NET core these are not included by default in OpenShift

1. Create the build/runtime ImageStream.  This is the ImageStreams which the S2I buildconfig will utilize
    ```
    cd ~/git/dotnet-labs
    oc new-project $(oc whoami)-demo
    oc create -f https://raw.githubusercontent.com/redhat-developer/s2i-dotnetcore/master/dotnet_imagestreams_rhel8.json
    ```
1. Create Application ImageStream.  This ImageStream will be used to host the output of the buildconfig.  This utilizes the internal registry in OpenShift.
    ```
    oc create -f ./build-pipeline/imagestream.yaml
    ```

### Create BuildConfig
We will now create the build config so that the S2I build runs in the OpenShift Cluster instead of locally

1. Create the BuildConfig
    ```
    oc create -f ./build-pipeline/build.yaml
    ```

### Run the Build From Local Files
This will run the build using the local current folder as the source for the builder image.  It will upload the contents of the directory to the remote pod in the cluster, at the end it will push the image to the internal quay registry.

1. Start the build
    ```
    cd ~/git/s2i-dotnetcore-ex/app
    oc start-build dotnet-build --from-dir=. -F
    ```

### Run the Build with Jenkins
This will be very similar to running locally with the exception we will be using the GIT repo as the source for the build.

1. Create the Jenkins instance in the workspace
    ```
     oc new-app --param=ENABLE_OAUTH=false jenkins-ephemeral
    ```
1. Create the Jenkins Build Pipeline
    ```
    cd ~/git/dotnet-labs
    oc create -f ./build-pipeline/build-git.yaml
    oc create -f ./build-pipeline/jenkins-build.yaml
    ```
1. Start a Jenkins build which automates the command we ran in "Run the Build From Local Files"
    ```
    oc start-build dotnetcore-ex-pipeline
    ```
1. Watch the progress by going to the link outputted from the command below using username (admin/password) to login to jenkins
    ```
    echo open $(oc get build dotnetcore-ex-pipeline-1 -o=jsonpath="{ .metadata.annotations['openshift\.io/jenkins-console-log-url'] }") to follow build
    ```

## Deployment POC
This lab will walk through using helm to provision the application deployment to OpenShift using the above build

1. Install the helm chart to OpenShift
    ```
    helm install helm --generate-name
    ```
1. Watch the application deployment in OpenShift
1. Access the application running on OpenShift by executing the following to get the url
    ```
    echo "http://$(oc get route -l "app.kubernetes.io/name=helm" -o jsonpath="{.items[0].spec.host}")"
    ```

## Deployment and Build POC
This lab will walk through using helm to create both the build items and deployment in a single namespace

1. Create a new project to demonstrate this relies on nothing else we have done so far
    ```
    oc new-project $(oc whoami)-dotnet-helm
    oc create -f https://raw.githubusercontent.com/redhat-developer/s2i-dotnetcore/master/dotnet_imagestreams_rhel8.json
    ```
1. Deploy the OpenShift manifests with Helm
    ```
    helm install demo helm-build
    ```
1. Access the new app using the link from the following output
    ```
    echo "http://$(oc get route -l "app.kubernetes.io/name=helm" -o jsonpath="{.items[0].spec.host}")"
    ```