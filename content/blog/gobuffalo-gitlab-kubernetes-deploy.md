---
title: "Gobuffalo Kubernetes Deploy"
date: 2019-03-26T08:36:27-04:00
tags: ["devops", "gobuffalo", "kubernetes"]
draft: false
---

I wanted to deploy gobuffalo on kubernetes using gitlab-ci and found several guides but still had to peice together some info on my own and wanted to share that in case anyone else is looking to do the same thing.   I initially started with [Gitlab Auto DevOps](https://docs.gitlab.com/ee/topics/autodevops/) but it came up a bit inflexible and tricky to use for some of my use cases.   The biggest one being that I wanted to pass in build-args into my docker build so that I could pass in API key secrets.  AutoDevOps didn't seem to support this.

If you need to setup a kubernetes cluster I recommend checking out my last blog post on [setting up kubernetes on digitalocean using terraform](/terraform-digitalocean-kubernetes).  It'll get you setup with a kubernetes cluster running on digitalocean that supports gitlab, has [Traefik](https://traefik.io) setup as an ingress controller and a digitalocean load balancer.

Following this guide you'll setup:

* a brand new gobuffalo app by running
* a local postgres server running in docker-compose
* Docker build runs tests too so you can replicate CI at your desk
* a gitlab-ci.yml that will build and test your app on a pull request
* once you merge to master it will build and test your app and then deploy it to kubernetes
* The deployment will include adding the ingress with Traefik and setting up 2 kubernetes secrets
* Simple!  This doesn't have review apps, canary, etc.  Its few lines of code so much easier to understand for a first deployment

## Steps:

1. Create your brand new buffalo app:
{{<highlight sh>}}
buffalo new cokek8s
{{</highlight>}}
2. Create a new gitlab project and push this code to it
3. Add a `docker-compose.yml` file at the root of your project so you can test locally
{{<highlight dockerfile>}}
version: '3'
services:
  postgres:
    image: postgres:11
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_USER=postgres
    volumes:
      - db-data:/var/lib/posgresql/data
    ports:
      - "5432:5432"

volumes:
  db-data:
{{</highlight>}}
3. Modify your `Dockerfile` to be able to run buffalo test inside.  Add this line before `buffalo build`:
{{<highlight dockerfile>}}
    RUN service postgresql start && sleep 8 && buffalo test
{{</highlight>}}
4. Run your tests locally:
{{<highlight shell>}}
    docker-compose up -d
    docker build .
{{</highlight>}}
5. Add a `.gitlab-ci.yml` file at the root of your project:
{{<highlight yml>}}
stages:
  - build
  - deploy

variables:
  DOCKER_HOST: tcp://docker:2375/
  DOCKER_DRIVER: overlay2
  CONTAINER_DEV_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

services:
  - docker:dind

build_app:
  image: docker:latest
  stage: build
  script:
    - docker login -u gitlab-ci-token -p ${CI_BUILD_TOKEN} ${CI_REGISTRY}
    # if you have arguments such as API keys you want to pass into your docker container then you can add them as --buid-arg MYKEY=$KEYVALUE
    #- docker build --pull -t ${CONTAINER_DEV_IMAGE} --build-arg MYKEY="$KEYVALUE" .
    - docker build --pull -t ${CONTAINER_DEV_IMAGE} .
    - docker push ${CONTAINER_DEV_IMAGE}

deploy:
  stage: deploy
  image: alpine
  environment:
    name: production
    url: https://example.com   # TODO: change me to your domain
  only:
    refs:
      - master
  script:
    - apk add --no-cache curl
    - curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
    - chmod +x ./kubectl
    - mv ./kubectl /usr/local/bin/kubectl
    - kubectl version
    - 'printf "apiVersion: v1\nkind: Secret\n$(kubectl create secret docker-registry gitlab-registry --docker-server=$CI_REGISTRY --docker-username=$GITLAB_DEPLOY_TOKEN_USER --docker-password=$GITLAB_DEPLOY_TOKEN_PASSWORD --docker-email=$GITLAB_USER_EMAIL -o yaml --dry-run)" | kubectl apply -f -'
    - sed 's/_APP_NAME_/'"$CI_PROJECT_NAME"'/g; s/_VERSION_/'"$CI_COMMIT_SHA"'/g; s/_HOST_NAME_/example.com/g' k8s/kubernetes.tpl.yml > k8s/kubernetes.yml;
    - sed 's/_CI_PROJECT_NAME_/'"$CI_PROJECT_NAME"'/g; s/_CI_PROJECT_ID_/'"$CI_PROJECT_ID"'/g; s/_DATABASE_URL_/'"$DATABASE_URL"'/g; s/_SESSION_SECRET_/'"$SESSION_SECRET"'/g' k8s/secrets.tpl.yml > k8s/secrets.yml;
    - kubectl apply -f k8s/secrets.yml
    - kubectl apply -f k8s/kubernetes.yml
{{</highlight>}}
6.  Modify the `.gitlab-ci.yml` file to change `example.com` to your domain
7.  `mkdir k8s` and place these two files inside:
    * k8s/kubernetes.tpl.yml: this file defines the deployment and ingress
    {{<highlight yml>}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: _APP_NAME_
  labels:
    app: _APP_NAME_
spec:
  replicas: 1
  selector:
    matchLabels:
      app: _APP_NAME_

  # Pod template
  template:
    metadata:
      labels:
        app: _APP_NAME_
    spec:
      containers:
        - name: cokek8s
          image: registry.gitlab.com/calenator/cokek8s:_VERSION_
          imagePullPolicy: Always
          ports:
            - containerPort: 80
          env:
            - name: HOST
              value: "_HOST_NAME_"
            - name: HOST_URL
              value: "https://_HOST_NAME_"
            - name: PORT
              value: "3000"
            - name: SESSION_SECRET
              valueFrom:
                secretKeyRef:
                  name: my-secrets
                  key: SESSION_SECRET
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: my-secrets
                  key: DATABASE_URL
      imagePullSecrets:
        - name: gitlab-registry
---
apiVersion: v1
kind: Service
metadata:
  name: _APP_NAME_
  labels:
    app: _APP_NAME_
spec:
  ports:
    - port: 80
      targetPort: 5000
      protocol: TCP
      name: http
  selector:
      app: _APP_NAME_
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: _APP_NAME_-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.frontend.passHostHeader: "true"
    traefik.backend.loadbalancer.sticky: "true"
spec:
  rules:
  - host: _HOST_NAME_
    http:
      paths:
      - path: /
        backend:
          serviceName: _APP_NAME_
          servicePort: http
    {{</highlight>}}
    * k8s/secrets.tpl.yml: this file defines your secrets such as DATABASE_URL
    {{<highlight yml>}}
apiVersion: v1
kind: Secret
metadata:
  name: my-secrets
  namespace: _CI_PROJECT_NAME_-_CI_PROJECT_ID_
type: Opaque
stringData:
  DATABASE_URL: _DATABASE_URL_
  SESSION_SECRET: _SESSION_SECRET_
    {{</highlight>}}
8.  Modify `kubernets.tpl.yml` as needed to replace cokek8s with our own project
9. Add gitlab env vars
    1.  Go to your gitlab project
    2.  Click on settings -> Repository
    3.  Expand Deploy tokens and create a new one with name "gitlab-deploy-token" with "read_registry" permission ![screenshot](/img/gobuffalo-gitlab-kubernetes-deploy/gobuffalo-gitlab-kubernetes-deploy1.png)
    4.  Go to CI/CD settings and then add two new environment variables:  GITLAB_DEPLOY_TOKEN_USER and GITLAB_DEPLOY_TOKEN_PASSWORD
    5.  Add DATABASE_URL and SESSION_SECRET environment variables to the CI/CD settings page.  These map to the `secrets.tpl.yml` file
10. git add and git push to gitlab master.  Your pipeline should run and should be green. ![pipeline](/img/gobuffalo-gitlab-kubernetes-deploy/pipeline.png)


You can find the source code for this project here: [https://github.com/dcardamo/cokek8s](https://github.com/dcardamo/cokek8s)
