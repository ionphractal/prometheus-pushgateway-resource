Concourse Resource for the Prometheus Push Gateway
==================================================

A Concourse Resource for sending metrics to the [Prometheus Push Gateway](https://github.com/prometheus/pushgateway).


[TOC levels=4]: # "## Contents"

## Contents
- [Resource Usage](#resource-usage)
    - [Sample Pipeline](#sample-pipeline)
    - [Resource Configuration](#resource-configuration)
        - [The `resources.type:pushgateway.source` Section](#the-resourcestypepushgatewaysource-section)
        - [The `jobs.plan.task.on_success|on_failure.params` section](#the-jobsplantaskon_successon_failureparams-section)
- [Developer's Guide](#developers-guide)
    - [Pusing a new resource Image to Docker Hub](#pusing-a-new-resource-image-to-docker-hub)
    - [Spinning up a local Development Environment with `docker-compose`](#spinning-up-a-local-development-environment-with-docker-compose)
        - [Generating Keys for Concourse](#generating-keys-for-concourse)
    - [Running the Tests](#running-the-tests)
    - [Building and pushing the Docker Image for the Resource](#building-and-pushing-the-docker-image-for-the-resource)
    - [Setting up the CI Pipeline for the Resource](#setting-up-the-ci-pipeline-for-the-resource)
    - [Troubleshooting & Debugging](#troubleshooting--debugging)
- [TODOs](#todos)
- [Resources](#resources)


Resource Usage
--------------

### Sample Pipeline

Here is a sample usage of the Pushgateway resource

```yaml
---
resource_types:
  - name: pushgateway
    type: docker-image
    source:
      repository: michaellihs/prometheus-pushgateway-resource
      tag: 1.0.0

resources:
  - name: pushgateway
    type: pushgateway
    source:
      url: http://pushgw:9091
      job: concourse-pushgw-demo

jobs:
  - name: pushgw-metric
    plan:
      - task: a-successful-task
        config:
          platform: linux
          image_resource:
            type: docker-image
            source: {repository: busybox}
          run:
            path: echo
            args:
              - hello world
        on_success:
          put: pushgateway
          params:
            metric: successful_metric 1
```


### Resource Configuration

#### The `resources.type:pushgateway.source` Section

| Parameter | Type   | Required | Default | Description                                                                                        |
|:----------|:-------|:---------|:--------|:---------------------------------------------------------------------------------------------------|
| `url`     | URL    | yes      |         | URL of the Pushgateway server to send metrics to                                                   |
| `debug`   | String | no       | `false` | If set to `true`, the resource will output only debug information                                  |
| `job`     | String | no       |         | Job name of the metrics ( `metric{job="THIS VALUE",...}`), overridden by value in `params` section |


#### The `jobs.plan.task.on_success|on_failure.params` section

| Parameter | Type   | Required | Default | Description                                                                                       |
|:----------|:-------|:---------|:--------|:--------------------------------------------------------------------------------------------------|
| `metric`  | String | yes      |         | The metric to send to the Pushgateway                                                             |
| `job`     | String | no       |         | Job name of the metrics ( `metric{job="THIS VALUE",...}`) - overrides value in `resource` section |


Developer's Guide
-----------------

This section provides some information for those who want to join development on this resource.


### Pusing a new resource Image to Docker Hub

In case you want to build and push a new Docker image for the resource via our [Circle CI job](https://circleci.com/gh/michaellihs/prometheus-pushgateway-resource), to the following:

1. Create an annotated tag with the new (semantic version)

    ```bash
    git tag -a 1.3.5 -m "version 1.3.5"
    ```

2. Push the tag to GitHub

    ```bash
    git push origin 1.3.5
    ```

Whenever CircleCI builds a commit that has a semantic version tag on it, it will automatically push the image to Docker Hub.


### Spinning up a local Development Environment with `docker-compose`


1. Make sure to have Docker and Docker Compose installed on your workstation
2. Create a host entry in your `/etc/hosts` file

    ```
    127.0.0.1       concourse.dev.localhost
    ```

3. `cd` into the `dev` directory and use the provided shell script to spin up the dev environment

    ```
    cd dev
    ./concourse-up.sh
    ```

4. After a while, the Pushgateway website and Concourse should open up in your browser. You can login with user `test` and password `test`


> **Warning**: For convenience, this repository comes with a set of default keys used by Concourse. Make sure to re-create those keys if you want to run Concourse in a more production setup.


#### Generating Keys for Concourse

Follow steps in https://concourse-ci.org/concourse-generate-key.html - this is just a reminder of what I did to generate the keys:

```
# Inside the Concourse web container

root@f39bb0c9da87:/concourse-keys# /usr/local/concourse/bin/concourse generate-key -t ssh -f ./worker_key
wrote private key to ./worker_key
wrote ssh public key to ./worker_key.pub
root@f39bb0c9da87:/concourse-keys# cd /concourse-keys && /usr/local/concourse/bin/concourse generate-key -t ssh -f ./worker_key
wrote private key to ./worker_key
wrote ssh public key to ./worker_key.pub
root@f39bb0c9da87:/concourse-keys# cd /concourse-keys && /usr/local/concourse/bin/concourse generate-key -t ssh -f ./tsa_host_key
wrote private key to ./tsa_host_key
wrote ssh public key to ./tsa_host_key.pub
root@f39bb0c9da87:/concourse-keys# cd /concourse-keys && /usr/local/concourse/bin/concourse generate-key -t ssh -f ./session_signing_key
wrote private key to ./session_signing_key
wrote ssh public key to ./session_signing_key.pub
root@f39bb0c9da87:/concourse-keys# cp worker_key.pub authorized_worker_keys
```


### Running the Tests

The resource ships with a bunch of integration tests in the `test` folder, in order to run them:

```bash
test/all.sh
```

The tests are also part of the `Dockerfile` and will run with every build of the image. Build will fail if tests fail.


### Building and pushing the Docker Image for the Resource

```bash
./build.sh VERSION REPOSITORY
```


### Setting up the CI Pipeline for the Resource

The `ci` folder contains a Concourse pipeline that builds the resource and pushes it to a Docker registry.

```bash
cd ci
export CONCOURSE_FQDN='http://your.concourse.server'
export CONCOURSE_USER='concourse_username'
export CONCOURSE_PASSWORD='concourse_p455w0rd'
export DOCKER_REPO='yourdockerregistry'
export DOCKER_USER='your_user_on_dockerhub'
export DOCKER_PASSWORD='your_password_on_dockerhub'
./setup-pipeline.sh
```


### Troubleshooting & Debugging

* hijacking the resource container in the dev pipeline

    ```bash
    cd dev
    ./fly -t prometheus-pushgateway hijack -j prometheus-pushgateway-dev/pushgw-metric -c prometheus-pushgateway-dev/pushgateway
    ```


TODOs
-----

- [ ] add tests with using env vars (`$BUILD_ID`...)


Resources
---------

* [Promethues Pushgateway](https://github.com/prometheus/pushgateway)
* [Docker Compose Stack for Prometheus](https://github.com/vegasbrianc/prometheus)
* [Developing a custom Concourse Resource](https://content.pivotal.io/blog/developing-a-custom-concourse-resource)
