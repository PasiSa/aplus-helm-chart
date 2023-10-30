# Deploying A+ in a fresh Kubernetes environment

This document describes the steps taken to install A+ using the helm charts in
this repository in a fresh Kubernetes setup. It first describes how a small
[MicroK8s](https://microk8s.io/) Kubernetes environment of three nodes was
installed on three Ubuntu 22.04 Linux machines, and in the second part, how A+
was set up on the Kubernetes environment using the helm charts. If you already
have a running Kubernetes environment available, you can skip the first part.

This document works as a journal of what the undersigned did, with not so much
Kubernetes experience, and there may be better ways to do some of these steps --
comments and suggestions for improvements are very welcome! The document is not
designed to be a comprehensive manual on the related matters, but hopefully it is
still useful.

The following steps took about few hours, including quite much googling and
studying Kubernetes basics from different sources. Doing everything again would
be faster.

## Setting up MicroK8s environment

Three Ubuntu 22.04 machines were set up for this exercise. The goal was to
experiment with setting up A+ in a small, yet usable server environment for
small-scale use. Canonical's MicroK8s was selected, because it was easy to set
up (at least on Ubuntu machines), but seems extensible and scalable also for
more serious use.

### First steps

Following the [Getting started
instructions](https://microk8s.io/docs/getting-started) were quite straight
forward. I.e., first installing microK8s using snap on each of the three nodes:

    $ sudo snap install microk8s --classic --channel=1.28
    $ sudo usermod -a -G microk8s $USER
    $ sudo chown -f -R $USER ~/.kube
    $ su - $USER

Then some initial add-ons are installed on all of the three nodes:

    microk8s enable dns
    microk8s enable helm
    microk8s enable ingress

DNS and Helm might be installed by default, but just to be sure about it, the
above commands (even if redundant) won't harm. Adding also the ingress
controller add-on to all three machines does not harm, and could in fact be
useful in some cases for redundancy. More information about the available
add-ons can be found from [MicroK8s web site](https://microk8s.io/docs/addons).

After these steps Kubernetes could be used as a single-node entity, e.g. for
testing and development needs, but we want to connect the three nodes into a
cluster.

### Building the cluster

Again the [instructions at the MicroK8s web
site](https://microk8s.io/docs/clustering) are good and easy to follow, but to
summarize, here is a quick recap. First we pick the cluster master node and tell
it about the matter:

    microk8s add-node

In return, some follow-up instructions are provided on the terminal.
Particularly, there is a token that allows other nodes to join the cluster. The
token is for single use only: if several nodes are attached, a new token is
needed for each, by repeating the above command.

After this the other two nodes are connected to the master. The below IP address
and token are not real, but instead you should copy them from the output that
resulted from the above command.

    microk8s join 192.168.1.230:25000/92b2db237428470dc4fcfc4ebbd9dc81/2c0cb3284b05

Note that Kubernetes uses specific ports for communication within the cluster.
You may need to check whether the default firewall rules allow this, and if
needed do the necessary modifications in the firewall rules.

To add another node, you would call `microk8s add-node` again at the master to
get a fresh token and repeat above on the other machine. Note that the
instructions also show the option to add node to cluster as worker (with
additional `--worker` flag), but here we wanted to enable the high availability
features, i.e. making all three nodes able to support the control plane features.

In MicroK8s examples kubectl commands are used through the microk8s command,
e.g.: `microk8s kubectl get nodes`, but for convenience, an alias is useful:

    alias kubectl='microk8s kubectl'

Usually you would also want to operate kubectl from your local machine outside the
cluster. In local configuration the Kubernetes config file is needed, and it can
be generated in the following way (selecting the target file as appropriate in
your case):

    microk8s config >~/.kube/config

This config file can then be copied to client machines, usually in
`~/.kube/config`. How kubectl is installed in different systems, varies per
system. In Windows 10 host with WSL2 installed, kubectl for WSL2 comes as
bundled with Docker Desktop. Note that the firewall rules may prevent direct
kubectl access from outside the cluster's network domain.

### Setting up NFS

We follow quite closely the [MicroK8s
documentation](https://microk8s.io/docs/nfs) regarding NFS setup. An existing
NFS server was available for our use, but if you need to set up one by yourself,
there are instructions for that in the aforementioned documentation. The
MicroK8s instructions were followed to set up a NFS-CSI driver:

    microk8s helm3 repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
    microk8s helm3 install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
        --namespace kube-system \
        --set kubeletDir=/var/snap/microk8s/common/var/lib/kubelet
    microk8s kubectl wait pod --selector app.kubernetes.io/name=csi-driver-nfs --for condition=ready --namespace kube-system

Then, further following the instructions, NFS-CSI storage class was created, but
few modifications were done to the yaml configuration. Our version was
(obviously, you will need to set the server address and share properly for your
environment):

    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
        name: nfs-csi
    provisioner: nfs.csi.k8s.io
    parameters:
        server: nfs.server.com
        share: /share/path
        subDir: ${pvc.metadata.name}
    reclaimPolicy: Retain
    volumeBindingMode: Immediate
    mountOptions:
    - hard
    - nfsvers=4.2

`reclaimPolicy` was set to "*Retain*", so that the volumes persist if the
cluster is shut down. By default NFS-CSI generates seemingly random names for
the subdirectories in the NFS share when the cluster is started, but with the
given `subDir` setting the subdirectories are named according to the PVC names
set in the values file (for example, "aplusMedia", "gitmanagerBuild", and so
on). This is necessary in addition to the reclaimPolicy setting so that the data
remains usable if cluster is shut down and later restarted.

This configuration can then be applied in the Kubernetes cluster:

    kubectl apply -f - < sc-nfs.yaml


## Deploying A+ from Helm charts

### Setting up the local Helm configuration

A local *values.yaml* file was created for local configuration. Fairly small
changes are needed compared to example to the example given in the repository:
primarily, the hostnames need to be configured appropriately. Note that the
names need to be available also in the DNS system so that requests from the
world can find their way to the cluster. The DNS records should be configured to
point to the cluster ingress IP address, that will then pass the traffic further
inside the cluster to a correct service. (*It seems that MicroK8s supports
multiple ingresses, and as long as the traffic reaches any of them, it is
directed to a correct service. Something to be tried later: DNS records could be
configured to contain all alternative ingress IPs to improve fault tolerance.
The DNS spec recommends that clients rotate through IP addresses if connection to
first address does not work for some reason.*)

Also the SSH host key of the Git server(s) used by Gitmanager need to be
configured using `knownHosts` attribute to allow git cloning work (if ssh is to
be used as the access method).

In the case of the earlier mentioned NFS-CSI configuration, the
`storageClassName` needs to be set to `nfs-csi`. If you follow the example of
values-minikube-testing.yaml, just replace "standard" with the correct
storageClassName setting.

Also setting `secretKey` and `dbPassword` is recommended, although not needed
for initial trials. These can be any random strings.

### Starting the Kubernetes deployment

We follow the steps described already in the main README file: after the above
steps, running `helm install` should work as given in the example, with your own
values file. However, at least in this case, also `helm dependency build` was
needed before install worked, to setup the helm chart dependencies properly.
Initially there was a problem that some pods were stuck in pending state. At
least in this case the reason was problem in wrong Persistent Volume
configurations. Using `kubectl describe` is useful in debugging problems if such
arise.

After a short while, one can check if there are pods running, and if there are
problems with any of the pods:

    kubectl -n aplus get pods

There should be at least the following pods (the pod names are appended with
some random characters to ensure name uniqueness):

* aplus (A+ front)
* grader (The MOOC Grader)
* gitmanager (Gitmanager)
* aplus-db (Postgres database)
* redis (Message broker / datastore used by Gitmanager)
* aplus-celery-worker (For executing asynchronous / timed Celery tasks for the
  A+ front)

Additionally, there typically are some temporary pods for cleanup tasks, etc.

To uninstall the Kubernetes deployment, the following can be used:

    helm uninstall -n aplus aplus

If configurations change, they can be updated in the running deployment as follows:

    helm upgrade -n aplus -f values-dice.yaml aplus .

After the upgrade the pods may need to be restarted by force in some cases to
make the changes effective:

    kubectl rollout restart -n aplus deploy aplus

### Setting up A+

The [A+ deployment
instructions](https://github.com/apluslms/a-plus/blob/master/doc/DEPLOYMENT.md)
are written for a traditional server use case, but in the case of Kubernetes and
pre-built Docker images, many of the needed steps are already done in the
Docker images available in the apluslms Docker Hub. However, a few steps are
still needed:

To open bash shell in the aplus container, the following can be used:

    kubectl exec -n aplus -it $(kubectl get po -n aplus -l app=aplus -o name) -c aplus -- bash

Inside the container's bash shell, the first step is to run database migrations
to make the DB structure to be up to date:

    python3 manage.py migrate

An initial superuser needs to be created for A+ (you will be prompted for
username and password):

    python3 manage.py createsuperuser

You can now exit the shell, e.g. by typing 'exit' or using Ctrl-D.

Also the database in Gitmanager needs have the migrations run:

    kubectl exec -n aplus -it $(kubectl get po -n aplus -l app=gitmanager -o name) -c gitmanager -- bash
    python3 manage.py migrate

While in Gitmanager shell, you should also create a JWT token that is needed to
access the Gitmanager web interface, for configuring a new course in A+:

    echo '[]' | python3 manage.py gentoken 'YOUR NAME HERE' 10h

`10h` refers to the token lifetime, after which you will need to create a new
token. The command will output an token that you will need to copy-paste to
access Gitmanager web interface.

### Creating the first A+ course

A good test case for the first course is the [A+
Manual](https://github.com/apluslms/aplus-manual), but any other course could be
used as well. Note that in order to be able to set the deployment keys, etc. as
described below, you should probably fork the A+ Manual Git repository to a
place where you can configure these settings.

You should now be able to login to A+ web interface using the address configured
in the values file in the beginning, and the superuser account created earlier.
There you should go to admin view at `https://your.dns.name/admin`. There you
can first navigate to the Courses table, to create the course, and then to the
Course Instances table to create the first course instance for the course. For
the course instance, you need to remember the instance ID. When you view the
instance details in admin view, near the end of the URL there is an integer that
represents the ID.

After this you need to log in to Gitmanager, using the Gitmanager URL provided
in the values.yaml configuration, and the JWT token created earlier. In
Gitmanager you can add the configuration needed to fetch the course content from
Git. Press the "*New*" button and enter the form fields. "*Key*" is a unique
string identifier used by the Gitmanager. A good practice is to construct this
based on e.g. the course code and some label that identifies the instance.
"Remote id* is the A+ instance identifier, i.e. the integer mentioned in the
previous paragraph. Git origin and branch refer to the git repository where the
content is. 'Update hook' can be left empty.

In the Git repository, you will need to configure the SSH deployment key to
provide access to the repository, that is shown on the front page, and the web
hook details to allow automatic updates when git repository is updated. The hook
URL and secret are shown in the course information page that opens when you
click on the course key link in the main view.

Once these are configured, you can click the "*Trigger*" button that causes the
course content to be cloned from git, and starts the course build process. When
you refresh the page after a while, you will see whether the build succeeded,
and some log information about the build process. One common problem initially
is, that if ssh access method is used for Git clone, the host key is not
properly configured. This can be pre-configured in the Helm values file.

Note the A+ configuration JSON field: this URL needs to be set in A+ course
setup, so that A+ can find the built course content from the correct location.
When you press "*Apply*" in A+, the course should become visible, if everything
went successfully.