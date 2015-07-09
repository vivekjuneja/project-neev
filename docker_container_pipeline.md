#Purpose :-

Demonstrate that we can setup a simple Deployment Pipeline that builds a Project, and deploys consistently to any type of infrastructure

#Why :-

To build Agile infrastructure, it is important to have a valid Demonstration that underlies the core idea of a fast feedback loop :- consistent, automated development to deployment loops. 

#Setup :-

**We need the following components for the system** :-

1. Jenkins for CI and CD Orchestration
2. Mesos Cluster with Marathon - that supports Docker Containerizer
3. Git Repository (basically any Source Repository will do)
4. Docker Private Registry
5. Bakery Server
6. VAMP Cluster (for Canary Release as of now)

**Overall steps to use the system for a Project** :-

1. Dockerize the current Project (example :- Monolithic : Single application has all the dependencies, can use external data sources, caching etc.)

 - How to dockerize a Project ?

 - How to test if the Dockerization worked ?

2. Use the Jenkins Project template to arrive at a customized Project build and deployment template

2. Create a Deployment JSON for the Current Project - will be used to deploy on Mesos 
 - Each Project is deployed via a Deployment JSON file that is essential a payload to communicate with Marathon to initiate deployment
 - Let us look at a Sample Deployment JSON for a simple Java Spring based web Application Petclinic :-
```
{
    "id": "/deployed_service_name",
    "groups": [
        {
            "id": "/deployed_service_name/data",
            "apps": [
                {
                    "id": "/deployed_service_name/data/mysql-db",
                    "cpus": 0.5,
                    "mem": 512,
                    "container": {
                        "type": "DOCKER",
                        "docker": {
                            "image": "mysql",
                            "network": "BRIDGE",
                            "portMappings": [
                                {
                                    "containerPort": 3306,
                                    "servicePort": 13306
                                }
                            ]
                        }
                    },
                    "env": {
                        "MYSQL_ROOT_PASSWORD": "root",
			             "MYSQL_DATABASE" : "petclinic"
                    },
  		   "healthChecks" : [{
			"protocol" : "COMMAND",
			"command" : { "value" : "mysql -u root -proot -h 172.17.42.1 -P 13306"},
 			"maxConsecutiveFailures" : 3
		   }]
                }
            ]
        },
        {
            "id": "/deployed_service_name/app",
            "dependencies" : ["/deployed_service_name/data"],
            "apps": [
                {
                    "id": "/deployed_service_name/app/spring-app",
                    "cpus": 0.5,
                    "mem": 512,
                    "container": {
                        "type": "DOCKER",
                        "docker": {
                            "image": "image_registry_location/deployed_service_name:latest",
                            "network": "BRIDGE",
                            "portMappings": [
                                {
                                    "containerPort": 8080,
                                    "servicePort": 4569
                                }
                            ]
                        }
                    }
                }
            ]
        }
    ]
}
```


**Preparing Jenkins Infrastructure** :-

Jenkins is run as a Docker Container. Its a Standalone system for now. 

```
docker run -d -v <LOCATION_OF_THE_DOCKER_SOCKET>:/var/run/docker.sock -v <LOCATION_OF_DOCKER_BINARY>:/usr/bin/docker -v <LOCATION_OF_THE_JENKINS_DATA_DIRECTORY>:/var/jenkins_home -v <LOCATION_OF_THE_SSH_KEY_DIRECTORY_CONFIGURED_FOR_GIT>:/var/jenkins/.ssh <JENKINS_DOCKER_IMAGE> 
```

Eg:- 

```
docker run -v /var/run/docker.sock:/var/run/docker.sock -v $(which docker):/usr/bin/docker -v /var/local/jenkins_data:/var/jenkins_home -v /Users/testuser/.ssh/:/var/jenkins_home/.ssh/ localhost:5000/jenkins
```

In the above command, we allow Jenkins (running in Docker itself) to use Docker to perform Build and Push to Registry. We also provide Jenkins the access to the SSH Keys that will be used to connect to Github


In a typical Jenkins Project, following are the key Things that are put in as "Post" Build Activities :-
```
sudo docker build -t ${image_registry_location}/${deployed_service_name}:${BUILD_ID} . #Build the Docker Container Image for the deployable

sudo docker push ${image_registry_location}/${deployed_service_name}:${BUILD_ID} #Push the built Docker Image to the Docker Registry

curl -X DELETE -H "Content-Type: application/json" http://${deployment_endpoint}/v2/groups/${deployed_service_name} #Remove the current deployed Application on the Mesos Cluster using Marathon API
```

The below steps are to modify the Deployment JSON file that exists in each Project directory. This Deployment JSON contains the Marathon Deployment information for that application. 

```
cp -f deploy_app.json deploy_app.json.tmp # Take a copy of the JSON before modifying 

sed -i "s/latest/${BUILD_ID}/g" deploy_app.json.tmp #Set the Jenkins Build number to the Image version 

sed -i "s/image_registry_location/${image_registry_location}/g" deploy_app.json.tmp #Set the Image registry location

sed -i "s/deployed_service_name/${deployed_service_name}/g" deploy_app.json.tmp #Set the Service Name to be used for deployment 

curl -X POST -H "Content-Type: application/json" http://${deployment_endpoint}/v2/groups?force=true -d@deploy_app.json.tmp #Make a Call to Marathon to deploy a new Copy of the Application instance
```

**Preparing Mesos Infrastructure (Standalone)** :-

Mesos Cluster with Marathon is installed on an individual VM that uses Docker Containerizer for deploying workload

1. Use the Vagrant Mesos to set it up on Virtual Box : https://github.com/everpeace/vagrant-mesos

2. Docker must be configured to run with insecure private docker registry (if not secured)

	-Option 1 - Modify the Docker configuration to use the private docker registry

	-Option 2 - Stop the pre-existing Docker service, and manually run the Docker runtime
	- /usr/bin/docker -d --host=unix:///var/run/docker.sock --restart=false --insecure-registry <REGISTRY_HOST>:<IP_ADDRESS>

3. Test if the Docker Containers can be deployed on Mesos slave via the Marathon API : https://mesosphere.github.io/marathon/docs/native-docker.html

4. Setup HAProxy to handle the Service Discovery for Multi-Apps ) : https://mesosphere.github.io/marathon/docs/service-discovery-load-balancing.html

5. Sample Applications that can be deployed on this Mesos Cluster to test if everything is working :-

6. Commands that can be handy when testing Marathon :-

 a. Deploying a Single App
  ```
  curl -X POST -H "Content-Type: application/json" http://<MARATHON_URL>/v2/apps -d@<DEPLOYMENT_JSON_FILE>
  ```
   
 b. Deploying an Application that is composed of many tiers - app, db, caching etc. 
  ```
   curl -X POST -H "Content-Type: application/json" http://<MARATHON_URL>/v2/groups -d@<DEPLOYMENT_JSON_FILE>
  ```
  
 c. Remove the pre-existing application deployed on Mesos
 ```
  curl -X DELETE -H "Content-Type: application/json" http://<MARATHON_URL>/v2/apps/<APP_DESCRIPTOR> -d@<DEPLOYMENT_JSON_FILE>
  ```

**Preparing the Docker Registry** :-

Docker Registry is run as a Docker container

1. Create a Self Signed Certificate 
```
mkdir -p certs && openssl req \
	-newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key \
	-x509 -days 365 -out certs/domain.crt
```

use the CN as the Host Name of the Registry Server / Registry Host in the above command.

2. Copy the Domain certificate "domain.crt" to the /etc/docker/certs.d/<REGISTRY_HOST>:5000/ca.crt

3. Run the Docker Registry 
```
  docker run -d -p 5000:5000 \
    -e REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/var/lib/registry \
    -v <LOCATION_TO_MOUNT_THE_CONTAINER_FILE_SYSTEM>:/var/lib/registry \
    --restart=always --name registry registry:2
```

**EXPERIMENTS : Using VAMP to allow Canary release** :-

VAMP is a micro-services platform that allows Netflix style Canary Release, SLA management including metrics, and overall Service Management. 

As of now, we want to use VAMP to perform Canary Releases for the applications that use this Pipeline.

*How do we use VAMP in our Infrastructure*

a. We deploy the VAMP router as mentioned on http://vamp.io/documentation/getting-started/installation/

Call the Marathon URL to deploy the VAMP Router:-

```
curl -v -H "Content-Type: application/json" -X POST --data @vamp-router.json http://10.143.22.49:8080/v2/apps
```

The vamp-router.json is as following :-


```
 {
      "id": "main-router",
      "container": {
          "docker": {
              "image": "magneticio/vamp-router:latest",
              "network": "HOST"
          },
          "type": "DOCKER"
      },
      "instances": 1,
      "cpus": 0.5,
      "mem": 512,
      "env": {},
      "ports": [
          0
      ],
      "args": [
          ""
      ]
    }
```

Now, we will deploy the VAMP core on an another Docker container, using the instructions as mentioned in the above link.

Set some environment variables that VAMP core would use :-

```
export MARATHON_MASTER=<MARATHON_HOST_IP>
export VAMP_MARATHON_URL=http://$MARATHON_MASTER:8080
export VAMP_ROUTER_HOST=<VAMP_ROUTER_IP_DEPLOYED_ON_MESOS>
export VAMP_ROUTER_PORT=10001
export VAMP_ROUTER_URL=http://$VAMP_ROUTER_HOST:$VAMP_ROUTER_PORT
```

Now, we will deploy VAMP Core on a local container :-

```
docker run -d --name=vamp -p 81:80 -p 8081:8080 -p 10002:10001 -p 8084:8083 -e VAMP_MARATHON_URL=http://$MARATHON_MASTER:8080 -e VAMP_ROUTER_URL=http://$VAMP_ROUTER_HOST:10001 -e VAMP_ROUTER_HOST=$VAMP_ROUTER_HOST magneticio/vamp-mesosphere:latest
```

For the sample application here, Spring PetClinic, following is the Deployment JSON we use to deploy the Application. In order to do this, we have to issue REST API call to the VAMP CORE  :-

```
curl -v -X POST --data-binary @deploy_spring_petclinic.yaml -H "Content-Type: application/x-yaml" http://<VAMP_CORE_HOST>:8080/api/v1/deployments
```

Here, the VAMP_CORE_HOST is the Host IP address where the VAMP Core Docker Container is currently running. 

The "deploy_spring_petclinic.yaml" file holds the deployment manifest that is used by VAMP to deploy on Mesos. 

For demonstrating the Canary Release, we made changes to the Spring PetClinic Application. The changed and unchanged version of the Spring Petclinic are bundled as independent Docker Container Images identified by different version number. With that background, let us look at the "deploy_spring_petclinic.yaml" :-

```
name: petclinic:1.0

endpoints:
  petclinic.port: 9050

clusters:

  petclinic:
    services: # services is now a list of breeds
        breed:
          name: petclinic-frontend:1.0.0
          deployable: 10.52.179.232:5000/spring-petclinic-app:24
          ports:
            port: 8080/http
          environment_variables:
            DATABASE_BACKEND: $database_backend.host:$database_backend.ports.port
          dependencies:
            database_backend: petclinic-backend:1.0.0
        scale:
          cpu: 0.5
          memory: 512
          instances: 1          
        routing: 
          weight: 50  # weight in percentage   
       breed:
          name: petclinic-frontend:1.0.1
          deployable: 10.52.179.232:5000/spring-petclinic-app:23
          ports:
            port: 8080/http
          environment_variables:
            DATABASE_BACKEND: $database_backend.host:$database_backend.ports.port
          dependencies:
            database_backend: petclinic-backend:1.0.0
        scale:
          cpu: 0.5
          memory: 512
          instances: 1          
        routing: 
          weight: 50  # weight in percentage 	

  database_backend: 
    services:
        breed:
          name: petclinic-backend:1.0.0
          deployable: mysql
          ports:
            port: 3306/http
          environment_variables:
            MYSQL_ROOT_PASSWORD: root
        scale:
          cpu: 0.5
          memory: 512
          instances: 1          
        routing: 
          weight: 100  # weight in percentage  

```

In the above example, both the changed and unchanged service use the same database. That means, there has been No change to the Database. 


**BAKERY Server** :-

The purpose of Bakery Server :-

1. Access to Package Repository (Chef recipes, ```ZINST packages```, Puppet Scripts)
2. Create Base Images that use Standard Package tool for package deployment
3. Create smallest images that are ready to use

How does the Bakery work :-

1. On Bakery, Baker pulls out standard Base (Golden) Container Images from the Private Docker registry
2. Based on the specification, installs the standard packages needed to support the spec
3. Create Test cases to test the Baked images
4. *Push the Image to Jenkins CI to push it to Private Registry*

Example:-
1. Standard Tomcat Container images are NOT Good enough. We need to customize it to make it configurable when deploying applications

The Baker configures the "setenv.sh" file that sets a JAVA_OPT variable containing the JDBC URL of the Application. Note the $DATABASE_BACKEND used in the jdbc.url. This environment variable will be set at Runtime, when Container is deployed. 

```
export JAVA_OPTS="-Djdbc.url=jdbc:mysql://$DATABASE_BACKEND/petclinic"
```

The Dockerfile for new version of Tomcat is baked like this :-

```
FROM 10.52.179.232:5000/tomcat:8.0
WORKDIR /usr/local/tomcat
COPY setenv.sh bin/

EXPOSE 8080
CMD ["catalina.sh", "run"]
```

Now, when this Tomcat Container Image is used, it will be able to configure the JDBC URL based upon the runtime configuration. For example :-

```
docker run -v /target/petclinic.war:/usr/local/tomcat/webapps/petclinic.war -e DATABASE_BACKEND=192.168.33.12:31000 tomcat:revised
```

**Monitor and Manage the entire Stack** :-

We want to establish some important considerations when using this pipeline in real projects :-

a. Centralized log aggregation for all Application Containers deployed on this stack.

b. Log Dashboard for all components :-
   - Jenkins
   - Mesos, Marathon
   - Docker Daemon
   - Docker Private Registry
   - Application Containers running over Mesos
   
   We use Elastic Search, Logstash, Kibana for log aggregation and dashboard. 
   
c. Unified Service Health Check for all components :-
   - Jenkins
   - Mesos, Marathon
   - Docker Daemon
   - Docker Private Registry
   - Application Containers running over Mesos
   
