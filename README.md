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
     
## Project Steps

Completing the project involves several steps:

1. Write a Dockerfile for a simple Flask API
2. Build and test the container locally
3. Create an EKS cluster
4. Store a secret using AWS Parameter Store
5. Create a CodePipeline pipeline triggered by GitHub checkins
6. Create a CodeBuild stage which will build, test, and deploy your code

For more detail about each of these steps, see the project lesson [here](https://classroom.udacity.com/nanodegrees/nd004/parts/1d842ebf-5b10-4749-9e5e-ef28fe98f173/modules/ac13842f-c841-4c1a-b284-b47899f4613d/lessons/becb2dac-c108-4143-8f6c-11b30413e28d/concepts/092cdb35-28f7-4145-b6e6-6278b8dd7527).

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
arn:aws:iam::218700020755:role/UdacityFlaskDeployCBKubectlRole

## Grant the Role Access to the Cluster
1. The 'aws-auth ConfigMap' is used to grant role-based access control to your cluster. When your cluster is first created, the user who created it is given sole permission to administer it. You need to add the role you just created so that CodeBuild can administer it as well. Get the current configmap and save it to a file:
```
kubectl get -n kube-system configmap/aws-auth -o yaml > aws-auth-patch.yml
```

1. Do not follow udacity. Following this [link](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html). Just execute
```
kubectl edit -n kube-system configmap/aws-auth
```
a txt editor will appear and you can edit the file there.
