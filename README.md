# kubernetes-configuration

Kubernetes object configuration files for [Nelson][nelson].

## Architecture

* Nelson deployed inside the Kubernetes cluster it's manager - multi-cluster management is not included in this repo
    * Objects created will be namespaced under `nelson-system`
* Nelson deployed as a Deployment with 1 replica
    * Allows leveraging of Kubernetes features like reconcilliation - if the node Nelson is deployed on fails,
      Kubernetes will attempt to re-deploy
* Nelson UI served through a LoadBalancer + Ingress + Service
    * LoadBalancer will be environment-specific (e.g. OpenStack vs. GCE/GKE)
* RBAC for Nelson
    * Nelson gets permissions to touch deployments, jobs, etc.
    * Service owners interface with Nelson through `nelson` CLI, no need for per-owner/team RBAC

## Tutorial

### Requirements

This tutorial was tested on a Kubernetes 1.8.x cluster with LoadBalancer support and RBAC enabled. Older versions of
Kubernetes have not been tested, but LoadBalancer support and RBAC is definitely required.

Additionally, your GitHub (Enterprise) deployment must have line of sight to your LoadBalancer's external IP address,
and your Kubernetes cluster must have line of sight to your GitHub (Enterprise) deployment.

All objects we create will be done through `kubectl apply -f` - to undo a specific action you can run the corresponding
`kubectl delete -f`.

Finally while Nelson on Kubernetes is in a WIP mode, you will need to clone the repository and publish Nelson
internally. Here are some quick instructions, please refer to the Nelson [development documentation][devDocs]
for more information.

Make sure you are in your Nelson checkout and that Docker is running before starting.

```bash
$ sbt publishLocal
$ docker tag verizon/nelson:latest <YOUR REGISTRY ENDPOINT HERE>/verizon/nelson:latest
$ docker push <YOUR REGISTRY ENDPOINT HERE>/verizon/nelson:latest
```

### Namespace

Everything we create will be under the `nelson-system` namespace so to start off, let's create that.

```bash
$ kubectl apply -f namespace/nelson.yaml
```

### RBAC

Create the `nelson-system` RBAC objects.

```bash
$ kubectl apply -f serviceAccount/nelson.yaml
$ kubectl apply -f clusterRole/nelson.yaml
$ kubectl apply -f clusterRoleBinding/nelson.yaml
```

### Configuration

Read the Nelson [Operator Guide][nelsonOperatorGuide] and edit `configMap/nelson.yaml` according to the
instructions. Note that the `*.cfg` file is embedded in the ConfigMap object file, this is so we can
continuously run `kubectl apply -f configMap/nelson.yaml` to update the config. Kubernetes has support
for reading from a file *on creation* but seems to have trouble with updating from files. It's possible,
but just not as simple as `apply -f`, so ConfigMap.

After that's done, create it.

```bash
$ kubectl apply -f configMap/nelson.yaml
```

### Deploy Nelson

Before we create the Nelson Deployment we need to point to the image we published in the Requirements
section and setup Persistent Volumes for Nelson to store its
H2 database and logs. Open up `deployment/nelson.yaml` and edit it to configure the image location and
volumes for your deployment.

Once that is done we can create the Deployment. This is what will actually deploy (a single instance of)
Nelson in the cluster and open up port 9000 (the UI and API) and 5775 (monitoring) on the corresponding Pod.

```bash
$ kubectl apply -f deployment/nelson.yaml
```

Verify that the pod is up and running with

```bash
$ kubectl --namespace=nelson-system get pods
```

If it looks like there's an issue you can view the logs by running

```bash
$ kubectl --namespace=nelson-system logs <POD ID>
```

Add a `-f` to the end of that if you want to continuously watch it - this is useful for debugging, especially
when you want to verify GitHub is talking to Nelson, Nelson is deploying workflows, etc.

### Expose Nelson

Create the Service.

```bash
$ kubectl apply -f service/nelson.yaml
```

This gives it a stable in-cluster IP, but does not make it accessible outside the cluster which GitHub needs
to be able to do. To do this we create a corresponding Ingress object - this is where you need to have a
functioning LoadBalancer so inbound traffic gets routed to the correct service.

```bash
$ kubectl apply -f ingress/nelson.yaml
```

### Start playing!

At this point Nelson and it's UI should be reachable at your LoadBalancer's external IP address. From here
you should be able to follow Nelson's [User Guide][nelsonUserGuide] to try it out on your new deployment!

### Cleanup

To undo everything we did in this guide:

```bash
$ kubectl --namespacec=nelson-system delete configMap,deployment,ingress,service,serviceAccount nelson
$ kubectl delete namespace nelson-system
$ kubectl delete clusterRole nelson
$ kubectl delete clusterRoleBinding nelson
```

## Credits

This repository was largely inspired by Kelsey Hightower's [istio-ingress-tutorial][istioTutorial].

[devDocs]: https://verizon.github.io/nelson/development.html
[istioTutorial]: https://github.com/kelseyhightower/istio-ingress-tutorial
[nelson]: https://verizon.github.io/nelson/
[nelsonOperatorGuide]: https://verizon.github.io/nelson/#operator-guide
[nelsonUserGuide]: https://verizon.github.io/nelson/#user-guide
[persistentVolumes]: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
[rbac]: https://kubernetes.io/docs/admin/authorization/rbac/
