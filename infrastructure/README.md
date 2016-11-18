# Infrastructure

This readme will describe in details how to map the infrastructure in code and how **Climulon** uses all of files to provision the infrastructure.

## File hierarchy
This is the ideal file hierarchy under which all the infrastructure files are stored :
- **templates** : This folder contains the cloudformation templates that describes each stack (networking, ecs, redis...)
- **taskDefs** : This folder contains the description of the [task definitions](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_defintions.html) that describes the container groups
- **services** : This folder contains the description of the [services](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html) that will start the [task definitions](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_defintions.html) in ECS
- **root** : The root folder contains the **master config files**, that **Climulon** uses to know which set of files to use, and contains the variables that will override those that are in all of the other files (templates, taskDefs and services).

_Note : This file hierarchy is totally optional. You can put any file anywhere you like. The **master config files** contain the path to all of the files that **Climulon** will need written in them, so their location is not important, as long as they are in the repo._

## Json, everywhere
Every file that **Climulon** uses should be written in Json, including :
- The Cloudformation templates
- The task definitions
- The service descriptions
- The master config files

## Templating language
**Climulon** is using the same templating language that Cloudformation uses for variable substitution.

Example :
```json
{
    "Environment": {
        "Ref": "foobar"
    }
}
```
The `Environment` value is gonna be replaced by the value of the `foobar` parameter.

## Files, in detail :

### Templates

**Climulon** uses Cloudformation templates to provision ressources in AWS.  

Here are some guidelines on how to write your Cloudformation templates :  

1. In order to harness the full potential of **Climulon**, each Cloudformation template should be as decoupled as possible.
Compute ressources should be kept in their own template, each persistency layer on its own template (on-disk/in-memory), and shared networking template should contain the ressources that do net get frequently updated and are required by multiple templates.  
The shared template should also include glue ressources that allow stacks to communicate with each other.
  - Example : The ECS stack need to be able to reach the REDIS stack. In order to allow communication without coupling, a security group is to be created in the shared to template, and both the ECS and REDIS template should use it for communication. If the security group is added in any of the ECS or REDIS stack, the deletion of the stack containing it will be blocked, since the other stack will be dependent on the said security group. Adding the security group to the shared template lowers the coupling, and allows simplifies stack provision/decommission.
2. Templates should be as alterable as possible via their parameters. That will allow using the same templates for multiple environments (prod, stage, test...). 

### Task definitions
Task definitions are described in json.
Every task definition requires four keys to be present in their description :
- family
- taskRoleArn
- containerDefinitions
- volumes

More details on how to write task definitions can be found [here](http://boto3.readthedocs.io/en/latest/reference/services/ecs.html#ECS.Client.register_task_definition).

The task definition used for the app in this repo can be found [here](taskDefs/taskDef-climulon-nodejs.json)

Example :
```json
{
    "family": "string",
    "taskRoleArn": "string",
    "networkMode": "bridge/host/none",
    "containerDefinitions": [
        {
            "name": "string",
            "image": "string",
            "cpu": 123,
            "memory": 123,
            "memoryReservation": 123,
            "links": [
                "string",
            ],
            "portMappings": [
                {
                    "containerPort": 123,
                    "hostPort": 123,
                    "protocol": "tcp/udp"
                },
            ],
            "essential": "True/False",
            "entryPoint": [
                "string",
            ],
            "command": [
                "string",
            ],
            "environment": [
                {
                    "name": "string",
                    "value": "string"
                },
            ],
            "mountPoints": [
                {
                    "sourceVolume": "string",
                    "containerPath": "string",
                    "readOnly": "True/False"
                },
            ],
            "volumesFrom": [
                {
                    "sourceContainer": "string",
                    "readOnly": "True/False"
                },
            ],
            "hostname": "string",
            "user": "string",
            "workingDirectory": "string",
            "disableNetworking": "True/False",
            "privileged": "True/False",
            "readonlyRootFilesystem": "True/False",
            "dnsServers": [
                "string",
            ],
            "dnsSearchDomains": [
                "string",
            ],
            "extraHosts": [
                {
                    "hostname": "string",
                    "ipAddress": "string"
                },
            ],
            "dockerSecurityOptions": [
                "string",
            ],
            "dockerLabels": {
                "string": "string"
            },
            "ulimits": [
                {
                    "name": "core/cpu/data/fsize/locks/memlock/msgqueue/nice/nofile/nproc/rss/rtprio/rttime/sigpending/stack",
                    "softLimit": 123,
                    "hardLimit": 123
                },
            ],
            "logConfiguration": {
                "logDriver": "json-file/syslog/journald/gelf/fluentd/awslogs/splunk",
                "options": {
                    "string": "string"
                }
            }
        },
    ],
    "volumes": [
        {
            "name": "string",
            "host": {
                "sourcePath": "string"
            }
        },
    ]
}
```

### Services
Services are described in json.
More details on how to write services can be found [here](http://boto3.readthedocs.io/en/latest/reference/services/ecs.html#ECS.Client.create_service).

The service used for the app in this repo can be found [here](services/service-climulon-nodejs.json)

Example :
```json
{
    "cluster": "string",
    "serviceName": "string",
    "taskDefinition": "string",
    "loadBalancers": [
        {
            "targetGroupArn": "string",
            "loadBalancerName": "string",
            "containerName": "string",
            "containerPort": 123
        },
    ],
    "desiredCount": 123,
    "clientToken": "string",
    "role": "string",
    "deploymentConfiguration": {
        "maximumPercent": 123,
        "minimumHealthyPercent": 123
}
```

### Master config files (MCFs)
Master config files (MCFs) are used to define environments. Each single MCF defines a whole environment (prod, stage, test ...).
Each MCF :
- under the key `infrastructureTemplates` : contains the paths of the Cloudformation templates it needs, the variables that will define in which region those templates are gonna create stacks, their priority when created and the parameter override that are gonna be applied on the stack during creation. 
- under the key `taskDefsTemplates` : contains the paths of the task definition files
- under the key `servicesTemplates` : contains the paths of the service files
- under the key `generalParameters` : contains a list of key/values that are gonna be used to override any variable that can be found in the templates, task definitions and services.

Each MCF requires these 4 keys :
- infrastructureTemplates (set)
- taskDefsTemplates (set)
- servicesTemplates (set)
- generalParameters (dict/key-value)

The MCF used for the app in this repo can be found [here](climulon-nodejs.json)

MCF skeleton :
```json
{
    "infrastructureTemplates": [
       {
            "StackTemplate": "file path",
            "StackName": "namey-mcnameface",
            "StackOrder": "1",
            "StackRegion": "eu-central-1",
            "ComputeStack": "False",
            "StackParameters": {
                "param1": "value1",
                "param2": "value2"
            }
      }
    ],
    "taskDefsTemplates": [
        "file path"
    ],
    "servicesTemplates": [
        "file path"
    ],
    "globalParameters": {
        "param3": "value3",
        "param4": "value4"
    }
}
```

#### infrastructureTemplates
This key describes what templates to use and how to modify them to suit our needs.  
The `infrastructureTemplates` is a set of dicts. Each dict describes a single stack.  

Each stack can be described this way :
```json
{
    "StackTemplate": "file path",
    "StackName": "namey-mcnameface", 
    "StackOrder": "1",
    "StackRegion": "eu-central-1",
    "ComputeStack": "False",
    "StackParameters": {
        "param1": "value1",
        "param2": "value2"
    }
}
```

- StackTemplate : (string) where the template of the stack is, relatively to the MCF's location
- StackName : (string) Name of the stack
- StackOrder : (number) The order of creation of the stack. The stack with order 1 will be created first and so on
- StackRegion : (string) AWS region where the stack is gonna be created
- ComputeStack : (boolean) If this key is set to true, creating the stack will trigger the creation of an ECS cluster, services and task definitions
- StackParameters : (dict) Parameters used to override the default parameters of the Cloudformation template
Each of the keys above are mandatory.

#### taskDefsTemplates
This key is a set of paths to the task definitions files used when provisionning.

#### servicesTemplates
This key is a set of paths to the services files used when provisionning.

#### globalParameters
This key is a dict of keys/values. It contains variables that will be used by **Climulon** to substitute any references that it finds in the MCF, task definition files and service files.

Example
```json
{
    "infrastructureTemplates": [
       {
            "StackTemplate": "templates/infrastructureECS.json",
            "StackName": "cronenberged",
            "StackOrder": "1",
            "StackRegion": "eu-central-1",
            "ComputeStack": "true",
            "StackParameters": {
                "instanceType": {
                    "Ref": "INST_TYPE"
                }
            }
      }
    ],
    "taskDefsTemplates": [
        "taskDefs/taskDef.json"
    ],
    "servicesTemplates": [
        "services/service.json"
    ],
    "globalParameters": {
        "INST_TYPE": "t2.micro"
    }
}
```

**Climulon** is gonna substitute the reference and replace it by the value of the `INST_TYPE` parameter.
Replacing `"instanceType": { "Ref": "INST_TYPE" }` by `"instanceType": "t2.micro"`

**Climulon** is also gonna look for any references in the MCFs, the task definition files and service files, and substitute them with their values.

## Reference substitution

As stated above, **Climulon** substitutes references using the same pattern that Cloudformation uses.
I.e
```json
{
    "Environment": {
        "Ref": "foobar"
    }
}
```

**Climulon** does not resolve all the references in one go, it only resolves them when it needs to. If **Climulon** is to start provisionning a stack, it will try to resolve all the variables that are related to that stack, then proceed with creating it.  
**Climulon** gets the subtitution values from three sources :
- The output keys/values of the previously created/existing stacks
- The StackParameters keys/values of the stacks in the MCF
- The globalParameters keys/values in the MCF
