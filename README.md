# Kubernetes Examples

This respository is for demonstrating the very basics of manually setting up a kubernetet cluster with vagrant.

## Prereqesuites

You need to have packer and vagrant installed together with virtualbox.

Find packer here: https://packer.io
Find vagrant here: https://vagrant.io

To manage the cluster and deploy services, you will need to have kubectl and helm on your local machine.

Find how to install kubectl here: https://v1-18.docs.kubernetes.io/docs/tasks/tools/install-kubectl/
Find helm here: https://helm.sh/


## Build vagrant box

A packer file has been prepared to build a vagrant box with all everything necessary packages install for running kubernetes:

```bash
$ packer build kubernetes_box.pk
```

## Bring up virtual machines

Now when you have built the vagrant box, you are ready to bring the virtual machines up that will form the kubernetes cluster.

```bash
$ vagrant up
```

## Create kubernetes cluster

Now its time to create the kubernetes cluster with flannel as CNI (container network interface) for seamless communication between containers in all nodes in the cluster.

Use vagrant to ssh to the first virtual machine:
```bash
$ vagrant ssh node-1
```

Figure out your public ip adress with the ip command:
```bash
$ ip addr ip -4 addr show enp0s8 | grep -oP '(?<=inet\s)\d+(\.\d+){3}'
```

In this example I use 192.168.100.2 as advertise address.

Then create the cluster:
```bash
$ sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-advertise-address 192.168.100.2
```

--pod-network-cidr is to define which network cidr flannel should use.
--apiserver-advertise-address is to tell on which address the api server should listen for incoming connections from other nodes.

Now export the kubeonfig to your local machine by:
```bash
$ vagrant ssh node-1 -c 'sudo cat /etc/kubernetes/admin.conf' > ~/.kube/config
```

*Please backup any existing config file before you do this. It is possible to merge multiple kube configurations but I will not bring it up in this demo.*

From now on, you can run all kubectl from your local machine.

Now install flannel:
```bash
$ kubectl apply -f kube-flannel.yml
```

Now you are ready to add nodes to your cluster.

For all other virtul machines (node-2, node-3, node4...) join the cluster with the command that kubeadm init gives you on the first machine. It is important that you copy the command from your first machine as the token is unique for each installation.

```bash
$ sudo kubeadm join 192.168.100.2:6443 --token yua7q1.4b6jqauh3wxkppad \
    --discovery-token-ca-cert-hash sha256:82a6824e0a87cecb3f9bad060ce66f522908e036cfa8d066012a4be6534e152f
```

Now you can monitor the node status to see when your cluster is ready.

```bash
$ kubectl get nodes
```

When the cluster is ready, all nodes should have the status _Ready_. Now you are ready to deploy a service.

## Deploy the dashboard

Deploying the kubernetes dashboard is a good example for showing how deployments are done.

```bash
$ helm install kubernetes-example-dashboard kubernetes-dashboard/kubernetes-dashboard
```

You can monitor the status of the pods by

```bash
$ kubectl get pods
```

And you should see something like this when the service is up and running:
```
NAME                                                              READY   STATUS    RESTARTS   AGE
kubernetes-example-dashboard-kubernetes-dashboard-9d58fc7b5g4kp   1/1     Running   0          2d18h
```

In order to reach the service we can port-forward to it.

```bash
$ kubectl -n default port-forward $POD_NAME 8443:8443
```

Now you can reach the dashboard by browsing to http://127.0.0.1:8443 . But that is of course not good enough for handling external traffic to your services.

In order to be able to login to the dashboard, you will need a secret token. Dashboard is using the deployment-controller-token secret which you will find this way:

```bash
$ kubectl -n kube-system get secret
```

Show your secret token by

```bash
$ kubectl -n kube-system describe secret deployment-controller-token-vpjk6
```

## Setting up Ingress

Install the ingress-nginx controller (our virtualbox machines counts as bare-metal):

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.43.0/deploy/static/provider/baremetal/deploy.yaml
```

Then wait until the nginx controller is running.

```bash
$ kubectl get pods -n ingress-nginx \
  -l app.kubernetes.io/name=ingress-nginx --watch
```

Then we need to create an ingress

```bash
$ kubectl apply -f ingress.yaml 
```

## Adding example services

In the example-services directory are two examples services together with ingress.

Install the services

```bash
$ kubecctl apply -f appple.yml
$ kubectl apply -f banana.yml
```

Now install an ingress for these services
```bash
$ kubectl apply -f ingess.yml
```

Now you need to wait for the ingress to get an address...

```bash
$ kubectl get ingress --watch
NAME                   CLASS   HOSTS   ADDRESS         PORTS   AGE
apple-banana-ingress   nginx   *       192.168.10.84   80      20m
```

Now try to curl to the specified address:

```bash
$ curl http://192.168.10.84/apple
apple
$ curl http://192.168.10.84/banana
banana
```
