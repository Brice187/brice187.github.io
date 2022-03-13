+++
categories = ["cloud-native"]
comments = false
date = "2020-09-07T4:59:49+01:00"
draft = false
showpagemeta = false
showcomments = true
slug = ""
tags = ["k3s", "docker", "local"]
title = "Push local docker images to your (air-gapped) k3s cluster"
description = "A post about the amazingness of Gavin Belson"

+++

I am tipping my toe in the landscape of Kubernetes and found the lightweight, production-ready (and opionated)
k3s distribution is the perfect fit for my on-premises setup. Consider this as an experimental guide and use
it at your own risk (as you might know when using arbitrary code from the internet :)).

I found online only tutorials for running locally developed images without a container registry for minikube,
which can not be adapted to k3s or any other kubernetes distribution. I think this itches many developers
and there are no online ressources out there, so I decided to write my first technical blog post.

## Introduction

I ran into the problem that my locally developed docker images which were manually transfered to the desired host,
were ignored by the node running the kubelet. I found out that even with `imagePullPolicy:Never` in the deployment
manifest, the docker images where ignored because k3s uses containerd as container runtime.

At first, I thought that k3s only works with images downloaded from a public or private container registry
by the k3s node. But I found a practical solution in the next chapter. Let me first give you some context
about the IT environment:

Our network is separated into multiple segments. The k3s cluster serving our web applications is located in
the demilitarized zone with no access to our container registry (the single source of truth regarding our container images)
in the development segment. Digging a hole in the packetfilter to access the internal network from the DMZ was
never an option. To avoid running an other infrastructure component like a registry in the DMZ or pay for
a private cloud registry (with possible security implications of public facing services serving the source of the business logic), I had to find an other approach to deploy containerd images on our webservers.

## Using local images

Make sure your deployment manifest has the following pull policy:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my_app-deployment
  labels:
    app: my_app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my_app
  template:
    metadata:
      labels:
        app: my_app
    spec:
      containers:
      - name: my_app
        image: my_app:latest
        imagePullPolicy: Never
```

k3s ships with its own containerd runtime, which opens a socket at `/run/k3s/containerd/containerd.sock`.
If you are running k3s on your local workstation, you can simply import your local docker image (called
my_app for example) into containerd runtime with the following command `docker save my_app:latest | ctr -a /run/k3s/containerd/containerd.sock -n=k8s.io images import`

It is important to use the namespace **k8s.io**, because the kubelet will only use images from this namespace.

So running a local developed application on the development workstation seems not a big deal, but consider the
next chapter to provision your whole cluster.

{{< figure src="/img/container.jpg"
    width="750"
    margin="auto"
>}}
### Deploy local images to hosts in your network<

Make sure that you have ssh access to the k3s hosts from the workstation or dev-server where you build the image.

Now create a system user/group called `k3s` for the image deployment on every kubelet node (or ask your sysadmin nicely).
Make sure you put your public ssh key into the `/home/k3s/.ssh/authorized_keys` and do a
`chown root:k3s /run/k3s/containerd/containerd.sock` on every node, so you can control the containerd runtimer
with the system user.

The last step is to extend the command of the previous chapter: `docker save my_app:latest | ssh k3s@node1 'ctr -a /run/k3s/containerd/containerd.sock -n=k8s.io images import -'`
Now the kubelet node will use this manually deployed image instead of trying to pull the image from hub.docker.com or running into famous `ImagePullBackOff`.

## Bonus: Automating image deployment with Gitlab CI/CD

So this is pretty cool already, right? But how about automatic deployment with a CI/CD pipeline.
Make sure your gitlab_runner has the correct permission to access the k3s system user on the kubelet
nodes over ssh. Now you can add this deployment stage to your `.gitlab-ci.yml` in your software repository:

```yaml
deploy_app:
  stage: deploy
  tags:
    - shell # Tag for using a definded gitlab runner config
  script:
    - export IMAGE_TIMESTAMP=`date +%Y-%m-%d_%H-%M`
    # Get standard images from Docker Hub
    - docker image pull mongo:4.2.3-bionic
    # Build frontend app
    - docker build -t my_app src
    #### Optional: Push app to development registry myregistry.local
    - docker tag my_app myregistry.local/lk012/my_app:$IMAGE_TIMESTAMP
    - docker push myregistry.local/lk012/my_app:$IMAGE_TIMESTAMP
      for host in node1 node2 node3; do
        echo \"#### Provisioning $host... ####\"
        docker save mongo:4.2.3-bionic | ssh k3s@$host 'ctr -a /run/k3s/containerd/containerd.sock -n=k8s.io images import -'
        docker save my_app:latest | ssh k3s@$host 'ctr -a /run/k3s/containerd/containerd.sock -n=k8s.io images import -'
        echo \"##### Successfully provisioned $host... ####\"
      done
```
## Conclusion

In this post we have learned how to get rid of local container registries at all. As you have mentioned in the gitlab manifest,
it is also possible to push standard images from the official docker hub to your kubelet nodes in the same manner. Feel free to try it out.

### Update

Just watched the [TALK](https://www.youtube.com/watch?v=aR12Oij4CYw) of Darren Shepherd, the Founder of **k3s**,
at Kubecon Europe and learned about the capability to deploy docker image tarballs to `${k3s}/images/app` (look at the skipped slide about pipelines at 26:26). Nevertheless, my method takes fewer steps to deploy an image, so may be this post has still a small take away for you ;)

If anything in this article is unclear or you want to add some useful informations, write a comment or just mention me on [Twitter](https://twitter.com/lk012)