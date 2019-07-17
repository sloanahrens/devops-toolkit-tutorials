# Part 6: Updating a Kubernetes cluster and microservice dependencies from CI

In this exercise, we'll finish building out Kubernetes infrastructure that will be needed by our application stacks when we build and deploy them in Part 7.

We will also move some of the cluster maintenance into our CI pipeline.

### Route53 domain registration

To follow along with the last part of the tutorial--and actually deploy a working applicaiton, accessible from the Internet, you will need a [Route53 hosted zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingHostedZone.html).

If you do not, you will need to create a registered domain, and you can start [here](https://console.aws.amazon.com/route53/home#DomainListing:).
It does cost money, but probably not very much.
A lot of people like to use their full name for a domain-name, but anything you feel inclined to buy should work.
Mine is [sloanahrens.com].

### Complete Parts 1-5

Make sure you've completed all the parts of the tutorial up to this point.

I'll assume that you still have all the files in place from the previous exercises, contained in your `source` directory at the top level of your local clone of the `devops-toolkit` [repository](https://github.com/sloanahrens/devops-toolkit).

You also need to have a running cluster, set up as in Part 5.

### `devops` Development Environment

In this exercise we'll use the `devops` development environment, so make sure you can run it as described [here](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/0-local-dev-env-devops.md).

Make sure you have built the `devops` Docker image, by running the following command from your `devops-toolkit` directory:

```bash
docker build -t devops -f docker/devops/Dockerfile .
```

Go to your `source` directory now, and and start the development environment Docker container with (we will need port `8001` later):

```bash
docker run -it \
  -p 8001:8001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $PWD:/src \
  --name local_devops \
  --rm \
  devops \
  /bin/bash
```

Remember that in this command, the argument `-v $PWD:/src` connects the directory on the host OS from which you _run_ the command, to the `/src` directory inside the container.

Run `ls` from inside the container and should see the files you created in the `source` directory:

```
root@1bc94b877039:/src# ls
container_environments	django	docker
```

### Environment variables in `devops` container

The `devops` container environment already has `kops` installed, so we can run the `kops create` command once we set up a few environment variables.

In [Part 4](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-4-ci-circleci-aws.md) you will have created an AWS IAM user, for which you should have credentials.
We need to add those credentials to the environment with (replace "REPLACE" with your user's key values):

```bash
export AWS_ACCESS_KEY_ID=REPLACE
export AWS_SECRET_ACCESS_KEY=REPLACE
```

We also need a few more variables.
For the purposes of this exercise, I'm going to use the AWS `us-west-2` (Oregon) region.
The code already in the `devops-toolkit` repo uses `us-east-2`.

If the AWS region you are using is different than `us-west-2`, you'll need to replace it in what follows.

```bash
export REGION=us-west-2
export ZONES=${REGION}b
export CLUSTER_NAME=devops-toolkit-${REGION}.k8s.local
export BUCKET_NAME=devops-toolkit-k8s-state-${REGION}
```

### Validate cluster

To be sure your cluster is working as expected, you can run the following from your `devops` container:

```bash
cd /src/kubernetes/$REGION/cluster && \
export KUBECONFIG=$PWD/kubecfg.yaml && \
kops validate cluster --name $CLUSTER_NAME --state s3://$BUCKET_NAME
```

Your output should look similar to:

```
Validating cluster devops-toolkit-us-west-2.k8s.local

INSTANCE GROUPS
NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
master-us-west-2b	Master	c4.large	1	1	us-west-2b
nodes			Node	t2.medium	2	2	us-west-2b

NODE STATUS
NAME						ROLE	READY
ip-172-20-41-131.us-west-2.compute.internal	node	True
ip-172-20-49-53.us-west-2.compute.internal	master	True
ip-172-20-58-98.us-west-2.compute.internal	node	True

Your cluster devops-toolkit-us-west-2.k8s.local is ready
```

### AWS-EFS

We will need to create some Kubernetes spec files, the first of which will require the [resource ID]() of the EFS volume we created in Part 5.

From your `source` directory, in the running `devops` container, run:

```bash
cd /src/kubernetes/$REGION/cluster && \
terraform init && \
terraform plan
```

At the end of the output, you should see something similar to:

```
...
Releasing state lock. This may take a few moments...

Outputs:

cluster_name = devops-toolkit-us-west-2.k8s.local
efs_id = fs-8b3aa320
master_autoscaling_group_ids = [
    master-us-west-2b.masters.devops-toolkit-us-west-2.k8s.local
]
master_security_group_ids = [
    sg-02b717f7340a95030
]
masters_role_arn = arn:aws:iam::421987441365:role/masters.devops-toolkit-us-west-2.k8s.local
masters_role_name = masters.devops-toolkit-us-west-2.k8s.local
node_autoscaling_group_ids = [
    nodes.devops-toolkit-us-west-2.k8s.local
]
node_security_group_ids = [
    sg-0bdce40fe7ec5d851
]
node_subnet_ids = [
    subnet-0502688fda48b806b
]
nodes_role_arn = arn:aws:iam::421987441365:role/nodes.devops-toolkit-us-west-2.k8s.local
nodes_role_name = nodes.devops-toolkit-us-west-2.k8s.local
region = us-west-2
route_table_private-us-west-2b_id = rtb-01ad2caeafbc8ea6a
route_table_public_id = rtb-0b7c9c37cae4e019b
subnet_us-west-2b_id = subnet-0502688fda48b806b
subnet_utility-us-west-2b_id = subnet-097fb5f6b277bc402
vpc_cidr_block = 172.20.0.0/16
vpc_id = vpc-01be3123b45e68107
```

Except all of your IDs will be different.
We need the value of `efs_id` for what follows.

The following code came from [here](https://github.com/kubernetes-incubator/external-storage/tree/master/aws/efs), originally.

Create `source/kubernetes/$REGION/specs/aws-efs.yaml` with the contents:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: efs-provisioner
  labels:
    stack: cluster
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: efs-provisioner-runner
  labels:
    stack: cluster
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events", "endpoints"]
    verbs: ["list", "get", "watch", "create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-efs-provisioner
  labels:
    stack: cluster
subjects:
  - kind: ServiceAccount
    name: efs-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: efs-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: efs-provisioner
  labels:
    stack: cluster
data:
  file.system.id: fs-8b3aa320
  aws.region: us-west-2
  provisioner.name: example.com/aws-efs
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: efs-provisioner
  labels:
    stack: cluster
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: efs-provisioner
    spec:
      serviceAccountName: efs-provisioner
      containers:
        - name: efs-provisioner
          image: quay.io/external_storage/efs-provisioner:latest
          env:
            - name: FILE_SYSTEM_ID
              valueFrom:
                configMapKeyRef:
                  name: efs-provisioner
                  key: file.system.id
            - name: AWS_REGION
              valueFrom:
                configMapKeyRef:
                  name: efs-provisioner
                  key: aws.region
            - name: PROVISIONER_NAME
              valueFrom:
                configMapKeyRef:
                  name: efs-provisioner
                  key: provisioner.name
          volumeMounts:
            - name: pv-volume
              mountPath: /persistentvolumes
      volumes:
        - name: pv-volume
          nfs:
            server: fs-8b3aa320.efs.us-west-2.amazonaws.com
            path: /
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: aws-efs
  labels:
    stack: cluster
provisioner: example.com/aws-efs
```

You will need to replace the EFS ID in the line `  file.system.id: fs-e0531199`.

You will also need to update the following line to match your EFS ID and AWS Region:

```yaml
server: fs-e0531199.efs.us-west-2.amazonaws.com
```

We can now add this microservice to our running Kubernetes cluster, with the following command:

```bash
kubectl apply -f /src/kubernetes/$REGION/specs/aws-efs.yaml
```

You can see the running pod and accompanying objects with `kubectl get all`:

```
root@8655a7cde2c0:/src/kubernetes/us-west-2/cluster# kubectl get all
NAME                                   READY   STATUS    RESTARTS   AGE
pod/efs-provisioner-64977c5cb6-tgjfr   1/1     Running   0          17s


NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   100.64.0.1   <none>        443/TCP   51m


NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/efs-provisioner   1         1         1            1           17s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/efs-provisioner-64977c5cb6   1         1         1       17s
```

### External-DNS

Next we're going to add a microservice to our cluster that will help with networking.
It will automatically connect our application stacks to the Internet for us, via Amazon's Route53 service.

This code originally came from [here](https://github.com/kubernetes-incubator/external-dns/blob/master/docs/tutorials/aws.md).

Create `source/kubernetes/specs/external-dns.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.opensource.zalan.do/teapot/external-dns:v0.5.1
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=sloanahrens.com # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
        - --provider=aws
        - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
        - --aws-zone-type=public # only look at public hosted zones (valid values are public, private or no value for both)
        - --registry=txt
        - --txt-owner-id=my-identifier
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
```

Now create the microservice with:

```bash
kubectl apply -f /src/kubernetes/specs/external-dns.yaml
```

Your output should match:

```
root@8655a7cde2c0:/src/kubernetes/us-west-2/cluster# kubectl apply -f /src/kubernetes/specs/external-dns.yaml
clusterrole.rbac.authorization.k8s.io/external-dns created
clusterrolebinding.rbac.authorization.k8s.io/external-dns-viewer created
deployment.extensions/external-dns created
serviceaccount/external-dns created
```

### Nginx Ingress controller

As part of the strategy we will use to route HTTP traffic to our various app stacks, we will need to set up an [Nginx](https://www.nginx.com/) K8s [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/).

The following code originally came from [here](https://github.com/kubernetes/ingress-nginx).

Create `source/kubernetes/specs/nginx-ingress-controller.yaml` with the contents:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx

---

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    app.kubernetes.io/name: default-http-backend
    app.kubernetes.io/part-of: ingress-nginx
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: default-http-backend
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: default-http-backend
        app.kubernetes.io/part-of: ingress-nginx
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: default-http-backend
          # Any image is permissible as long as:
          # 1. It serves a 404 page at /
          # 2. It serves 200 on a /healthz endpoint
          image: k8s.gcr.io/defaultbackend-amd64:1.5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 30
            timeoutSeconds: 5
          ports:
            - containerPort: 8080
          resources:
            limits:
              cpu: 10m
              memory: 20Mi
            requests:
              cpu: 10m
              memory: 20Mi

---
apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: default-http-backend
    app.kubernetes.io/part-of: ingress-nginx
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app.kubernetes.io/name: default-http-backend
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.19.0
          args:
            - /nginx-ingress-controller
            - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1

```

Now create the controller with:

```bash
kubectl apply -f /src/kubernetes/specs/nginx-ingress-controller.yaml
```

Your output should look like:

```
root@8655a7cde2c0:/src/kubernetes/us-west-2/cluster# kubectl apply -f /src/kubernetes/specs/nginx-ingress-controller.yaml
namespace/ingress-nginx created
deployment.extensions/default-http-backend created
service/default-http-backend created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
deployment.extensions/nginx-ingress-controller created
```

### Nginx load balancer

We need another piece to our networking puzzle, and this one is cluster-specific.

_(Notice the `$REGION` in the following path.)_

Create `source/kubernetes/$REGION/specs/nginx-ingress-load-balancer.yaml` with the contents:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:us-west-2:421987441365:certificate/02632508-b328-4831-962e-3bad22df580c"
    # the backend instances are HTTP
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
    # Map port 443
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "https"
    # Ensure the ELB idle timeout is less than nginx keep-alive timeout. By default,
    # NGINX keep-alive is set to 75s. If using WebSockets, the value will need to be
    # increased to '3600' to avoid any potential issues.
    service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "60"
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  ports:
    - name: http
      port: 80
      targetPort: http
    - name: https
      port: 443
      targetPort: http

```

We can't create this load-balancer yet, though, because we first need to set up an SSL Cert.
You will not be able to do this without a domain name, so make sure you've created one.

### Route53 public hosted zone

If you already have an AWS registered domain, you can set up a hosted zone for it.
Go to [https://console.aws.amazon.com/route53/home#hosted-zones:](https://console.aws.amazon.com/route53/home#hosted-zones:) and hit the "Create Hosted Zone" button and follow the instructions.

Once you have a public hosted zone set up for your domain, we will need its `Hosted Zone ID`, so make a note of that.

### SSL Cert for load balancer

We will need to create an [SSL certificate]() for the Nginx load balancer we will create in the next section.

This can be done via the AWS web console, at: [https://us-west-2.console.aws.amazon.com/acm/](https://us-west-2.console.aws.amazon.com/acm/) (or your appropriate region).

Once you have the Certificate ready, we will need its ARN, which you can copy from the console.
Paste the ARN for your cert into the quotes in the value for the yaml key `service.beta.kubernetes.io/aws-load-balancer-ssl-cert` in the `nginx-ingress-load-balancer.yaml` file above.

### Create Nginx load balancer

With DNS and SSL all set up, we are finally ready to create an Nginx load-balancer for our cluster.
When we deploy app stacks in the next section, they will have their web traffic routed through our Nginx Ingress load-balancer.

Now create the load-balancer with:

```bash
kubectl apply -f /src/kubernetes/$REGION/specs/nginx-ingress-load-balancer.yaml
```

You should see this output:

```
root@8655a7cde2c0:/src/kubernetes/us-west-2/cluster# kubectl apply -f /src/kubernetes/$REGION/specs/nginx-ingress-load-balancer.yaml
service/ingress-nginx created
```

### Kubernetes Dashboard

We will also add [kubernetes dashboard](https://github.com/kubernetes/dashboard) to our cluster because it can be handy.

Create a file `source/kubernetes/specs/kubernetes-dashboard.yaml` with the contents (or copy from the GitHub page):

```yaml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# ------------------- Dashboard Secret ------------------- #

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque

---
# ------------------- Dashboard Service Account ------------------- #

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system

---
# ------------------- Dashboard Role & Role Binding ------------------- #

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
rules:
  # Allow Dashboard to create 'kubernetes-dashboard-key-holder' secret.
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
  # Allow Dashboard to create 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs"]
  verbs: ["get", "update", "delete"]
  # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["kubernetes-dashboard-settings"]
  verbs: ["get", "update"]
  # Allow Dashboard to get metrics from heapster.
- apiGroups: [""]
  resources: ["services"]
  resourceNames: ["heapster"]
  verbs: ["proxy"]
- apiGroups: [""]
  resources: ["services/proxy"]
  resourceNames: ["heapster", "http:heapster:", "https:heapster:"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard-minimal
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system

---
# ------------------- Dashboard Deployment ------------------- #

kind: Deployment
apiVersion: apps/v1beta2
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - name: kubernetes-dashboard
        image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
          # Uncomment the following line to manually specify Kubernetes API server Host
          # If not specified, Dashboard will attempt to auto discover the API server and connect
          # to it. Uncomment only if the default does not work.
          # - --apiserver-host=http://my-address:port
        volumeMounts:
        - name: kubernetes-dashboard-certs
          mountPath: /certs
          # Create on-disk volume to store exec logs
        - mountPath: /tmp
          name: tmp-volume
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: kubernetes-dashboard-certs
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule

---
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard

---
# SA, 11/1/2018:
# the following gives the dashboard full admin permissions, which is a bad idea security-wise,
# so we don't want to open this up to the internet (don't put an ingress/LB in front of it!)
# see: https://blog.heptio.com/on-securing-the-kubernetes-dashboard-16b09b1b7aca
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
```

Now we can install the dashboard with:

```bash
kubectl apply -f /src/kubernetes/specs/kubernetes-dashboard.yaml
```

You should see:

```
root@8655a7cde2c0:/src/kubernetes/us-west-2/cluster# kubectl apply -f /src/kubernetes/specs/kubernetes-dashboard.yaml
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
```

To use the dashboard, run:

```bash
cd /src/kubernetes/$REGION/cluster && \
export KUBECONFIG=$PWD/kubecfg.yaml && \
kubectl proxy
```

Now, in your browser, go to [http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/).

Once you're done with the dashboard, hit `ctl-c` to kill the proxy server.

### CircleCI Integration

We can integrate the [idempotent](https://en.wikipedia.org/wiki/Idempotence) parts of what we've done in this exercise into our CircleCI platform.
We're going to add some code to the CircleCI config file.

Edit `source/.circleci/config.yaml` to match (replace instances of `us-east-2` and `us-west-2` with your desired region(s)):

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

workflows:
  version: 2
  build-and-deploy:
    jobs:
      - update_kubernetes_cluster
      - image_build_test_push
      - tagged_image_test:
          requires:
            - image_build_test_push
```

### Encrypt `kubecfg` file

To make our communication from CI to AWS secure, we have to encrypt the `kubecfg` file that contains our SSH key.

A simple way to do this is to use `gpg`.

Pick a long, [durable pass-phrase](https://www.xkcd.com/936/) and run:

```bash
KUBECONFIG_PASSPHRASE=some-long-string-of-random-words-more-secure-than-this
cd /src/kubernetes/$REGION/cluster
gpg --symmetric --batch --yes --passphrase $KUBECONFIG_PASSPHRASE kubecfg.yaml

```

This pass-phrase has to be added to CircleCI as well, in the settings for the Project for your GitHub repo.

The URL of the settings page will have the form: [https://circleci.com/gh/user-name/repo-name/edit#env-vars](https://circleci.com/gh/user-name/repo-name/edit#env-vars).

Once you click on the "Add Variable" button, create the `KUBECONFIG_PASSPHRASE` environment variable, like:

![alt text](https://github.com/sloanahrens/devops-toolkit-tutorials/raw/master/images/circleci_passphrase_env_var.png "Set kubecfg pass-phrase in CircleCI")

### Push changes to Github

Now we need to push all these changes to GitHub so we can see the new CircleCI job run.

Push your code changes to GitHub with (from your host OS, in your `source` directory):

```bash
git add -A
git commit -m "add k8s"
git push origin master
```

Now go to your CircleCI dashboard.
After all the jobs have finshed, the one called `update_kubernetes_cluster` should look similar to:

![alt text](https://github.com/sloanahrens/devops-toolkit-tutorials/raw/master/images/circleci_k8s_update.png "Successful CI K8s update")

In the next exercise, we will deploy the `stockpicker` application to Kubernetes, and run tests against our deployments.

[Prev: Part 5](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-5-orchestration-kubernetes-kops.md)
[Next: Part 7](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-7-microservices-deploy-test.md)
