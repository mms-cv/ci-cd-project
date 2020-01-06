# Simple CD pipeline demo at scale with Jenkins Swarm,Docker Swarm and Goland


[![CD pipeline demo at scale with Jenkins Swarm,Docker Swarm and Goland](https://img.youtube.com/vi/USxRrMWzK1s/0.jpg)](https://youtu.be/USxRrMWzK1s "The video is a step by step tutorial for a simple Golang application and how to create a continuous delivery pipeline using  Jenkins Swarm ,Docker swarm, Docker Secrets")

### [Original blog post link](https://www.vip-consult.solutions/post/easy-docker-swarm-jenkins-continuous-deployment-at-scale#content)

## First let's create the jenkins master

	docker service create --name jenkins-master \
		-p 50000:50000 \
		-p 80:8080 jenkins

* **port 50000** used by all joining slaves created in the next step.
* **created as a service** so that docker swarm will manage the container and reschedule it if needed. 
* **/var/jenkins_home** is used for all data so in a real deployment this need to be either a mount a folder from the host or use some docker storage driver. 


### Required plugins
* **Pipeline** the new Pipeline as Code plugin that allows to specify the steps in a Jenkinsfile
* **GitHub plugin** to checkout and push to github
* **Self-Organizing Swarm Plug-in Modules** deploy each Docker swarm node as a jenkins slave
* **GitHub Branch Source Plugin** scan a github account and automatically create jobs for repos that have a Jenkinsfile in the root directory. It also creates a github webhook so every repo push will trigger a build.


## Now let's add some jenkins slaves

first create a secret that will be used by the Jenkins slaves to connect to the master
	
	echo "-master http://10.0.0.101 -password admin -username admin"|docker secret create jenkins-v1 -
* **10.0.0.101** replace it with actual Jenkins master IP
* *jenkins-v1 instead of just jenkins so that we can rotate the passwords - [Docker docs](https://docs.docker.com/engine/swarm/secrets/#example-rotate-a-secret)*


**On the testing docker swarm**

	docker service create \
		--mode=global \
		--name jenkins-swarm-agent \
		-e LABELS=docker-test \
		--mount "type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock" \
		--mount "type=bind,source=/tmp/,target=/tmp/" \
		--secret source=jenkins-v1,target=jenkins \
		vipconsult/jenkins-swarm-agent

**On the staging docker swarm**

	docker service create \
		--mode=global \
		--name jenkins-swarm-agent \
		-e LABELS=docker-stage \
		--mount "type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock" \
		--mount "type=bind,source=/tmp/,target=/tmp/" \
		--secret source=jenkins-v1,target=jenkins \
		vipconsult/jenkins-swarm-agent
	
**On the production Docker swarm**

	docker service create \
		--mode=global \
		--name jenkins-swarm-agent \
		-e LABELS=docker-prod \
		--mount "type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock" \
		--mount "type=bind,source=/tmp/,target=/tmp/" \
		--secret source=jenkins-v1,target=jenkins \
		vipconsult/jenkins-swarm-agent
	
* **[vipconsult/jenkins-swarm-agent](https://hub.docker.com/r/vipconsult/jenkins-swarm-agent)**  is simple image that runs the jenkins swarm plugin which connects to a given master using parameters from  the Docker secret 
* **mode=global** when we add new nodes to the Docker cluster they will all connect to the jenkins master to serve as slaves
* **mount "type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock" ** we mount the Docker socket so that we can use the Docker client inside a container to start new containers or services directly on the host. This is a better approach than Docker in Docker
* **mount "type=bind,source=/tmp/,target=/tmp/"**  all containers can share files generated by the same Jenkins job
* **LABELS=docker-test**  the jenkins slave label. This is used when you define where to run each pipeline step - the labels need to match the ones used in the Jenkinsfile - node("docker-test") , node("docker-stage") , node("docker-prod")
* **secret source=jenkins-v1,target=jenkins** using the secret created in the previous step, needs to match a user on the jenkins master


## Finally lets start building!
Create a new GitHub Organization job using the Jenkins GUI - this will scan all repositories for a Jenkinsfile in the root directory and add it as a new job.
* **Owner** this is the github account or organization - **vipconsult** in this case
* **Scan credentials** need to add some github credentials - anonymous github api requests are limited and will fail to scan all repositories.
* The Jenkinsfile is set to use credentials with ID "*DockerHub*" so also need to create those.


## Automate the builds with Github Webhooks
Using the Github Webhooks every repo change will trigger a new build.

In my case the jenkins master is behind a router so the webhook was created with my internal IP so I needed to open a firewall port so that the webhook can hit the Jenkins master. 
Here is a little nifty tool that simplifies the whole process and works even when you don't have access to the router.
It uses UPNP protocol to automatically open and map a port to your Jenkins master machine
	
	apt-get install miniupnpc
	upnpc -a 10.0.0.101 80 8081 tcp
* **10.0.0.101** Jenkins master internal IP
* **80** Jenkins Master port
* **8081** external port useg by Github webhooks
so after this Github can send webhooks using your external ip and the external port like http://216.58.213.78:8081/


## Required :) - Fork it and give it a try
Just [fork the project](https://github.com/krasi-georgiev/cd-demo) and change the docker hub username at the top of the Jenkinsfile.