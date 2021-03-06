---
title: Deploying Docker containers on ECS
description: Deploying Docker containers on ECS
keywords: Docker, AWS, ECS, Integration, context, Compose, cli, deploy, containers, cloud
toc_min: 1
toc_max: 2
---

## Overview

The Docker Compose CLI enables developers to use native Docker commands to run applications in Amazon EC2 Container Service (ECS) when building cloud-native applications.

The integration between Docker and Amazon ECS allows developers to use the Docker Compose CLI to:

* Set up an AWS context in one Docker command, allowing you to switch from a local context to a cloud context and run applications quickly and easily
* Simplify multi-container application development on Amazon ECS using Compose files

## Prerequisites

To deploy Docker containers on ECS, you must meet the following requirements:

1. Download and install Docker Desktop Stable version 2.3.0.5 or later, or Edge version 2.3.2.0 or later.

    - [Download for Mac](https://desktop.docker.com/mac/edge/Docker.dmg){: target="_blank" rel="noopener" class="_"}
    - [Download for Windows](https://desktop.docker.com/win/edge/Docker%20Desktop%20Installer.exe){: target="_blank" rel="noopener" class="_"}

    Alternatively, install the [Docker Compose CLI for Linux](#install-the-docker-compose-cli-on-linux).

2. Ensure you have an AWS account.

Docker not only runs multi-container applications locally, but also enables
developers to seamlessly deploy Docker containers on Amazon ECS using a
Compose file with the `docker compose up` command. The following sections
contain instructions on how to deploy your Compose application on Amazon ECS.

## Run an application on ECS

### Create AWS context

Run the `docker context create ecs myecscontext` command to create an Amazon ECS Docker
context named `myecscontext`. If you have already installed and configured the AWS CLI,
the setup command lets you select an existing AWS profile to connect to Amazon.
Otherwise, you can create a new profile by passing an
[AWS access key ID and a secret access key](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys){: target="_blank" rel="noopener" class="_"}.

After you have created an AWS context, you can list your Docker contexts by running the `docker context ls` command:

```console
NAME   DESCRIPTION  DOCKER ENDPOINT  KUBERNETES ENDPOINT ORCHESTRATOR
myecscontext *
default  Current DOCKER_HOST based configuration   unix:///var/run/docker.sock     swarm
```

### Run a Compose application

You can deploy and manage multi-container applications defined in Compose files
to Amazon ECS using the `docker compose` command. To do this:

- Ensure you are using your ECS context. You can do this either by specifying
the `--context myecscontext` flag with your command, or by setting the
current context using the command `docker context use myecscontext`.

- Run `docker compose up` and `docker compose down` to start and then
stop a full Compose application.

  By default, `docker compose up` uses the `docker-compose.yaml` file in
  the current folder. You can specify the Compose file directly using the
  `--file` flag.

  You can also specify a name for the Compose application using the `--project-name` flag during deployment. If no name is specified, a name will be derived from the working directory.

- You can view services created for the Compose application on Amazon ECS and
their state using the `docker compose ps` command.

- You can view logs from containers that are part of the Compose application
using the `docker compose logs` command.

## Rolling update

To update your application without interrupting production flow you can simply
use `docker compose up` on the updated Compose project.
Your ECS services are created with rolling update configuration. As you run
`docker compose up` with a modified Compose file, the stack will be
updated to reflect changes, and if required, some services will be replaced.
This replacement process will follow the rolling-update configuration set by
your services [`deploy.update_config`](https://docs.docker.com/compose/compose-file/#update_config)
configuration.

AWS ECS uses a percent-based model to define the number of containers to be
run or shut down during a rolling update. The Docker Compose CLI computes
rolling update configuration according to the `parallelism` and `replicas`
fields. However, you might prefer to directly configure a rolling update
using the extension fields `x-aws-min_percent` and `x-aws-max_percent`.
The former sets the minimum percent of containers to run for service, and the
latter sets the maximum percent of additional containers to start before
previous versions are removed.

By default, the ECS rolling update is set to run twice the number of
containers for a service (200%), and has the ability to shut down 100%
containers during the update.

## View application logs

The Docker Compose CLI configures AWS CloudWatch Logs service for your
containers.
By default you can see logs of your compose application the same way you check logs of local deployments:

```console
# fetch logs for application in current working directory
docker compose logs

# specify compose project name
docker compose logs --project-name PROJECT

# specify compose file
docker compose logs --file /path/to/docker-compose.yaml
```

A log group is created for the application as `docker-compose/<application_name>`,
and log streams are created for each service and container in your application
as `<application_name>/<service_name>/<container_ID>`.

You can fine tune AWS CloudWatch Logs using extension field `x-aws-logs_retention`
in your Compose file to set the number of retention days for log events. The
default behavior is to keep logs forever.

You can also pass `awslogs` driver parameters to your container as standard
Compose file `logging.driver_opts` elements.

## Private Docker images

The Docker Compose CLI automatically configures authorization so you can pull private images from the Amazon ECR registry on the same AWS account. To pull private images from another registry, including Docker Hub, you’ll have to create a Username + Password (or a Username + Token) secret on the [AWS Secrets Manager service](https://docs.aws.amazon.com/secretsmanager/){: target="_blank" rel="noopener" class="_"}.

For your convenience, the Docker Compose CLI offers the `docker secret` command, so you can manage secrets created on AWS SMS without having to install the AWS CLI.

```console
docker secret create dockerhubAccessToken --username <dockerhubuser>  --password <dockerhubtoken>
arn:aws:secretsmanager:eu-west-3:12345:secret:DockerHubAccessToken
```

Once created, you can use this ARN in you Compose file using using `x-aws-pull_credentials` custom extension with the Docker image URI for your service.

```yaml
services:
  worker:
    image: mycompany/privateimage
    x-aws-pull_credentials: "arn:aws:secretsmanager:eu-west-3:12345:secret:DockerHubAccessToken"
```

> **Note**
>
> If you set the Compose file version to 3.8 or later, you can use the same Compose file for local deployment using `docker-compose`. Custom extensions will be ignored in this case.

## Service discovery

Service-to-service communication is implemented by the [Security Groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html){: target="_blank" rel="noopener" class="_"} rules, allowing services sharing a common Compose file “network” to communicate together. This allows individual services to run with distinct constraints (memory, cpu) and replication rules. However, it comes with a constraint that Docker images have to handle service discovery and wait for dependent services to be available.

### Service names

Services are registered by the Docker Compose CLI on [AWS Cloud Map](https://docs.aws.amazon.com/cloud-map/latest/dg/what-is-cloud-map.html){: target="_blank" rel="noopener" class="_"} during application deployment. They are declared as fully qualified domain names of the form: `<service>.<compose_project_name>.local`. Services can retrieve their dependencies using this fully qualified name, or can just use a short service name (as they do with docker-compose).

### Dependent service startup time and DNS resolution

Services get concurrently scheduled on ECS when a Compose file is deployed. AWS Cloud Map introduces an initial delay for DNS service to be able to resolve your services domain names. As a result, your code needs to be adjusted to support this delay by waiting for dependent services to be ready, or by adding a wait-script as the entrypoint to your Docker image, as documented in [Control startup order](https://docs.docker.com/compose/startup-order/).

Alternatively, you can use the [depends_on](https://github.com/compose-spec/compose-spec/blob/master/spec.md#depends_on){: target="_blank" rel="noopener" class="_"} feature of the Compose file format. By doing this, dependent service will be created first, and application deployment will wait for it to be up and running before starting the creation of the dependent services.

## Volumes

ECS integration supports volume management based on Amazon Elastic File System (Amazon EFS).
For a Compose file to declare a `volume`, ECS integration will define creation of an EFS
file system within the CloudFormation template, with `Retain` policy so data won't
be deleted on application shut-down. If the same application (same project name) is
deployed again, the file system will be re-attached to offer the same user experience
developers are used to with docker-compose.

A basic compose service using a volume can be declared like this:

```yaml
services:
  nginx:
    image: nginx
    volumes:
      - mydata:/some/container/path
volumes:
  mydata:
```

With no specific volume options, the volume still must be declared in the `volumes`section for
the compose file to be valid (in the above example the empty `mydata:` entry)
If required, the initial file system can be customized using `driver-opts`:

```yaml
volumes:
  my-data:
    driver_opts:
      # Filesystem configuration
      backup_policy: ENABLED
      lifecycle_policy: AFTER_14_DAYS
      performance_mode: maxIO
      throughput_mode: provisioned
      provisioned_throughput: 1024
```

File systems created by executing `docker compose up` on AWS can be listed using
`docker volume ls` and removed with `docker volume rm <filesystemID>`.

An existing file system can also be used for users who already have data stored on EFS
or want to use a file system created by another Compose stack.

```yaml
volumes:
  my-data:
    external: true
    name: fs-123abcd
```

Accessing a volume from a container can introduce POSIX user ID
permission issues, as Docker images can define arbitrary user ID / group ID for the
process to run inside a container. However, the same `uid:gid` will have to match
POSIX permissions on the file system. To work around the possible conflict, you can set the volume
`uid` and `gid` to be used when accessing a volume:

```yaml
volumes:
  my-data:
    driver_opts:
      # Access point configuration
      uid: 0
      gid: 0
```

## Secrets

You can pass secrets to your ECS services using Docker model to bind sensitive
data as files under `/run/secrets`. If your Compose file declares a secret as
file, such a secret will be created as part of your application deployment on
ECS. If you use an existing secret as `external: true` reference in your
Compose file, use the ECS Secrets Manager full ARN as the secret name:

```yaml
services:
  webapp:
    image: ...
    secrets:
      - foo

secrets:
  foo:
    name: "arn:aws:secretsmanager:eu-west-3:1234:secret:foo-ABC123"
    external: true
```

Secrets will be available at runtime for your service as a plain text file `/run/secrets/foo`.

The AWS Secrets Manager allows you to store sensitive data either as a plain
text (like Docker secret does), or as a hierarchical JSON document. You can
use the latter with Docker Compose CLI by using custom field `x-aws-keys` to
define which entries in the JSON document to bind as a secret in your service
container.

```yaml
services:
  webapp:
    image: ...
    secrets:
      - foo

secrets:
  foo:
    name: "arn:aws:secretsmanager:eu-west-3:1234:secret:foo-ABC123"
    keys:
      - "bar"
```

By doing this, the secret for `bar` key will be available at runtime for your
service as a plain text file `/run/secrets/foo/bar`. You can use the special
value `*` to get all keys bound in your container.

## Auto scaling

The Compose file model does not define any attributes to declare auto-scaling conditions.
Therefore, we rely on `x-aws-autoscaling` custom extension to define the auto-scaling range, as
well as cpu _or_ memory to define target metric, expressed as resource usage percent.

```yaml
services:
  foo:
    deploy:
      x-aws-autoscaling:
        min: 1
        max: 10 #required
        cpu: 75
        # mem: - mutualy exlusive with cpu
```

## IAM roles

Your ECS Tasks are executed with a dedicated IAM role, granting access
to AWS Managed policies[`AmazonECSTaskExecutionRolePolicy`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)
and [`AmazonEC2ContainerRegistryReadOnly`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecr_managed_policies.html).
In addition, if your service uses secrets, IAM Role gets additional
permissions to read and decrypt secrets from the AWS Secret Manager.

You can grant additional managed policies to your service execution
by using `x-aws-policies` inside a service definition:

```yaml
services:
  foo:
    x-aws-policies:
      - "arn:aws:iam::aws:policy/AmazonS3FullAccess"
```

You can also write your own [IAM Policy Document](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html)
to fine tune the IAM role to be applied to your ECS service, and use
`x-aws-role` inside a service definition to pass the
yaml-formatted policy document.

```yaml
services:
  foo:
    x-aws-role:
      Version: "2012-10-17"
      Statement:
        - Effect: "Allow"
          Action:
            - "some_aws_service"
          Resource":
            - "*"
```

## Tuning the CloudFormation template

The Docker Compose CLI relies on [Amazon CloudFormation](https://docs.aws.amazon.com/cloudformation/){: target="_blank" rel="noopener" class="_"} to manage the application deployment. To get more control on the created resources, you can use `docker compose convert` to generate a CloudFormation stack file from your Compose file. This allows you to inspect resources it defines, or customize the template for your needs, and then apply the template to AWS using the AWS CLI, or the AWS web console.

## Using existing AWS network resources

By default, the Docker Compose CLI creates an ECS cluster for your Compose application, a Security Group per network in your Compose file on your AWS account’s default VPC, and a LoadBalancer to route traffic to your services. If your AWS account does not have [permissions](https://github.com/docker/ecs-plugin/blob/master/docs/requirements.md#permissions){: target="_blank" rel="noopener" class="_"} to create such resources, or if you want to manage these yourself, you can use the following custom Compose extensions:

- Use `x-aws-cluster` as a top-level element in your Compose file to set the ARN
of an ECS cluster when deploying a Compose application. Otherwise, a
cluster will be created for the Compose project.

- Use `x-aws-vpc` as a top-level element in your Compose file to set the ARN
of a VPC when deploying a Compose application.

- Use `x-aws-loadbalancer` as a top-level element in your Compose file to set
the ARN of an existing LoadBalancer.

- Use `external: true` inside a network definition in your Compose file for
Docker Compose CLI to _not_ create a Security Group, and set `name` with the
ID of an existing SecurityGroup you want to use for network connectivity between
services:

```yaml
networks:
  back_tier:
    external: true
    name: "sg-1234acbd"
```

## Local simulation

When you deploy your application on ECS, you may also rely on the additional AWS services.
In such cases, your code must embed the AWS SDK and retrieve API credentials at runtime.
AWS offers a credentials discovery mechanism which is fully implemented by the SDK, and relies
on accessing a metadata service on a fixed IP address.

Once you adopt this approach, running your application locally for testing or debug purposes
can be difficult. Therefore, we have introduced an option on context creation to set the
`ecs-local` context to maintain application portability between local workstation and the
AWS cloud provider.

```console
$ docker context create ecs --local-simulation ecsLocal
Successfully created ecs-local context "ecsLocal"
```

When you select a local simulation context, running the `docker compose up` command doesn't
deploy your application on ECS. Therefore, you must run it locally, automatically adjusting your Compose
application so it includes the [ECS local endpoints](https://github.com/awslabs/amazon-ecs-local-container-endpoints/).
This allows the AWS SDK used by application code to
access a local mock container as "AWS metadata API" and retrieve credentials from your own
local `.aws/credentials` config file.

## Install the Docker Compose CLI on Linux

The Docker Compose CLI adds support for running and managing containers on ECS.

### Install Prerequisites

[Docker 19.03 or later](https://docs.docker.com/get-docker/)

### Install script

You can install the new CLI using the install script:

```console
curl -L https://raw.githubusercontent.com/docker/compose-cli/main/scripts/install/install_linux.sh | sh
```

## FAQ

**What does the error `this tool requires the "new ARN resource ID format"` mean?**

This error message means that your account requires the new ARN resource ID format for ECS. To learn more, see [Migrating your Amazon ECS deployment to the new ARN and resource ID format](https://aws.amazon.com/blogs/compute/migrating-your-amazon-ecs-deployment-to-the-new-arn-and-resource-id-format-2/){: target="_blank" rel="noopener" class="_"}.

## Feedback

Thank you for trying out the Docker Compose CLI. Your feedback is very important to us. Let us know your feedback by creating an issue in the [Compose CLI](https://github.com/docker/compose-cli){: target="_blank" rel="noopener" class="_"} GitHub repository.
