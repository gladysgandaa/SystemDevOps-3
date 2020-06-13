Name: Gladys Ganda
Student ID: s3679389

## Analysis of the Problem

The problem that ACME corp is facing at the moment is to expand their CI build to be able to create container, deploy  the application and implement the pipeline. We will be using Helm to create chart, Kubernetes to deploy the acme application, and CircleCI to build the pipeline, AWS to handle cluster, container and the rest of cloud configuration.

## Create Helm Chart 

- We will first start by creating a directory called helm and change to that directory 

```
$> mkdir helm
$> cd helm
```

- After changing to that direcotry, we will generate a new helm chart, which in this case we are calling it acme

``` 
$> helm create acme
```

- Next, we're going to change directory to acme folder that we have just generate, and we will delete unnecessary files.

```
$> rmdir charts
$> rm -rf templates/*
$> echo "" > values.yml
```

- Now we are going to include both deployment and service manifest

#### Deployment

- Create a new file deployment.yml inside templates folder

``` 
$> touch templates/deployment.yml
```

- Add content to deployment.yml as below 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: "acme-app"
spec:
  selector:
    matchLabels:
      app: "acme-app"
  replicas: {{ .Values.replicaCount }}
  template: 
    metadata:
      labels:
        app: "acme-app"
    spec:
      containers:
      - image: {{ .Values.image }}
        name: "acme-app"
        env: 
          - name: DB_HOSTNAME
            value: {{ .Values.dbhost }}
          - name: DB_USERNAME
            value: {{ .Values.dbuser }}
          - name: DB_NAME
            value: {{ .Values.dbname }}
          - name: DB_PASSWORD
            value: {{ .Values.dbpassword }}
          - name: "PORT"
            value: "80"
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
```

- We are going to set some database and image variable on values.yml inside the acme folder which are going to be pass to deployment.yml 

```
replicaCount: 1
image: acme-app
env: prod
dbhost: localhost
dbname: servian
dbuser: postgres
dbpassword: password
```

#### Service

- Create a new file called service.yml inside the templates folder. Make sure you're in helm directory

```
$> touch templates/service.yml
```

- Add this content to service.yml 

```
apiVersion: v1
kind: Service
metadata:
  name: "acme-app"
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: LoadBalancer
  selector:
    app: "acme-app"
```

## Deploy Application into Non-Production environment

- On .circleci/config.yml, we are creating a new job called deploy-test.

```
 deploy-test:
    docker: 
      - image: cimg/base:2020.01
    environment:
      ENV: test
    steps:
      - attach_workspace:
          at: ./

      - setup-cd 

      - run:
          name: deploy infra
          command: |
            cd artifacts/infra
            make init
            make up
            terraform output endpoint > ../dbendpoint.txt
    
      - run:
          name: deploy app
          command: |
            kubectl get namespace test || kubectl create namespace test 
            helm upgrade app artifacts/acme-0.1.0.tgz -i -n test --wait --set image=$(cat artifacts/image.txt),dbusername=postgres,dbpassword=password,dbhost=$(cat artifacts/dbendpoint.txt),env=test

      - run: 
          name: upgrade and seed DATABASE
          command: |
            sleep 10
            kubectl exec deployment/acme-app -n test -- env NODE_ENV=production node_modules/.bin/sequelize db:migrate  
```

- We use this command to create namespace. The left side of the pipe is checking whether namespace 'test' exists. If exists, then it will not create the namespace, but if it doesn't it will create a namespace called 'test'. 

```
          name: deploy app
          command: |
            kubectl get namespace test || kubectl create namespace test 
```

- Deploy HELM Chart and database to back of the application. We are re-using the image and the db that we have pass from the previous job. and set the database username, host and setting the environment to test. 

```
        name: deploy app
        command: |
            kubectl get namespace test || kubectl create namespace test 
            helm upgrade app artifacts/acme-0.1.0.tgz -i -n test --wait --set image=$(cat artifacts/image.txt),dbusername=postgres,dbpassword=password,dbhost=$(cat artifacts/dbendpoint.txt),env=test
```

- Run database migration script against the database. Sleep 10 is a command to ask the job to sleep for 10 seconds. 

```
      - run: 
          name: upgrade and seed DATABASE
          command: |
            sleep 10
            kubectl exec deployment/acme-app -n test -- env NODE_ENV=production node_modules/.bin/sequelize db:migrate  
```

## Change End to End Testing against non-production environment

- On e2e job inside config.yml, change the content as below
- Set up the environment by installing sudo, followed by kubernetes and kops , and jq (JSON processor). Next, run the e2e testing by setting the endpoint to the url of the external-ip of the application from production environment

```
 e2e:
    docker:
      - image: qawolf/qawolf:v0.9.2
    environment:
      QAW_HEADLESS: true

    steps: 
      - checkout

      - run:
          name: set up environment
          command: |
            apt-get install -y sudo

            # install kubectl
            curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
            chmod +x ./kubectl 
            sudo mv ./kubectl /usr/local/bin/kubectl

            # configure kops
            curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
            chmod +x kops-linux-amd64
            sudo mv kops-linux-amd64 /usr/local/bin/kops
            kops export kubecfg rmit.k8s.local --state s3://rmit-kops-state-09wd7g

            apt-get install jq -y

      - run: 
          name: End 2 end tests
          no_output_timeout: 2m
          command: |
            export ENDPOINT=http://$(kubectl get service/acme-app -n test -o json | jq '.status.loadBalancer.ingress[0].hostname | sed -e 's/^"//' -e 's/"$//')
            cd src
            npm install
            npm run test-e2e
```

### Deploy Application into Production Environment 

- Create another job called 'deploy-prod' which will have the same content as the deploy-test job. We will create a new namespace called prod, and set the environment prod. 

```
  deploy-prod:
    docker:
      - image: cimg/base:2020.01
    environment:
      ENV: prod
    steps:
      - attach_workspace:
          at: ./
      
      - setup-cd 

      - run:
          name: deploy infra
          command: |
            cd artifacts/infra
            make init
            make up
            terraform output endpoint > ../dbendpoint.txt
      
      - run:
          name: deploy app on production
          command: | 
            kubectl get namespace prod || kubectl create namespace prod 
            helm upgrade app artifacts/acme-0.1.0.tgz -i -n prod --wait --set image=$(cat artifacts/image.txt),dbusername=postgres,dbpassword=password,dbhost=$(cat artifacts/dbendpoint.txt),env=prod

      - run:
          name: upgrade database on production
          command: |
            sleep 10
            kubectl exec deployment/acme-app -n prod -- env NODE_ENV=production node_modules/.bin/sequelize db:migrate 
```

- Next, create a stage gate before deployment allowed to production. It will request for approval before proceeding to deploy-prod.

```
      - deploy-test:
          requires:
              - package 
      - approval:
          type: approval
          requires:
            - deploy-test
      - deploy-prod:
          requires:
            - approval
```




    