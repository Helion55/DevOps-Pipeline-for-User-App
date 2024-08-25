# NextJS-User-Management-App-DevOps

![Diagram](https://github.com/Helion55/NextJS-User-Management-App-Deployment/blob/main/NextJS.jpg?raw=true)

## Project Overview
User Management Application doing CRUD opertaions in Database. NextJS is used to build the frontend, ExpressJS useed to build the backend, Prisma is used for ORM and Postgres Is used as Database.
Dockerizing the application on GitLabCI pipeline, Testing the application on docker-compose stack and deploying the application on Kubernetes using ArgoCD.

## Tech Stack
- NextJS
- ExpressJS
- Postgres
- Docker
- ArgoCD
- GItLabCI
- Kubernetes

## Steps
1. Dockerizing the application
2. Local Testing with Docker Compose
3. Building the CI Pipeline using GitLabCI
4. Building the CD Pipeline using ArgoCD
5. Creating Kubernetes Manifest files

## 1. Dockerizing the application
Frontend and Backend directory containing Dockerfiles for each application. Each Dockerfiles following these steps same...
- Pulling the base image
- Installing the dependencies
- Running the application
- Exposing the Port
But backend application conatains a ```startscript.sh``` file to run the application after doing database migrations. Frontend is using the default NextJS Dockerfile from NextJS itself.
### - Frontend Dockerfile
```
FROM node:18-alpine AS base

# Install dependencies only when needed
FROM base AS deps
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Install dependencies based on the preferred package manager
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm i --frozen-lockfile; \
  else echo "Lockfile not found." && exit 1; \
  fi


# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Next.js collects completely anonymous telemetry data about general usage.
# Learn more here: https://nextjs.org/telemetry
# Uncomment the following line in case you want to disable telemetry during the build.
# ENV NEXT_TELEMETRY_DISABLED 1

RUN yarn build && ls -l /app/.next


# If using npm comment out above and use below instead
# RUN npm run build

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
# Uncomment the following line in case you want to disable telemetry during runtime.
# ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public

# Set the correct permission for prerender cache
RUN mkdir .next
RUN chown nextjs:nodejs .next

# Automatically leverage output traces to reduce image size
# https://nextjs.org/docs/advanced-features/output-file-tracing
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT 3000
# set hostname to localhost
ENV HOSTNAME "0.0.0.0"

# server.js is created by next build from the standalone output
# https://nextjs.org/docs/pages/api-reference/next-config-js/output
CMD ["node", "server.js"]
```
### - Backend Dockerfile
```Dockerfile
FROM node:20

WORKDIR /app

ADD package*.json ./

RUN npm install

ADD prisma ./prisma

RUN npx prisma generate

ADD . .

EXPOSE 4000

RUN chmod +x startscript.sh
```
### - Startscript for backend
```bash
#!/bin/sh
npx prisma migrate dev --name init
npm run start
```
Using the command...
```
docker build -t IMAGE-NAME<FRONTEND/BACKEND> .
```
to build the images.

## 2. Local Testing with Docker Compose
The Docker Compose file will pull the Postgres Image, build and run the local application image and connect them using a Docker network.
This is the compose file...
```compose
version: '3.9'
services:
  database:
    container_name: database
    image: postgres:12
    restart: always
    environment:
      POSTGRES_USER: USER
      POSTGRES_PASSWORD: PASSWORD
      POSTGRES_DB: MY-DATABASE
    ports:
      - 5432:5432
    volumes:
      - postgres-data:/var/lib/postgresql/data

  backend:
    container_name: backend
    image: backend
    build:
      context: ./backend
    ports:
      - 4000:4000
    environment:
      - DATABASE_URL=postgresql://USER:PASSWORD@database:5432/MY-DATABASE?schema=public
    command: ./startscript.sh
    depends_on:
      - database

  frontend:
    container_name: frontend
    image: frontend
    build:
      context: ./frontend
    ports:
      - 3000:3000
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:4000
    restart: always
    depends_on:
      - backend
  
volumes:
  postgres-data:
```
and using the command...
```bash
docker compose up
```
to start the application stack.


## 3. Building the CI Pipeline using GitLabCI
In this project GitLab default Runners are Used. Pipeline is having two stages named backend and frontend and it is configured to run automatically when commit is triggred on the main branch.
Using the Docker Image to build the Dockerfiles which is known as docker-in-docker concept. Now accessing the GitLab container registry with the help of environment variables to store the images.
Here is the pipeline file...
```yaml
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH != "main" && $CI_PIPELINE_SOURCE != "merge_request_event"      
      when: never
    - when: always

stages:
  - frontend
  - backend

frontend building and pushing:
  stage: frontend
  
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  
  script:
    - cd ./frontend
    - ls
    - docker build -t $CI_REGISTRY_IMAGE/app/frontend:1.0 .
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE/app/frontend:1.0

backend building and pushing:
  stage: backend
  
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  
  script:
    - cd ./backend
    - docker build -t $CI_REGISTRY_IMAGE/app/backend:1.0 .
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE/app/backend:1.0
```
This completes the CI part of the project.

## 4. Building the CD Pipeline using ArgoCD
Now ArgoCD will pull the manifest files from another code repository and apply them on the cluster. 3 steps is followed for ArgoCD setup
- Creating a ArgoCD Application
This ArgoCD Application will manage our main application. It is creating using this file...
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: MY-APPLICATION-NAME
  namespace: argocd
spec:
  project: default
  source:
    repoURL: KUBERNETES-MANIFESTS-REPOSITORY-URL
    targetRevision: HEAD
    path: ./MY-FOLDER
  destination: 
    server: https://kubernetes.default.svc
    namespace: MY-NAMESPACE
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    automated:
      selfHeal: true
      prune: true
```
and applied using ```kubectl apply``` command.

- Creating a Secret to access the Kubernetes manifest file Repositry
To pull the main application's manifest files ArgoCD need to Access the manifest repository. This Secret is storing credentials of that Repositry and it is applied on argocd namepace.
```yaml
apiVersion: v1
kind: Secret
metadata:
 name: manifest-repository-access-secret
 namespace: argocd
 labels:
  argocd.argoproj.io/secret-type: repository
stringData:
 type: git
 url: KUBERNETES-MANIFESTS-REPOSITORY-URL
 username: REPOSITORY-USERNAME
 password: REPOSITORY-ACCESS-TOKEN
```

- Applying a Secret on the main application's namespace to pull the application image
ArgoCD created a namespace for the application and on that namespace this secret is applied using the command
```bash
kubectl create secret docker-registry SECRET-NAME   --docker-server=registry.gitlab.com  --docker-username=GITLAB-USERNAME   --docker-password="ACCESS-TOKEN" -n APPLICATION-NAMESPACE
```
These steps completes the CD part of the application.
## 5. Creating Kubernetes Manifest files
The Main application have a backend, forntend and a database. These Kubernetes Objects are created to run the application. All the files are in Kubernetes Folder.
- Deployment
Frontend, Backend and Database are deployed as Deployment with 1 Replicaset. 
- Service
This 3 Deployments have Services to connect the pods and connecting with each other. The Forntend is using NodePort service and other 2 are ClusterIp service.
- Persistent Volume
A Persistent Volume of 500Mi is created for the database. Other 2 does not need it.
- Persistent Volume Claim
A Persistent Volume Claim is bound with the PV which will connect with database pod.
- Ingress
An Ingress is created for future upgrade of the application in cloud. This have the route setup for frontend.
- Configmap
A Configmap is used to connect the backend with the frontend. It contains the service address of the backend which is used in frontend deployment file.

After Completing these Steps and Configuration this application is Deployed...🚀


## Contributing
I will be very glad to accept any contribution which will take this whole project one step further.
