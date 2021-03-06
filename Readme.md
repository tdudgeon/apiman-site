# About
This is a Docker project that illustrates how to set up a Dockerised environment for Apiman containing:

- [PostgreSQL database](http://www.postgresql.org/)
- [Elasticsearch](https://www.elastic.co/products/elasticsearch)
- [Keycloak](http://keycloak.jboss.org/) authentication server
- [Apiman](http://www.apiman.io/) for API management

The aim is to generate a fully dockerised and scaleable environment that can be used as a template for a production system.
The core of this is Apiman, and this uses a number of separate subsystems (Keycloak, a relational database, Elasticsearch), and Apiman itself has 2 major parts, the API mananger and the API gateway.
In the default "quickstart" deployment all of these parts are present as one big system, but this is not so suitable for production use, so the aim of this work is to break it out into the separate parts, each as a Docker container and to get these to play nicely with each other.

Other aspects that we want to handle:

1. provision of SSL keys (self signed for testing, real ones for production)
2. how to generate integral backups of the entire system
3. defining and documenting how to lock down security in a manner suitable for a production system

# Status

IMPORTANT: this is not currently suitable for production use.

This work is incomplete. So far we have:

- separate Docker containers for PostgreSQL, Elasticsearch, Keycloak and Apiman
- Keycloak and Apiman using PostgreSQL for persistence

Much is still to be done, including

- separate containers for Apimman manager and Apiman gateway
- set up of SSL keys
- describe how to make system secure
- describe how to backup and restore 

# Instructions
The process is divided into 4 stages:

1. quickstart with mostly default settings to get a basic (insecure) system running
2. example of how to set up a service with Apiman
3. instructions for what needs to be changed to make the sytem suitable for production
4. handling backup and disaster recovery

# Quickstart

## Docker
Install Docker and Docker Compose [see here](https://docs.docker.com/compose/install/)

## Edit configuration files

Create a keystore.jks containing your keystore. In production this would be your public certificate (signed by a root authority). In testing it would be a self-signed certificate. Self signed certificates for localhost and 192.168.59.103 are provided - copy one of these to keystore.jks or create your own as needed.

Also, if your server name is not 192.168.59.103 you need to edit these files:

- apiman/apiman.properties - edit the apiman-gateway.public-endpoint property at the bottom of this file to point to your server
- apiman/standalone-apiman.xml - edit the kc:auth-server-url element (around line 425) to point to your server
- apiman-realm.json - edit the redirect URLs

In testing the server will likely be localhost or the IP address of the docker host (if using boot2docker or docker machine). In production it would be the name or IP address of your public server.

## Build Docker images
The main docker-compose.yml file is present in the top level directory. This uses docker builds that are contained in subdirectories, currently ones for:

- /postgres
- /keycloak
- /apiman

The standard elacticsearch docker image from Dockerhub is used for the Elasticsearch container.

`docker-compose build`

This uses the docker-compose.yml file in the top directory to build the different Docker images that are needed.
NOTE: docker-compose by default uses the parent directory name as the root stem for the images/container names. If your root directory name is not apiman-site then some of the commands that follow will need adjusting, or you can use the -p argument for docker-compose to set the name to something different from the directory name.
 
## Deploy database
`docker-compose up -d postgres elasticsearch`

This deploys a Docker container with the PostgreSQL database.

```sh
$ docker-compose up -d postgres elasticsearch
Starting apimansite_elasticsearch_1...
Starting apimansite_postgres_1...
$ docker ps
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                    NAMES
1cd1cfa14019        elasticsearch         "/docker-entrypoint.s"   57 seconds ago      Up 18 seconds       9200/tcp, 9300/tcp       apimansite_elasticsearch_1
503c46e75738        apimansite_postgres   "/usr/lib/postgresql/"   3 minutes ago       Up 18 seconds       0.0.0.0:5432->5432/tcp   apimansite_postgres_1
```

## Import apiman realm into Keycloak
To import the realm definition that Apiman needs into Keycloak you need to start the keycloak server with special arguments that tell it to import a real definition from a json file. We can't do that when we build the image as it is persisted in the database, so we first start the database, and then fire up a temporary Keycloak container that populates the database with the real configuration.

The apiman-realm.json file contains the definition of the apiman realm that is expected by Apiman.
To achieve this start the Keycloak container with these special startup args. This starts keycloak and imports the realm definition.

```sh
docker run -it --link apimansite_postgres_1:postgres -e POSTGRES_DATABASE=keycloak -e POSTGRES_USER=keycloak -e POSTGRES_PASSWORD=keycloak --rm -v $PWD:/tmp/json apimansite_keycloak /opt/jboss/keycloak/bin/standalone.sh -b 0.0.0.0 -Dkeycloak.migration.action=import -Dkeycloak.migration.provider=singleFile -Dkeycloak.migration.file=/tmp/json/apiman-realm.json -Dkeycloak.migration.strategy=OVERWRITE_EXISTING
```

(replace apimansite_postgres_1 and apimansite_keycloak with the appropriate names if your root directory name is different, and change usernames and passwords as needed).

Once done Ctrl-C to shutdown that container. We don't need it any more as its job is done.
The result of this exercise is that the PostgreSQL keycloak database has had the apiman realm definition added. Next time we start Keycloak that realm will be present.

## Deploy all containers
Now start all the containers using `docker-compose up -d`.

```sh
$ docker-compose up -d
Creating apimansite_elasticsearch_1...
Creating apimansite_apiman_1...
Recreating apimansite_postgres_1...
Creating apimansite_keycloak_1...
$
```

You should now have a basic functioning system. 
Check this by connecting to keycloak using admin/admin as credentials (you are prompted to change the password):

https://192.168.59.103:8443/auth/admin/

You should see the apiman realm that was imported.
Now check you the apiman manger using admin/admin123! as credentials:

https://192.168.59.103:9443/apimanui/

You should se the API Management console.

So far so good. We have a functioning setup, so let's use it to set up a service.

# Running Echo service through apiman

## Setting up echo service
We'll use the apiman echo service sample app to test things. For this we'll run the echo app to wildfly in the apiman container.
The is done by default in the apiman Dockerfile, but if you need to use a different version you can build as follows:
Start an apiman container.
Install maven into the apimman contianer (as root):

```sh
docker exec -it -u root apimansite_apiman_1 bash -c 'curl -o /etc/yum.repos.d/epel-apache-maven.repo https://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo; yum install -y apache-maven'
```

Now build the echo-service war file:

```sh
docker exec -it apimansite_apiman_1 bash -c 'cd /opt/jboss/wildfly/apiman/quickstarts/echo-service; mvn clean package'
```

Now copy it from the container:

```sh
docker cp apimansite_apiman_1:/opt/jboss/wildfly/apiman/quickstarts/echo-service/target/apiman-quickstarts-echo-service-1.1.8.Final.war apiman/
```

(change version number as needed).
Finally edit apiman/Dockerfile to reflect the updated war file to deploy, and rebuild the Docker image:

```sh
docker-compose build apiman
```

One you have a running container check the echo service is there:

```sh
$ curl http://192.168.59.103/apiman-echo/bla/bla
{
  "method" : "GET",
  "resource" : "/apiman-echo/bla/bla",
  "uri" : "/apiman-echo/bla/bla",
  "headers" : {
    "Host" : "192.168.59.103",
    "User-Agent" : "curl/7.37.1",
    "Accept" : "*/*"
  },
  "bodyLength" : null,
  "bodySha1" : null
```

## Configuring apiman to use the echo service
We'll take a bare bones approach here and set it up as a public service.

1. Log in to apiman at https://192.168.59.103/apimanui using admin/admin123!
2. Create an organisation
3. Create a new service in that organisation, name it echoservice
4. For the implementation choose http://localhost:8080/apiman-echo as a REST service with no API security. In this case the service is running within the same container so the host should be localhost and the port the port on which wildfly is actually running, not the port that Docker exposes. In a real world case you would point to a service running on a different server.
5. In the Plans section check the "Make this service public" option.
6. Publish the service.
7. Find out what the endpoint is. Should be something like this: https://192.168.59.103/apiman-gateway/MyOrganisation/echoservice/1.0
8. Test it works:

```sh
$ curl -k https://192.168.59.103/apiman-gateway/MyOrganisation/echoservice/1.0/hello
{
  "method" : "GET",
  "resource" : "/apiman-echo/hello",
  "uri" : "/apiman-echo/hello",
  "headers" : {
    "Host" : "localhost:8080",
    "Accept-Encoding" : "gzip",
    "User-Agent" : "curl/7.37.1",
    "Accept" : "*/*",
    "Connection" : "Keep-Alive"
  },
  "bodyLength" : null,
  "bodySha1" : null
}$
```

# Branding
You'll probably want to change the branding so that at the very least it contains your logos, or maybe to completely change the L&F.
A full description is out of scope, but here's a brief guide to making some simple changes to basic things like icons. For this we take the approach of copying the keyloak theme and making a few changes to it.

For full instructions on keycloak themes see [here](http://keycloak.github.io/docs/userguide/html/themes.html).

## Copying the keycloak theme
Create a themes directory and copy the existing keycloak theme configuration to it and give it the name banana (choose your own name!).

```sh
mkdir themes
docker cp apimansite_keycloak_1:/opt/jboss/keycloak/standalone/configuration/themes/keycloak themes/
mv themes/keycloak themesthemes/banana
```

Edit the content as required (for basic stuff you probably just want to change the icons and css). the very minimum you may want to change is:

- login/resources/img/keycloak-logo.png
- account/resources/img/keycloak-logo.png
- admin/resources/img/keyclok-logo.svg (note the typo)
- welcome/resources/keycloak_logo.png

But you might well also need to look at the css and other content.

Then mount that dir into your keycloak image as volume. First change docker-compose.yml so that the keycloak container mounts the directory as a volume.
Add this to your keycloak cotnainer definition:

```
    volumes:
    - "themes/banana:/opt/jboss/keycloak/standalone/configuration/themes/banana"
```

## Optional: create a customised keycloak-server.json 

This file lets you define your theme as the default theme and to change the caching properties which can be useful as it allows you to make changes and see the impact directly.

You might want to edit keycloak/keycloak-server.json. This is present in the github repo, but if you need to update it to a newer version you can do so like this:

```sh
docker cp apimansite_keycloak_1:/opt/jboss/keycloak/standalone/configuration/keycloak-server.json keycloak/
```

Then modify the theme section to look like this (according to your needs):

```
    "theme": {
        "default": "banana",
        "staticMaxAge": -1,
        "cacheTemplates": false,
        "cacheThemes": false,
        "folder": {
          "dir": "${jboss.server.config.dir}/themes"
        }
    },
```

The standard keycloak Dockerfile copies this file to the container. If you don't edit this file you will need to set the themes manually using the keycloak UI.

Then restart the keycloak container:

```sh
docker-compose stop keycloak
docker-compose rm -f keycloak
docker-compose build keycloak
docker-compose up -d --no-recreate keycloak
```


# Making the setup more secure
So far we've mostly used default settings which means that passwords and certificates are those found in the github repository, so this setup is far from suitable for a prodcution system. In this section we address this, and other aspects relating to security.

TODO - write this section.

# Handling backups and disaster recovery
 
TODO - write this section.
