# Udacity CloudDeveloper: Project Module3 - Microservices
This is the the root for the projects repository of the Udacity Microservice.  The [origin](https://github.com/scheeles/cloud-developer/tree/06-ci/course-03/exercises)  of was copied into [this]([https://github.com/mitschen/udacity-c3-pro-microservice](https://github.com/mitschen/udacity-c3-pro-microservice)) repository. 
The repository contains four subfolders:
* [udacity-c3-deployment](https://github.com/mitschen/udacity-c3-pro-microservice/tree/master/udacity-c3-deployment) contains the details for kubernetes deployment and the docker container setup. Last is used for CI via [Travis](https://travis-ci.org/mitschen/udacity-c3-pro-microservice/branches) or can be used to run locally (see instructions below)
* [udacity-c3-frontend](https://github.com/mitschen/udacity-c3-pro-microservice/tree/master/udacity-c3-frontend) containing the angular-sources for a html frontend 
* [udacity-c3-restapi-feed](https://github.com/mitschen/udacity-c3-pro-microservice/tree/master/udacity-c3-restapi-feed) contains the sources for image feed service
* [udacity-c3-restapi-user](https://github.com/mitschen/udacity-c3-pro-microservice/tree/master/udacity-c3-restapi-user) contains the sources for user authentication

All the README inside the subfolders are used to build and run the different services locally. 

## CI/CD
The repository is connected to the [Travis](https://travis-ci.org/mitschen/udacity-c3-pro-microservice/branches) CI/CD system. It actually builds any changes on master and develop branch. For building, docker-compose is used which refers to the compose-files located in the udacity-c3-deployment folder.
For continious delivery I've adjusted exactly one service, the udacity-c3-restapi-feed in order to demonstrate the delivery branch. The reason I've only chosen one is, that DockerHub allows free usage limited to exactly one repository. That means in all other cases the docker containers fetched from the origin (author scheele).


The .travis.yml file is used to trigger the CI/ CD tasks. Only for master branch, the `docker push` towards DockerHub is realized.


## Prerequisite
In order to run these services, you must have an AWS account and have to prepare a S3, a RDS DB and a AWS user. Please refer to the udacity cloud developer lessons for more details.

### Docker
In order to run it locally using [docker](https://www.docker.com/), make sure to
* install docker according to this [https://docs.docker.com/install/](https://docs.docker.com/install/)
* make sure to export the following variables
```
export POSTGRESS_USERNAME=<aws_postgress_username>
export POSTGRESS_PASSWORD=<aws_postgress_password>
export POSTGRESS_DB=<aws_postgress_database>
export POSTGRESS_HOST=<aws_postgress_endpoint>
export POSTGRESS_DIALECT=postgress
export AWS_REGION=<aws_region>
export AWS_PROFILE=default
export AWS_BUCKET=<aws_s3_media_bucketname>
export JWT_SECRET=<a JWT secret (use something)>
```
**Caution:** udacity changed some of the environment variables from module 2 to module 3.
### Kubernetes Service setup

In order to use kubernetes on your local machine, udacity refers to kubeone. Please follow the [quickstart-aws.md](https://github.com/kubermatic/kubeone/blob/master/docs/quickstart-aws.md) tutorial to install kubeone on your machine.
**Please note**: in addition you must install 
* [terraform](https://www.terraform.io/downloads.html) 
* and you need to make [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux) available

**CAUTION:** Kubernetes requires medium EC2 instances - so these are not covered by the free tier license. In other words, make sure to shutdown the instances following the section [Deleting The Cluster
](https://github.com/kubermatic/kubeone/blob/master/docs/quickstart-aws.md#deleting-the-cluster). Furthermore, doublecheck your EC2 instances in the AWS console - sometimes they weren't stopped automatically!

#### Troubleshooting
* **Missing rsa-key?** Create one by calling `ssh-kegen`which will generate a key for you located by default in `~\.ssh\id_rsa`. You might need to associate the key with your account by typing `eval 'ssh-agent -s'` and afterwards `ssh-add ~/.ssh/id_rsa`. See further details [here ](https://github.com/kubermatic/kubeone/blob/master/docs/ssh.md).
* Getting errormessage like ... call `eval 'ssh-agent -s'` and `ssh-add ~/.ssh/id_rsa`
```
INFO[12:40:11 CEST] Installing prerequisites…
ERRO[12:40:11 CEST] failed to connect to 18.197.96.194: could not open socket "env:SSH_AUTH_SOCK": dial unix env:
```

* I was facing the issue, that the AWS-Credentials where not available/ exported (even though they were). So maybe you must explicitly making them available 


### Credentials for Kubernetes Deployment
In [/udacity-c3-deployment/k8s](https://github.com/mitschen/udacity-c3-pro-microservice/tree/master/udacity-c3-deployment/k8s) you'll find some secret and credential yaml files which expects a base64 representation of your login credentials. In order to do so, apply the following command on your e.g. ~/.aws/credential files to get the corresponding base64 representation
`base64 -w 0 ~/.aws/credentails ~/.aws/credentials.base64`
I was spending hours in finding an issue during deployment on kubernetes, where all my backend-services run into error situation. If you're facing a similar issue, check the logs by
```
kubectl get pods
kubectl logs <the_pod_crashing>
```

This allows to get some details about the reason. If you're facing the problem with sequelize/ authentication issues, it is most propably that your base64 credentials aren't working as expected. Check [this page](https://www.base64encode.org/) to make an online base64 en/decoding.

**Caution:** do not publish them in github

### Scaling
The default deployment won't run with the default setup provided by the [Kubernetes](https://github.com/kubermatic/kubeone/blob/master/docs/quickstart-aws.md) quickstart tutorial. On my machine it results in 3 masters and 1 worker node. This isn't enought power to make the 3 feed, 2 user, 2 frontend and 2 reverse proxies up and running. There are two ways on how you can adjust that
* changing the kubernetes default worker machine - see therefore the `variables.tf` content described in [here](https://github.com/kubermatic/kubeone/blob/master/docs/quickstart-aws.md#creating-infrastructure).
* or to rescale the number of workers subsequent (this is what I've done)
```
// get the machinedeployment name
kubectl get -n kube-system machinedeployment
// rescale the number of workers (will take a while)
kubectl scale -n kube-system machinedeployment/<your_pool> --replicas=<number of replicates>
```

**CAUTION:** as mentioned - don't forget that these machines aren't part for the AWS free tier!

#### Troubleshooting
**Rescale not working**: Please note that the `rescale command` isn't working with `kubectl 1.15.x` - if you're facing an error message, try to get back to `kubectl 1.14.x`
```
//get the kubectl version
kubectl version
//install version 1.14.x
sudo apt-get install kubectl=1.14.\*
```

**Reverseseproxy stay's in CrashLoopBackOff**: unfortunately the tutorial and lessons missed some parts which aren't clear right from the getgo. If you're running in the `reverseproxy-deployment`in the kubenetes, the reverseproxy tries to connect to the backend-services - not to the pods! In otherwords: the deyploment will only get up and running if you're deploying the services in addition!

