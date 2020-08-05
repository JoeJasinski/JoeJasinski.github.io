---
layout: post
title: "Istio: IstioOperator Install and Use"
date: 2020-08-02
categories: kubernetes istio
excerpt_separator: <!--more-->
---
# Istio Operator Basic Use

The Istio community has been in the process of changing its install procedure for a 
number of releases. As of Istio 1.6, the IstioOperator has graduated to 
[stable](https://discuss.istio.io/t/istios-helm-support-in-2020/5535/25), and is one
of the preferred install methods. The [documentation](https://istio.io/latest/docs/setup/install/standalone-operator/) for
the IstioOperator left me with a lot of questions, so I am writing here to try to brain
dump what I have found out about how it works.
<!--more-->

## Install methods

### IstioOperator Install

The IstioOperator is a Custom Resource Definition that 
lets you define and install all of the Istio components that you want in 
your cluster. So say you want to install istiod, istio-ingressgateway, istio-egressgateway,
and Graphana. You can define those selections as part of an IstioOperator, apply it
with kubectl, and Istio will take care of installing the Istio components as defined in
the operator.

To install the IstioOperator, you can run the following command. After a few seconds,
this will install the IstioOperator CRD and install the `istio-operator` controller pod
inside of a new `istio-operator` namespace. This controller will look for any IstioOperator
resources and will install Istio if it finds one.

{% highlight bash linenos %}
istioctl operator init
{% endhighlight %}


After installing the CRD and controller, you can define and apply an IstioOperator
Custom Resource (CR) to specify exactly what Istio components you want installed. 

### istioctl install command

Istio also provides users another installation method: the `istioctl install` command.
This command also takes care of installing and configuring Istio. Behind the
scenes, `istioctl install` creates an instance of an IstioOperator.
(You can see it with `kubectl get  istiooperator -n istio-system`.) Perhaps this operator
definition is used by istioctl as an intermediary definition to specify should be installed; 
however, as far as I can tell, changes to that IstioOperator resource after installation
are ignored.

To install Istio using this approach, simply run the following command. This will install
istio into the `istio-system` namespace. 

{% highlight bash linenos %}
istioctl install
{% endhighlight %}

### Helm install method (Removed)

Istio used to support a Helm installation method, but Istio has deprecated that method in
Istio 1.5 and has removed it in 1.6. (You can install the IstioOperator CRD
and controller with a Helm chart, but the Helm charts that allow you to install all of the
Istio resources has been removed.) Clearly, Istio is moving away from Helm.

## The case for using the IstioOperator install method

While the `istioctl install` method is simple and allows you to customize the installation
with the `--set` flag, I prefer the IstioOperator install method. Defining an 
IstioOperator Custom Resource lets you be very explicit as to what you want installed in
a declarative and concise way. 

Once you define the Istio component state that you want in the
form of an IstioOperator CR, Istio and Kubernetes will do everything they need to do to 
instantiate that declared state. Also, if you make changes to that IstioOperator CR, 
Istio and Kubernetes will change the installed resources to match the IstioOperator state.
To uninstall Istio, all you need to do is delete the IstioOperator CR.

This article will focus on the IstioOperator install method.

## Simple Usage

First I want to demo the simple use-case from the docs to demo the simplest use case.

Create the istio-system namespace. This is where Istio will be installed when the custom
resource is created.

{% highlight bash linenos %}
kubectl create ns istio-system
{% endhighlight %}

Then create and apply the yaml.

{% highlight yaml linenos %}
# filename: istio-simple.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: simple-istio
spec:
  profile: default
{% endhighlight %}

{% highlight bash linenos %}
kubectl apply -f istio-system.yaml
{% endhighlight %}

Once you apply this resource, it will take several minutes for the controller to recognize
and start installing all of the Istio components. Try running `kubectl get po -n istio-system`
to monitor the installation progress.

If you do not see any change, tail the logs in the operator pod.

{% highlight bash linenos %}
kubectl logs -f -n istio-operator -l name=istio-operator
{% endhighlight %}

## Configuring the operator

The IstioOperator can be configured according to the following 
[API](https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/#IstioOperatorSpec)
documentation. This documentation helps you to configure things such as what Istio 
plugins and components to install as well as the, k8s resources, autoscaling policies,
labels and annotations, and lots of other properties associated with those components. 

For example:

{% highlight yaml linenos %}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istio
spec:
  components:
    ingressGateways:
    - name: istio-ingressgateway
      namespace: istio-system
      # set labels on the ingress gateway pods
      label:
        app: ingress-gateway
      # enable the ingress gateway
      enabled: True
      k8s:
        # Set the HPA minimum and maximum replicas
        hpaSpec:
          minReplicas: 1
          maxReplicas: 5
        resources:
          # define the min and max size of the ingress gateway pods
          requests:
            memory: "100M"
            cpu: "100m"
          limits:
            memory: "4G"
            cpu: "2"
    egressGateways:
    - name: egress-gateway
      namespace: istio-system
      # set labels on the egress gateway pods
      label:
        app: egress-gateway
      # enable the egress gateway
      enabled: True
  addonComponents:
    # enable the grafana addon
    grafana:
      enabled: true
{% endhighlight %}

However, the api documentation does not detail how to configure individual settings 
specific to given components.

To do that, you can use the Helm passthrough API and define attributes under the values
attribute. The available values that you can set can be found in the 
[manifests/charts](https://github.com/istio/istio/tree/1.6.7/manifests/charts) directory
of the source code. Checkout the values file [here](https://github.com/istio/istio/blob/1.6.7/manifests/charts/global.yaml).


{% highlight yaml linenos %}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istio
spec:
  ...
  values:
    ...
{% endhighlight %}

## Examples

You can find some examples of IstioOperators [here](https://github.com/istio/istio/tree/c71f6b45ca24e5d5aec01e7f5685245955f1c5b0/operator/cmd/mesh/testdata/manifest-generate/data-snapshot/profiles).

Here is a full example.


{% highlight yaml linenos %}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istio
spec:
  values:
    sidecarInjectorWebhook:
      rewriteAppHTTPProbe: true
    global:
      logging:
        level: default:info
      proxy:
        # Configures the access log for each sidecar.
        # Options:
        #   "" - disables access log
        #   "/dev/stdout" - enables access log
        accessLogFile: ""
        logLevel: ""
        #componentLogLevel: "lua:info"
        # With autoinject disabled, you need to apply sidecar.istio.io/inject: true
        # annotation to each pod AND label the namespace with istio-injection=enabled
        autoInject: enabled
  components:
    pilot:
      enabled: True
    ingressGateways:
    - name: istio-ingressgateway
      namespace: istio-system
      label:
        app: ingress-gateway
      enabled: True
      k8s:
        hpaSpec:
          minReplicas: 1
          maxReplicas: 5
        resources:
          requests:
            memory: "100M"
            cpu: "100m"
          limits:
            memory: "4G"
            cpu: "2"
    egressGateways:
    - name: egress-gateway
      namespace: istio-system
      label:
        app: egress-gateway
      enabled: True
      k8s:
        hpaSpec:
          minReplicas: 1
          maxReplicas: 5
        resources:
          requests:
            memory: "100M"
            cpu: "100m"
          limits:
            memory: "4G"
            cpu: "2"
    sidecarInjector:
      enabled: True
    citadel:
      enabled: True
  addonComponents:
    grafana:
      enabled: true
      k8s:
        imagePullPolicy: Always
        # nodeSelector:
        replicaCount: 1
        resources:
          limits:
            cpu: '1'
            memory: '1G'
          requests:
            cpu: '100m'
            memory: '100M'
{% endhighlight %}

## Conclusion

I'm still learning about how the IstioOperator works, but this is what I have gathered
so far. 
