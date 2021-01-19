# Deploying a Flask API

This is the project starter repo for the fourth course in the [Udacity Full Stack Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004): Server Deployment, Containerization, and Testing.

In this project you will containerize and deploy a Flask API to a Kubernetes cluster using Docker, AWS EKS, CodePipeline, and CodeBuild.

The Flask app that will be used for this project consists of a simple API with three endpoints:

- `GET '/'`: This is a simple health check, which returns the response 'Healthy'. 
- `POST '/auth'`: This takes a email and password as json arguments and returns a JWT based on a custom secret.
- `GET '/contents'`: This requires a valid JWT, and returns the un-encrpyted contents of that token. 

The app relies on a secret set as the environment variable `JWT_SECRET` to produce a JWT. The built-in Flask server is adequate for local development, but not production, so you will be using the production-ready [Gunicorn](https://gunicorn.org/) server when deploying the app.

## Initial setup
1. Fork this project to your Github account.
2. Locally clone your forked version to begin working on the project.

## Dependencies

- Docker Engine
    - Installation instructions for all OSes can be found [here](https://docs.docker.com/install/).
    - For Mac users, if you have no previous Docker Toolbox installation, you can install Docker Desktop for Mac. If you already have a Docker Toolbox installation, please read [this](https://docs.docker.com/docker-for-mac/docker-toolbox/) before installing.
 - AWS Account
     - You can create an AWS account by signing up [here](https://aws.amazon.com/#).
     
## Run locally.
```cmd
SET JWT_SECRET='random_secret_1'
SET LOG_LEVEL=DEBUG
python main.py
```

```
curl -H "Content-Type:application/json" -d "{\"email\":\"hello@hello.com\",\"password\":\"123456\"}" -X POST http://localhost:8080/auth

curl --request GET http://127.0.0.1:8080/contents -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2MTIxMDYxMzMsIm5iZiI6MTYxMDg5NjUzMywiZW1haWwiOiJoZWxsb0BoZWxsby5jb20ifQ.sK9mIEYhxg3Odko_77p5Sw1zrYn3iYGJKv7aro2327I"

```

## Build the docker
```
docker build -t "jwt-api-test" .
docker image ls
docker run --env-file=.env_file -p 80:8080 jwt-api-test
docker container ls
docker stop [OPTIONS] CONTAINER [CONTAINER...]
```

## Test running on docker
```
curl -H "Content-Type:application/json" -d "{\"email\":\"hello@hello.com\",\"password\":\"123456\"}" -X POST http://localhost:80/auth

curl --request GET http://localhost:80/contents -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2MTIxMDY5OTYsIm5iZiI6MTYxMDg5NzM5NiwiZW1haWwiOiJoZWxsb0BoZWxsby5jb20ifQ.oBIPgNJt6HOV36-ycCoU53w8E9LSD3JfURpWBG70u40"
```

## create a EKS cluster
```
eksctl create cluster --name simple-jwt-api
```

## Create a IAM Role
`arn:aws:iam::111111111111:role/UdacityFlaskDeployCBKubectlRole`

## Grant the Role Access to the Cluster
The 'aws-auth ConfigMap' is used to grant role-based access control to your cluster. When your cluster is first created, the user who created it is given sole permission to administer it. You need to add the role you just created so that CodeBuild can administer it as well. Get the current configmap and save it to a file:
```
kubectl get -n kube-system configmap/aws-auth -o yaml > aws-auth-patch.yml
```

Do not follow udacity. Following this [link](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html). Just execute
```
kubectl describe configmap -n kube-system aws-auth
kubectl edit -n kube-system configmap/aws-auth
```
a txt editor will appear and you can edit the file there.<br>
The given tutorial might have problems. Use the following one to revise. 
check [this].(https://github.com/jungleBadger/FSND-Deploy-Flask-App-to-Kubernetes-Using-EKS/blob/master/troubleshooting/deploy.md#step-4---create-an-additional-role-and-fetch-aws-auth-file)

## Set a Secret using AWS Parameter Store
1. add the following to your `buildspec.yml` file:
```
env:
     parameter-store:         
       JWT_SECRET: JWT_SECRET
```
2.  Put secret into AWS Parameter Store
```
aws ssm put-parameter --name JWT_SECRET --value "randomsecret" --type SecureString
```
3. when needed, can use the following to delete from parameter store
```
aws ssm delete-parameter --name JWT_SECRET
```
## Create a Cloudformation stack using template. 
**Important**: The given template has some problems. The solution is given [here](https://github.com/jungleBadger/FSND-Deploy-Flask-App-to-Kubernetes-Using-EKS/blob/master/troubleshooting/codepipeline_creation_issue.md). To be more specific. Line 158 should be changed from:
```
headers = {'content-type': '', "content-length": len(response_body) }
```
 to: 
 ```
 headers = {'content-type': '', "content-length": str(len(response_body))}
 ```


## Test the built pipeline
Get the api endpoints. 
```
kubectl get services simple-jwt-api -o wide
```
```
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP                                                                PORT(S)        AGE     SELECTOR
simple-jwt-api   LoadBalancer   10.100.16.200   af4d4703de6bc46e49f9ac1ad5b7ebec-1299799129.eu-north-1.elb.amazonaws.com   80:31646/TCP   4h57m   app=simple-jwt-api
```

Post a email and password to api
```
curl -H "Content-Type:application/json" -d "{\"email\":\"hello@hello.com\",\"password\":\"123456\"}" -X POST af4d4703de6bc46e49f9ac1ad5b7ebec-1299799129.eu-north-1.elb.amazonaws.com/auth

curl --request GET af4d4703de6bc46e49f9ac1ad5b7ebec-1299799129.eu-north-1.elb.amazonaws.com/contents -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2MTIyOTMzNzMsIm5iZiI6MTYxMTA4Mzc3MywiZW1haWwiOiJoZWxsb0BoZWxsby5jb20ifQ.T-q8t50iDCpX52MKTHwzfzxhF0c1M49gluDXtdem8PY"
```

