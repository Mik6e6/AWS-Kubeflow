# AWS-Kubeflow


![](https://raw.githubusercontent.com/kubeflow/kfserving/51f4f8c0131bc687b8299446e8efe11aade7f903/docs/diagrams/kfserving.png)


### Cluster Setup:

1. Installing eksctl 

``` 

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

```

2.Cluster Configuration

  2.1 cluster.yaml 
  
  ```apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig


metadata:
    name: {CLUSTER_NAME}
    region: us-east-1

nodeGroups:

  - name: ng
    instanceType: m5.large
    desiredCapacity: 2

```


2.2 Create Clsuter (takes a while)

```
eksctl create cluster -f cluster.yaml
eksctl utils associate-iam-oidc-provider --cluster={CLUSTER_NAME} --approve
```

2.3 Once the Cluster is created, we can connect to it with the following

```

aws eks --region us-east-1 update-kubeconfig --name 

```

3. Installing kubectl

```

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

kubectl version --client

```

4. Deploying the K8S Dashboard


```

wget https://api.github.com/repos/kubernetes-sigs/metrics-server/tarball/v0.3.6

tar zxvf v0.3.6

kubectl apply -f kubernetes-sigs-metrics-server-d1f4f6f/deploy/1.8+

```

To verify the metrics server is working

```

kubectl get deployment metrics-server -n kube-system

```


5. User Creation eks-admin-service-account.yaml

```

apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
  
```


6. Accessing the Dashboard

```

kubectl proxy

```

Generating the Token

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')

```


Login to the Dashboard with the Token from the previous step


http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login



7. Adding Load Balancer

```

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/rbac-role.yaml

```

Creating the Role

```

aws iam create-policy \
    --policy-name ALBIngressControllerIAMPolicy \
    --policy-document https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/iam-policy.json
    
   
```

This return the arn of the role


```

eksctl create iamserviceaccount 
       --cluster {CLUSTER_NAME} 
       --namespace kube-system 
       --name alb-ingress-controller 
       --attach-policy-arn=arn:aws:iam::xxxx:policy/ALBIngressControllerIAMPolicy \
       --override-existing-serviceaccounts 
       --approve
       
curl -sS "https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/alb-ingress-controller.yaml" \
     | sed "s/# - --cluster-name=devCluster/- --cluster-name=attractive-gopher/g" \
     | kubectl apply -f -
     
     
```


To verify

```
kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o alb-ingress[a-zA-Z0-9-]+)

```


8. Installing kfctl


```


wget -O kfctl_aws.yaml https://raw.githubusercontent.com/kubeflow/manifests/v1.2-branch/kfdef/kfctl_aws.v1.2.0.yaml

```

Modify the file

In case a role should be used, it could be fetched like:


```

  | jq -r ".Roles[] \
  | select(.RoleName \
  | startswith(\"eksctl-$AWS_CLUSTER_NAME\") and contains(\"NodeInstanceRole\")) \
 .RoleName"

```

The region must be changed as well

8.1 Installing AWS-AUTHENTICATOR

```

curl -o aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/aws-iam-authenticator

hmod +x ./aws-iam-authenticator

mkdir -p $HOME/bin && cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$PATH:$HOME/bin

echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc


```

8.2 

```

kfctl apply -V -f kfctl_aws.yaml

```


Verify Kubeflow Installation 

```

kubectl -n kubeflow get all

```


9. Installing Knative

```

kubectl apply --filename https://github.com/knative/serving/releases/download/v0.18.0/serving-crds.yaml


kubectl apply --filename https://github.com/knative/serving/releases/download/v0.18.0/serving-core.yaml


```

10. Get Ingress gateway

```

kubectl --namespace istio-system get service istio-ingressgateway

```

you can use proxy the dashboard to localhost as well

```

kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80

```

We can use the default creds in kfctl_aws.yaml

admin@kubeflow.org
12341234

11. Kf Serving

```

mkdir app
cd app

```

Create flask app app.py

```

import os

from flask import Flask

app = Flask(__name__)

@app.route('/')
def predict():
   return 'Hello {}!\n'.format(target)

if __name__ == "__main__":
   app.run(debug=True,host='0.0.0.0',port=8080)
   
```


Creating the Dockerfile

```


FROM python:3.7-slim

# Allow statements and log messages to immediately appear in the Knative logs
ENV PYTHONUNBUFFERED True

# Copy local code to the container image.
ENV APP_HOME /app
WORKDIR $APP_HOME
COPY . ./

# Install production dependencies.
RUN pip install Flask gunicorn

# Run the web service on container startup. Here we use the gunicorn
# webserver, with one worker process and 8 threads.
# For environments with multiple CPU cores, increase the number of workers
# to be equal to the cores available.
CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 --timeout 0 app:app

```

Creating service.yaml

```
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: helloworld-python
  namespace: default
spec:
  template:
    spec:
      containers:
        - image: docker.io/{username}/hello-wolrd-predict
          env:
            - name: TARGET
              value: "Python Sample v1"
```


```

# Build the container 
docker build -t {username}/hello-wolrd-predict .

# Push the container to docker registry
docker push {username}/hello-wolrd-predict


kubectl apply --filename service.yaml

kubectl get ksvc hello-wolrd-predict  --output=custom-columns=NAME:.metadata.name,URL:.status.url


```


Temporary DNS configuration

```

curl -H "Host: http://hello-wolrd-predict.default.example.com" ip:port

```
