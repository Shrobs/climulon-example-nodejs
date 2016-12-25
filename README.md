# Climulon example NodeJS

[![Build Status](https://travis-ci.org/Shrobs/climulon-example-nodejs.svg?branch=master)](https://travis-ci.org/Shrobs/climulon-example-nodejs)

This is a micro nodejs/redis app that showcases how **Climulon** is used to provision/decommission infrastructure and run docker apps in a single command.  
The Node app this showcase is based on was written by [Pascal Cremer](https://github.com/b00giZm)

## Description

This repo contains a micro nodejs express app, that stores the number of times you accessed its index page in redis, and displays it in the page itself.  
The repo also contains the description of the infrastructure, in code, that **Climulon** is gonna use to provision and run the app.

## Getting started

### Prerequesites

Make sure you have **Climulon** installed in your machine (or use its docker images).  
You need to have your AWS credentials set and ready for use, like you do for the [AWS cli](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) or [boto](http://boto3.readthedocs.io/en/latest/guide/configuration.html).  
Make sure you have power user/admin access attached to the credentials you are using.

### How do I provision the infrastructure and deploy this app with **Climulon**

To provision the infrastructure and deploy this app with **Climulon**, ou just have to run this command :
```
climulon provision -c ./infrastructure/climulon-nodejs.json
```

Then, check the progress of the docker containers deployment via this command :
```
climulon status -c ./infrastructure/climulon-nodejs.json
```

Once completely deployed, you will be able to access the app via the value of the ```ELBURL``` variable in the output of the provision command you just ran. Paste it in your browser, and voila!

### What did I provision ?
**Climulon** created three Cloudformation stacks:
- ECS stack : This stack contains all what is gonna run the application
  - Instances
  - Instance security groups
  - Instance auto-scaling group
  - Instance launch configuration
  - Centralised log group
  - Instance/container IAM roles
- REDIS stack : This stack contains the redis cluster that is gonna store the app data, using the AWS elasticache service
  - Redis cache cluster
  - Redis cluster sub group
  - Redis cluster security group
- INFRA stack : This stack contains the networking infrastructure that is gonna hold all the other stacks, as well as *glue components* that will allow stacks to communicate, without being tied to each other.
  - VPC
  - Internet gateway
  - Two subnets
  - Routing tables
  - Front-facing ELB 
  - ELB security group
  - A shared security group that will allow the ECS stack and REDIS stack to communicate without being tied to each other.

**Climulon** also created an ECS service cluster, and created Task definitions and services running docker containers inside it.

### Why do I need three templates ?
In order to have maximum flexibility, the infrastructure is divided into small subsets(stacks) that can be duplicated, changed at will, and decommissioned without the need to take down the whole infrastructure.  
By having independent stacks we are reducing coupling and increasing cohesion.

Climulon allow the provision/decommission of a single/multiple stacks inside the whole infrastructure, making operating changes easy.

Example: Decommissioning the ECS cluster, but keeping the networking infrastructure and Redis running :
```
climulon decommission -c ./infrastructure/climulon-nodejs.json --stacks ECS-Climulon-NodeJS
```
Recreating the ECS cluster, and connecting it with the existing networking infrastructure and Redis stacks :
```
climulon provision -c ./infrastructure/climulon-nodejs.json --stacks ECS-Climulon-NodeJS
```

Possible use cases :
- Seamless Redis migration by having two Redis running in parallel.
- Adding new components without impacting the already running infrastructure.
- Feature Flag deployments, by having two ECS stacks running in parallel with different versions.
- Blue/Greem deployments, without operating any change on the current running ECS stack

### Why is the ELB in the INFRA stack and not in the ECS stack ?
The front-facing ELB is in the networking stack in order :
- to have the app always accessible via the same ELB/DNS name combo.
- for the ELB to be used by multiple ECS stacks behind it. The ECS stacks will go up and down, while the ELB is always gonna be same and serving whatever is behind it.

### I want to know moar
Head this way, for a more detailed description of how the infrastructure config/files work, and how to write them ==> [Infrastructure](infrastructure/README.md)

### I want to remove everything Climulon created, what do I do ?
To decommission the whole infrastructure running this app, run the following command :
```
climulon decommission -c ./infrastructure/climulon-nodejs.json
```
