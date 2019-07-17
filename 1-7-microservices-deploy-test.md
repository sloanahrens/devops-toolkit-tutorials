# Part 6: Deploying and testing application stacks in K8s from CI

In this exercise, we'll deploy our `stockpicker` application to Kubernetes via [spec templates](), test the deployed stacks with the testing infrastructure from earlier exercises, and ultimately set up a simple [continuous-deployment]() system.

### Complete Parts 1-6

Make sure you've completed all the parts of the tutorial up to this point.

I'll assume that you still have all the files in place from the previous exercises, contained in your `source` directory at the top level of your local clone of the `devops-toolkit` [repository](https://github.com/sloanahrens/devops-toolkit).

You also need to have a running cluster, set up as in Part 5 and finished in Part 6.

All we need to do now to finish up is create a bunch of files.
We'll need some spec templates, some ci-scripts, and and updated CircleCI config file.

### Spec Templates

**1)** `source/kubernetes/spec_templates/redis.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    stack: STACK_NAME
spec:
  clusterIP: None
  ports:
  - port: 6379
  selector:
    type: redis
    service: redis
---
apiVersion: apps/v1beta2
kind: StatefulSet
metadata:
  name: redis
  labels:
    stack: STACK_NAME
spec:
  serviceName: redis
  selector:
    matchLabels:
      type: redis
      service: redis
  template:
    metadata:
      labels:
        type: redis
        service: redis
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: redis
        image: redis
        imagePullPolicy: IfNotPresent
        envFrom:
        - configMapRef:
            name: stack-environment-variables
```

**2)** `source/kubernetes/spec_templates/rabbitmq.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-management
  labels:
    stack: STACK_NAME
spec:
  clusterIP: None
  ports:
  - port: 15672
  selector:
    type: rabbitmq
    service: rabbitmq
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
  labels:
    stack: STACK_NAME
spec:
  clusterIP: None
  ports:
  - port: 5672
  selector:
    type: rabbitmq
    service: rabbitmq
---
apiVersion: apps/v1beta2
kind: StatefulSet
metadata:
  name: rabbitmq
  labels:
    stack: STACK_NAME
spec:
  serviceName: rabbitmq
  selector:
    matchLabels:
      type: rabbitmq
      service: rabbitmq
  template:
    metadata:
      labels:
        type: rabbitmq
        service: rabbitmq
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: rabbitmq
        image: rabbitmq:3.6
        imagePullPolicy: IfNotPresent
        envFrom:
        - configMapRef:
            name: stack-environment-variables
```

**3)** `source/kubernetes/spec_templates/postgres.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    stack: STACK_NAME
spec:
  clusterIP: None
  ports:
  - port: 5432
  selector:
    type: postgres
    service: postgres
---
apiVersion: apps/v1beta2
kind: StatefulSet
metadata:
  name: postgres
  labels:
    stack: STACK_NAME
spec:
  serviceName: postgres
  selector:
    matchLabels:
      type: postgres
      service: postgres
  template:
    metadata:
      labels:
        type: postgres
        service: postgres
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: postgres
        image: postgres:9.4
        imagePullPolicy: IfNotPresent
        envFrom:
        - configMapRef:
            name: stack-environment-variables
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: pg-data
      volumes:
      - name: pg-data
        persistentVolumeClaim:
          claimName: postgres-data
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgres-data
  labels:
    stack: STACK_NAME
  annotations:
    volume.beta.kubernetes.io/storage-class: "aws-efs"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

**4)** `source/kubernetes/spec_templates/celerybeat.yaml`:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: celerybeat
  labels:
    stack: STACK_NAME
    type: django-app
    service: celerybeat
spec:
  clusterIP: None
  selector:
    type: django-app
    service: celerybeat
---
apiVersion: apps/v1beta2
kind: StatefulSet
metadata:
  name: celerybeat
  labels:
    stack: STACK_NAME
spec:
  serviceName: celerybeat
  replicas: 1
  selector:
    matchLabels:
      type: django-app
      service: celerybeat
  template:
    metadata:
      labels:
        type: django-app
        service: celerybeat
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: worker
        image: 421987441365.dkr.ecr.us-east-2.amazonaws.com/stockpicker/celery:IMAGE_TAG
        imagePullPolicy: Always
        envFrom:
        - configMapRef:
            name: stack-environment-variables
        env:
        - name: STACKNAME
          value: STACK_NAME
        - name: IMAGETAG
          value: IMAGE_TAG
        - name: ENVFILE
          value: ENV_FILE
        command: ["/bin/bash"]
        args: ["-c", "sleep 180 && celery beat --app=stockpicker.celery --loglevel=info"]
```

In this file, you need to replace `421987441365` with your AWS ECR repo-id, and `us-east-2` with your appropriate region.

**5)** `source/kubernetes/spec_templates/celeryworker.yaml`:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: celeryworker
  labels:
    stack: STACK_NAME
    type: django-app
    service: celeryworker
spec:
  clusterIP: None
  selector:
    type: django-app
    service: celeryworker
---
apiVersion: apps/v1beta2
kind: StatefulSet
metadata:
  name: celeryworker
  labels:
    stack: STACK_NAME
spec:
  serviceName: celeryworker
  replicas: WORKER_REPLICAS
  selector:
    matchLabels:
      type: django-app
      service: celeryworker
  template:
    metadata:
      labels:
        type: django-app
        service: celeryworker
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: worker
        image: 421987441365.dkr.ecr.us-east-2.amazonaws.com/stockpicker/celery:IMAGE_TAG
        imagePullPolicy: Always
        envFrom:
        - configMapRef:
            name: stack-environment-variables
        env:
        - name: STACKNAME
          value: STACK_NAME
        - name: IMAGETAG
          value: IMAGE_TAG
        - name: ENVFILE
          value: ENV_FILE
        command: ["/bin/bash"]
        args: ["-c", "celery worker -O fair -c 1 --app=stockpicker.celery --loglevel=info"]
```

In this file, you need to replace `421987441365` with your AWS ECR repo-id, and `us-east-2` with your appropriate region.

**6)** `source/kubernetes/spec_templates/webapp.yaml`:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: webapp
  labels:
    stack: STACK_NAME
    type: django-app
    service: webapp
spec:
  clusterIP: None
  selector:
    type: django-app
    service: webapp
  ports:
  - name: web
    port: 8000
    targetPort: 8000
---
apiVersion: apps/v1beta2
kind: StatefulSet
metadata:
  name: webapp
  labels:
    stack: STACK_NAME
spec:
  serviceName: webapp
  replicas: WEBAPP_REPLICAS
  selector:
    matchLabels:
      type: django-app
      service: webapp
  template:
    metadata:
      labels:
        type: django-app
        service: webapp
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: webapp
        image: 421987441365.dkr.ecr.us-east-2.amazonaws.com/stockpicker/webapp:IMAGE_TAG
        imagePullPolicy: Always
        envFrom:
        - configMapRef:
            name: stack-environment-variables
        env:
        - name: STACKNAME
          value: STACK_NAME
        - name: IMAGETAG
          value: IMAGE_TAG
        - name: ENVFILE
          value: ENV_FILE
        readinessProbe:
          httpGet:
            path: /health/database/
            port: 8000
          initialDelaySeconds: 120
          timeoutSeconds: 1
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/tickers-loaded/
            port: 8000
          initialDelaySeconds: 120
          timeoutSeconds: 1
          periodSeconds: 10
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: webapp
  labels:
    stack: STACK_NAME
    type: django-app
    service: webapp
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  rules:
  - host: STACK_NAME.sloanahrens.com
    http:
      paths:
      - backend:
          serviceName: webapp
          servicePort: 8000
```

In this file, you need to replace `sloanahrens.com` with your domain name, `421987441365` with your AWS ECR repo-id, and `us-east-2` with your appropriate region.

### CI Scripts

**1)** `source/ci_scripts/deploy_k8s_app_stack.sh`:

```yaml
#!/usr/bin/env bash

STACK_NAME="$1"
IMAGE_TAG="$2"
ENV_FILE="$3"
WEBAPP_REPLICAS="$4"
WORKER_REPLICAS="$5"

echo "Deploying k8s stack:"
echo "STACK_NAME: ${STACK_NAME}"
echo "IMAGE_TAG: ${IMAGE_TAG}"
echo "ENV_FILE: ${ENV_FILE}"
echo "WEBAPP_REPLICAS: ${WEBAPP_REPLICAS}"
echo "WORKER_REPLICAS: ${WORKER_REPLICAS}"

kubectl create namespace ${STACK_NAME}

kubectl -n ${STACK_NAME} create cm stack-environment-variables --from-env-file=./container_environments/${ENV_FILE}

cat kubernetes/spec_templates/redis.yaml \
  | sed -e  "s@STACK_NAME@${STACK_NAME}@g" \
  | kubectl -n ${STACK_NAME} create -f -

cat kubernetes/spec_templates/rabbitmq.yaml \
  | sed -e  "s@STACK_NAME@${STACK_NAME}@g" \
  | kubectl -n ${STACK_NAME} create -f -

cat kubernetes/spec_templates/postgres.yaml \
  | sed -e  "s@STACK_NAME@${STACK_NAME}@g" \
  | kubectl -n ${STACK_NAME} create -f -

cat kubernetes/spec_templates/webapp.yaml \
  | sed -e  "s@STACK_NAME@${STACK_NAME}@g" \
  | sed -e  "s@IMAGE_TAG@${IMAGE_TAG}@g" \
  | sed -e  "s@ENV_FILE@${ENV_FILE}@g" \
  | sed -e  "s@WEBAPP_REPLICAS@${WEBAPP_REPLICAS}@g" \
  | kubectl -n ${STACK_NAME} create -f -

cat kubernetes/spec_templates/celeryworker.yaml \
  | sed -e  "s@STACK_NAME@${STACK_NAME}@g" \
  | sed -e  "s@IMAGE_TAG@${IMAGE_TAG}@g" \
  | sed -e  "s@ENV_FILE@${ENV_FILE}@g" \
  | sed -e  "s@WORKER_REPLICAS@${WORKER_REPLICAS}@g" \
  | kubectl -n ${STACK_NAME} create -f -

cat kubernetes/spec_templates/celerybeat.yaml \
  | sed -e  "s@STACK_NAME@${STACK_NAME}@g" \
  | sed -e  "s@IMAGE_TAG@${IMAGE_TAG}@g" \
  | sed -e  "s@ENV_FILE@${ENV_FILE}@g" \
  | kubectl -n ${STACK_NAME} create -f -

echo "Deployed stack \"${STACK_NAME}\"."
```

**2)** `source/ci_scripts/test_k8s_app_stack.sh`:

```yaml
#!/usr/bin/env bash

STACK_NAME="$1"

echo "Testing: ${STACK_NAME}.sloanahrens.com"

docker run -e SERVICE="https://${STACK_NAME}.sloanahrens.com" stacktest ./integration-tests.sh \
    || \
    (echo "*** PODS:" && echo "$(kubectl -n ${STACK_NAME} get pods)" && \
    WEBAPP_POD=$(kubectl -n ${STACK_NAME} get pod -l service=webapp -o jsonpath="{.items[0].metadata.name}") && \
    WORKER_POD=$(kubectl -n ${STACK_NAME} get pod -l service=celeryworker -o jsonpath="{.items[0].metadata.name}") && \
    echo "*** WORKER LOGS:" && echo "$(kubectl -n ${STACK_NAME} logs $WORKER_POD)" && \
    echo "*** WEBAPP LOGS:" && echo "$(kubectl -n ${STACK_NAME} logs $WEBAPP_POD)" && \
    echo "Integration Tests Failed. See response and logs output above." && exit 1)
```

**3)** `source/ci_scripts/delete_k8s_app_stack.sh`:

```yaml
#!/usr/bin/env bash

STACK_NAME="$1"

echo "Deleting K8s resources for STACK_NAME: ${STACK_NAME}"
kubectl -n ${STACK_NAME} delete service,deployment,ingress,statefulset,pod,pvc --all --grace-period=0 --force
kubectl delete ns ${STACK_NAME}  --ignore-not-found=true

aws route53 list-resource-record-sets  --hosted-zone-id Z1CDZE44WDSMXZ | jq -c '.ResourceRecordSets[]' |
while read -r resourcerecordset; do
  read -r name type <<<$(echo $(jq -r '.Name,.Type' <<<"$resourcerecordset"))
  if [[ $name = ${STACK_NAME}* ]]; then
        echo "Deleting Route53 records for STACK_NAME: ${STACK_NAME}"
        aws route53 change-resource-record-sets \
          --hosted-zone-id Z1CDZE44WDSMXZ \
          --change-batch '{"Changes":[{"Action":"DELETE","ResourceRecordSet":
              '"$resourcerecordset"'
            }]}' \
          --output text --query 'ChangeInfo.Id'
  fi
done
```

### CircleCI Config with K8s-stack tests and continuous deployment

```yaml
defaults: &defaults
  docker:
    - image: sloanahrens/devops-toolkit-ci-dev-env:0.0.3

version: 2
jobs:

  update_kubernetes_cluster:
    <<: *defaults
    steps:
      - checkout
      - run:
          name: Terraform Apply
          command: |
            set -x
            cd kubernetes/us-west-2/cluster
            terraform init
            terraform plan
            terraform apply --auto-approve
      - run:
          name: Kops Rolling Update
          command: |
            set -x
            CLUSTER_NAME=devops-toolkit-us-west-2.k8s.local
            BUCKET_NAME=devops-toolkit-k8s-state-us-west-2
            cd kubernetes/us-west-2/cluster
            gpg --decrypt --batch --yes --passphrase $KUBECONFIG_PASSPHRASE kubecfg.yaml.gpg > kubecfg.yaml
            export KUBECONFIG=$PWD/kubecfg.yaml
            kops rolling-update cluster --name $CLUSTER_NAME --state s3://$BUCKET_NAME --yes
      - run:
          name: Kops Validate Cluster
          command: |
            set -x
            sleep 30
            CLUSTER_NAME=devops-toolkit-us-west-2.k8s.local
            BUCKET_NAME=devops-toolkit-k8s-state-us-west-2
            cd kubernetes/us-west-2/cluster
            gpg --decrypt --batch --yes --passphrase $KUBECONFIG_PASSPHRASE kubecfg.yaml.gpg > kubecfg.yaml
            export KUBECONFIG=$PWD/kubecfg.yaml
            kops validate cluster --name $CLUSTER_NAME --state s3://$BUCKET_NAME
      - run:
          name: Update K8s Dependencies
          command: |
            set -x
            cd kubernetes/us-west-2/cluster
            gpg --decrypt --batch --yes --passphrase $KUBECONFIG_PASSPHRASE kubecfg.yaml.gpg > kubecfg.yaml
            export KUBECONFIG=$PWD/kubecfg.yaml
            cd ../../..
            kubectl apply -f kubernetes/specs/kubernetes-dashboard.yaml
            kubectl apply -f kubernetes/specs/external-dns.yaml
            kubectl apply -f kubernetes/specs/nginx-ingress-controller.yaml
            kubectl apply -f kubernetes/us-west-2/specs/nginx-ingress-load-balancer.yaml
            kubectl apply -f kubernetes/us-west-2/specs/aws-efs.yaml

  image_build_test_push:
    <<: *defaults
    steps:
      - checkout
      - setup_remote_docker:
          version: 17.03.0-ce
      - run:
          name: Cache Directory
          command: |
            set -x
            mkdir -p /caches
      - restore_cache:
          key: v1-{{ .Branch }}-{{ checksum "django/requirements.txt" }}
          paths:
            - /caches/baseimage.tar
      - run:
          name: Load Base Image From Cache
          command: |
            set -x
            if [ -e /caches/baseimage.tar ]; then
              docker load -i /caches/baseimage.tar
            fi
      - run:
          name: Build Base Image
          command: |
            set -x
            docker build --cache-from=baseimage -t baseimage -f docker/baseimage/Dockerfile .
      - run:
          name: Save Base Image To Cache
          command: |
            set -x
            docker save -o /caches/baseimage.tar baseimage
      - save_cache:
          key: v1-{{ .Branch }}-{{ checksum "django/requirements.txt" }}
          paths:
            - /caches/baseimage.tar
      - run:
          name: Build Webapp Image
          command: |
            set -x
            docker build -t webapp -f docker/webapp/Dockerfile .
      - run:
          name: Build Celery Image
          command: |
            set -x
            docker build -t celery -f docker/celery/Dockerfile .
      - run:
          name: Build Stack Tester Image
          command: |
            set -x
            docker build -t stacktest -f docker/stacktest/Dockerfile .
      - run:
          name: Run Django Unit Tests
          command: |
            set -x
            docker-compose -f docker/docker-compose-unit-test.yaml run unit-test
            docker-compose -f docker/docker-compose-unit-test.yaml down
      - run:
          name: Run Integration Tests Against Local Docker-Compose Stack
          command: |
            set -x
            docker-compose -f docker/docker-compose-local-image-stack.yaml up -d
            docker run -e SERVICE="localhost:8001" --network container:stockpicker_webapp stacktest ./integration-tests.sh \
                || \
                (echo "*** WORKER LOGS:" && echo "$(docker logs stockpicker_worker)" && \
                echo "*** WEBAPP LOGS:" && echo "$(docker logs stockpicker_webapp)" && exit 1)
            docker-compose -f docker/docker-compose-local-image-stack.yaml down
      - run:
          name: Push Docker Images
          command: |
            set -x
            export PATH="$PATH:$(python -m site --user-base)/bin"
            $(aws ecr get-login --no-include-email --region us-east-2)
            IMAGE_TAG=$(echo "$CIRCLE_BRANCH" | sed 's/[^a-zA-Z0-9]/-/g'| sed -e 's/\(.*\)/\L\1/')
            AWS_REGION=us-east-2
            ECR_ID=421987441365
            docker tag webapp $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/webapp:$IMAGE_TAG
            docker tag celery $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/celery:$IMAGE_TAG
            docker push $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/webapp:$IMAGE_TAG
            docker push $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/celery:$IMAGE_TAG

  tagged_image_test:
    <<: *defaults
    steps:
      - checkout
      - setup_remote_docker:
          version: 17.03.0-ce
      - run:
          name: Build Stack Tester Image
          command: |
            set -x
            docker build -t stacktest -f docker/stacktest/Dockerfile .
      - run:
          name: Run Integration Tests With Pushed Images
          command: |
            set -x
            export IMAGE_TAG=$(echo "$CIRCLE_BRANCH" | sed 's/[^a-zA-Z0-9]/-/g'| sed -e 's/\(.*\)/\L\1/')
            export AWS_REGION=us-east-2
            export ECR_ID=421987441365
            $(aws ecr get-login --no-include-email --region us-east-2)
            docker-compose -f docker/docker-compose-tagged-image-stack.yaml up -d
            docker run -e SERVICE="localhost:8001" --network container:stockpicker_webapp stacktest ./integration-tests.sh \
                || \
                (echo "*** WORKER1 LOGS:" && echo "$(docker logs stockpicker_worker1)" && \
                echo "*** WORKER2 LOGS:" && echo "$(docker logs stockpicker_worker2)" && \
                echo "*** WEBAPP LOGS:" && echo "$(docker logs stockpicker_webapp)" && \
                exit 1)
            docker-compose -f docker/docker-compose-tagged-image-stack.yaml down

  kubernetes_image_test:
    <<: *defaults
    steps:
      - checkout
      - setup_remote_docker:
          version: 17.03.0-ce
      - run:
          name: Build Stack Tester Image
          command: |
            set -x
            docker build -t stacktest -f docker/stacktest/Dockerfile .
      - run:
          name: Build and Test K8s Stack
          command: |
            set -x
            cd kubernetes/us-west-2/cluster
            gpg --decrypt --batch --yes --passphrase $KUBECONFIG_PASSPHRASE kubecfg.yaml.gpg > kubecfg.yaml
            export KUBECONFIG=$PWD/kubecfg.yaml
            cd ../../..
            IMAGE_TAG=$(echo "$CIRCLE_BRANCH" | sed 's/[^a-zA-Z0-9]/-/g'| sed -e 's/\(.*\)/\L\1/')
            STACK_NAME=ci-test-stockpicker-$CIRCLE_BUILD_NUM
            ENV_FILE=test-stack.yaml
            ./ci_scripts/delete_k8s_app_stack.sh $STACK_NAME
            sleep 10
            ./ci_scripts/deploy_k8s_app_stack.sh $STACK_NAME $IMAGE_TAG $ENV_FILE 1 2
            sleep 240 # hack for now
            ./ci_scripts/test_k8s_app_stack.sh $STACK_NAME
            ./ci_scripts/delete_k8s_app_stack.sh $STACK_NAME

  prod_stack_deployment:
    <<: *defaults
    steps:
      - checkout
      - setup_remote_docker:
          version: 17.03.0-ce
      - run:
          name: Build Stack Tester Image
          command: |
            set -x
            docker build -t stacktest -f docker/stacktest/Dockerfile .
      - run:
          name: Build and Test Production K8s Stack
          command: |
            set -x
            cd kubernetes/us-west-2/cluster
            gpg --decrypt --batch --yes --passphrase $KUBECONFIG_PASSPHRASE kubecfg.yaml.gpg > kubecfg.yaml
            export KUBECONFIG=$PWD/kubecfg.yaml
            cd ../../..
            IMAGE_TAG=master
            STACK_NAME=stockpicker
            ENV_FILE=test-stack.yaml
            ./ci_scripts/delete_k8s_app_stack.sh $STACK_NAME
            sleep 10
            ./ci_scripts/deploy_k8s_app_stack.sh $STACK_NAME $IMAGE_TAG $ENV_FILE 1 2
            sleep 240 # hack for now
            ./ci_scripts/test_k8s_app_stack.sh $STACK_NAME

workflows:
  version: 2
  build-and-deploy:
    jobs:
      - update_kubernetes_cluster
      - image_build_test_push
      - tagged_image_test:
          requires:
            - image_build_test_push
      - kubernetes_image_test:
          requires:
            - update_kubernetes_cluster
            - tagged_image_test
      - prod_stack_deployment:
          requires:
            - kubernetes_image_test
          filters:
            branches:
              only:
                - master
```

### Push changes to Github

Now we need to push all these changes to GitHub so we can see the new CircleCI job run.

Push your code changes to GitHub with (from your host OS, in your `source` directory):

```bash
git add -A
git commit -m "add k8s"
git push origin master
```

Now go to your CircleCI dashboard.

### Manual stack deployment and testing

Make sure you have built the `devops` Docker image, by running the following command from your `devops-toolkit` directory:

```bash
docker build -t devops -f docker/devops/Dockerfile .
```

Go to your `source` directory now, and and start the development environment Docker container with:

```bash
docker run -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $PWD:/src \
  --name local_devops \
  --rm \
  devops \
  /bin/bash
```

Remember that in this command, the argument `-v $PWD:/src` connects the directory on the host OS from which you _run_ the command, to the `/src` directory inside the container.

Now from inside the running container, pick a value of `IMAGE_TAG` and `STACK_NAME` (or leave them unchanged) and run the following to deploy a stack to our Kubernetes cluster:

```bash
IMAGE_TAG=master && \
STACK_NAME=stockpicker && \
ENV_FILE=test-stack.yaml && \
./ci_scripts/deploy_k8s_app_stack.sh $STACK_NAME $IMAGE_TAG $ENV_FILE 1 2
```

To run the integration tests against that stack, run:

```bash
./ci_scripts/test_k8s_app_stack.sh $STACK_NAME
```

To delete the stack, run:

```bash
./ci_scripts/delete_k8s_app_stack.sh $STACK_NAME
```

[Prev: Part 6](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-6-ci-cluster-maintenance.md)
