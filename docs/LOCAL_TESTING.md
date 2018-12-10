# Running Casablanca Quoting Service Locally

This document is intended to guide a user through the steps required to run the casablanca quoting service locally.

## Prerequisites

Ensure you have to following:

 1. A linux operating system (either native or a VM - ubuntu is a reasonable choice if you arent fussy)
 2. Docker CE installed (e.g. see https://docs.docker.com/install/linux/docker-ce/ubuntu/)
 3. A user account for the casablanca docker registry (see Infrastructure page on Confluence)

## Get the Docker images

1. Authenticate with artifactory:

    `$ docker login casablanca-casa-docker-release.jfrog.io`

    Enter your account credentials when prompted.

2. Pull the required docker images locally:

    `$ docker pull mysql/mysql-server:5.7`
    `$ docker pull casablanca-casa-docker-release.jfrog.io/central-ledger-init:latest`
    `$ docker pull casablanca-casa-docker-release.jfrog.io/quoting-service-init:latest`
    `$ docker pull casablanca-casa-docker-release.jfrog.io/quoting-service-api-adapter:latest`

## Create and Configure a Database Server Container:

1. Start a mysql 5.7 server container:

    `$ docker run -d --name mysql mysql/mysql-server:5.7`

2. Create a new user in the database server for the quote service to use:

    1. Get the autogenerated root password for the mysql server:
    `$ docker logs mysql`
    This will print some text to the terminal containing a randomly generated root password.

    2. Open a command shell in the database container:
    `$ docker exec -it mysql /bin/sh`

    3. Start the mysql command line interface:
    `sh-4.2# mysql -p`
    Enter the root password you got in step 1 above

    4. Change the root user password (REQUIRED!):
    `mysql> alter user 'root'@'localhost' identified by 'root';`

    5. Create the database user and grant it permissions:
    `mysql> create user 'casa'@'%' identitfied by 'casa';`
    `mysql> grant all privileges on *.* to 'casa'@'%';`
    `mysql> flush privileges;`

    6. Create the core database schema:
    `mysql> create database central_ledger character set utf8 collate utf8_general_ci;`
    `mysql> exit;`
    `sh-4.2# exit`

    7. Find the IP address of the database container:
    `$ docker inspect mysql | grep \"IPAddress\"`
    This will print the container IP address to the terminal

    8. Initialise the core database schema:
    Run the following command but substitute the IP address you got from step 7 above with [IPADDRESS] (remove the square brackets also):
    `$ docker run -e "CLEDG_DATABASE_URI=mysql://casa:casa@[IPADDRESS]:3306/central_ledger" --name cli casablanca-casa-docker-release.jfrog.io/central-ledger-init:latest`
    Check the output for errors. There should not be any.

    9. Initialise the quoting service database schema:
    Run the following command but substitute the IP address you got from step 7 above with [IPADDRESS] (remove the square brackets also):
    `$ docker run -e "CLEDG_DATABASE_URI=mysql://casa:casa@[IPADDRESS]:3306/central_ledger" --name qsi casablanca-casa-docker-release.jfrog.io/quoting-service-init:latest`
    Check the output for errors. There should not be any.

    >*Note: If you need to re-run steps 8 and/or 9 above you will need to remove the existing containers by running:*
    `$ docker rm cli`
    `$ docker rm qsi`

## Create and Start the Quoting Service

1. Start an instance of the quoting-service-api-adapter container:

    `$ docker run -d -e "DATABASE_HOST=172.17.0.2" --name qapi casablanca-casa-docker-release.jfrog.io/quoting-service-api-adapter:latest`

2. Optional: You can watch the quoting service logs by running the following command:

    `$ docker logs --follow qapi`

## Run Tests

1. Find the IP address of the quoting service container:

    `$ docker inspect qapi | grep \"IPAddress\"`

2. Use Postman or similar to make calls to the quoting service API:

    1. Note that the API base address will be of the format:
    `http://[IPADDRESS]:3000/quotes`

3. Details of the API exposed by the quoting service is available as swagger yaml in the casablanca github here:

    `https://github.com/casablanca-project/quoting-service-api-adapter/blob/master/src/config/fspiop-rest-v1.0-OpenAPI-implementation-quoting.yaml`