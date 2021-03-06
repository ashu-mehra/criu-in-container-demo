# criu-in-container-demo

This project is an attempt to demostrate use of CRIU inside docker containers to improve startup time of applications.
It uses AcmeAir Java application as the use-case. You can read more about AcmeAir [here](https://github.com/sabkrish/acmeair).

## Prereqs For AcmeAir

**1. Install Java 8**
Get it from AdoptOpenJDK: https://adoptopenjdk.net/
**2. Install gradle**\
[Gradle 4.1](https://services.gradle.org/distributions/gradle-4.1-bin.zip) seems to work fine.

## AcmeAir execution process

Process for starting AcmeAir involves following steps:

### 1. Execute setup_acmeair.sh

`$ ./setup_acmeair.sh`

This script performs following steps:

**1.1 Sets up docker network**

Creates a docker network of type `bridge` with name `acmeair-net`.

**1.2 Pulls mongo db docker image**

**1.3 Clones AcmeAir repository**

Clones AcmeAir from the repository `git@github.com:ashu-mehra/acmeair.git`

**1.3 Builds AcmeAir**
**1.4 Creates AcmeAir docker image**

By default the AcmeAir image is named as `acmeair_liberty:latest` which can be configured in `common_env_vars.sh`.

### 2. Execute run_acmeair.sh

`$ ./run_acmeair.sh [docker-image] [container-name]`\
where `docker-image` is the AcmeAir docker image to be used for starting AcmeAir application (default=acmeair_liberty:latest)
and `container-name` is the name of the container to be used for the AcmeAir app container (default=acmaeir-app)

Verify the AcmeAir started by checking logs of the container as:\
`docker logs -t --follow acmeair-app`

You can check the server logs for the time it took to start the application.\
In the output of above command, search for the message `The server defaultServer is ready to run a smarter planet`.\
The timestamp for this message minus the timestamp for the first message would give you the startup time.

Now you should be able to access `http://localhost/`.

A sample output generated by `docker logs`:
```
$ docker logs -t acmeair-server
2019-03-15T08:19:46.645067700Z 
2019-03-15T08:19:47.235712655Z Launching defaultServer (WebSphere Application Server 19.0.0.2/wlp-1.0.25.cl190220190222-1311) on IBM J9 VM, version 8.0.5.30 - pxa6480sr5fp30-20190207_01(SR5 FP30) (en_US)
2019-03-15T08:19:47.392605173Z [AUDIT   ] CWWKE0001I: The server defaultServer has been launched.
2019-03-15T08:19:47.403278238Z [AUDIT   ] CWWKE0100I: This product is licensed for development, and limited production use. The full license terms can be viewed here: https://public.dhe.ibm.com/ibmdl/export/pub/software/websphere/wasdev/license/base_ilan/ilan/19.0.0.2/lafiles/en.html
2019-03-15T08:19:47.886158936Z [AUDIT   ] CWWKG0093A: Processing configuration drop-ins resource: /opt/ibm/wlp/usr/servers/defaultServer/configDropins/defaults/keystore.xml
2019-03-15T08:19:50.782259872Z [AUDIT   ] CWWKZ0058I: Monitoring dropins for applications.
2019-03-15T08:19:54.029871485Z [AUDIT   ] CWWKT0016I: Web application available (default_host): http://70a7c9c472d6:80/
2019-03-15T08:19:54.033891368Z [AUDIT   ] CWWKZ0001I: Application acmeair-monolithic started in 2.284 seconds.
2019-03-15T08:19:54.138431716Z [AUDIT   ] CWWKF0012I: The server installed the following features: [managedBeans-1.0, servlet-3.1, jndi-1.0, json-1.0, cdi-1.2, websocket-1.1, jaxrs-2.0, jaxrsClient-2.0].
2019-03-15T08:19:54.139383718Z [AUDIT   ] CWWKF0011I: The server defaultServer is ready to run a smarter planet.
```

Here the time take for startup can be obtained by `08:19:54.139383718-08:19:46.645067700` which is approximately `7.5 seconds`.

## Using CRIU

This project shows how to incorporate CRIU into the mix and use it to startup AcmeAir real quick.

### Idea

Basic idea is to incorporate the checkpoint of the application in the docker image itself, so that when a new container is started, the application can be restored from the checkpoint. This way it won't have to go through the startup process.

### Scripts

To achieve this, copied the generic scripts from https://github.com/ashu-mehra/criu-for-docker.
Then updated `app.sh` and `run_app_docker_image.sh` based on the AcmeAir Liberty setup.

Now run `driver.sh` as:

`$ ./driver.sh acmeair_liberty_checkpoint`

On success it would create a new container image with name `acmeair_liberty_checkpoint:latest` which contains the AcmeAir application checkpoint.

Use this image to start the AcmeAir as:
`$ ./run_acmeair.sh acmeair_liberty_checkpoint acmeair-app`

You should now be able to access `http://localhost/`.
