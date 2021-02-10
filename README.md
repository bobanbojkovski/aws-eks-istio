# AWS EKS &amp; Istio service mesh

Motivation of the following sample is to setup Istio on EKS cluster and work with circuit breaker, internal retries, controlled rollouts features.

Thank you IBM, [O’Reilly ebook, Istio Explained](https://www.ibm.com/downloads/cas/XWN1WV9Q)


### Install required tools 

Here we use [eksctl](https://eksctl.io/) to manage the AWS EKS, [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to work with the Kubernetes, [helm](https://helm.sh/docs/intro/install/) to manage the Kubernetes applications.


### Create AWS EKS Cluster

Note: VPC has been previously configured.

We use eksctl and cluster definition in yaml file to setup cluster.  
Sample file, eks-sample.yaml.

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: <cluster>
  region: <region>
  version: "1.18"

vpc:
  cidr: <cidr>
  id: <vpc-id>
  subnets:
    private:
      private-<region>a:
        id: <subnet-private-<region>a>
      private-<region>b:
        id: <subnet-private-<region>b>
      private-<region>c:
        id: <subnet-private-<region>c>
    public:
      public-<region>a:
        id: <subnet-public-<region>a>
      public-<region>b:
        id: <subnet-public-<region>b>
      public-<region>c:
        id: <subnet-public-<region>c>

nodeGroups:
  - name: <nodegroup>
    instanceType: "t3.medium"
    minSize: 0
    maxSize: 3
    desiredCapacity: 2
    disableIMDSv1: true
    privateNetworking: true
    iam:
      withAddonPolicies:
        autoScaler: true
```

Note: 
For test purpose we use t3.medium, it is recommended to use bigger instance types, to have for example 8GB of memory and 4 CPUs free in the cluster.

Create the eks cluster.
```
AWS_PROFILE=<aws-profile> eksctl create cluster -f eks-sample.yaml
```


### Install istioctl cli

[Getting Started](https://istio.io/latest/docs/setup/getting-started/)

Download latest Istio package.
```
curl -L https://istio.io/downloadIstio | sh -
```

Move to the Istio package directory & inspect the content.
```
cd istio-<version>
```

Add the istioctl client to your path.
```
echo "export PATH=$PWD/bin:$PATH" >> ~/.bash_profile
or
echo "export PATH=$PWD/bin:$PATH" >> ~/.zshrc
```
Reopen the shell, to load the updated PATH setup.


### Install istio components using config profiles

[Installation Configuration Profiles](https://istio.io/latest/docs/setup/additional-setup/config-profiles/)

istioctl profile list
```
Istio configuration profiles:
    default
    demo
    empty
    minimal
    openshift
    preview
    remote
```

Install demo profile. Different approaches are available to install the components.  
For example.
```
istioctl profile dump demo > demo.yaml

AWS_PROFILE=<aws-profile> istioctl install --set profile=demo
or
AWS_PROFILE=<aws-profile> istioctl install -f demo.yaml

This will install the Istio demo profile with ["Istio core" "Istiod" "Ingress gateways" "Egress gateways"] components into the cluster. Proceed? (y/N) y
✔ Istio core installed
✔ Istiod installed
✔ Egress gateways installed
✔ Ingress gateways installed
✔ Installation complete
```

List the installed components.
```
AWS_PROFILE=<aws-profile> kubectl get pod,svc -n istio-system
```

Analyze Kubernetes cluster.
```
AWS_PROFILE=<aws-profile> istioctl analyze --all-namespaces

Info [IST0102] (Namespace default) The namespace is not enabled for Istio injection. Run 'kubectl label namespace default istio-injection=enabled' to enable it, or 'kubectl label namespace default istio-injection=disabled' to explicitly mark it as not needing injection.
```

We'll use namespace `demo` instead of `default` for the sample application.  
Define the namespace.
```
cat << EOF > namespace-demo.yaml
---
kind: Namespace
apiVersion: v1
metadata:
  name: demo
  labels:
    name: demo
EOF
```

Create namespace `demo`.
```
AWS_PROFILE=<aws-profile> kubectl apply -f namespace-demo.yaml
namespace/demo created
```

Enable Istio Sidecar Injection.  
Istio uses a namespace label to inject the Envoy proxy sidecar.

Describe the created namespace `demo`.
```
AWS_PROFILE=<aws-profile> kubectl describe ns demo
```

Add a namespace label.
```
AWS_PROFILE=<aws-profile> kubectl label namespace demo istio-injection=enabled
namespace/demo labeled

AWS_PROFILE=<aws-profile> kubectl get namespaces --show-labels
```

Run istio analyze for the `demo` namespace.
```
AWS_PROFILE=<aws-profile> istioctl analyze --namespace demo
✔ No validation issues found when analyzing namespace: demo.
```

#### Install [addon components](https://istio.io/latest/docs/ops/integrations/)

Basic installation of observability tools, Kiali, Jaeger, Grafana, Prometheus, Zipkin.

Kiali

In 1.8 version errors are introduced. We first upgrade kiali image from version 1.26.0 to 1.29.0.

Download the sample kiali.yaml
```
wget https://raw.githubusercontent.com/istio/istio/release-<version>/samples/addons/kiali.yaml
```
then replace the image version to 1.29.0 in kiali.yaml, and apply the configuration.
```
AWS_PROFILE=<aws-profile> kubectl apply -f kiali.yaml
customresourcedefinition.apiextensions.k8s.io/monitoringdashboards.monitoring.kiali.io unchanged
serviceaccount/kiali unchanged
configmap/kiali unchanged
clusterrole.rbac.authorization.k8s.io/kiali-viewer unchanged
clusterrole.rbac.authorization.k8s.io/kiali unchanged
clusterrolebinding.rbac.authorization.k8s.io/kiali unchanged
service/kiali unchanged
deployment.apps/kiali unchanged
monitoringdashboard.monitoring.kiali.io/envoy created
monitoringdashboard.monitoring.kiali.io/go created
monitoringdashboard.monitoring.kiali.io/kiali created
monitoringdashboard.monitoring.kiali.io/micrometer-1.0.6-jvm-pool created
monitoringdashboard.monitoring.kiali.io/micrometer-1.0.6-jvm created
monitoringdashboard.monitoring.kiali.io/micrometer-1.1-jvm created
monitoringdashboard.monitoring.kiali.io/microprofile-1.1 created
monitoringdashboard.monitoring.kiali.io/microprofile-x.y created
monitoringdashboard.monitoring.kiali.io/nodejs created
monitoringdashboard.monitoring.kiali.io/quarkus created
monitoringdashboard.monitoring.kiali.io/springboot-jvm-pool created
monitoringdashboard.monitoring.kiali.io/springboot-jvm created
monitoringdashboard.monitoring.kiali.io/springboot-tomcat created
monitoringdashboard.monitoring.kiali.io/thorntail created
monitoringdashboard.monitoring.kiali.io/tomcat created
monitoringdashboard.monitoring.kiali.io/vertx-client created
monitoringdashboard.monitoring.kiali.io/vertx-eventbus created
monitoringdashboard.monitoring.kiali.io/vertx-jvm created
monitoringdashboard.monitoring.kiali.io/vertx-pool created
monitoringdashboard.monitoring.kiali.io/vertx-server created
```
Note: Run the command 2 times.

Jaeger
```
wget https://raw.githubusercontent.com/istio/istio/release-<version>/samples/addons/jaeger.yaml
```
Update the image version, all-in-one:1.20 to all-in-one:1.21
```
AWS_PROFILE=<aws-profile> kubectl apply -f jaeger.yaml
deployment.apps/jaeger created
service/tracing created
service/zipkin created
service/jaeger-collector created
```

Grafana
```
AWS_PROFILE=<aws-profile> kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-<version>/samples/addons/grafana.yaml
serviceaccount/grafana created
configmap/grafana created
service/grafana created
deployment.apps/grafana created
configmap/istio-grafana-dashboards created
configmap/istio-services-grafana-dashboards created
```

Prometheus
```
AWS_PROFILE=<aws-profile> kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-<version>/samples/addons/prometheus.yaml
serviceaccount/prometheus created
configmap/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
service/prometheus created
deployment.apps/prometheus created
```

Zipkin
```
AWS_PROFILE=<aws-profile> kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-<version>/samples/addons/extras/zipkin.yaml
deployment.apps/zipkin created
service/tracing configured
service/zipkin configured
```

List the current pods and services.
```
AWS_PROFILE=<aws-profile> kubectl get pod,svc -n istio-system
```


#### Istio is language agnostic, install sample application written in different languages.  
[Bookinfo App](https://istio.io/latest/docs/examples/bookinfo/)

```
AWS_PROFILE=<aws-profile> kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -n demo
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created
```

Istio creates classic elb.

Check whether the gateway has been created, otherwise use bookinfo-gateway.yaml to create L4/6 & L7 routes.
```
AWS_PROFILE=<aws-profile> kubectl get gateway -n demo
AWS_PROFILE=<aws-profile> kubectl apply -f  samples/bookinfo/networking/bookinfo-gateway.yaml -n demo
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
```

Get the ingress ip, elb dns.
```
AWS_PROFILE=<aws-profile> kubectl get service istio-ingressgateway -n istio-system
```

Use the EXTERNAL-IP, elb dns and check the bookinfo site.  
Access the bookinfo site.
```
http://<EXTERNAL-IP>/productpage
```

Open Kiali UI.
```
AWS_PROFILE=<aws-profile> istioctl dashboard kiali
```


### AWS Load Balancer Controller setup

Configure AWS LB Controller, to expose the app to external traffic.  
[Introducing the AWS Load Balancer Controller](https://aws.amazon.com/about-aws/whats-new/2020/10/introducing-aws-load-balancer-controller/)

Create IAM OIDC provider.
```
AWS_PROFILE=<aws-profile> eksctl utils associate-iam-oidc-provider \
  --region <region> \
  --cluster <cluster> \
  --approve
```

Download IAM policy for the AWS Load Balancer Controller.
```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/install/iam_policy.json
```

Create an IAM policy, AWSLoadBalancerControllerIAMPolicy.
```
AWS_PROFILE=<aws-profile> aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
```

Create a IAM role and ServiceAccount for the AWS Load Balancer controller.
```
AWS_PROFILE=<aws-profile> eksctl create iamserviceaccount \
  --region <region> \
  --cluster=<cluster> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

Add the controller to EKS cluster.
```
helm repo add eks https://aws.github.io/eks-charts
AWS_PROFILE=<aws-profile> kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
customresourcedefinition.apiextensions.k8s.io/targetgroupbindings.elbv2.k8s.aws created
```

Install aws load balancer controller
```
AWS_PROFILE=<aws-profile> helm upgrade -i aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set image.repository=amazon/aws-alb-ingress-controller \
  --set clusterName=<cluster> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<vpc-id>
```


### External DNS

Note: In our case Route53 hosted zone has been previously configured.

External DNS provisions hosted zones and DNS records.

[Setup External DNS](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.1/guide/integrations/external_dns/)  
[Setting up ExternalDNS for Services on AWS](https://github.com/kubernetes-incubator/external-dns/blob/master/docs/tutorials/aws.md#iam-permissions)

Create IAM Policy to allow ExternalDNS to update Route53 configurations.  
Sample IAM Policy, external-dns-policy.json
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

Create the IAM policy.
```
AWS_PROFILE=<aws-profile> aws iam create-policy \
  --policy-name AWSExternalDNSIAMPolicy \
  --policy-document file://external-dns-policy.json
```

Create IAM Role, Kubernetes Service Account & Associate IAM Policy.
```
AWS_PROFILE=<aws-profile> eksctl create iamserviceaccount \
  --region <region> \
  --cluster <cluster> \
  --namespace kube-system \
  --name aws-external-dns \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/AWSExternalDNSIAMPolicy \
  --approve 
```

[Install external DNS](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/istio.md)

Specify `- --domain-filter` value in the external-dns-istio.yaml file.
```
args:
  - --source=service
  - --source=ingress
  - --source=istio-gateway
  - --source=istio-virtualservice
  - --domain-filter=example.com # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
  - --provider=aws
  - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
  - --aws-zone-type=public # only look at public hosted zones (valid values are public, private or no value for both)
  - --registry=txt
  - --txt-owner-id=my-identifier
```

Change namespace from `default` to `kube-system`
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: kube-system
```

Deploy the external-dns configuration.
```
AWS_PROFILE=<aws-profile> kubectl apply -f external-dns-istio.yaml -n kube-system
serviceaccount/external-dns created
clusterrole.rbac.authorization.k8s.io/external-dns created
clusterrolebinding.rbac.authorization.k8s.io/external-dns-viewer created
deployment.apps/external-dns created
```

Attach AWSExternalDNSIAMPolicy policy to NodeInstanceRole role.
```
AWS_PROFILE=<aws-profile> aws iam attach-role-policy \
  --role-name eksctl-<cluster>-<nodegroup>-NodeInstanceRole-<id> \
  --policy-arn arn:aws:iam::<account-id>:policy/AWSExternalDNSIAMPolicy
```

Update the istio gateway and virtual service hosts values.  
In samples/bookinfo/networking/bookinfo-gateway.yaml add host domain.
```
hosts:
  - "bookinfo.example.com"
```

Apply the changes.
```
AWS_PROFILE=<aws-profile> kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml -n demo
gateway.networking.istio.io/bookinfo-gateway configured
virtualservice.networking.istio.io/bookinfo unchanged
```

Verify in Route53 the records have been created.  
Access the bookinfo site using the url `http://bookinfo.example.com/productpage`

#### Create AWS Network ELB

Generate istio `demo` profile configuration file.
```
istioctl profile dump demo > demo.yaml
```

Update the demo.yaml adding serviceAnnotations to ingressGateways component.
```
serviceAnnotations:
  proxy.istio.io/config: '{"gatewayTopology" : { "numTrustedProxies": 2 } }'
  service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: '*'
  service.beta.kubernetes.io/aws-load-balancer-type: nlb
```

Apply the changes.
```
AWS_PROFILE=<aws-profile> istioctl install -f demo.yaml
This will install the Istio demo profile with ["Istio core" "Istiod" "Ingress gateways" "Egress gateways"] components into the cluster. Proceed? (y/N) y
✔ Istio core installed
✔ Istiod installed
✔ Egress gateways installed
✔ Ingress gateways installed
✔ Installation complete
```

#### Proceed with Traffic Management

[Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)

