# Spring Rest - A RESTful API written in Spring Boot


## Usage to Run Locally

1. `git clone`
2. `mvn spring-boot:run`

curl -v localhost:8080/api/books
curl -v localhost:8080/api/health - use for readiness probe



This demonstrates how to implement a full end-to-end Jenkins Pipeline for a Java application in OpenShift Container Platform. This sample demonstrates the following capabilities:

* Deploying an integrated Jenkins server inside of OpenShift
* Running both custom and oob Jenkins slaves as pods in OpenShift
* "One Click" instantiation of a Jenkins Pipeline using OpenShift's Jenkins Pipeline Strategy feature
* Building a Jenkins pipeline with library functions from our [pipeline-library](https://github.com/redhat-cop/pipeline-library)

## Architecture

The following breaks down the architecture of the pipeline deployed, as well as walks through the manual deployment steps

### OpenShift Templates

The components of this pipeline are divided into two templates.

The first template, `openshift/templates/build.yml` is what we are calling the "Build" template. It contains:

* A `jenkinsPipelineStrategy` BuildConfig
* An `s2i` BuildConfig
* An ImageStream for the s2i build config to push to

The build template contains a default source code repo for a java application compatible with this pipelines architecture

The second template, `openshift/templates/deployment.yml` is the "Deploy" template. It contains:

* A tomcat8 DeploymentConfig
* A Service definition
* A Route

The idea behind the split between the templates is that I can deploy the build template only once (to my build project) and that the pipeline will promote my image through all of the various stages of my application's lifecycle. The deployment template gets deployed once to each of the stages of the application lifecycle (once per OpenShift project).

### Pipeline Script

This project includes a sample `Jenkinsfile` pipeline script that could be included with a Java project in order to implement a basic CI/CD pipeline for that project, under the following assumptions:

* The project is built with Maven
* The OpenShift projects that represent the Application's lifecycle stages are of the naming format: `<app-name>-dev`, `<app-name>-stage`, `<app-name>-prod`.

## Bill of Materials

* One or Two OpenShift Container Platform Clusters
  * OpenShift 3.5+ is required
  * [Red Hat OpenJDK 1.8](https://access.redhat.com/containers/?tab=overview#/registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift) image is required
* Access to GitHub

## Deployment Instructions

### 1. Create Lifecycle Stages

For the purposes of this demo, we are going to create three stages for our application to be promoted through.

- `basic-spring-boot-build`
- `basic-spring-boot-dev`
- `basic-spring-boot-stage`
- `basic-spring-boot-prod`

In the spirit of _Infrastructure as Code_ we have a YAML file that defines the `ProjectRequests` for us. This is as an alternative to running `oc new-project`, but will yeild the same result.

```
$ oc create -f openshift/projects/projects.yml
projectrequest "basic-spring-boot-build" created
projectrequest "basic-spring-boot-dev" created
projectrequest "basic-spring-boot-stage" created
projectrequest "basic-spring-boot-prod" created
```

### 2. Stand up Jenkins master in build project

For this step, the OpenShift default template set provides exactly what we need to get jenkins up and running.

```
$ oc process openshift//jenkins-ephemeral | oc apply -f- -n basic-spring-boot-build
route "jenkins" created
deploymentconfig "jenkins" created
serviceaccount "jenkins" created
rolebinding "jenkins_edit" created
service "jenkins-jnlp" created
service "jenkins" created
```



A _deploy template_ is provided at `openshift/templates/deployment.yml` that defines all of the resources required to run our Tomcat application. It includes:

* A `Service`
* A `Route`
* An `ImageStream`
* A `DeploymentConfig`
* A `RoleBinding` to allow Jenkins to deploy in each namespace.


### 3. Instantiate deployment template for each environment

This template should be instantiated once in each of the namespaces that our app will be deployed to. For this purpose, we have created a param file to be fed to `oc process` to customize the template for each environment.

Deploy the deployment template to all three projects.
```
$ oc process -f openshift/templates/deployment.yml -p=APPLICATION_NAME=basic-spring-boot -p NAMESPACE=basic-spring-boot-dev -p=SA_NAMESPACE=basic-spring-boot-build -p=READINESS_PATH="/api/health" -p=READINESS_RESPONSE="status.:.UP"  | oc apply -f-
service "spring-rest" created
route "spring-rest" created
imagestream "spring-rest" created
deploymentconfig "spring-rest" created
rolebinding "jenkins_edit" configured
$ oc process -f openshift/templates/deployment.yml -p=APPLICATION_NAME=basic-spring-boot -p NAMESPACE=basic-spring-boot-stage -p=SA_NAMESPACE=basic-spring-boot-build -p=READINESS_PATH="/api/health" -p=READINESS_RESPONSE="status.:.UP" | oc apply -f-
service "spring-rest" created
route "spring-rest" created
imagestream "spring-rest" created
deploymentconfig "spring-rest" created
rolebinding "jenkins_edit" created
$ oc process -f openshift/templates/deployment.yml -p=APPLICATION_NAME=basic-spring-boot -p NAMESPACE=basic-spring-boot-prod -p=SA_NAMESPACE=basic-spring-boot-build -p=READINESS_PATH="/api/health" -p=READINESS_RESPONSE="status.:.UP" | oc apply -f-
service "spring-rest" created
route "spring-rest" created
imagestream "spring-rest" created
deploymentconfig "spring-rest" created
rolebinding "jenkins_edit" created
```

A _build template_ is provided at `templates/build.yml` that defines all the resources required to build our java app. It includes:

* A `BuildConfig` that defines a `JenkinsPipelineStrategy` build, which will be used to define out pipeline.
* A `BuildConfig` that defines a `Source` build with `Binary` input. This will build our image.

### 4. Instantiate Pipeline

Deploy the pipeline template in build only.
```
$ oc process -f openshift/templates/build.yml -p=APPLICATION_NAME=basic-spring-boot -p NAMESPACE=basic-spring-boot-build -p=SOURCE_REPOSITORY_URL="https://github.com/fwmarkcheung/spring-rest-pipeline-example.git" -p=APPLICATION_SOURCE_REPO="https://github.com/fwmarkcheung/spring-rest-pipeline-example.git" | oc apply -f-
buildconfig "spring-rest-pipeline" created
buildconfig "spring-rest" created
```

The pipeline build should start automatically or you can do it manually by going to the Web Console and follow the pipeline by clicking in your `basic-spring-boot-build` project, and going to *Builds* -> *Pipelines*. At several points you will be prompted for input on the pipeline. You can interact with it by clicking on the _input required_ link, which takes you to Jenkins, where you can click the *Proceed* button. By the time you get through the end of the pipeline you should be able to visit the Route for your app deployed to the `myapp-prod` project to confirm that your image has been promoted through all stages.

## Cleanup

Cleaning up this example is as simple as deleting the projects we created at the beginning.

```
oc delete project basic-spring-boot-build basic-spring-boot-dev basic-spring-boot-prod basic-spring-boot-stage
```
