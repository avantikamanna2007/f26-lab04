# Deployment Evidence

Fill this in as you go. Paste real output, not descriptions of output. A TA reads this
file with you at recitation.

## 1. Deployed URL and instance id

<!-- The ServiceUrl and InstanceId outputs. Paste both here every time
describe-stacks prints them, for the healthy deploy and for scenario 2. Both
change on every recreate, and you will need them for curls and sessions. -->

**Milestone 1: healthy deployment**

http://ec2-54-145-228-68.compute-1.amazonaws.com:8080

InstanceId: i-0c164233c5c1cc002

**Milestone 2: scenario-2 deployment**

| OutputKey | OutputValue |
| --- | --- |
| InstanceId | i-08537eb2226e3e8d7 |
| ServiceUrl | http://ec2-13-221-253-37.compute-1.amazonaws.com:8080 |

**Milestone 2: redeployment with healthy parameters**

| OutputKey | OutputValue |
| --- | --- |
| InstanceId | i-0ce4b799ab64fc4e4 |
| ServiceUrl | http://ec2-52-72-39-109.compute-1.amazonaws.com:8080 |

## 2. External health check

Run the check from your own machine, not from the instance. Paste the command and the
response.

```
$ curl http://ec2-54-145-228-68.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 3. What the template created

The template created a `t3.micro` EC2 instance running Amazon Linux 2023 to host the service.
It also created a security group that allows incoming internet traffic on port 8080 for the service and port 22 for SSH, with outgoing traffic allowed by default.
When the instance starts, a user-data script installs and starts Docker, then runs the course service container with `PORT=8080` and forwards the instance's port 8080 to the container's port 8080.

## 4. Scenario 2 diagnosis

**The failing curl** (command and output):

```
$ curl http://ec2-13-221-253-37.compute-1.amazonaws.com:8080/api/health
curl: (7) Failed to connect to ec2-13-221-253-37.compute-1.amazonaws.com port 8080 after 47 ms: Couldn't connect to server
```

**The log line that told you what was wrong:**

```
$ sudo docker logs lab04-service
lab04-service listening on 9090
```

The running container's `sudo docker ps` output showed this port mapping:

```text
0.0.0.0:8080->8080/tcp, :::8080->8080/tcp
```

**What was wrong, and the fix you applied:**

The scenario-2 parameters set `PortOverride=9090`, so the application listens on
container port 9090 (confirmed by the log), while Docker forwards host port 8080
to container port 8080, where the application is not listening.
I deleted the broken stack and recreated it using
`infra/params-healthy.json`, whose empty `PortOverride` makes the application
listen on 8080 to match the port mapping and security group.

**The healthy curl after the fix:**

```
$ curl http://ec2-52-72-39-109.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 5. Teardown proof

The delete and wait commands completed without errors. The subsequent describe
command confirmed that the stack no longer exists.

```
$ aws cloudformation delete-stack --stack-name lab04-service
$ aws cloudformation wait stack-delete-complete --stack-name lab04-service
$ aws cloudformation describe-stacks --stack-name lab04-service
aws: [ERROR]: An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id lab04-service does not exist
```
