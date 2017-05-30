# Docker Cloud

Docker Cloud is 100% web based CaaS (Container as a Service) platform hosted by Docker which allows to easily manage containerized applications.

It allows to:

- create image repositories (the place where the versions of the images are stored)
- configure a continuous integration pipeline triggering the creation of an image and some application testing when code is pushed to source control
- create an application and deploy it on the infrastructure of several cloud providers
- setup a continuous deploiement pipeline which automates the deploiement of the application once the tests passed

On top of this, Teams and Organizations can be created so applications components and their underlying infrastructure can be isolated from each other.

In short, Docker Cloud allows to setup a fully CI/CD pipeline in an easy and secure way.

Still in beta version at the date of this writing is the management of a fleet of swarm directly within Docker Cloud. This feature will be developed later in this section.

## Sample application

To illustrate the usage of Docker Cloud, we will consider a sample Node.js api that exposes a single endpoint and retuns a json message inviting you to visit a random city.
The code is available on [https://github.com/lucj/api](https://github.com/lucj/api). Basically, the application is composed of:

- app.js: definition of the http server
- index.js: entry point which calls app.js
- package.json: list of the application dependencies, definition of the `start` and `test` actions.
- Dockerfile
- test: folder containing a simple test scenario

## Login onto Docker Cloud

The Docker Cloud interface is available at [https://cloud.docker.com](https://cloud.docker.com). If you already have an account on Docker Hub, the same credentials must be used.

![cloud.docker.com](./images/Login.png)

When logged on, the interface shows a menu on the left (we'll go through those elements later on) and a main panel with several tabs. Each of the tab details a step to deploy an application and its full CI/CD pipeline in the cloud.

To help in the setup, each tab also provides links towards [Docker online documentation](https://docs.docker.com).

## Creation of the repository

![CloudRegistry01](./images/CloudRegistry-01.png)

From the `Cloud registry` tab, we can create repository which will hold the images of our Node.js api. Let's create a repository and name it `api`.

At this stage, there is not a lot of information to provide, we only need:
- the name of the repository
- a description (optional)
- the visibility of the repository (we leave the default value (`public`) so that anyone can use it)

![CloudRegistry02](./images/CloudRegistry-02.png)

Once it's created, the `general` tab which provides basic information of the repository

![CloudRegistry03](./images/CloudRegistry-03.png)

The `tags` tab shows the tags (each tag representing a version of the image) pushed to the repository. As no image have been pushed yet, the list is empty.
We are also provided the command to use in order to push an image to the repository.

![CloudRegistry04](./images/CloudRegistry-04.png)

## Setup the continuous integration

![CI00](./images/CI-00.png)

From detailed view of the repository, the `build` tab is an interesting one as it allows to setup an automated build of the image. The idea behind this is to be able to trigger a build of the image each time some code is pushed to the corresponding repository of the underlying source control system ([GitHub](https://github.com) or [Bitbucket](https://bitbucket.org/)).

![CI01](./images/CI-01.png)

### Setup the automated build

As the source code of our Node.js api is in GitHub, we need to link the GitHub account to the Docker Cloud account. When we click on the `Link to GitHub` icon, we have access to the `Source providers` section of the `Cloud Settings` menu. From there we can easily connect both accounts.

![CI02](./images/CI-02.png)

Once this is done, we can go back to the `build` tab of the `api` repository and configure the build as follow:

- SOURCE REPOSITORY: select the GitHub repository which holds the source code (lucj/api in this example).
- BUILD LOCATION: make it use the smallest machine of the Docker infrastructure
- DOCKER VERSION: we use the default value (latest Docker EE)
- AUTOTEST: we leave this option deactivated for now on, we will use it in a next step
- BUILD RULES: this is where the magic happens. By default, each push on the master branch in GitHub will result in the build of the `latest` tag of the image. By default, the Dockerfile is expected to be at the root of the project repository. The `Autobuild` option is also enabled by default (which makes sense as we want to setup an automated build).

![CI03](./images/CI-03.png)

We can click on `Save` and we should have the automated build setup correctly.

Now the link is established between the GitHub repository and the Docker Cloud one, we can do some changes to the code and push it to GitHub. We will observe how the image creation is triggered.

```
$ git push -u origin master
Counting objects: 7, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (6/6), done.
Writing objects: 100% (7/7), 1.05 KiB | 0 bytes/s, done.
Total 7 (delta 0), reused 0 (delta 0)
To git@github.com:lucj/api.git
 * [new branch]      master -> master
Branch master set up to track remote branch master from origin.
```

From the `build` tab of the repository's details, we can see the build goes from the `EMPTY` status
![CI04](./images/CI-04.png)

... to the `BUILDING` status...
![CI05](./images/CI-05.png)

... and finally to the `SUCCESS` status
![CI06](./images/CI-06.png)

From the log of the build we can see all the instructions of the Dockerfile that were executed to build the image.

![CI07](./images/CI-07.png)

Going back to the repository, we can see the new tag that has been created.

Note: as we did not specified any tag, each time a new image is created, it will overwrite the last version (the one with the `latest` tag). This is just for the example as this is not a suitable approach in a enterprise environment.

![CI08](./images/CI-08.png)

From the `timeline` tab of the repository, we can see the history of the action performed. In the current state, we have linked the repository to GitHub and then triggered a build.
![CI09](./images/CI-09.png)

### Setup the automated test

At this stage, we have setup Docker Cloud in a way that a push to a GitHub's repository triggers the creation of a new image. Let's now go one step further and have some tests running on the newly created image.

The source code embeds a simple test scenario which:
- runs the application
- sends a request to the default endpoint
- check taht the application replies with a 200 HTTP Code

The test can be run locally with the command `npm test`.

```
$ npm test

> Travel@0.0.1 test /Users/luc/api
> mocha test/functional.js

  City
info: server listening on port 3000
info: new request at [2017-05-30T12:27:24.661Z]
    ✓ should return a dummy city

  1 passing (50ms)
```

In order for the test to be run in Docker Cloud, we need to create a `docker-compose.test.yml` file which contains a service named `sut` that runs the tests. In the current example, the docker-compose.test.yml file is pretty simple and contains only the following.

```
sut:
  build: .
  command: npm test
```

Note: if the application was dependent on other services, we would add them in the docker-compose.test.yml file as well.

In the `build` tab of the repository, we need to activate the `Autotest` option in order for the test to be run automatically.

![CI10](./images/CI-10.png)

The workflow is the following one: 

- code is pushed to the GitHub repository
- the image is created
- the test are run on the new image
- if the tests are successfull, the image is pushed to the repository

The screenshot below illustrates the trigger of a new build.

![CI11](./images/CI-11.png)

We can see from the build's log that the test are performed after the image is built. Then the image is pushed to the repository.

![CI12](./images/CI-12.png)

In only a few steps, we have setup a continuous integration pipeline with Docker Cloud. Pretty neat !

## Manage the infrastructure

The third tab of the main panel is dedicated to the deploiment of the application. So let's see, how we can deploy the image we have created above.
As for the previous steps, we are provided a couple of link which details what needs to be done for our application to be deployed.

### Link Docker Cloud with a cloud provider

Docker Cloud allows to deploy an application on the infrastructure of the following cloud providers:

- [Amazon Web Services](https://aws.amazon.com)
- [DigitalOcean](https://digitalocean.com)
- [Microsoft Azure](https://azure.microsoft.com)
- [SoftLayer](http://www.softlayer.com)
- [Packet](https://www.packet.net)

From the `Cloud providers` tab of the `Cloud Settings` menu item, we connect DigitalOcean to our Docker Cloud account. This will allow us to create DO Droplet and deploy our application on them.

### Create a DigitalOcean droplet

Using the `Node Clusters` menu within the `Infrastructure` item in the menu on the left, we create one node selecting DigitalOcean as a provider. The type/size of the node needs to be provided as well, we will go for the smallest one.

IMAGE

Less than one minute after, the node is ready to run our application.

### Create a service

As defined in the Docker's documentation: "a service is a group of container that use the same IMAGE:TAG". Basically, a service is instanciated into one or more containers and can be scaled very easily.

Let's go ahead and create a new service based on the image we created in the previous steps.

The image of the service can be selected from several sources:
- templates that are presented by default
- images from the Docker Hub
- repositories created in our Docker Cloud account

We will go for the last option as the image we want to deploy is the one we created earlier in the `api` repository.

The menu on the right lists all the configuration that can be done on the service we are about to deploy:

- some general settings (service name, image the service is based on, deploiement strategy and constraints, restart policies, ...)
- some container's configuration options (entrypoint and command, RAM and CPU limits)
- the ports published
- the environment variables required
- configuration of volume so data are decoupled from the container's lifecycle

In order to create a service based on the `api`, we will provide the following pieces of information:

- the name of the image to use (`lucj/api:latest`). As we did not specify any tag when we create the image, `latest` is the default one.
- the name of the service
- a port published on the host (and mapped to the port 80 of the service)

A couple of seconds later, the service is in RUNNING state and ready to be used by the external world through the endpoint provided on the interface.

```
$ curl http://api.9443adc3.svc.dockerapp.io:8080
{"msg":"api-1 suggests to visit Isnozek"}
```

In the `APPLICATION` section of the menu on the right, we can see the services (only the `api` in this case), and the containers running on the node.

The details view of the container shows several tabs:
- the general view of the containers, meaning its configuration options
- the logs that allows to see the call we did above using curl
- the timeline of the action done on this container
- a terminal that allows to perform operation in the running container (which can be handy for debugging purposes)

From the service details, we can easily scale the number of containers. 

```
$ curl http://api-1.15179480.cont.dockerapp.io:32770
{"msg":"api-1 suggests to visit Guplebzuz"}
luc at neptune in ~
$ curl http://api-2.15179480.cont.dockerapp.io:32771
curl: (6) Could not resolve host: api-2.15179480.cont.dockerapp.io

luc at neptune in ~
$ curl http://api-2.806291e9.cont.dockerapp.io:32771
{"msg":"api-2 suggests to visit Zafipic"}
luc at neptune in ~
$ curl http://api-3.a5ead573.cont.dockerapp.io:32772
{"msg":"api-3 suggests to visit Buhdezo"}
```

#### Launching a load-balancer



## Handle Swarm mode cluster
