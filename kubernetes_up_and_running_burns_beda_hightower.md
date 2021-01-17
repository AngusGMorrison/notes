# Kubernetes Up & Running

# Chapter 1: Introduction

Kubernetes provides tools to move quickly while staying available. The core concepts are:

- Immutability
- Declarative configuration
- Online self-healing systems

## Immutability

In an immutable system, rather than a series of incremental updates and changes, an entirely new, complete image is built, where the update simply replaces the entire image in a single operation.

The artifact you create, and the record of how you created it, make it easy to understand exactly the differences in some new version and, if something goes wrong, to determine what has changed and how to fix it.

Building a new image rather than modifying an existing one means that old image is still around, and can quickly be used for a rollback if an error occurs.

## Declarative configuration

Everything in Kubernetes is a declarative configuration object that represents the desired state of the system. It is the job of Kubernetes to ensure that the actual state of the world matches this desired state.

The idea of storing declarative configuration in source control is referred to as "infrastructure as code".

## Decoupling

In a decoupled architecture, each component is separated from other components by defined APIs and service load balancers.

Decoupling components via load balancers makes it easy to scale the programs that make up your service, because increasing the size of the program can be done without adjusting or reconfiguring any of the other layers of your service.

## Scaling Development Teams with Microservices

Kubernetes provides numerous abstractions and APIs that make it easier to build these decoupled microservices architectures:

- **Pods**, or groups of containers, can group together container images developed by different teams into a single deployable unit.
- Kubernetes **services** provide load balancing, naming, and discovery to isolate one microservice from another.
- **Namespaces** provide isolation and access control, so that each microservice can control the degree to which other services interact with it.
- **Ingress** objects provide an easy-to-use frontend that can combine multiple microservices into a single externalized API surface area.

## Separation of Concerns for Consistency and Scaling

- The application developer relies on the service-level agreement (SLA) delivered by the container orchestration API, without worrying about the details of how this SLA is achieved. Likewise, the container orchestration API reliability engineer focuses on delivering the orchestration API's SLA without worrying about the applications that are running on top of it.

## Efficiency

Kubernetes provides tools that automate the distribution of applications across a cluster of machines, ensuring higher levels of utilization than are possible with traditional tooling.

# Chapter 2: Creating and Running Containers

Container images bundle a program and its dependencies into a single artifact under a root filesystem.

## Container Images

The Docker image format is made up of a series of filesystem layers. Each layer adds, removes or modifies files from the preceding layer in the filesystem. This is an example of an overlay filesystem.

### Container Layering

The image isn't a single file but rather a specification for a manifest file that points to other files. Associated with this format is an API for uploading and downloading images to an image registry.

Container images are typically combined with a container configuration file, which provides instructions on how to set up the container environment and execute an application entry point. The container configuration often includes information on how to set up networking, namespace isolation, resource constraints (cgroups), and what `syscall` restrictions should be placed on a running container instance.

**System containers** seek to mimic virtual machines and often run a full boot process. They often include a set of system services typically found in a VM, such as `ssh`, `cron`, and `syslog`. Over time, they have come to be seen as poor practice.

**Application containers** commonly run a single program.

## Building Application Images with Docker

### Dockerfiles

The **Dockerfile** is a recipe for how to build the container image, while `.dockerignore` defines the set of files that should be ignored when copying files into the image.

*Example: .dockerignore*

---

```docker
node_modules
```

*Dockerfile*

---

```docker
# Start from a Node.js 10 (LTS) image.   (1)
FROM node:10

# Specify the directory inside the image in which all commands will run.   (2)
WORKDIR /usr/src/app

# Copy package files and install dependencies.   (3)
COPY package*.json ./
RUN npm install

# Copy all of the app files into the image.   (4)
COPY . .

# The default command to run when starting the container.    (5)
CMD ["npm", "start"]
```

1. Every Dockerfile builds on other container images. This is a preconfigured image with Node.js 10.
2. Sets the work directory, in the container image, for all following commands.
3. Initializes the dependencies for Node.js. First we copy the package files into the image. The `RUN` command then runs the correct command *in the container* to install the dependencies.
4. Copies the rest of the program files into the image, apart from those in `.dockerignore`.
5. Specifies the command that should be run when the container is run.

Build and tag a docker image: `docker build -t simple-node .`

Run an image, removing it once done: `docker run --rm -p 3000:3000 simple-node`

### Optimizing Image Sizes

Files that are removed by subsequent layers in the system are actually still present in the images; they're just inaccessible.

Each layer is an independent delta from the layer below it. Every time you change a layer, it changes every layer that comes after it. Changing the preceding layers mean that they need to be rebuilt, repushed and repulled to deploy your image to development.

In general, you want to order your layers from least likely to change to most likely to change in order to optimize the image size for pushing and pulling.

### Image Security

Don't build containers with passwords baked into any layer. Deleting a file in one layers doesn't delete the file from preceding layers. It still takes up space and can be accessed with the right tools.

## Multistage Image Builds

One of the most common ways to accidentally build large images is to do the actual program compilation as part of the construction of the application container image.

**Multistage builds** resolve this problem. Rather than producing a single image, a Docker file produces multiple images, with each considered a separate stage. Artifacts can be copied from preceding stages to the current stage.

```docker
# STAGE 1: Build
FROM golang:1.11-alpine AS build

# Install Node and NPM
RUN apk update && apk upgrade && apk add --no-cache git nodejs bash npm

# Get dependencies for Go part of build
RUN go get -u github.com/jteeuwen/go-bindata/...
RUN go get github.com/tools/godep

WORKDIR /go/src/github.com/kubernetes-up-and-running/kuard

# Copy all sources in
COPY . .

# This is a set of variables that the build script expects
ENV VERBOSE = 0
ENV PKG=github.com/kubernetes-up-and-running/kuard
ENV ARCH=amd64
ENV VERSION=test

# Do the build. Script is part of incoming sources.
RUN build/build.sh

# STAGE 2: Deployment
FROM alpine
USER nobody:nobody
COPY --from=build /go/bin/kuard /kuard
CMD [ "/kuard" ]
```

This Dockerfile produces two images. The first is the build image, which contains the Go compiler, React.js toolchain, and source code for the program. The second is the deployment image, which simply contains the compiled binary.

## Storing Images in a Remote Registry

Once logged in, you can **tag** an image by prepending the target Docker registry. You can also append another identifier that is usually used for the version or variant of that image, separated by a colon.

```bash
docker tag kuard [652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_u](http://652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_ur)r/kuard-amd64:blue
```

To **push** the image:

```bash
docker push [652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_u](http://652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_ur)r/kuard-amd64:blue
```

## The Docker Container Runtime

Kubernetes provides and API for describing an application deployment, but relies on a container runtime to set up an application container using the container-specific APIs native to the target OS.

### Running Containers with Docker

To deploy a container from a remote image:

```bash
docker run -d --name kuard --publish 8080:8080 652455772073[.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_u](http://652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_ur)r/kuard-amd64:blue
```

This command starts the `kuard` container and maps ports 8080 on your local machine to 8080 in the container.

The `--publish` option can be shortened to `p`.

This forwarding is necessary because each container gets its own IP address, so listening on `[localhost](http://localhost)` inside the container doesn't cause you to listen on your machine. Without port forwarding, connections are inaccessible to your machine.

To remove a container, run `docker rm kuard`

## Limiting Resource Usage

Docker provides the ability to limit the amount of resources used by applications by exposing the underlying cgroup technology provided by the Linux kernel. These capabilities are likewise used by Kubernetes to limit the resources used by each Pod.

```bash
docker run -d --name kuard \
--publish 8080:8080 \
--memory 200m \
--memory-swap 1G \
--cpu-shares 1024 \
652455772073[.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_u](http://652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_ur)r/kuard-amd64:blue
```

## Cleanup

Delete images with `docker rmi [tag-name | image-id]`.

The image ID can be shortened as long as it remains unique. Generally only three or four characters of the ID are necessary.

Unless you explicitly delete an image it will live on your system forever, even if you build a new image with an identical name. Building this new image simply moves the tag to the new image.

`docker system prune` removes all stopped containers, all untagged images, and all unused image layers cached as part of the build process.

# Chapter 3: Deploying a Kubernetes Cluster

## Minikube

The `minikube` tool is a way to get a local Kubernetes cluster up and running in a VM on your local machine.

`minikube` only creates a single-node cluster.

You need to have a hypervisor installed to use `minikube`. This is generally `virtualbox`.

Create a local VM, provision Kubernetes, and create a local `kubectl` confguration that points to the cluster: `minikube start`.

Stop the cluster: `minikube stop`.

Delete the cluster: `minikube delete`.s

## Installing Kubernetes on Amazon Web Services

The easiest way to create an EKS cluster is via the open source `eksctl` command-line took.

To create a cluster: `eksctl create cluster --fargate --name kuar-cluster ...`

For more details on installation options such as node size: `eksctl create cluster --help`.

## Running Kubernetes in Docker

The kind project uses Docker containers to simulate multiple Kubernetes nodes instead of running everything in a virtual machine.

## The Kubernetes Client

The official Kubernetes client is `kubectl`: a command-line tool for interacting with the Kubernetes API.

Check cluster version of local `kubectl` tool and the version of the Kubernetes API server: `kubectl version`.

Get simple cluster diagnostics: `kubectl get componentstatuses`.

`controller-manager`: responsible for running various controllers that regulate behaviour in the cluster; for example, ensuring that all of the replicas of a service are available and healthy.

`scheduler`: responsible for placing different Pods onto different nodes in the cluster.

`etcd` server: storage for the cluster where all API objects are stored.

### Listing Kubernetes Worker Nodes

Master nodes contain containers like the API server, scheduler, etc., which manage the cluster.

Worker nodes are where containers will run.

Kubernetes generally won't schedule work onto master nodes to ensure that user workloads don't harm the overall operation of the cluster.

Get more information about a specific node: `kubectl describe nodes <name>`:

- Basic info about the node
- Info about the operation of the node, e.g. current conditions like `OutOfDisk` and `MemoryPressure`
- Information about the capacity of the machine itself
- Information about the software on the node, including the version of Docker, Kubernetes and the Linux kernel
- Information about Pods currently running on the node.

Kubernetes tracks about the requests and upper limits for resources for each Pod that runs on a machine. Resources requested by a Pod are guaranteed to be present on the node, while a Pod's limit is the maximum amount of a given resource that a Pod can consume.

A pod's limit can be higher than its request, in which case the extra resources are supplied on a best-effort basis.

## Cluster Components

Many of the components that make up the Kubernetes cluster are actually deployed by Kubernetes itself.

All of these components run in the `kube-system` namespace.

### Kubernetes Proxy

The Kubernetes proxy is responsible for routing network traffic to load-balanced services in the Kubernetes cluster.

The proxy must be present on every node in the cluster. Kubernetes has an API object named DaemonSet that is used in many clusters to accomplish this.

If your cluster runs the Kubernetes proxy with a DaemonSet, you can see the proxies by running `kubectl get daemonSets --namespace=kube-system kube-proxy`.

### Kubernetes DNS

Kubernetes runs a DNS server which provides naming and discovery for the services that are defined in the cluster.

The DNS server is run as a replicated service – you may see one or more running in the cluster.

The DNS server is run as a Kubernetes **deployment**, which manages these replicas.

View DNS deployment: `kubectl get deployments --namespace=kube-system core-dns`

There is also a Kubernetes **service** that performs load balancing for the DNS server: 
`kubectl get services --namespace=kube-system core-dns`.

### Kubernetes UI

The UI is run as a single replica, but is still managed by a Kubernetes deployment for reliability and upgrades.

See the UI server using:
`kubectl get deployments --namespace=kube-system kubernetes-dashboard`.

The dashboard also has a service that performs load balancing:
`kubectl get services --namespace=kube-system kubernetes-dashboard`.

Use `kubectl proxy` and look up the Kubenetes dashboard URL to see the UI.

Use the interface to explore your cluster, create new containers and more.

# Common kubectl Commands

## Namespaces

Each namespace is like a folder that holds a set of objects.

By default, the `kubectl` command line tool interacts with the `default` namespace.

Pass the `--all-namespaces` flag to interact with all namespaces.

## Contexts

Use a context to change the default namespace permanently. This gets recorded in a `kubectl` configuration file, usually located at `$HOME/.kube/config`.

The configuration file also stores how to both find and authenticate to your cluster.

Create a context with a different default namespace using:

`kubectl config set-context my-context --namespace=mystuff`

This creates a new context, but doesn't start it. To use it:

`kubectl config use-context my-context`

Contexts can also be used to manage different clusters or different users for authenticating to those clusters using the `--users` or `--clusters` flags with the `set-context` command.

## Viewing Kubernetes API Objects

Everything contained in Kubernetes is is represented by a RESTful resource, referred to as Kubernetes objects. Each object exists at a unique HTTP path. E.g. 
*https[:](https://your-k8s.com/api/v1/namespaces/default/pods/my-pod)//your-k8s.com/api/v1/namespaces/default/pods/my-pod* leads to the representation of a Pod named `my-pod` in the default namespace.

The `kubectl` command makes HTTP requests to these URLs to access the Kubernetes objects at these paths.

### `get`

- List all resources in namespace: `kubectl get <resource-name>.`
- Show specific object: `kucectl get <resource-name> <object-name>`.
- The `-o wide` flag provides more details on a longer line.
- View raw JSON/YAML: `-o json` / `-o yaml`.
- Remove headers from `kubectl` table displays (e.g. for piping output): `--no-headers`
- Extract specific fields using JSONPath: `-o jsonpath --template={.status.podIP}`

### `describe`

- Provide detailed information about a particular object: `kubectl describe <resource-name> <object-name>`

### Creating, Updating and Destroying Kubernetes Objects

Objects in the Kubernetes API are represented as JSON or YAML files, either returned by the server in response to a query or posted to the server as part of an API request.

### `apply`

Given a simple objects stored in *obj.yaml*, create this object in Kubernetes by running:
`kubectl apply -f obj.yaml`.

You don'y need to specify the resource type of an object. It's obtained from the object file itself.

Use the same command to make changes to the object.

The `apply` tool only modifies objects if there are differences between the existing object and the input file. If they match, `apply` will exit successfully without making any changes. This makes it useful for loops where yu want to ensure the state of the cluster matches the state of the filesystem.

Dry run: `--dry-run`.

Make interactive edits instead of using a file: `kubectl edit <resource-name> <object-name>`.

`apply` records the history of previous configurations in an annotation within the object. Manipulate these records with `edit-last-applied`, `set-last-applied` and `view-last-applied`. E.g. `kubectl apply -f myobj.yaml view-last-applied` will show you that last state that was applied to the object.

### `delete`

To delete an object: `kubectl delete -f obj.yaml` or `kubectl delete <resource> <obj>`.

## Labeling and Annotating Objects

Labels and annotations are tags for your objects.

To add the `color=red` label to a Pod named `bar`: `kubectl label pods bar color=red`.

Use `--overwrite` to overwrite an existing label.

## Debugging Commands

See logs for a running container: `kubectl logs <pod>`

If you have multiple containers in your pod, choose the container to view with the `-c` flag.

To continuously stream logs back to the terminal, add the `-f` (follow) command line flag.

Execute a command in a running container: `kubectl exec -it <pod> -- bash`.

If you don't have bash or some other terminal available within your container, you can `attach` to the running process: `kubectl attach -it <pod>`.

`attach` is similar to `kubectl logs` but allows you to send input to the running process, assuming that process is set up to read from standard input.

Copy files to and from a container: `kubectl cp <pod>:<remote-path> <local-path>`.

- Copies file from running container to local machine.

To access a Pod via the network: `kubectl port-forward <pod> 8080:80`.

- Forwards network traffic from the local machine to the Pod.
- Enables you to securely tunnel network traffic through to containers that might not be exposed anywhere on the public network.

You can also use `port-forward` to with services by specifying `services/<service-name>` instead of `<pod>`.

- Requests will only ever be forwarded to a single Pod in that service. They will not go through the service load balancer.

See the resources in use by either nodes or pods: `kubectl top nodes/pods`

# Pods

## Pods in Kubernetes

A pod represents a collection of application containers and volumes running in the same execution environment. They are the smallest deployable artifact in the Kubernetes cluster.

All of the containers in a pod always land on the same machine.

Each container within a pod runs in its own cgroup.

Applications running in the same Pod share the same IP address and port space (network namespace), have the same hostname (UTS namespace), and can communicate using native interprocess communication channels over System V IPC or POSIX message queues (IPC namespace).

Applications in different pods are isolated from each other; they have different IP addresses. hostnames and more. They might as well be on different servers.

## Thinking with Pods

The right question to ask when designing pods is, "Will these containers work correctly if they land on different machines?". If the answer is "no", a Pod is the correct grouping for the containers.

## The Pod Manifest

Pods are described in a Pod manifest – a text-file representation of the Kubernetes API object.

The Kubernetes API server accepts and processes Pod manifests before storing them in persistent storage (`etcd`).

The scheduler uses the Kubernetes API to find Pods that haven't been scheduled to a node, then places them onto nodes depending on the resources and other constraints expressed in the Pod manifests.

Multiple pods can be placed on the same machine as long as a there are sufficient resources.

Scheduling multiple replicas of the same application onto the same machine is worse for reliability, since the machine is a single failure domain.

The scheduler tries to ensure Pods from the same application are distributed onto different machines for reliability in the presence of such failures.

Once scheduled to a node, Pods don't move and must be explicitly destroyed and rescheduled.

### Creating a Pod

The simplest way to create a Pod is via the imperative `kubectl run` command.

```bash
kubectl run kuard --generator=run-pod/v1 --image=652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_ur/kuard-amd64
```

See the status of the pod by running: `kubectl get pods`.

Delete the pod: `kubectl delete pods/kuard`.

## Creating a Pod Manifest

Pod manifests include a `metadata` section for describing the Pod and its labels, a `spec` section for describing volumes, and a list of containers that will run in the Pod.

*Example: kuard-pod.yaml*

---

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: kuard
spec:
	containers:
		- image: 652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_ur/kuard-amd64:blue
			name: kuard
			ports:
				- containerPort: 8080
					name: http
					protocol: TCP
```

## Running Pods

To turn a manifest into a Pod: `kubectl apply -f kuard-pod.yaml`.

The Kubernetes server will schedule the Pod to run on a healthy node in the cluster, where it will be monitored by the `kubelet` daemon process.

The `Pending` state indicates that the Pod has been submitted but hasn't been scheduled yet.

### Pod Details

`kubectl describe pods kuard` outputs information about the Pod in sections:

1. Basic information about the Pod.
2. Information about containers running in the Pod.
3. Events related to the Pod, such as when it was scheduled, when its image was pulled, and if/when it had to be restarted due to failing health checks.

## Deleting a pod

Delete by name: `kubectl delete pods/kuard`

Delete using creation manifest: `kubectl delete -f kuard-pod.yaml`

When a Pod is deleted, it isn't immediately killed. The pod is placed in `Terminating` state for 30 seconds by default. It no longer receives new requests, but may finished active requests.

Any data stores in the containers associated with that Pod will be deleted as well. To persist data across multiple instances of a Pod, you need to use `PersistentVolumes`.

## Accessing Your Pod

### Using Port Forwarding

To access a specific pod even if it's not serving traffic on the internet:
`kubectl port-forward kuard 8080:8080`.

For as long as the pod runs, you can access it at *http:localhost:8080/<local-port>.*

### Getting More Info with Logs

The `kubectl logs` command always tries to get logs from the current running container.

Adding the `--previous` flag will get logs from a previous instance of the container. This is useful if containers are continuously restarting due to a problem at container startup.

It's generally useful to use a log aggregation service like fluentd and elasticsearch. They provide greater capacity for storing a longer duration of logs, as well as rich log filtering and search capabilities. They often provide the ability to aggregate logs from multiple Pods into a single view.

### Running Commands in Your Container with `exec`

To execute commands in the context of the container itself: `kubectl exec kuard -- date`.

To get an interactive session: `kubectl exec -it kuard -- ash`.

### Copying Files to and from Containers

Copy to pod: `kubectl cp <pod-name>:path ./host-path`

Generally, copying files into a container is an anti-pattern. The contents of a container should be treated as immutable, otherwise it will be overwritten on a subsequent rollout.

## Health Checks

Containers are automatically kept alive using a ***process health check**. If the main process of your application stops running, Kubernetes restarts it.*

If the process has deadlocked and is unable to serve requests, a process health check will still believe that your application is healthy since its process is still running.

**Liveness health checks** run application-specific logic (e.g., loading a web pasge) to verify that the application is not just still running, but is functioning properly.

Liveness health checks are defined in your Pod manifest.

### Liveness Probe

Liveness probes are defined per container, which means each container inside a Pod is health-checked separately.

The following manifest adds a liveness probe to the `kuard `container, which runs an HTTP request against the `/healthy` path.

*Example: kuard-pod-health.yaml*

---

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: kuard
spec:
	containers:
		- image: 652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_ur/kuard-amd64:blue
			name: kuard
			livenessProbe:
				httpGet:
					path: /healthy
					port: 8080
				initialDelaySeconds: 5
				timeoutSeconds: 1
				periodSeconds: 10
				failureThreshold: 3
			ports:
				- containerPort: 8080
					name: http
					protocol: TCP
```

- Uses an `httpGet` probe to perform an HTTP GET request against the `/healthy` endpoint on port 8080 of the `kuard` container.
- Sets in `initialDelaySeconds` of 5: it won't be called until 5 seconds after all the containers in the Pod are created.
- Probe must respond within the 1-second timeout, and HTTP status code must be `≥ 200 && < 400`.
- Calls the probe every 10 seconds.
- If more than 3 consecutive probes fail, the container will restart.

The pod's behaviour on a failed liveness check is defined by its `restartPolicy`:

- `Always` (default)
- `OnFailure` (restart only on liveness failure or nonzero process exit code)
- `Never`

### Readiness Probe

Liveness determines if an application is running properly. Readiness describes when a container is ready to server user requests.

Containers that fail readiness checks are removed from service load balancers.

### Types of Health Checks

In addition to HTTP checks, Kubernetes supports:

- `tcpSocket` health checks, which open a TCP socket; if the connection is successful, the probe succeeds.
    - Useful for non-HTTP applications, such as databases.
- `exec` probes, which execute a script or run a program in the context of the contaner. If the script returns a 0 exit code, the probe succeeds.
    - Often used for custom application validation logic that doesn't fit neatly into an HTTP call.

## Resource Management

Efficiency is measured by **utilization:** the amount of a resource actively being used divided by the amount of a resource that has been purchased.

Resource **requests** specify the minimum amount of a resource that an application can consume.

Resource **limits** specify the maximum amount of a resource that an application can consume.

### Resource Requests: Minimum Required Resources

To request that the `kuard` container lands on a machine with half a CPU free and gets 128 MB of memory allocated to it, we define the following Pod.

*Example: kuard-pod-resreq.yaml*

---

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: kuard
spec:
 containers:
		- image: 652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_ur/kuard-amd64:blue
			name: kuard
			resources:
				requests:
					cpu: "500m"
					memory: "128Mi"
			ports:
				- containerPort: 8080
					name: http
					protocol: TCP
```

Resources are requested per container, not per Pod. The total resources requested by the Pod is the same of all resources requested by all containers within the Pod.

CPU requests are implemented using the `cpu-shares` functionality in the Linux kernel.

If a container is over its memory request, the OS can't just remove memory from the process, because it's been allocated. Consequently, when the system runs out of memory, the `kubelet` terminates containers whose memory usage is greater than their requested memory. The containers are automatically restarted, but with less available memory on the machine for the container to consume.

### Capping Resource Usage With Limits

In this example, we extend the previous manifest to add a limit of 1.0 CPU and 256 MB of memory.

*Example: kuard-pod-reslim.yaml*

---

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: kuard
spec:
 containers:
		- image: 652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_ur/kuard-amd64:blue
			name: kuard
			resources:
				requests:
					cpu: "500m"
					memory: "128Mi"
				limits:
					cpu: "1000m"
					memory: "256Mi"
			ports:
				- containerPort: 8080
					name: http
					protocol: TCP
```

## Persisting Data with Volumes

When a Pod is deleted or a container restarts, all data in the container's filesystem is also deleted.

### Using Volumes with Pods

`spec.volumes` defines all of the volumes that may be accessed by containers in the Pod manifest.

Not all containers are required to mount all volumes defined in the Pod.

The `volumeMounts` array in the container definition defines that volumes that are mounted into a particular container, and the path where each volume should be mounted.

Different containers in a Pod can mount the same volume at different mount paths.

This example defines a single new volume named `kuard-data`, which the `kuard` container mounts to the `/data` path.

*Example: kuard-pod-vol.yaml*

---

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: kuard
spec:
	volumes:
		- name: "kuard-data"
			hostpath: "/var/lib/kuard"
containers:
	- image: 652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_ur/kuard-amd64:blue
		name: kuard		
		ports:
			- containerPort: 8080
				name: http
				protocol: TCP
		volumeMounts:
			- mountPath: "/data"
				name: "kuard-data"
			
```

### Different Ways of Using Volumes with Pods

**Communication/synchronization**

For example, two containers using a shared volume to serve a site while keeping it synchronized to a remote Git location.

To achieve this, the Pod uses an `emptyDir` volume. These are scoped to the Pod's lifespan but can be shared between containers.

**Cache**

A volume that is valuable for performance but is not required for correct operation of the application. E.g. a store of prerendered thumbnails for larger images. Such a cache must survive a container restart due to a health-check failure.

`emptyDir` works well for this use case too.

**Persistent data**

To achieve data persistence beyond the lifespan of a Pod which is able to move between nodes in a cluster in the event of failure, Kubernetes supports a variety of remote network storage volumes, including NFS, iSCSI and cloud provider network storage like AWS Elastic Block Store and Elastic File System (NFS).

**Mounting the host filesystem**

Some applications don't need a persistent volume, but do need access to the underlying host filesystem. E.g. access to `/dev` to perform raw, block-level access to a device on the system.

For these cases, Kubernetes supports the `hostPath` volume type, which can mount arbitrary locations on the worker node into the container.

### Persisting Data Using Remote Disks

Often, you want the data a Pod is using to stay with the Pod, even if it is restarted on a different host machine.

To achieve this, mount a remote network storage volume into your Pod.

When using network-based storage, Kubernetes automatically mounts and unmounts the appropriate storage whenever a Pod using that volume is scheduled onto a particular machine.

*Example: Mounting an NFS server*

---

```yaml
...
volumes:
	- name: "kuard-data"
		nfs:
			server: my.nfs.server.local
			path: "/exports"
```

## Putting it all Together

*Example: kuard-pod-full.yaml*

---

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: kuard
spec:
	volumes:
		- name: "kuard-data"
			nfs:
				server: my.nfs.server.local
				path: "/exports"
	containers:
		- image: 652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_ur/kuard-amd64:blue
			name: kuard
			ports:
				- containerPort: 8080
				  name: http
					protocol: TCP
			resources:
				requests:
					cpu: "500m"
					memory: "128Mi"
				limits:
					cpu: "1000m"
					memory: "256Mi"
			volumeMounts:
				- name: "kuard-data"
					mountPath: "/data"
			livenessProbe:
				httpGet:
					path: /healthy
					port: 8080
				initialDelaySeconds: 5
				timeoutSeconds: 1
				periodSeconds: 10
				failureThreshold: 3
			readinessProbe:
				httpGet:
					path: /ready
					port: 8080
				initialDelaySeconds: 30
				timeoutSeconds: 1
				periodSeconds: 10
				failureThreshold: 3
```

# Chapter 6: Labels and Annotations

**Labels** are key/value pairs that can be attached to Kubernetes objects such as Pods and ReplicaSets.

Labels provide identifying information and provide the foundation for grouping objects.

**Annotations** provide a storage mechanism that resembles labels: key/value pairs designed to hold non-identifying information that can be leveraged by tools and libraries.

## Labels

Labels provide identifying metadata for objects. These are fundamental qualities of the object that will be used for grouping, viewing and operating.

Further reading: *Site Reliability Engineering*, Beyer et al. (O'Reilly)

Production abhors a singleton. When deploying software, users will often start with a single instance. However, as the application matures, these singletons often multiply and become sets of objects. Kubernetes uses labels to deal with sets of objects instead of single instances.

Any hierarchy imposed by the system will fall short for many users. In addition, user groupings and hierarchies change over time. Kubernetes labels are flexible enough to adapt to these situations and more.

Label keys can be broken down into two parts: an optional prefix and a name, separated by a slash.

- The prefix, if specified, must be a DNS subdomain with a 253-character limit.
- The key name is required and must be shorter than 63 characters.
- Names must start and end with an alphanumeric character and permit -, _ and . between characters.
- Label values are strings with max length 63 characters. The contents follow the same rules as for label keys.

*Example: label examples*

---

```bash
acme.com/app-version
appVersion
app.version
kubernetes.io/cluster-service
```

When domain names are used in labels and annotations, they are expected to be aligned to that particular entity in some way.

### Applying Labels

*Example: applying labels to a deployment*

---

```bash
kubectl create deployment alpaca-prod --image=652455772073.dkr.ecr.eu-west-1.amazonaws.com/kubernetes_ur/kuard-amd64:blue \

kubectl label deployments/alpaca-prod ver=1 name=alpaca env=prod
```

View an object with labels: `kuectl get deployments --show-labels`.

### Modifying Labels

Labels can be applied and updated on objects after creation:
`kubectl label deployments alpaca-prod ver=2`.

- This only changes the label on the deployment itself. It won't affect the ReplicaSets and Pods the deployment creates. To change those, change the template embedded in the deployment.

You can also use the `-L` option to `kubectl get` to show a label value as a column:

`kubectl get deployments -L canary`.

Remove labels with a dash suffix: `kubectl label deployments alpaca-prod canary-`

### Label Selectors

The label `pod-template-hash` is applied by the deployment so it can keep track of which Pods were generated from which template versions.

To list only objects that have a certain label: `kubectl get deployments —selector="ver=2"`.

To ask whether a label is one of a set of values:
`kubectl get deployments —selector="app in (alpaca,bandicoot)`.

To ask whether a selector is set at all: `kubectl get deployments --selector="canary"`.

There are negative versions of each selector, e.g. `notin`, `key!=value`, `!key`... E.g.
`kubectl get deployments —selector='!canary'`.

You can combine positive an negative selectors with the selector flag, `-l`:

`kubectl get deployments -l 'ver=2,!canary'`.

### Label Selectors in API Objects

For historical reasons, there are two forms.

*Example: more recent selector operations*

---

```yaml
selector:
	matchLabels:
		name: alpaca
	matchExpression:
		- {key: ver, operator: In, values: {1, 2}}
```

- All terms are evaluated as a logical AND. The only way to represent `!=` is to convert it to a `NotIn` expression with a single value.

*Example: older form*

---

```yaml
selector:
	app: alpaca
	ver: 1
```

- Used in `ReplicationControllers` and services
- Only supports the `=` operator
- A simple set of key-value pairs that must all match a target object to be selected.

### Labels in the Kubernetes Architecture

Kubernetes is a purposefully decouple architecture. There is no hierarchy and all components operate independently. However, in many cases objects need to relate to each other, and these relationships are defined by labels and label selectors.

- E.g. `ReplicaSets` find the Pods they are managing via a selector. Service load balancers finds the Pods it should bring traffic to via a selector query. When a Pod is created, it can use a node selector to identify a particular set of nodes that it can be scheduled onto. When people want to restrict network traffic in their cluster, they use `NetworkPolicy` in conjunction with specific labels to identify Pods that should or should not be allowed to communicate with each other.

## Annotations

Annotations provide a place to store additional metadata for Kubernetes objects with the sole purpose of assisting tools and libraries.

Annotations provide extra information about where an object came from, how to use it, or policy around that object.

Annotations are used to:

- Keep track of a reason for the latest update to an object.
- Communicate a specialized scheduling policy to a specialized scheduler.
- Extend data to about the last tool to update the resource and how it was updated (used for detecting changes by other tools and doing a smart merge).
- Attach build, release or image information that isn't appropriate for labels (may include a Git hash, timestamp, PR number, etc.)
- Enable the Deployment object to keep track of `ReplicaSets` that it is managing for rollouts.
- Provide extra data to enhance the visual quality or usability of a UI (e.g. objects could include a link to an icon).
- Prototype alpha functionality in Kubernetes (instead of creating a first-class API field, the parameters for that functionality are encoded in an annotation).

The primary use case is rolling deployments, where annotations are used to track rollout status and provide the necessary information required to roll back a deployment to a previous state.

Avoid using the Kubernetes API server as a general-purpose database. Annotations are good for small pieces of data that are highly associated with the given resource.

### Defining Annotations

Annotation keys use the same format as label keys. However, because they are often used to communicate information between tools, the "namespace" part of the key is more important.

Annotations are defined in the common metadata section in every object.

*Example: Defining annotations*

---

```yaml
...
metadata:
	annotations:
		example.com/icon-url: "https://example.com/icon.png"
...
```

## Summary

Using labels and annotations properly unlocks the true power of Kubernetes' flexibility and provides the starting point for building automation and deployment workflows.

# Chapter 7: Service Discovery

## What is Service Discovery?

Service discovery tools help solve the problem of finding which processes are listening at which addresses for which web services.

A good service discovery system:

- Enables users to resolve this information quickly and reliably.
- Is low-latency: clients are updated soon after the information associated with a service changes.
- Can store a richer definition of what that service is, e.g. whether there are multiple ports associated with a service.

The Domain Name System (DNS) is the traditional system of service discovery on the internet, designed for relatively stable name resolution with wide and efficient caching.

DNS falls short in the dynamic world of Kubernetes: many systems (e.g. Java, by default) look up a name in DNS directly and never re-resolve. This leads to clients caching stale mappings and talking to the wrong IP.

There are natural limits to the amount and type of information that can be returned in a DNS query too. Things start to break past 20-30 A records for a single name, while SRV records are often very hard to use. Finally, the way that clients handle multiple IPs in a DNS record is usually to take the first IP address and rely on the DNS server to randomize or round-robin the order of records.

- A DNS A record points a domain or subdomain to an IP address.
- An SRV record defines a symbolic name and transport protocol used as part of the domain name. It defines the priority, weight, port and target for the service in the record content.

## The Service Object

A Service object is a way to create a named label selector.

Expose a service: `kubectl expose deployment bandicoot-prod`.

View services and associated selectors: `kubectl get services -o wide`.

The Kubernetes service is automatically created for you so that you can find and talk to the Kubernetes API from within the app.

At a high level, a service simply gives a name to a selector and specifies which ports to talk to for that service. `kubectl expose` pulls both the label selector and the relevant ports from the deployment definition.

The service is assigned a new type of virtual IP called a **cluster IP**. This is a special IP address the system will load-balance across all of the Pods that are identified by the selector.

To port forward to a Pod within a service:

```bash
ALPACA_POD=$(kubectl get pods -l app=alpaca-prod) \
-o jsonpath='{.items[0].metadata.name}')

kubectl port-forward $ALPACA_POD 48858:8080
```

### Service DNS

Because the cluster IP is virtual, it is stable, and it is appropriate to give it a DNS address. All of the issues around clients caching DNS results no longer apply.

Within a namespace, it is as easy as using the service name to connect to one of the Pods identified by a service.

Kubernetes provides a DNS service exposed to Pods running in the cluster, which is installed as a system component when the cluster is created.

The Kubernetes DNS service provides DNS names for cluster IPs.

The full DNS name will be something like: `alpaca-prod.default.svc.cluster.local.`.

- `alpaca-prod`: the name of the service.
- `default`: the namespace the service is in.
- `svc`: recognizes that this is a service, allowing Kubernetes to expose other types of things as DNS in future.
- `cluster.local.`: The base domain name for the cluster. Administrators may change this to allow unique DNS names across multiple clusters.

When referring to a service in your own namespace you can just use the service name.

Refer to a service in another namespace with `alpaca-prod.default`.

You can also use the fully qualified service name: `alpaca-prod.default.svc.cluster.local.`.

### Readiness Checks

The Service object tracks which Pods are ready via a readiness check. Only ready Pods will be sent traffic.

*Example: deployment with readiness check*

---

```yaml
spec:
	...
	template:
		...
		spec:
			containers:
				...
				name: alpaca-prod
				readinessProbe:
					httpGet:
						path: /ready
						port: 8080
					periodSeconds: 2
					intialDelaySeconds: 0
					failureThreshold: 3
					successThreshold: 1
```

Endpoints are a lower-level way of finding what a service is sending traffic to. The `--watch` option causes the `kubectl` command to output updates as they happen:

`kubectl get endpoints alpaca-prod --watch`

The readiness check is a way for an overloaded or sick server to signal to the system that it doesn't want to receive traffic anymore. This is a great way to implement a graceful shutdown.

## Looking Beyond the Cluster

The most portable way to allow new traffic into the cluster is to use a feature called **NodePorts**. In addition to a cluster IP, the system picks a port, and every node in the cluster then forwards traffic to that port to the service.

If you can reach any node in the cluster, you can contact a service. You use the NodePort without knowing where any of the Pods for that service are running.

This can be integrated with hardware or software load balancers to expose the service further.

To create a NodePort service, change the `spec.type` field to `NodePort`, or specify `--type=NodePort` if creating the service via the command line.

The system will assign a NodePort to the service. Now we can hit any of our cluster nodes on that port to access the service.

- If you are sitting on the same network, you can access it directly. If your cluster is in the cloud, you can use SSH tunnelling with something like `ssh <node> -L 8080:localhost:32711` (requires additional AWS configuration).

## Cloud Integration

The `LoadBalancer` type builds on the NodePort type by configuring the cloud to create a new load balancer and direct it at nodes in your cluster. This will generate a public address assigned by your Cloud.

## Advanced Details

### Endpoints

For every Service object, Kubernetes creates a buddy Endpoints object that contains the IP addresses for that service: `kubectl describe endpoints alpaca-prod`.

To use a service, an advanced application can talk to the Kubernetes API directly to look up endpoints and call them.

The Kubernetes API can watch objets and be notified as soon as they change. In this way, a client can react immediately as soon as the IPs associated with a service change:
`kubectl get endpoints alpaca-prod —watch`.

### Manual Service Discovery

Kubernetes services are built on top of label selectors over Pods. That means you can use the Kubernetes API to do rudimentary service discovery without using a Service object at all. You can always use labels to identify the set of Pods you're interested in, get all of the Pods for those labels, and dig out the IP address.

### kube-proxy and Cluster IPs

Cluster IPs are stable virtual IPs that load-balance traffic across all of the endpoints in a service. This is performed by a component running on every node in the cluster called the `kube-proxy`.

The `kube-proxy` watches for new services in the cluster via the API server. It then programs a set of `iptables` rules in the kernel of that host to rewrite the destinations of packets so they are directed at one of the endpoints for that service.

- If the set of endpoints for a service changes (due to Pods coming and going or due to a failed readiness check), the set of `iptables` rules is rewritten.

The cluster IP is usually assigned by the API server when the service is created, but the user can specify one when creating the service.

Once set, the cluster IP can't be modified without deleting and recreating the Service object.

The Kubernetes service address range is configured using the `--service-cluster-ip-range` flag on the `kube-apiserver` binary.

- Any explicit cluster IP requested must come from that range and not already be in use.

### Cluster IP Environment Variables

Injecting a set of environment variables into Pods as they start up is an older mechanism to find cluster IPs.

The two main environment variables are `ALPACA_PROD_SERVICE_HOST` and `ALPACA_PROD_SERVICE_PORT`.

This approach requires resources to be created in a specific order. The services must be created before the Pods that reference them. This introduces complexity.

## Connecting with Other Environments

When connecting Kubernetes to legacy resources outside of the cluster, you can use selector-less services to declare a Kubernetes service with a manually assigned IP address that is outside of the cluster.

- Kubernetes service discovery via DNS works as expected, but the network traffic itself flows to the external resource.

Connecting external resources to Kubernetes is trickier. If your cloud provider supports it, the easiest thing to do is to create an "internal" load balancer that lives in your virtual private network and can deliver traffic from a fixed IP address into the cluster. You can then use traditional DNS to make this IP address available to the external resource.

## Summary

Once your application can dynamically find services and react to the dynamic placement of those applications, you are free to stop worrying about where things are running and when they move.

# Chapter 8: HTTP Load Balancing with Ingress

The Service object operates at Layer 4 (according to the OSI model). This means that it only forwards TCP and UDP connections and doesn't look inside of those connections.

- Hosting many applications on a service uses many exposed services.
    - When services are `NodePort`s, you have to have clients connect to a unique port per service.
    - When services are type `LoadBalancer`, you'll be allocated cloud resources for each service.

For HTTP (Layer 7)-based services, we can do better.

When solving a similar problem without Kubernetes, users turn to the idea of "virtual hosting" – hosting many HTTP sites on a single IP address.

- Uses a load balancer or reverse proxy to accept incoming connections on HTTP (80) and HTTPS (443) ports.
- That program parses the connect and, based on the `Host` header and the URL path requested, proxies the HTTP call to some other program.
- User has to manage the load balancer configuration file. In a dynamic environment and as the set of virtual hosts expands, this can be very complex.

**Ingress** is a Kubernetes-native way to implement this pattern.

- Simplifies it by standardizing the configuration, moving it to a standard Kubernetes object, and merging multiple Ingress objects into a single config file for the load balancer.

The Ingress **controller** is a software system exposed outside the cluster using a service of `type: LoadBalancer`. It then proxies requests to upstream servers.

- The configuration for how it does this is the result of reading and monitoring Ingress objects.

## Ingress Spec Versus Ingress Controllers

Ingress is split into a common resource specification and a controller implementation.

There is no standard Ingress controller built in to Kubernetes, so the user must install one of many optional implementations.

Users can create and modify Ingress objects like every other object. But, by default, there is no code running to actually act on those objects. It is up to the users to install and manage an outside controller.

## Installing Contour

Contour is a controller built to configure the open source load balancer called Envoy. Envoy is built to be dynamically configured bia an API. The Contour Ingress controller takes care of translating the Ingress objects into something that Envoy can understand.

Install Contour with:

`kubectl apply -f https://j.hept.io/contour-deployment-rbac`

This creates a namespace called `projectcontour`. Inside the namespace it creates a deployment (with two replicas) and an external-facing service of type `LoadBalancer`.

- Also sets up correct permissions via a service account and installs a CustomResourceDefinition for some extended capabilities.

Fetch the external address of Contour with:
`kubectl get -n projectcontour service envoy -o wide`

### Configuring DNS

To make Ingress work well, you need to configure DNS entries to the external address for your load balancer.

You can map multiple hostnames to a single external endpoint, and the Ingress controller will play traffic cop and direct incoming requests to the appropriate upstream service based on that hostname.

If you have an IP address for your external load balancer, you'll want to create A records. If you have a hostname, create CNAME records..

### Configuring a Local `hosts` File

If you don't have a domain or you are using a local solution such as `minikube`, you can set up a local configuration by editing your `/etc/hosts` file to add an IP address.

Add a line like: `<ip-address> alpaca.example.com bandicoot.example.com`, filling in the external IP address for contour.

If all you have is a hostname, you can get an IP address (that may change in future) by executing `host -t a <address>`.

## Simplest usage

The simplest way to use Ingress is to have it just blindly pass everything that it sees through to an upstream service.

*Example: simple-ingress.yaml*

---

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
	name: simple-ingress
spec:
	backend:
		serviceName: alpaca
		servicePort: 8080
```

`kubectl apply -f simple-ingress.yaml`

`kubectl get ingress`

`kubectl describe ingress simple-ingress`

This sets things up so that *any* HTTP request that hits the Ingress controller is forwarded on to the `alpaca` service. You can now access the `alpaca` instance of `kuard` on any of the raw IPs/CNAMEs of the service: either `[alpaca.example.com](http://alpaca.example.com)` or `bandicoot.example.com`.

## Using Hostnames

We can direct traffic based on the properties of the request. The most common example is to have the Ingress system look at the HTTP host header (which is set to the DNS domain in the original URL).

*Example: host-ingress.yaml*

---

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
	name: host-ingress
spec:
	rules:
	- host: alpaca.angus-morrison.com
		http:
			paths:
			- backend:
					serviceName: alpaca
					servicePort: 8080
	- host: bandicoot.angus-morrison.com
		http:
			paths:
			- backend:
					serviceName: bandicoot
					servicePort: 8080
```

### Using Paths

In this example, we direct everything coming into *[http://bandicoot.angus-morrison.com](http://bandicoot.angus-morrison.com)* to the bandicoot service, but we also send *[http://bandicoot.angus-morrison.com/a](http://bandicoot.angus-morrison.com/a)* to the alpaca service.

This type of scenario can be used to host multiple services on different paths of a single domain.

*Example: path-ingress.yaml*

---

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
	name: path-ingress
spec:
	rules:
	- host: bandicoot.example.com
		http:
			paths:
			- path: "/"
				backend:
					serviceName: bandicoot
					servicePort: 8080
			- path: "/a/"
				backend:
					serviceName: alpaca
					servicePort: 8080
```

The upstream server must be configured to handle traffic on that subpath (e.g. `/a/`).

## Advanced Ingress Topics and Gotchas

### Running Multiple Ingress Controllers

To run multiple Ingress controllers on a single cluster, you specify which Ingress object is meant for which Ingress controller using the `[kubernetes.io/ingress.class](http://kubernetes.io/ingress.class)` annotation on the controller and objects.

If the annotation is missing, the behaviour is undefined. It is likely that multiple controllers will fight to satisfy the Ingress and write the `status` field of the Ingress objects.

### Multiple Ingress Objects

If you specify multiple Ingress objects, the Ingress controllers should read them all and try to merge them into a coherent configuration.

If you specify duplicate and conflicting configurations, the behaviour is undefined.

### Ingress and Namespaces

An Ingress object can only refer to an upstream service in the same namespace. You can't use an Ingress object to point to a subpath in another namespace.

Multiple Ingress objects in different namespaces can specify subpaths for the same host. These objects are merged together to come up with the final config for the Ingress controller.

This means that Ingress must be coordinated globally across the cluster, otherwise Ingress objects in one namespace could cause undefined behaviour in other namespaces.

### Path Rewriting

Some Ingress controller implementations support path rewriting, modifying the HTTP request as it gets proxied.

This is usually specified by an annotation on that Ingress object and applies to all requests specified by that objet. E.g. with NGINX Ingress controller, the annotation is:
`nginx.ingress.kubernetes.io/rewrite-target: 

This can make upstream services work on paths they weren't built to handle.

Path rewriting can lead to bugs. Many web applications assume that they can link within themselves using absolute paths. The app in question may be hosted on `/subpath` but have requests show up to it on `/`. It may then send a user to `/app-path`. There is then the question of whether that is an internal link for the app (in which case it should be `/subpath/app-path`), or a link to some other app.

It is best to avoid subpaths if you can help it.

### Serving TLS

First, users need to specify a secret with their TLS certificate and keys.

*Example: tls-secret.yaml*

---

```yaml
apiVersion: v1
kind: Secret
metadata:
	createTimestamp: null
	name: tls-secret-name
type: kubernetes.io/tls
data:
	tls.crt: <base64 encoded certificate>
	tls.key: <base64 encoded private key>
```

You can also specify a secret imperatively:

`kubectl create secret tls <secret-name> --cert <cert-file> --key <private-key-file>`

Once the secret is uploaded, you can reference it in an Ingress object. This specifies a list of certificates along with the hostnames that those certificates should be used for.

If multiple Ingress objects specify certificates for the same hostname, the behaviour is undefined.

*Example: tls-ingress.yaml*

---

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
	name: tls-ingress
spec:
	tls:
	- hosts:
		- alpaca.example.com
		secretName: tls-secret-name
	rules:
	- host: alpaca.example.com
		http:
			paths:
			- backend:
				serviceName: alpaca
				servicePort: 8080
```

There is a non-profit called [Let's Encrypt](https://letsencrypt.org) running a free Certificate Authority that is API-driven, making it possible to set up a Kubernetes cluster that automatically fetches and installs TLS certificates for you.

The missing piece is an open source project called `[cert-manager](https://github.com/jetstack/cert-manager)`.

## Alternate Ingress Implementations

Each cloud provider has an Ingress implementation that exposes the specific cloud-based L7 load balancer for that cloud. Instead of configuring a software load balancer running in a Pod, these controllers take Ingress objects and use them to configure, via an API, the cloud-based load balancers.

The most popular generic Ingress controller is the [NGINX ingress controller](https://github.com/kubernetes/ingress-nginx).

# Chapter 9: ReplicaSets

Replication is desirable because:

- Redundancy: Multiple running instances mean failure can be tolerated.
- Scale: Multiple running instances mean that more requests can be handled.
- Sharding: Different replicas can handle different parts of a computation in parallel.

A ReplicaSet acts as a cluster-wide Pod manager, ensuring that the right types and number of Pods are running at all times.

ReplicaSets are the building blocks used to describe common application deployment patterns and provide the underpinnings of self-healing for our applications at the infrastructure level.

The act of managing the replicated Pods is an example of a **reconciliation loop***.* Such loops are fundamental to the design and implementation of Kubernetes.

## Reconciliation Loops

The reconciliation loop is constantly running, observing the current state of the world and taking action to try to make the observed state match the desired state.

## Relating Pods and ReplicaSets

Though ReplicaSets create and manage Pods, they do not own the Pods they create.

ReplicaSets use label queries to identify the set of Pods they should be managing.

ReplicaSets that create multiple Pods and the services that load-balance to those Pods are also totally separate, decouple API objects.

### Quarantining Containers

When a server misbehaves, Pod-level health checks will automatically restart that Pod. But if your health checks are incomplete, a Pods can be misbehaving but still be part of the replicated set.

In these situations, while it would work to simply kill the Pod, that would leave devs with only logs to debug the problem.

Instead, you can modify the set of labels on the sick Pod. Doing so will dissociate it from the ReplicaSet (and service) so that you can debug the Pod.

## Designing with ReplicaSets

ReplicaSets are designed to represent a single, scalable microservice inside your architecture.

The key characteristic of ReplicaSets is that every Pod that is created by the ReplicaSet controller is entirely homogeneous. Typically, these Pods are then fronted by a Kubernetes service load balancer, which spreads traffic across the Pods that make up the service.

Generally speaking, ReplicaSets are designed for stateless or near-stateless services.

## ReplicaSet Spec

*Example: kuard-rs.yaml*

---

```yaml
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
	name: kuard
spec:
	replicas: 1
	template:
		metadata:
			labels:
				name: kuard
				version: "2"
		spec:
			containers:
				- name: kuard
					image: "gcr.io/kuar-demo/kuard-amd64:green"
```

### Labels

ReplicaSets monitor cluster state using a set of Pod labels, which are used to filter Pod listings and track Pods running within a cluster.

The labels used for filtering are defined in the ReplicaSet `spec` section and are key to understand how ReplicaSets work.

## Inspecting a ReplicaSet

### Finding a ReplicaSet from a Pod

The ReplicaSet controller adds an annotation to every Pod that it creates:
`kubernetes.io/created-by`.

Such annotations are best-effort. They are only created when the Pod is created by the ReplicaSet, and can be removed by a Kubernetes user at any time.

## Scaling ReplicaSets

ReplicaSets are scaled up or down by updating the `spec.replicas` key on the ReplicaSet object stored in Kubernetes.

### Imperative Scaling with `kubectl scale`

To scale up to four replicas, you could run: `kubectl scale replicasets kuard --replicas=4`

### Autoscaling a ReplicaSet

**Horizontal Pod Autoscaling** (HPA) requires the presence of the `heapster` Pod on your cluster. `heapster` keeps track of metrics and provides an API for consuming metrics that HPA uses when making scaling decisions. You can validate its presence by listing Pods in the `kube-system` namespace: `kubectl get pods --namespace=kube-system`.

Kubernetes makes a distinction between HPA, which involves creating additional replicas of a Pod, and vertical scaling, which involves increasing the resources required for a particular Pod (e.g., increasing the CPU required for the Pod).

- Vertical scaling is not currently implemented but is planned.

Many solutions also enable enable cluster autoscaling, where the number of machines in the cluster is scaled in response to resource needs.

**Autoscaling Based on CPU**

Scaling based on CPU usage is the most common use case for Pod autoscaling. Generally it is most useful for request-based systems that consume CPU proportionally to the number of requests they are receiving, while using a relatively static amount of memory.

To scale a ReplicaSet, you can run a command like:
`kubectl autoscale rs kuard —min=2 —max=5 —cpu-percent=80`.

This creates an autoscaler that scales between two and five replicas with a CPU threshold of 80%.

To get autoscalers: `kubectl get hpa`.

It's a bad idea to combine both autoscaling and imperative or declarative management of the number of replicas. If both you can an autoscaler are attempting to modify the number of replicas, it's highly likely you will clash, resulting in unexpected behaviour.

## Deleting ReplicaSets

`kubectl delete rs kuard`

By default, this also deletes Pods managed by the ReplicaSet. If you don't want this, set 
`--cascade=false`.

# Deployments

The Deployment object exists to manage the release of new versions.

- Enable easy movement from one version of code to the next.
- Waits for a user-configurable amount of time between upgrading individual Pods.
- Uses health checks to ensure that the new version of the application is operating correctly.

The actual mechanics of the software rollout performed by a deployment is controlled by a deployment controller that runs in the Kubernetes cluster itself. You can let a deployment proceed unattended  and it will operate safely and correctly.

## Your First Deployment

*Example: kuard-deployment.yaml*

---

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
	name: kuard
spec:
	selector:
		matchLabels:
			run: kuard
	replicas: 1
	template:
		metadata:
			labels:
				run: kuard
		spec:
			containers:
			- name: kuard
				image: gcr.io/kuar-demo/kuard-amd64:blue
```

### Deployment Internals

Just as we learned that ReplicaSets manage Pods, deployments manage ReplicaSets.

This relationship is defined by labels and a label selector. You can see the selector by looking at the Deployment object:

`kubectl get deployments kuard -o jsonpath --template {.spec.selector.matchLabels}`

We can resize the deployment with the imperative `scale` command:

`kubectl scale deployments kuard --replicas=2`

If you were to scale the underlying ReplicaSet, the number of replicas would remain unchanged because the top-level Deployment object is managing the ReplicaSet. The deployment controller takes action to return the number of replicas to the desired state.

To manage the ReplicaSet directly, you need to delete the deployment (setting `--cascade=false` to avoid deleting the Pods).

## Creating Deployments

To download a deployment (deprecated):
`kubectl get deployments kuard —export -o yaml > kuard-deployment.yaml`.

When replacing a deployment, the `--save-config` flag adds an annotation so that, when `apply`ing changes in the future, `kubectl` will know what the last applied configuration was for smarter merging of configs.

If you always use `kubectl apply`, this step is only required after the first time you create a deployment using `kubectl create -f`.

## Managing Deployments

See deployment details with: `k describe deployments kuard`.

The most important output is `OldReplicaSets` and `NewReplicaSet`. These point to ReplicaSet objects this deployment is currently managing. If the deployment is in the middle of a rollout, both fields will be set to a value. If a rollout is complete, `OldReplicaSets` will be set to `<none>`.

Use `kubectl rollout history deployment kuard` to obtain the history of rollouts for a deployment.

Use `kubectl rollout status ...` to obtain the current status of a rollout.

### Updating a Container Image

A common use case for updating a deployment is to roll out a new version of the software running in one or more containers. To do this, update the container image in the YAML file, and update or add the `[kubernetes.io/change-cause](http://kubernetes.io/change-cause)` annotation.

Don't update `change-cause` when doing simple scaling operations. A modification of `change-cause` is a significant change to the template and will trigger a new rollout.

After you update the deployment, it will trigger a rollout, which you can then monitor via the `kubectl rollout` command:

`kubectl rollout status deployments kuard`.

You can see the old and new ReplicaSets managed by the deployment along with the images being used. Both old and new ReplicaSets are kept around in case you want to roll back:

`kubectl rollout get replicasets -o wide`.

If you are in the middle of a rollout and need to temporarily pause it (e.g. seeing weird behaviour in the system), you can run `kubectl rollout pause deployments kuard`.

To resume, `kubectl rollout resume deployments kuard`.

### Rollout History

`kubectl rollout history deployment kuard`

For more details about a particular revision, add the revision flag, e.g. `--revision=2`.

To roll back to a previous release: `kubectl rollout undo deployments kuard`.

The `undo` command works regardless of the stage of the rollout.

When using declarative files to control your production systems, you want to, as far as possible, ensure that the checked-in manifests match what is actually running in your cluster. When you do a `kubectl rollout undo`, you are updating the production state in a way that isn't reflected in source control.

An alternative (preferred) way to undo a rollout is to revert your YAML file and `kubectl apply` the previous version.

When you roll back to a previous revisions, the deployment simply reuses the template and renumbers it so that it is the latest revision. What was revision 2 before could be reordered into revision 4.

You can roll back to a specific revision using the `--to-revision` flag.

Specifying a revision of 0 is a shorthand way of specifying the previous revision.

By default, the complete revision history of a deployment is kept attached to the Deployment object itself. Over time, this history can grow fairly large, so it is recommend that you set a maximum history size for the deployment revision history to limit the total size of the Deployment object. (The `revisionHistoryLimit` property in the deployment specification).

## Deployment Strategies

### Recreate Strategy

`Recreate` simply updates the ReplicaSet it manages to use the new image and terminates all of the Pods associated with the deployment.

This will almost certainly result in downtime. The `Recreate` strategy should only be used for test deployments where a service is not user-facing and a small amount of downtime is acceptable

### RollingUpdate Strategy

The `RollingUpdate` strategy updates a few Pods at a time, moving incrementally until all of the Pods are running the new version of your software.

**Managing Multiple Versions of your Service**

For a period of time, both the new and the old version of your service will be receiving requests and serving traffic. It is critically important that each version of your software, and all of its clients, is capable of talking interchangeably with both a slightly older and a slightly newer version of your software.

This doesn't just applying to JavaScript libraries – the same thing is true of client libraries that are compiled into other services that make calls to your service. Just because you updated doesn't mean they have updated their client libraries. This sort of backward compatibility is critical in decoupling your service from systems that depend on your service. If you don't formalize your APIs and decouple yourself, you are forced to carefully manage your rollouts with all of the other systems that call into your service.

**Configure a Rolling Update**

The `maxUnavailable` parameter sets the maximum number of Pods that can be unavailble during a rolling update.

It can be set as an absolute number or a percentage.

At its core, `maxUnavailable` helps tune how quickly a rolling update proceeds.

Using reduced capacity to achieve a successful rollout is useful either when your service has cyclical traffic patterns (e.g., much less traffic at night) or when you have limited resources, so scaling to larger than the current maximum number of replicas isn't possible.

`maxSurge` is for situations where you don't want to fall below 100% capacity.

Setting `maxSurge` to 100% is equivalent to a blue/green deployment.

### Slowing Rollouts to Ensure Service Health

The purpose of a staged rollout is to ensure that the rollout results in a healthy, stable service running the new software version. To do this, the deployment controller always waits until a Pod reports that it is ready before moving on to updating the next Pod.

The deployment controller examines the Pod's status as determined by its readiness checks. Without these checks, the deployment controller is running blind.

In most real-word scenarios, you want to wait a period of time to have high confidence that the new version is operating correctly before you move on to updating the next Pod. This is defined by the `minReadySeconds` parameter.

```yaml
...
spec:
	minReadySeconds: 60
...
```

You also want to set a timeout that limits how long the system will wait. If the new version of your service has a big and immediately deadlocks, it will never become ready and the deployment controller will stall your rollout forever.

```yaml
...
spec:
	progressDeadlineSeconds: 600
...
```

Progress is defined as any time the deployment creates or deletes a Pod. When that happens, the timeout clock is reset to zero.

## Deleting a Deployment

By default, deleting a deployment deletes the entire service. It will delete any ReplicaSets and Pods under management unless `--cascade=false`.

## Monitoring a Deployment

The status of a deployment can be obtained from the `status.conditions` array, where the will be a `Condition` whose `Type` is `Progressing` and whose `Status` is `False` (in the case of a failed' deployment).

# Chapter 11: DaemonSets

A DaemonSet ensures a copy of a Pod is running across a set of nodes in a Kubernetes cluster.

DaemonSets are used to deploy system daemons such as log collectors and monitoring agents, which typically must run on every node.

ReplicaSets should be used when your application is completely decoupled from the node and you can run multiple copies on a given node without special consideration.

DaemonSets should be used when a single copy of your application must run on all or a subset of the nodes in the cluster.

You can use labels to run DaemonSet Pods on specific nodes; for example, you may want to run specialized intrusion-detection software on nodes that are exposed to the edge network.

You can also use DaemonSets to install software on nodes in a cloud-based cluster. For many cloud services, an upgrade or scaling of a cluster can delete and/or recreate new virtual machines. This dynamic **immutable infrastructure** approach can cause problems if you want to have specific software on every node. To ensure that specific software is installed on every machine despite upgrades and scale events, a DaemonSet is the right approach.

## DaemonSet Scheduler

By default, a DaemonSet will create a copy of a Pod on every node unless a node selector is used, which will limit eligible nodes to those with a matching set of labels.

DaemonSets determine which node a Pod will run on at Pod creation time by specifying the `nodeName` field in the Pod spec.

Pods created by DaemonSets are ignored by the Kubernetes scheduler.

Like ReplicaSets, DaemonSets are managed by a reconciliation control loop that measures the desired state (a Pod is present on all nodes) with the observed state (is the Pod present on a particular node?).

## Creating DaemonSets

DaemonSets are created by submitted a DaemonSet configuration to the Kubernetes API server. This example creates a `fluentd` logging agent on every node in the target cluster.

*Example: fluentd.yaml*

---

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
	name: fluentd
	labels:
		app: fluentd
spec:
	selector:
		matchLabels:
			app: fluentd
	template:
		metadata:
			labels:
				app: fluentd
		spec:
			containers:
			- name: fluentd
				image: fluentd/fluentd:v0.14.10
				resources:
					limits:
						memory: 200Mi
					requests:
						cpu: 100m
						memory: 200Mi
				volumeMounts:
				- name: varlog
					mountPath: /var/log
				- name: varlibdockercontainers
					mountPath: /var/lib/docker/containers
					readOnly: true
			terminationGracePeriod: 30
			volumes:
			- name: varlog
				hostPath:
					path: /var/log
			- name: varlibdockercontainers
				hostPath:
					path: /var/lib/docker/containers
```

DaemonSets require a unique name across all DaemonSets in a given namespace.

Each DaemonSet must include a Pod template spec, which will be used to create Pods as needed.

## Limiting DaemonSets to Specific Nodes

### Adding Labels to Nodes

The first step in limiting DaemonSets to specific nodes is to add the desired set of labels to a subset of nodes:

`kubectl label nodes k0-default-pool-35609c18 ssd=true.`

### Node Selectors

This DaemonSet configuration limits NGINX to running only on nodes with `ssd=true`.

*Example: nginx-fast-storage.yaml*

---

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
	labels:
		app: nginx
		ssd: "true"
	name: nginx-fast-storage
spec:
	selector:
		matchLabels:
			app: nginx
	template:
		metadata:
			labels:
				app: nginx
				ssd: "true"
	spec:
		nodeSelector:
			ssd: "true"
		containers:
			- name: nginx
				image: nginx:1.10.0
```

Removing labels from a node that are required by a DaemonSet's node selector will cause the Pod being managed by that DaemonSet to be removed from the node.

## Updating a DaemonSet

### Rolling Update of a DaemonSet

DaemonSets can be rolled out using the same RollingUpdate strategy that deployments use.

You can configure the update strategy using the `spec.updateStrategy.type` field, which should have the value `RollingUpdate`.

When a DaemonSet has an update strategy of `RollingUpdate`, any change to the `spec.template` field (or subfields) in the DaemonSet will initiate a rolling update.

Two parameters control the rolling update of a `DaemonSet`:

- `spec.minReadySeconds`: determines how long a Pod must be ready before the rolling update proceeds to upgrade subsequent Pods.
- `spec.updateStrategy.rollingUpdate.maxUnavailable`: indicates how many Pods may be simultaneously updated by the rolling update.
    - Setting it to 1 is a safe, general-purpose strategy, but takes number of nodes * `minReadySeconds`.
    - A good strategy might be to set `maxUnavailable` to 1 and only increase it if users or administrators complain about DaemonSet rollout speed.

Once a rolling update has started, you can use the `kubectl rollout` commnads to see teh current status of a DaemonSet rollout.

## Deleting a DaemonSet

Deleting a DaemonSet will also delete all Pods being managed by that DaemonSet unless `--cascase=false`.

# Chapter 12: Jobs

Jobs are useful for things you only want to do once, such as database migrations or batch jobs.

If run as a regular Pod, your database migration task would run in a loop, continually repopulating the database after every exit.

## The Job Object

If the Pod fails before successful termination, the job controller will create a new Pod based on the Pod template in the job specification.

Given that Pods have to be scheduled, there is a change that your job will not execute if the required resources are not found by the scheduler.

Due to the nature of distributed systems, there is a small chance, during certain failure scenarios, that duplicate Pods will be created for a specific task.

## Job Patterns

Jobs are designed to manage batch-like workloads where work items are processed by one or more Pods.

Job patterns are defined by two attributes:

- `completions`: the number of times to successfully run a task;
- `parallelism`: the number of pods to tackle the task in parallel.

[Job patterns](https://www.notion.so/b254bed2c2164e78a70a3484d5933802)

### One Shot

First, a Pod must be created and submitted to the Kubernetes API using a Pod template defined in the job configuration.

Once a job is running, the Pod backing the job must be monitored for successful termination.

The job controller is responsible for recreating the Pod until a successful termination occurs.

The easiest way to create a one-shot job is with the `kubectl` command-line tool:

```bash
kubectl run -i oneshot \
--image=gcr.io/kuar-demo/kuard-amd64:blue \
--restart=OnFailure \
-- --keygen-enable \
   --keygen-exit-on-complete \
   --keygen-num-to-gen 10
```

- The `-i` option indicates that this is an interactive command. `kubectl` will wait until the job is running and then show log output from the first Pod in the job.
    - Note that `kubectl` often misses the first couple of lines of output with the `-i` option.
- `--restart=onFailure` is the option that tells `kubectl` to create a Job object.
- All of the options after `--` are command-line arguments to the container image. These instruct our test server (`kuard`) to generate 10 4,096-bit SSH keys and then exit.

After the job has completed, the Job object and related Pod are still around so that you can inspect the log output.

The other option for creating a one-shot job is using a configuration file:

*Example: job-oneshot.yaml*

---

```yaml
apiVersion: batch/v1
kind: Job
metadata:
	name: oneshout
spec:
	template:
		spec:
			containers:
				- name: kuard
					image: gcr.io/kuar-demo/kuard-amd64:1
					imagePullPolicy: Always
					args:
						- "--keygen-enable"
						- "--keygen-exit-on-completion"
						- "--keygen-num-to-gen=10"
			restartPolicy: OnFailure
```

You can view the results of the job by looking at the logs of the Pod that was created:
`kubectl logs oneshot-4kfdt`.

Because jobs have a finite beginning and ending, it is common for users to create many of them . This makes picking unique labels more difficult and more critical. For this reason, the Job object will automatically pick a unique label and use it to identify the Pods it creates.

**Pod Failure**

To see failing jobs: `kubectl get pod -l job-name=oneshot`.

Status `CrashLoopBackOff` indicates that the Pod is restarting because of a problem.

Kubernetes waits a little between restarts to avoid a crash loop eating resources on the node. This all handled local to the node by the `kubelet` without the job being involved at all.

If `restartPolicy` is set to `Never`, the `kubelet` is instructed not to restart the Pod on failure, but rather just declare the Pod as failed. The Job object then notices and creates a replacement Pod. This can create a lot of junk in the cluster, so `restartPolicy: OnFailure` is recommended to rerun failed Pods in-place.

Workers can get stuck and not make any forward progress. To handle this case, use liveness probes.

### Parallelism

To generate 100 keys by having 10 runs of `kuard` with each run generating 10 keys without swamping the cluster:

*Example: job-parallel.yaml*

```yaml
apiVersion: batch/v1
kind: Job
metadata:
	name: parallel
	labels:
		chapter: jobs
spec:
	parallelism: 5
	completions: 10
	template:
		metadata:
			labels:
				chapter: jobs
		spec:
			containers:
				- name: kuard
					image: gcr.io/kuar-demo/kuard-amd64:1
					imagePullPolicy: Always
					args:
						- "--keygen-enable"
						- "--keygen-exit-on-complete"
						- "--keygen-num-to-gen=10"
			restartPolicy: OnFailure
```

### Work Queues

A common use case for jobs is to process work from a work queue. In this scenario, some task creates a number of work items and publishes them to a work queue. A worker job can be run to process each work item until the work queue is empty.

**Starting a Work Queue**

We start by launching a centralized work queue service (`kuard` has a simple memory-based work queue system built in).

Next, we create a simple ReplicaSet to manage singleton work queue daemon. We are using a ReplicaSet to ensure that a new Pod will get created in the face of machine failure.

*Example: rs-queue.yaml*

---

```yaml
apiVersion: apps/v1beta1
kind: ReplicaSet
metadata:
	labels:
		app: work-queue
		component: queue
		chapter: jobs
	name: queue
spec:
	replicas: 1
	template:
		metadata:
			labels:
				app: work-queue
				component: queue
				chapter: jobs
		spec:
			containers:
				- name: queue
					image: gcr.io/kuar-demo/kuard-amd64:1
					imagePullPolicy: Always
```

Connect to the running queue daemon with:

```bash
QUEUE_POD=$(kubectl get pods -l app=work-queue,component=queue \
	-o jsonpath='{.items[0].metadata.name}')
kubectl port forward $QUEUE_POD 8080:8080
```

With the work queue server in place, expose it using a service. This makes it easy for producers and consumers to locate the work queue via DNS.

*Example: service-queue.yaml*

---

```yaml
apiVersion: v1
kind: Service
metadata:
	labels:
		app: work-queue
		component: queue
		chapter: jobs
	name: queue
spec:
	ports:
		- port: 8080
			protocol: TCP
			targetPort: 8080
	selector:
		app: work-queue
		component: queue
```

**Loading Up the Queue**

We are now ready to put items in the queue. For simplicity, we'll just use `curl` to drive the API for the work queue server and insert a bunch of work itself.

- `curl` will communicate to the work queue through the `kubectl port-forward` we set up earlier.

*Example: load-queue.sh*

---

```bash
# Create a work queue called 'keygen'.
curl -X PUT localhost:8080/memq/server/queues/keygen

# Create 100 work items and load up the queue.
for i in work-item-{0..99}; do
	curl -X POST localhost:8080/memq/server/queues/keygen/enqueue \
		-d "$i"
done
```

**Creating the Consumer Job**

`kuard` is also able to act in consumer mode. We can set it to draw work items from the work queue, create a key, and then exit once the queue is empty.

*Example: job-consumers.yaml*

---

```yaml
apiVersion: batch/v1
kind: Job
metadata:
	labels:
		app: message-queue
		component: consumer
		chapter: jobs
	name: consumers
spec:
	parallelism: 5
	template:
		metadata:
			labels:
				app: message-queue
				component: consumer
				chapter: jobs
		spec:
			containers:
				- name: worker
					image: gcr.io/kuar-demo/kuard-amd64:1
					imagePullPolicy: Always
					args:
						- "--keygen-enable"
						- "--keygen-exit-on-complete"
						- "--keygen-memq-server=http://queue:8080/memq/server"
						- "--keygen-memq-queue=keygen"
			restartPolicy: OnFailure
```

Here we are telling the job to start up five Pods in parallel.

As the `completions` parameter is unset, we put the job into a worker pool mode.

Once the first Pod exits with a zero exit code, the job will start winding down and will not start any new Pods. This means that none of the workers should exit until the work is done and they are all in the process of finishing up.

**Cleaning Up**

Using labels, we can delete everything created in this chapter:

`kubectl delete rs,svc,job -l chapter=jobs`.

## CronJobs

To schedule a job at a certain interval, declare a CronJob.

*Example: cron-job.yaml*

---

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
	name: example-cron
spec:
	# run every fifth hour
	schedule: "0 */5 * * *"
	jobTemplate:
		spec:
			template:
				spec:
					containers:
						- name: batch-job
							image: my-batch-image
					restartPolicy: OnFailure
```

# Chapter 13: ConfigMaps and Secrets

It is good practice to make container images as reusable as possible. The same image should be able to be used for development, staging, and production. It is even better if the same image is general-purpose enough to be used across applications and services. Testing and versioning get riskier and more complicated if images need to be recreated for each new environment.

ConfigMaps are used to provide configuration information for workloads.

- Can be fine-grained, in the form of a string, or a configuration file.

Secrets are similar but focused on making sensitive information available to the workload.

- Used for things like credentials or TLS certificates.

## ConfigMaps

A ConfigMap can be thought of as a Kubernetes object that defines a small filesystem, or a set of variables that can be used when defining the environment or command line for your containers.

The ConfigMap is combined with the Pod right before it is run. This means the container image and the Pod definition itself can be reused across many apps by just changing the ConfigMap.

### Creating ConfigMaps

Imperative:

*Example: my-config.txt*

---

```
# This is a sample config file that I might use to configure an application.

parameter1 = value1
parameter2 = value2
```

Next, create a ConfigMap with the file while adding a couple of additional keys. These are referred to as `literal` values on the command line:

```bash
kubectl create configmap my-config \
--from-file=my-config.txt \
--from-literal=extra-param=extra-value \
--from-literal=another-param=another-value
```

YAML:

*Example: yaml-config.yaml*

---

```bash
apiVersion: v1
data:
	extra-param: extra-value
	another-param: another-value
	my-config.txt: |
		# This is a sample config file that I might use to configure an application.
		parameter1 = value2
		parameter2 = value2
kind: ConfigMap
metdata:
	creationTimestamp: ...
	name: my-config
	namespace: default
	resourceVersion: "13556"
	selfLink: /api/v1/namespaces/default/configmaps/my-config
	uid: <uid> 
```

A ConfigMap is really just some key-value pairs stored in an object.

### Using a ConfigMap

There are three main ways to use a ConfigMap:

- Filesystem: You can mount a ConfigMap into a Pod. A file is created for each entry based on the key name. The contents of the file are set to the value.
- Environment variable: A ConfigMap can be used to dynamically set the value of an environment variable.
- Command-line argument: Kubernetes supports dynamically creating the command line for a container based on ConfigMap values.

The following manifest pulls all three approaches together:

*Example: kuard-config.yaml*

---

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: kuard-config
spec:
	containers:
		- name: test-container
			image: gcr.io/kuar-demo/kuard-amd64:blue
			imagePullPolicy: Always
			command:
			  - "/kuard"
				- "$(EXTRA_PARAM)"
			env:
				- name: ANOTHER_PARAM
					valueFrom:
						configMapKeyRef:
							name: my-config
							key: another-param
				- name: EXTRA_PARAM
					valueFrom:
						configMapKeyRef:
							name: my-config
							key: extra-param
			volumeMounts:
				- name: config-volume
					mountPath: /config
	volumes:
		- name: config-volume
			configMap:
				name: my-config
	restartPolicy: Never
```

- **Filesystem**: we create a new volume inside the Pod and give it the name `config-volume`. We define this volume to be a `ConfigMap` volume and point at the ConfigMap to mount.
    - We specify where this gets mounted into the `kuard` container with a `volumeMount`.
- **Environment** **variables** are specified with the `valueFrom` member. This references the ConfigMap and the data key to use within that ConfigMap.
- **Command-line arguments** build on environment variables. Kubernetes will perform the correct substitution with the `$(<env-var-name>)` syntax.

## Secrets

Secrets enable container images to be created without bundling sensitive data. This allows containers to remain portable across environments.

Secrets are exposed to Pods via explicit declaration in Pod manifests and the Kubernetes API. In this way, the Kubernetes secrets API provides an application-centric mechanism for exposing sensitive configuration information to applications in a way that's easy to audit and leverages native OS isolation primitives.

By default, secrets are stored in plain text in the `etcd` storage for the cluster. Anyone who has administration rights in your cluster will be able to read them. In recent versions, support has been added for encrypting the secrets with a user-supplied key, generally integrated into a cloud key store.

Most cloud key stores have integration with Kubernetes flexible volumes, enabling you to skip Kubernetes secrets entirely and rely exclusively on the cloud provider's key store.

### Creating Secrets

The first step is to obtain the raw data we want to store:

```bash
curl -o kuard.crt https://storage.googleapis.com/kuar-demo/kuard.crt
curl -o kuard.key https://storage.googleapis.com/kuar-demo/kuard.key
```

With the secret `crt` and `key` files stored locally, create a secret using the `create secret` command:

```bash
kuard create secret generic kuard-tls \
--from-file=kuard.crt \
--from-file=kuard.key
```

- Creates the `kuard-tls` secret with two data elements.
- Run `kuard describe secrets kuard-tls` for details.
- 

### Consuming Secrets

Secrets can be consumed using the Kubernetes REST API by applications that know how to call the API directly.

However, to keep applications portable, we can use a **secrets volume**.

**Secrets Volumes**

Secrets volumes are manages by the `kubelet` and are created at Pod creation time. Secrets are stored on tmpfs volumes (aka RAM disks), and are not written to disk on nodes.

Each data element of a secret is stored in a separate file under the target mount point specified in the volume mount.

The `kuard-tls` secret (above) contains two data elements. Mounting the `kuard-tls` secrets volumes to `/tls` results in the following files:

- `/tls/kuard.crt`
- `/tls/kuard.key`

*Example: kuard-secret.yaml*

---

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: kuard-tls
spec:
	containers:
		- name: kuard-tls
			image: gcr.io/kuar-demo/kuard-amd64:blue
			imagePullPolicy: Always
			volumeMounts:
				- name: tls-certs
					mountPath: "/tls"
					readOnly: true
	volumes:
		- name: tls-certs
			secret:
				secretName: kuard-tls
```

### Private Docker Registries

A special use case for secrets is to store access credentials for private Docker registries.

**Image pull secrets** leverage the secrets API to automate the distribution of private registry credentials.

Image pull secrets are stored like normal secrets but are consumed through the `spec.imagePullSecrets` Pod specification field.

Use `create secret docker-registry` to create image pull secrets:

```bash
kubectl create secret docker-registry my-image-pull-secret \
--docker-username=<username> \
--docker-password=<password> \
--docker-email=<email-address>
```

Enable access to the private repository by referencing the image pull secret in the Pod manifest file.

*Example: kuard-secret-ips.yaml*

---

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: kuard-tls
spec:
	containers:
		- name: kuard-tls
			image: gcr.io/kuard-demo/kuard-amd64:blue
			imagePullPolicy: Always
			volumeMounts:
				- name: tls-certs
					mountPath: /tls
					readOnly: true
	imagePullSecrets:
		- name: my-image-pull-secret
	volumes:
		- name: tls-certs
			secret:
				secretName: kuard-tls
```

## Naming Constraints

Key names may begin with a dot followed by a letter or number. Following characters include dots, dashes, and underscores. Dots cannot be repeated and dots and underscores or dashes cannot be adjacent to each other.

When selecting a key name, consider that these keys can be exposed to Pods via a volume mount. Pick a name that is going to make sense when specified on a command line or in a config file. Storing a TLS key as `key.pem` is more clear than `tls-key` when configuring applications to access secrets.

ConfigMap data values are simple UTF-8 text specified directly in the manifest.

Secret data values hold arbitrary data encoded using base64. The use of base64 encoding makes it possible to store binary data.

- This makes it more difficult to manage secrets that are stored in YAML files as the base64-encoded value must be put in the YAML.

The maximum size for a ConfigMap or secret is 1 MB.

## Managing ConfigMaps and Secrets

### Creating

The various wats to specify the data items that go into the secret or ConfigMap can be combined into single commands:

- `--from-file=<filename>`: Load from the file with the secret data key the same as the filename.
- `--from-file=<key>=<filename>`: Load from the file with the secret data key explicitly specified.
- `--from-file=<directory>`: Load all the files in the specified directory where the filename is an acceptable key name.
- `--from-literal=<key>=<value>`: Use the specified key-value pair directly.

### Updating

You can update a ConfigMap or secret and have it reflected in running programs. There is no need to restart if the application is configured to reread configuration values.

**Update From File**

If you have a manifest for your ConfigMap or secret, you can just edit it directly and push a new version with `kubectl replace -f <filename>`.

- You can also use `kubectl apply -f <filename>` if you previously created the resource with `kubectl apply`.

Due to the way that datafiles are encoded into these objects, updating a configuration can be a bit cumbersome as there is no provision in `kubectl` to load data from an external file. The data must be stored directly in the YAML manifest.

The most common use case is when the ConfigMap is defined as part of a directory or list of resources and everything is created and updated together. Often these manifests will be checked into source control.

**Recreate and Update**

If you store the inputs into your ConfigMaps or secrets as separate files on disk (as opposed to embedded into YAML directly), you an use `kubectl` to recreate the manifest and then use it to update the object:

```bash
kubectl create secret generic kuard-tls \
--from-file=kuard.crt --from-file=kuard.key \
--dry-run -o yaml | kubectl replace -f -
```

- By dumping the output of `create secret` to `stdout` and piping it to `kubectl replace` (telling it to read from `stdin` with `-f -`, we can update a secret from files on disk without having to manually base64-encode data.

**Edit Current Version**

The final way to update a ConfigMap is to use `kubectl edit` to bring up a version of the ConfigMap in your editor: `kubectl edit configmap my-config`.

- You could do this with a secret, but you'd be stuck managing the base64 encoding of values on your own.

**Live Updates**

Once of ConfigMap or secret is updated using the API, it'll be automatically pushed to all volumes that use that ConfigMap or secret. It may take a few seconds, but the file listing and contents of the files will be updated with these new values.

- Using this, you can update the configuration of applications without restarting them.

There is no built-in way to signal an application when a new version of a ConfigMap is deployed. It is up to the application to look for the config files to change and reload them.

# Chapter 14: Role-Based Access Control for Kubernetes

Role-based access control provides a mechanism for restricting both access to and actions on Kubernetes APIs in the cluster.

RBAC is a critical component to both harden access to the Kubernetes cluster where you are deploying your application and prevent unexpected accidents where one person in the wrong namespace mistakenly takes down production when they think they are destroying their test cluster.

RBAC is useful for limiting access to the Kubernetes API, it's important to remember that anyone who can run arbitrary code inside the Kubernetes cluster can effectively obtain root privileges on the entire cluster.

If you are focused on hostile multitenant security, you must isolate the Pods running in your cluster. Generally this is done with a hypervisor isolated container, some sort of container sandbox, or both.

Every request to Kubernetes is first authenticated. It could be as simple as saying that the request is unauthenticated, or it could integrate deeply with a pluggable authentication provider (e.g. Azure Active Directory) to establish an identity within that third-party system.

Kubernetes does not have a built-in identity store, focusing instead on integrating other identity sources within itself.

Authorization is a combination of the identity of the user, the resource (effectively HTTP path), and the verb or action the user is attempting to perform.

## Role-Based Access Control

### Identity in Kubernetes

Every request that comes to Kubenetes is associated with some identity. Even a request with no identity is associated with the `system:unauthenticated` group.

Kubernetes distinguishes between user identities and service account identities:

- Service accounts are created and managed by Kubernetes itself and are generally associated with components running inside the cluster.
- User accounts are all other accounts associated with actual users of the cluster, and often include automation like continuous delivery as a service that runs outside of the cluster.

Kubernetes uses a generic interface for authentication providers. Each of the providers supplies a username and optionally the set of groups to which the user belongs.

Kubernetes supports a number of different authentication providers, including:

- HTTP Basic Authentication (largely deprecated)
- x509 client certificates
- Static token files on the host
- Cloud authentication providers with AWS IAM
- Authentication webhooks

### Understanding Roles and Role Bindings

A **role** is a set of abstract capabilities. For example, the `appdev` role might represent the ability to create Pods and services.

A **role binding** is an assignment of a role to one or more identities. Binding the `appdev` role to user `alice` indicates that Alice has the ability to create Pods and services.

### Roles and Role Bindings in Kubernetes

`Role` resources are namespaced. You cannot use namespaces roles for non-namespaced resources (e.g. CustomResourceDefinitions), and binding a `RoleBinding` to a role only provides authorization within the Kubernetes namespace that contains both the `Role` and the `RoleDefinition`.

Here is a simple role that gives an identity the ability to create and modify Pods and services:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
	namespace: default
	name: pod-and-services
rules:
	- apiGroups: [""]
		resources: ["pods", "services"]
		verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
```

To bind this role the user `alice` and the group `mydevs`, we need to create a `RoleBinding` as follows:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
	namespace: default
	name: pods-and-services
subjects:
	- apiGroup: rbac.authorization.k8s.io
		kind: User
		name: alice
	- apiGroup: rbac.authorization.k8s.io
		kind: Group
		name: mydevs
roleRef:
	apiGroup: rbac.authorization.k8s.io
	kind: Role
	name: pod-and-services
```

To create a role that applies to the entire cluster, or limit access to cluster-level resources, use the `ClusterRole` and `ClusterRoleBinding` resources.

- Largely identical to namespaces versions but with larger scope.

[Verbs for Kubernetes Roles](https://www.notion.so/05ab0440c2254bb4bf4eea9c24ee8c9a)

**Using Built-In Roles**

Kubernetes has a number of built-in cluster roles. You can view these by running:
`kubectl get clusterroles`.

Most of these roles are for system utilities, but four are designed for generic end users:

- The `cluster-admin` role provides complete access to the entire cluster.
- The `admin` role provides complete access to a namespace.
- The `edit` role allows an end user to modify things in a namespace.
- The `view` role allows for read-only access to a namespace.

Most clusters already have numerous `ClusterRole` bindings set up, and you can view these bindings with `kubectl get clusterrolebindings`.

**Auto-Reconciliation of Built-In Roles**

If you modify any built-in cluster role, those modifications are transient. Whenever the API server is restarted (e.g., for an upgrade) your changes will be overwritten.

To prevent this, add that `[rbac.authorization.kubernetes.io/autoupdate](http://rbac.authorization.kubernetes.io/autoupdate)` annotation with a values of `false` to the built-in `ClusterRole` resource.

By default, the Kubernetes API server installs a cluster role that allows `system:unauthenticated` users access to the API server's API discovery endpoint. For any cluster exposed to a hostile environement (e.g., the public internet), this is a bad idea.

If you are running a Kubernetes services on the public internet or another hostile environment, ensure that the `--anonymous-auth=false` flag is set on your API server.

## Techniques for Managing RBAC

### Testing Authorization with `can-i`

In its simplest usage, `can-i` takes a verb and a resource. E.g., this command will indicate if the current `kubectl` user is authorized to create Pods: `kubectl auth can-i create pods`.

### Managing RBAC in Source Control

RBAC resources are modeled using JSON or YAML. Given this text-based representation, it makes sense to store these resources in version control.

The  strong need for audit, accountability, and rollback for changes to RBAC policy means that version control for RBAC resources is essential.

The `reconcile` command operates like `kubectl apply` and will reconcile a text-based set of roles and role bindings with the current state of the cluster:

`kubectl auth reconcile -f some-rbac-config.yaml`

- To see changes before they are made, use the `--dry-run` flag.

## Advanced Topics

### Aggregating `ClusterRoles`

Kubernetes RBAC supports the usage of an **aggregation rule** to combine multiple roles together in a new role.

- Changes to constituent subroles will automatically be propagated back into the aggregate role.

`CluterRoles` to be aggregated are specified using label selectors: the `aggregationRule` field in the `ClusterRole` resource contains a `clusterRoleSelector` field, which in turn is a label selector.

Best practice for managing `ClusterRole` resources is to create a number of fine-grained cluster roles and than aggregated them together to form higher-level or broadly-defined cluster roles.

This is how built-in cluster roles are defined. E.g. the built-in `edit` role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
	name: edit
	...
aggregationRule:
	clusterRoleSelectors:
	- matchLabels:
			rbac.authorization.k8s.io/aggregate-to-edit: "true"
```

### Using Groups for Bindings

Rather than adding role bindings to specific identities, it is best practice to use groups to manage the roles that define access to the cluster.

When you bind a group to a `ClusterRole` or a namespace `Role`, anyone who is a member of that group gains access to the resources and verbs defined by that role.

To enable any individual to gain access to the group's role, that individual needs to be added to the group.

To bind a group to a `ClusterRole` you can use a `Group` kind for the subject in the binding:

```yaml
...
subjects:
	- apiGroup: rbac.authorization.k8s.io
		kind: Group
		name: my-great-groups-name
...
```

Groups are supplied by authentication providers. There is no strong notion of a group within Kubernetes, only that an identity can be part of one or more groups, and those groups can be associated with a `Role` or `ClusterRole` via a binding.

# Chapter 15: Integrating Storage Solutions and Kubernetes

## Importing External Services

In many cases, you have an existing machine running in your network that has some sort of database running on it. You may not want to immediately move that database into containers and Kubernetes. However, you can still represent this server in Kubernetes.

- Provides built-in naming and service discovery primitives provided by Kubernetes.
- Enables you to configure all applications so it looks like the database is actually a Kubernetes service.

Use namespaces to maintain high fidelity between development and production:

```yaml
kind: Service
metadata:
	name: my-database
	# note 'test' namespace here
	namespace: test
...
```

```yaml
kind: Service
metadata:
	name: my-database
	# note 'prod' namespace here
	namespace: prod
...
```

When you deploy a Pod into the `test` namespace and it looks up the service named `my-database`, it will receive a pointer to `my-database.test.svc.cluster.internal`, which in turn points to the test database.

### Services Without Selectors

With external services, label queries won't work. Instead, you generally have a DNS name that points to the specific service running the database.

*Example: dns-service.yaml*

---

```yaml
kind: Service
apiVersion: 1
metadata:
	name: external-database
spec:
	type: ExternalName
	externalName: database.company.com
```

When a typical service is created, an IP address is also created and the DNS service is populated with an A record that points to that IP address. When you create a service of type `ExternalName`, the DNS service is populated with a CNAME record that points to the external name you specified.

Many cloud databases and other services provide you with a DNS name to use when accessing the database. You can use this as `externalName`.

When you just have an IP address, first you create a Service without a label selector, but also without the `ExternalName` type:

*Example: external-ip-service.yaml*

---

```yaml
kind: Service
apiVersion: v1
metadata:
	name: external-ip-database
```

Kubernetes will allocate a virtual IP address for this service and populate an A record for it. However, because there is no selector for the service, there will be no endpoints populated for the load balancer to redirect traffic to.

The user is responsible for populating the endpoints manually with an Endpoints resource:

*Example: external-ip-endpoints.yaml*

---

```yaml
kind: Endpoints
apiVersion: v1
metadata:
	name: external-ip-database
subsets:
	- addresses:
			- ip: 192.168.0.1
			ports:
				- port: 3306
```

Because the user has assumed responsibility for keeping the IP address of the server up to data, you need to either ensure that it never changes or make sure that some automated process updates the `Endpoints` record.

### Limitations of External Services: Health Checking

External services do not perform health checking. The user is responsible for ensuring the endpoint or DNS name supplied to Kubernetes is as reliable as necessary for the application.

## Running Reliable Singletons

Primitives like ReplicaSet expect that every container is identical and replaceable, but for most storage solutions this isn't the case. One option to address this is to use primitives, but not attempt to replicate the storage. Instead, simply run a single Pod that runs the database or other storage solution.

In general, this is no less reliable than running your database or storage infrastructure on a single virtual or physical machine.

If you structure the system properly, the only thing you sacrifice is potential downtime for upgrades or in case of machine failure.

### Running a MySQL Singleton

We are going to create 3 basic objects:

- A persistent volume to manage the lifespan of the on-disk storage independently from the lifespan of the running MySQL application.
- A MySQL Pod that will run the MySQL application.
- A service that will expose this Pod to other containers in the cluster.

A persistent volume is a storage location that has a lifetime independent of any Pod or container.

There are persistent volume drivers for all major public cloud providers. To use these, simply replace `nfs` with the appropriate cloud provider volume type, e.g. `awsElasticBlockStore`.

*Example: nfs-volume.yaml*

---

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
	name: database
	labels:
		volume: my-volume
spec:
	accessModes:
		- ReadWriteMany
	capacity:
		storage: 1Gi
	nfs:
		server: 192.168.0.1
		path: /exports
```

- Defines an NFS `PersistentVolume` object with 1 GB of storage space.

With the persistent volume created, we need to claim it for our Pod with a `PersistentVolumeClaim`.

*Example: nfs-volume-claim.yaml*

---

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
	name: database
spec:
	accessModes:
		- ReadWriteMany
	resources:
		requests:
			storage: 1Gi
	selector:
		matchLabels:
			volume: my-volume
```

By using volume claims, you can keep your Pod specifications Cloud-agnostic; simply create different volumes, specific to the cloud, and use a `PersistentVolumeClaim` to bind them together.

Use a `ReplicaSet` with `replicas` set to 1 to construct the singleton pod. This ensures that the Pod will be restarted in the event of failure.

*Example: mysql-replicaset.yaml*

---

```yaml
apiVersion: extensions/v1
kind: ReplicaSet
metadata:
	name: mysql
	# labels so that we can bind a Service to this Pod
	labels:
		app: mysql
spec:
	replicas: 1
	selector:
		matchLabels:
			app: mysql
	template:
		metadata:
			labels:
				app: mysql
		spec:
			containers:
				- name: database
					image: mysql
					resources:
						requests:
							cpu: 1
							memory: 2Gi
					env:
						# Environment variables are not best practice for security, but we're
						# using them here for brevity in the example.
						# See Chapter 13 for better options.
						- name: MYSQL_ROOT_PASSWORD
							value: some-password-here
					livenessProbe:
						tcpSocket:
							port: 3306
					ports:
						- containerPort: 3306
					volumeMounts:
						- name: database
							# /var/lib/mysql is where MySQL stores its databases
							mountPath: /var/lib/mysql
			volumes:
				- name: database
					persistentVolumeClaim:
						claimName: database
```

Once created, the ReplicaSet will in turn create a Pod running MySQL using the persistent disk we originally created.

The final step is to expose the Pod using a Service:

*Example: mysql-service.yaml*

---

```yaml
apiVersion: v1
kind: Service
metadata:
	name: mysql
spec:
	ports:
		- port: 3306
			protocol: TCP
	selector:
		app: mysql
```

Now we have a reliable singleton MySQL instance running in our cluster and exposed as a service named `mysql`, which we can access at the full domain name `mysql.svc.default.cluster`.

If your needs are simple and you can survive limited downtime in the face of a machine failure or when you need to upgrade the database software, a reliable singleton may be the right approach to storage for your application.

### Dynamic Volume Provisioning

With dynamic volume provisioning, the cluster operator creates one or more `StorageClass` objects.

*Example: storageclass.yaml*

---

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
	name: default
	annotations:
		storageclass.beta.kubernetes.io/is-default-class: "true"
	labels:
		kubernetes.io/cluster-service: "true"
provisioner: kubernetes.io/azure-disk
```

Once a storage class has been created for a cluster, you can refer to this storage class in your persistent volume claim, rather than referring to any specific persistent volume.

When the dynamic provisioner sees this storage claim, it uses the appropriate volume driver to create the volume and bind it to your persistent volume claim.

This is an example of a `PersistentVolumeClaim` that uses the `default` storage class we just defined to claim a newly created persistent volume:

*Example: dynamic-volume-claim.yaml*

---

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
	name: my-claim
	annotations:
		volume.beta.kubernetes.io/storage-class: default
spec:
	accessModes:
		- ReadWriteOnce
	resources:
		requests:
			storage: 10Gi
```

- The `[volume.beta.kubernetes.io/storage-class](http://volume.beta.kubernetes.io/storage-class)` annotation is what links this class back up to the storage class we created.

The lifespan of these persistent volumes is dictated by the reclamation policy of the `PersistentVolumeClaim` and the default is to bind that lifespan to the lifespan of the Pod that creates the volume. You need to be careful to ensure that you don't accidentally delete your persistent volumes.

## Kubernetes-Native Storage with Stateful Sets

### Properties of StatefulSets

StatefulSets are replicated groups of Pods, similar to ReplicaSets but with certain unique properties:

- Each replica gets a persistent hostname with a unique index (e.g., `database-0`, `database-1`, etc.).
- Each replica is created in order from lowest to highest index, and creation will block until the Pod at the previous index is healthy and available. This also applies to scaling up.
- When a stateful set is deleted, each of the managed replica Pods is also deleted in order from highest to lowest. This also applies to scaling down.

### Manually Replicated MongoDB with StatefulSets

To start, we'll create a replicated set of three MongoDB Pods using a StatefulSet object:

*Example: mongo-simple.yaml*

---

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
	name: mongo
spec:
	serviceName: mongo
	replicas: 3
	template:
		metadata:
			labels:
				app: mongo
		spec:
			containers:
				- name: mongodb
					image: mongo:3.4.1
					command: ["mongod", "--replSet", "rs0"]
					ports:
						- containerPort: 27017
							name: peer
```

Once the StatefulSet is created, we need to create a "headless" service to manage the DNS entries for the StatefulSet.

- A service is called "headless" if it doesn't have a cluster virtual IP address.
- Since with StatefulSets each Pod has a unique identity, it doesn't really make sense to have a load-balancing IP address for the replicated service.

Create a headless service using `clusterIP: None` in the service specification.

*Example: mongo-service.yaml*

---

```yaml
apiVersion: v1
kind: Service
metadata:
	name: mongo
spec:
	ports:
		- port: 27017
			name: peer
	clusterIP: None
	selector:
		app: mongo
```

As usual, `[mongo.default.svc.cluster.local](http://mongo.default.svc.cluster.local)` is created, but unlike with a standard service, doing a DNS lookup on this hostname provides all the addresses in the StatefulSet.

- Entries are also created for `mongo-0.mongo.default...`, `mongo-1.mongo`, etc.

Next, we're going to manually set up Mongo replication using these per-Pod hostnames, choosing `mongo-0.mongo` to be our initial primary. Run the `mongo` tool in that Pod:

```bash
$ kubectl exec -it mongo-0 mongo
> rs.initiate({
		_id: "rs0",
		members: [ { _id: 0, host: "mongo-0.mongo:27017 } ]
	});
```

- This command tells `mongodb` to initiate the ReplicaSet `rs0` with `mongo-0.mongo` as the primary replica.
- The name `rs0` is arbitrary, but must match the StatefulSet definition.

Once you have initiated the Mongo ReplicaSet, you can add the remaining replicas by running the following commands in the `mongo` tool on the `mongo-0.mongo` Pod:

```bash
> rs.add("mongo-1.mongo:27017");
> rs.add("mongo-2.mongo:27017");
```

### Automating MongoDB Cluster Creation

To configure this Pod without having to build a new Docker image, we're going to use a ConfigMap to add a script into the existing MongoDB image. Here's the container we're adding:

```yaml
...
				- name: init-mongo
					image: mongo:3.4.1
					command: ["bash", "/config/init.sh"]
					volumeMounts:
						- name: config
							mountPath: /config
				volumes:
					- name: config
						configMap:
							name: "mongo-init"
```

- This is mounting a ConfigMap volume whose name is `mongo-init` and which holds a script that performs our initialization.
- First, the script determines whether it is running on `mongo-0` or not. If it is, it creates that ReplicaSet using the same command we ran imperatively previously. If it is on a different Mongo replica, it waits until the ReplicaSet exists, and then it registers itself as a member of that ReplicaSet.

*Example: mongo-configmap.yaml*

---

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
	name: mongo-init
data:
	init.sh: |
		#!/bin/bash

		# Need to wait for the readiness health check to pass so that the
		# mongo names resolve. This is kind of wonky.
		until ping -c 1 ${HOSTNAME}.mongo; do
			echo "waiting for DNS (${HOSTNAME}.mongo)..."
			sleep 2
		done

		until /usr/bin/mongo --eval 'printjson(db.serverStatus())'; do
			echo "connecting to local mongo..."
			sleep 2
		done
		echo "connected to local."

		HOST=mongo-0.mongo:27017

		until /usr/bin/mongo --host=${HOST} --eval 'printjson(db.serverStatus())'; do
			echo "connecting to remote mongo..."
			sleep 2
		done
		echo "connected to remote."

		if [[ "${HOSTNAME}" != 'mongo-0' ]]; then
			until /usr/bin/mongo --host=${HOST} --eval="printjson(rs.status())" \
						| grep -v "no replset config has been received"; do
				echo "waiting for replication set initialization"
				sleep 2
			done
			echo "adding self to mongo-0"
			/usr/bin/mongo --host=${HOST} \
				--eval="printjson(rs.add('${HOSTNAME}.mongo'))"
		fi

		if [[ "${HOSTNAME}" == 'mongo-0' ]]; then
			echo "initializing replica set"
			/usr/bin/mongo --eval="printjson(rs.initiate(\
				{'_id': 'rs0', 'members': [{'_id': 0, 'host': 'mongo-0.mongo:27017'}]}\
			))
		fi
		echo "initialized"
		
		while true; do
			sleep 3600
		end
```

- Sleeps forever after initializing the cluster. Since we do not want our main Mongo container to be restarted, we need to have our initialization container run forever too, or else Kubernetes might think our Mongo Pod is unhealthy.

Putting it all together, this is the complete StatefulSet that uses the ConfigMap:

*Example: mongo.yaml*

---

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
	name: mongo
spec:
	serviceName: "mongo"
	replicas: 3
	template:
		metadata:
			labels:
				app: mongo
		spec:
			containers:
				- name: mongodb
					image: mongo:3.4.1
					command: ["mongod", "--replSet", "rs0"]
					ports:
						- containerPort: 27017
							name: web
				# This container initializes the mongodb server, then sleeps.
				- name: init-mongo
					image: mongo:3.4.1
					command: ["bash", "/config/init.sh"]
					volumeMounts:
						- name: config
							mountPath: /config
			volumes:
				- name: config
					configMap:
						name: mongo-init
```

Given all of these files, you can create a Mongo cluster with:

```bash
kubectl apply -f mongo-config-map.yaml
kubectl apply -f mongo-service.yaml
kubectl apply -f mongo.yaml
```

- Alternatively, combine them into a single YAML file where the individual objects are separated by `---`.
- Ensure you keep the same ordering, since the StatefulSet definition relies on the ConfigMap definition existing.

### Persistent Volumes and StatefulSets

For persistent storage, you need to mount a persistent volume into the */data/db* directory (mongo-specific).

```yaml
...
				volumeMounts:
					- name: database
						mountPath: /data/db
```

While this approach is similar to reliable singletons, but because the StatefulSet replicates more than one Pod, you cannot simply reference a persistent volume claim. Instead you need to add a **persistent volume claim template**.

A claim template is identical to the Pod template, but instead of creating Pods, it creates volume claims.

Add the following onto the bottom of the StatefulSet definition:

```yaml
volumeClaimTemplates:
	- metadata:
			name: database
			annotations:
				volume.alpha.kubernetes.io/storage-class: anything
		spec:
			accessModes: [ "ReadWriteOnce" ]
			resources:
				requests:
					storage: 100Gi
```

When you add a volume claim template to a StatefulSet definition, each time the StatefulSet controller creates a Pod that is part of the StatefulSet it will create a persistent volume claim based on this template as part of that Pod.

In order for these replicated persistent volumes to work correctly, you either need to have autoprovisioning set up for persistent volumes, or you need to prepopulate a collection of persistent volume objects for the StatefulSet controller to draw from.

If there are no claims that can be created, the StatefulSet controller will not be able to create the corresponding Pods.

### Readiness Probes

For the liveness checks, we can use the `mongo` tool itself by adding the following to the Pod template in the StatefulSet object:

```yaml
...
	livenessProbe:
		exec:
			command: [ "/usr/bin/mongo", "--eval", "db.serverStatus()" ]
		initialDelaySeconds: 10
		timeoutSeconds: 10
...
```

# Chapter 16: Extending Kubernetes

## What It Means to Extend Kubernetes

Cluster administrator privileges are required to extend a cluster.

## Points of Extensibility

This chapter focuses on the extensions to the API server via adding new resource types or admission controllers to API requests.

Container Network Interface/Container Storage Interface/Container Runtime Interface extensions are more commonly used by Kubernetes cluster providers as opposed to end users.

There are a number of ways to extend your cluster without modifying the API server at all:

- DaemonSets that install automatic logging and monitoring
- Tools that scan your services for cross-site scripting (XSS) vulnerabilities
- Etc.

Admission controllers are called prior to the API object being written into the backing storage. They can reject or modify API requests.

There are several admission controllers built into the Kubernetes API server; for example, the limit range admission controller that sets default limits for Pods without them.

The other form of extension is custom resources. Whole new API objects are added to the Kuberenetes API surface area. These new objects can be added to namespaces, are subject to RBAC, and can be accessed with existing tools like `kubectl` as well as via the Kubernetes API.

The first thing you do to create a custom resource is to create a CustomResourceDefinition.

- This is actually a meta-resource; that is, a resource that is the definition of another resource.

Consider defining a new resource to represent load tests in your cluster. When a new LoadTest resource is created, a load test is spun up in your Kubernetes cluster and drives traffic to a service.

*Example: loadtest-resource.yaml*

---

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
	name: loadtests.beta.kuar.com
spec:
	group: beta.kuar.com
	versions:
		- name: v1
			served: true
			storage: true
	scope: Namespaced
	names:
		plural: loadtests
		singular: loadtest
		kind: LoadTest
		shortNames:
			- lt
```

The name must follow the format `<resource-plural>.<api-group>` to ensure that each resource definition is unique in the cluster.

In addition to the name of the version (e.g. v1, v2, etc.), there are fields that indicate if that version is served by the API service and which version is used for storing data in the backing storage for API server.

- The `storage` field must be true for only one version of the resource.

To create this resource: `kubetl create -f loadtest-resource.yaml`.

To get this resource: `kubectl get loadtests`.

To create an instance of LoadTest:

*Example: loadtest.yaml*

---

```yaml
apiVersion: beta.kuar.com/v1
kind: LoadTest
metadata:
	name: my-loadtest
spec:
	service: my-service
	scheme: https
	requestsPerSecond: 1000
	paths:
		- /index.html
		- /login.html
		- /shares/my-shares/
```

Note that we never defined the schema for the custom resource in the CustomResourceDefinition.

- It's possible to provide an OpenAPI specification for a custom resource, but this complexity is generally not worth it for simple resource types.
- If you do want to perform validation, you can register a validating admission controller, as described later.

LoadTest objects created this way don't do anything yet because there is no controller present in the cluster to react and take action when a LoadTest object is defined.

The controller interacts with the API server to list LoadTests and watches for any changes that might occur.

The simplest controllers run a `for` loop and repeatedly poll for new custom object, and then take actions to create or delete the resources that implement those custom objects.

This polling-based approach is inefficient: the period of the polling loop adds unnecessary latency, and the overhead of polling may add unnecessary load on the API server.

A more efficient approach is to use the watch API on the API server, which provides a stream of updates when the occur, eliminating both the latency and overhead of polling.

Using this API correctly is complicated. It is highly recommend that you use a well-supported mechanism such as the `Informer` pattern exposed in the `client-go` library.

### Validation and Defaulting

Validation is the process of ensuring that LoadTest objects sent to the API server are well formed and can be used to create load tests, while defaulting makes it easier for people to use our resources by providing automatic, commonly used values by default.

One option for adding validation is via an OpenAPI specification for our objects. This can be useful for basic validation of the presence of required fields or the absence of unknown fields.

Generally speaking, an API schema is insufficient for validation of API objects. E.g. we may want to validate that LoadTests have a valid scheme (e.g., http or https) or that requestsPerSecond is a nonzero positive integer.

To accomplish this, we use a validating admission controller, which intercepts requests to the API server and rejects or modifies them in-flight.

A dynamic admission controller is a simple HTTP application. The API server connects to the admission controller via either a Kubernetes Service object or an arbitrary URL. This means that admission controllers can optionally run outside of the cluster on services like AWS Lambda.

To install our validating admission controller, we need to specify it as a Kubernetes ValidatingWebhookConfiguration. This specifies the endpoint where the admission controller runs, as well as the resource (in this case LoadTest) and the action (in this case CREATE) where the admission controller should be run.

*Example: validating-admission-controller.yaml*

---

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
	name: kuar-validator
webhooks:
	- name: validator.kuar.com
		rules:
			- apiGroups:
					- beta.kuar.com
				apiVersions:
					- v1
				operations:
					- CREATE
				resources:
					- loadtests
		clientConfig:
			# Substitute the appropriate IP address for your webhook
			url: https://192.168.1.233:8080
			# caBundle should be a base64-encoded CA certificate for your cluster.
			# You can find it in your ${KUBECONFIG} file
			caBundle: REPLACEME
```

Webhooks that are accessed by the Kubernetes API server can only be accessed via HTTPS.  This means we need to generate a certificate to server the webhook. The easiest way to do this is to use the cluster's ability to generate new certificates using its own certificate authority (CA).

First we need a private key and a certificate signing request (CSR). Page 201 provides a simple Go program that generates these.

- Run the go program with `go run csr-gen.go <URL-for-webhook>`.
- It will generate `server.csr` and `server-key.pem`.

You can then create a certificate signing request for the Kubernetes API server using the following:

*Example: admission-controller-csr.yaml*

---

```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
	name: validating-controller.default
spec:
	groups:
		- system:authenticated
	request: REPLACEME
	usages:
		- digital signature
		- key encipherment
		- key agreement
		- server auth
```

The `request` field needs to be replaced with the base64-encoded certificate signing request we produced in the preceding code:

```yaml
perl -pi -e s/REPLACEME/$(base64 server.csr | tr -d '\n')/ \
admission-controller-csr.yaml
```

With the certificate signing request ready, you can send it to the API server to get the certificate:

`kubectl create -f admission-controller-csr.yaml`.

Next, you need to approve that request:

`kubectl certificate approve validating-controller.default`.

Once approved, you can download the new certificate:

```yaml
kubectl get csr validating-controller.default -o json | \
jq -r .status.certificate | base64 -d > server.crt
```

With the certificate, you are finally ready to create an SSL-based admission controller.

When the admission controller code receives a request, it contains an object of type `AdmissionReview`, which contains metadata about the request as well as the body of the request itself.

In our validating admission controller, we have only registered for a single resource type and a single action (CREATE), so we don't need to examine the request metadata. Instead we dive directly into the resource itself and validate that `requestsPerSecond` is positive and the URL scheme is valid. If they aren't we return a JSON body disallowing the request.

Implementing an admission controller to provide defaulting is similar, but instead of using a ValidatingWebhookConfiguration you use a MutatingWebhookConfiguration, and you need to provide a JSONPatch object to mutate the request object before it is stored.

Here's a TypeScript snippet that you can add to your validating admission controller to add defaulting. (If the paths field in the `loadtest` is of length 0, add a single path for `/index.html`:

```tsx
if (needsPatch(loadtest)) {
	const patch = [
		{ 'op': 'add', 'path': '/spec/paths', 'value': ['/index.html'] },
	]
	response['patch'] = Buffer.from(JSON.stringify(patch)).toString('base64');
	response['patchType'] = 'JSONPatch';
}
```

You can then register this webhook as a MutatingWebhookConfiguration by simply changing the `kind` field in the YAML object and saving the file as `mutating-controller.yaml`. Then create the controller by running: `kubectl create -f mutating-controller.yaml`.

## Patterns for Custom Resources

### Just Data

These easiest pattern for API extension is the notion of "just data". You are simply using the API server for storage and retrieval of information for your application – information that helps you manage the control and runtime of your application, not for application data storage.

For example, configuration for canary deployments of your application, like directing 10% of all traffic to an experimental backend.

- In theory this configuration could be stored in ConfigMaps, but they are essentially untyped, and using a more strongly typed API extension object provided clarity and ease of use.

Extensions that are just data don't need a corresponding controller to activate them, but they may have validating or mutating admission controllers to ensure that they are well formed.

### Compilers

In this pattern, the API extension object represents a high-level abstraction that is "compiled" into a combination of lower-level Kubernetes objects.

The LoadTest extension in the previous example is an example. A user consumes the extension as a high-level concept, in this case a `loadtest`, but it comes into being by being deployed as a collection of Kubernetes Pods and services.

To achieve this, a compiled abstraction requires an API controller to be running somewhere in the cluster, to watch the current LoadTests and create the "compiled" representation (and likewise delete representations that no longer exist).

In contrast to the operator pattern described next, there is no online health maintenance for compiled abstractions; it is delegated to the lower-level objects.

### Operators

Extensions that use the operator pattern provide online, proactive management of the resources created by the extensions.

- These extensions likely provide a higher-level abstraction that is compiled down to a lower-level representation, but they also provide online functionality like snapshot backups of a database, or upgrade notifications when a new version of the software is available.

to achieve this, the controller not only monitors the extension API to add or remove things as necessary, but also monitors the running state of the application supplied by the extension (e.g., a database) and takes actions to remediate unhealthy databased, take snapshots, or restore from a snapshot if a failure occurs.

Operators are the most complicated pattern for API extension of Kubernetes, but they are also the most powerful, enabling users to get access to "self-driving" abstractions that are responsible not just for deployment, but also health checking and repair.

### Getting Started

The Kubebuilder project (*https://kubebuilder.io*) contains a library of code intended to help you easily build reliable Kubernetes API extensions.

# Chapter 17: Deploying Real-World Applications

## Jupyter

The Jupyter project is a web-based interactive scientific notebook for exploration and visualization.

We begin by creating a namespace to hold the Jupyter application:
`kubectl create namespace jupyter`.

And then create a deployment of size one with the program itself:

*Example: jupyter.yaml*

---

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
	labels:
		run: jupyter
	name: jupyter
	namespace: jupyter
spec:
	replicas: 1
	selector:
		matchLabels:
			run: jupyter
	template:
		metadata:
			labels:
				run: jupyter
		spec:
			containers:
				- image: jupter/scipy-notebook:9b87b1625445
					name: jupyter
			dnsPolicy: ClusterFirst # ClusterFirst is the default if not specified
			restartPolicy: Always
```

Deploy this file with `kubctl create -f jupyter.yaml`.

You can wait for the container to become ready using the `watch` command:
`watch kubectl get pods -—namespace jupyter`.

Once the Jupyter container is up and running, you need to obtain the initial login token. You can do this be looking at the logs for the container:

```bash
pod_name=$(kubectl get pods --namespace=jupyter --no-headers | awk '{print $1}')
kubectl logs --namespace jupyter ${pod_name}
```

You should then copy the token and set up port forwarding to the Jupyter container:

`kubectl port-forward ${pod_name} 8888:8888 --namespace jupyter` 

Finally, visit *localhost:8888/?token=<token>*. You should find the Jupyter dashboard loads in your browser.

## Parse

The Parse server is a cloud API dedicated to providing easy-to-use storage for mobile applications.

### Prerequisites

Parse uses a MongoDB cluster for its storage. This section assumes you have a three-replica Mongo cluster running in Kubernetes with the names `mongo-0.mongo`, `mongo-1.mongo` and `mongo-2.mongo`.

### Building the Parse Server

The open source `parse-server` comes with a Dockerfile by default for easy containerization.

First, clone the Parse repository: `git clone https://github.com/ParsePlatform/parse-server`.

Then move into that directory and build the image:

```bash
cd parse-server
docker build -t ${DOCKER_USER}/parse-server
```

### Deploying the `parse-server`

Parse looks for three environment variables when being configured:

- `PARSE_SERVER_APPLICATION_ID`: an identifier for authorizing your application.
- `PARSE_SERVER_MASTER_KEY`: an identifier that authorizes the master (root) user.
- `PARSE_SERVER_DATABASE_URI`: the URI for your MongoDB cluster.

*Example: parse.yaml*

---

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
	name: parse-server
	namespace: default
spec:
	replicas: 1
	template:
		metadata:
			labels:
				run: parse-server
		spec:
			containers:
				- name: parse-server
					image: ${DOCKER_USER}/parse-server
					env:
						- name: PARSE_SERVER_DATABASE_URI
							value: "mongodb://mongo-0.mongo:27017,\
								mongo-1.mongo:27017,mongo-2.mongo:27017/dev?replicaSet=rs0"
						-	name: PARSE_SERVER_APP_ID
							value: my-app-id
						- name: PARSE_SERVER_MASTER_KEY
							value: my-master-key

```

### Testing Parse

To test your deployment, you need to expose it as a Kubernetes service.

*Example: parse-service.yaml*

---

```yaml
apiVersion: v1
kind: Service
metadata:
	name: parse-server
	namespace: default
spec:
	ports:
		- ports: 1337
			protocol: TCP
			targetPort: 1337
	selector:
		run: parse-server
```

In any real application, you are likely to want to secure the connection with HTTPS. See the `parse-server` GitHub page for more details on such a configuration.

## Ghost

Ghost is a popular blogging engine with a clean interface written in JavaScript. It can either use a file-based SQLite database or MySQL for storage.

### Configuring Ghost

Ghost is configured with a JavaScript file that describes the server. We will store this file as a ConfigMap.

*Example: ghost-config.js*

---

```jsx
const path = require('path')

const config = {
	development: {
		url: 'http://localhost:2368',
		database: {
			client: 'sqlite3',
			connection: {
				filename: path.join(process.env.GHOST_CONTENT, '/data/ghost-dev.db')
			},
			debug: false
		},
		server: {
			host: '0.0.0.0',
			port: '2368'
		},
		paths: {
			contentPath: path.join(process.env.GHOST_CONTENT, '/')
		}
	}
}
```

`kubectl create cm ghost-config --from-file ghost-config.js`

As with the Parse example, we will mount this configuration file as a volume inside of our container and deploy Ghost as a Deployment object.

*Example: ghost.yaml*

---

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
	name: ghost
spec:
	replicas: 1
	selector:
		matchLabels:
			run: ghost
	template:
		metadata:
			labels:
				run: ghost
		spec:
			containers:
				- image: ghost
					name: ghost
					command:
						- sh
						- -c
						- cp /ghost-config/ghost-config.js /var/lib/ghost/config.js
							&& /usr/local/bin/docker-entrypoint.sh node current/index.js
					volumeMounts:
						- mountPath: /ghost-config
							name: config
				volumes:
					- name: config
						configMap:
							defaultMode: 420
							name: ghost-config

```

- We are copying the *config.js* file from a different location to where Ghost expects to find it, since the ConfigMap can only mount directories, not individual files.

Once the pod is up and running, you can expose it as a service with:
`kubectl expose deployments ghost —port=2368`.

Once the service is exposed, you can use the `kubectl proxy` command to access the ghost server at *http://localhost:8001/api/v1/namespaces/default/services/ghost/proxy.*

**Ghost + MySQL**

To use MySQL, first modify `config.js` to include:

```yaml

...
database: {
	client: "mysql",
	connection: {
		host: "mysql",
		user: "root",
		password: "root",
		database: "ghost_db",
		charset: "utf8"
	}
},
...
```

Next, create a new ConfigMap object:

`kubectl create configmap ghost-config-mysql --from-file ghost-config.js`.

Then update the Ghost deployment to change the name of the configMap to `config-map-mysql`.

Using the instructions from "Kuvernetes-Native Storage with StatefulSets" on page 186, deploy a MySQL server in your Kubernetes cluster. Make sure that it has a service named `mysql` defined as well.

You will need to create the database in the MySQL application:

```yaml
kubectl exec -it mysql-zzmlw -- mysql -u root -p
mysql> create database ghost_db;
```

Finally, because your Ghost server is now decoupled from its database, you can scale up your Ghost server and it will continue to share data across all replicas.

## Redis

Redis is a popular in-memory key/value store.

A reliable Redis installation is actually two programs working together: `redis-server`, which implements the key/value store, and `redis-sentinel`, which implements health checking and failover for a replicated Redis cluster.

When Redis is deployed in a replicated manner, there is a single master server that can be used for both read and write operations.

There are other replica servers that duplicate the data written to the master and can be used for load-balancing read operations.

Any of these replicas can fail over to become the master is the original master fails. This failover is performed by a Redis sentinel.

In our deployment, a Redis server and a redis sentinel are located in the same file.

### Configuring Redis

Redis needs separate configurations for the master and slave replicas.

*Example: master.conf*

---

```
bind 0.0.0.0
port 6379

dir /redis-data
```

- Directs Redis to bind all network interfaces on port 6379 and store its files in the */redis-data* directory.

The slave configuration adds a single `slaveof` directive:

*Example: slave.conf*

---

```
bind 0.0.0.0
port 6379

dir .

slaveof redis-0.redis 6379
```

- We will set up `redis-0.redis` using a service and a StatefulSet.

We also need a configuration for the Redis sentinel:

*Example: sentinel.conf*

---

```
bind 0.0.0.0
port 26379

sentinel monitor redis redis-0.redis 6379 2
sentinel parallel-syncs redis 1
sentinel down-after-milliseconds redis 10000
sentinel failover-timeout redis 20000
```

Next, we need wrapper scripts to use in our StatefulSet deployment.

The first script looks at the hostname for the Pod and determines whether this is the master or a slave, and launches Redis with the appropriate configuration.

*Example: init.sh*

---

```bash
#!/bin/bash
if [[ ${HOSTNAME} == 'redis-0' ]]; then
	redis-server /redis-config/master.conf
else
	redis-server /redis-config/slave.conf
fi
```

The other script is for the sentinel, and simply waits for the `redis-0.redis` DNS name to become available.

*Example: sentinel.sh*

```bash
#!/bin/bash
cp /redis-config-src/*.* /redis-config

while ! ping -c 1 redis-0.redis; do
	echo 'Waiting for server'
	sleep 1
done

redis-sentinel /redis-config/sentinel.conf
```

Now we need to package all of these files into a ConfigMap object:

```bash
kubectl create configmap redis-config \
--from-file=slave.conf=./slave.conf \
--from-file=master.conf=./master.conf \
--from-file=sentinel.conf=./sentinel.conf \
--from-file=init.sh=./init.sh \
--from-file=sentinel.sh=./sentinel.sh
```

### Creating a Redis Service

To provide naming and discovery for Redis replicas, we create a service without a cluster IP address.

*Example: redis-service.yaml*

---

```yaml
apiVersion: 1
kind: Service
metadata:
	name: redis
spec:
	selector:
		app: redis
	ports:
		- port: 6379
			name: peer
	clusterIP: None
```

### Deploying Redis

We deploy Redis using a StatefulSet:

*Example: redis.yaml*

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
	name: redis
spec:
	replicas: 3
	serviceName: redis
	selector:
		matchLabels:
			app: redis
	template:
		metadata:
			labels:
				app: redis
		spec:
			containers:
				- name: redis
					image: redis:4.0.11-alpine
					command: [sh, -c, source /redis-config/init.sh]
					ports:
						- name: redis
							containerPort: 6379
					volumeMounts:
						- mountPath: /redis-config
							name: config
						- mountPath: /redis-data
							name: data
				- name: sentinel
					image: redis:4.0.11-alpine
					command: [sh, -c, source /redis-config-src/sentinel.sh]
					volumeMounts:
						- mountPath: /redis-config-src
							name: config
						- mountPath: /redis-config
							name: data
			volumes:
				- configMap:
						defaultMode: 420
						name: redis-config
					name: config
				- emptyDir:
					name: data
```

- There are two containers in this Pod: one runs the `[init.sh](http://init.sh)` script and the main Redis server, and the other is the sentinel that monitors the servers.
- There are also two volumes: one is the volume that uses our ConfigMap to configure the two Redis applications, and the other is a simple `emptyDir` volume that is mapped into the Redis server container to hold the application data so it survives a container restart.
    - For a more reliable installation, this could be a network-attached disk.

### Playing with Our Redis Cluster

To determine which server the Redis sentinel believes is the master:

```yaml
kubectl exec redis-2 -c redis \
-- redis-cli -p 6379 sentinel get-master-addr-by-name redis
```

- Should print out the IP address of the `redis-0` pod, which should match that shown with `kubectl get pods -o wide`.

To confirm that replication is working:

1. Try to read the value `foo` from one of the replicas:
`kubectl exec redis-2 -c redis — redis-cli -p 6379 get foo`.
You should see no data in the response.
2. Try (and fail) to write data to a read-only replica:
`kubectl exec redis-2 -c redis — redis-cli -p 6379 set foo 10`.
3. Write the same data to `redis-0`, which is the master.
4. Now try the original read from a replica. It should show that the data has been replicated between master and slaves.

# Chapter 18: Organizing Your Application

## Principles to Guide Us

- Filesystems as the source of truth
- Code review to ensure the quality of changes
- Feature flags for staged roll forward and roll back

### Filesystems as the Source of Truth

Rather than viewing the state of the cluster – the data in `etcd` – as the source of truth, it is optimal to view the filesystem of YAML objects as the source of truth for your application.

The API objects deployed into your cluster(s) are then a reflection of the truth stored in the filesystem.

### The Role of Code Review

The second principle of our application layout is that it must facilitate the review of every change merged into the set of files the represents the source of truth for our cluster.

### Feature Gates and Guards

In larger projects, it often makes sense to separate the source code from the configuration to provide for a separation of concerns.

When some new feature is developed, that development should take place entirely behind a feature flag or gate. That looks something like:

```yaml
if (featureFlags.myFlag) {
 // Feature implementation
}
```

The use of feature flags both simplifies debugging problems in production and ensures that disabling a feature doesn't require a binary rollback to an older version of the code that would remove all of the bug fixes and other improvements made by the newer version of the code.

The third principle of application layout is that code lands in source control, by default off, behind a feature flag, and is only activated through a code-reviewed change to configuration files.

## Managing Your Application in Source Control

### Filesystem Layout

The first cardinality on which you want to organize your application is the semantic component or layer (e.g. frontend, batch work queue, etc.).

For an application with a frontend that uses two services, the filesystem might look like:

```yaml
frontend/
service-1/
service-2/
```

Within each of the directories, the configurations for each application are stored. These are YAML files which directly represent the state of the cluster. It's generally useful to include both the service name and the object type within the same file name.

While Kubernetes allows for the creation of YAML files with multiple objects in the same file, this should generally be considered an anti-pattern.

The filesystem might look like:

```yaml
frontend/
	frontend-deployment.yaml
	frontend-service.yaml
	frontend-ingress.yaml
...
```

### Managing Periodic Versions

It's useful to be able to store and maintain multiple different revisions of your configuration.

Given the file and version control approach, there are two different approaches you can use:

- Tags, branches and source control features. Convenient because it maps to the way that people manage revisions in source control, and it leads to a simpler directory structure.
- Clone the configuration within the filesystem and use directories for different revisions. Makes the simultaneous viewing of the configurations very straightforward.

**Versioning with Branches and Tags**

When you use branches and tags to manage the configuration revisions, the directory structure is unchanged from the example in the previous section.

When you are ready for a release, you place a source-control tag (e.g. `git tag v1.0`) in the configuration source-control system. The tag represents the configuration used for that version, and the HEAD of source control continues to iterate forward.

The world becomes somewhat more complicated when you need to update the release configuration, but the approach models what you would do in source control.

1. Commit the change to the HEAD of the repository.
2. Create a new branch named `v1` at the `v1.0` tag.
3. Cherry-pick the desired change onto the release branch (`git cherry-pick <edit>`).
4. Tag this branch with the `v1.1` tag to indicate a new point release.

One common error when cherry-picking fixes into a release branch is to only pick the change into the latest release. It's a good idea to cherry-pick it into all active releases. in case for some reason you need to roll back versions but the fix is still needed.

**Versioning with Directories**

An alternative approach to using source-control features is to use filesystem features.

Each versioned deployment exists within its own directory. For example:

```yaml
frontend/
	v1/
		frontend-deployment.yaml
		frontend-service.yaml
	current/
		frontend-deployment.yaml
		frontend-service.yaml
...
```

All deployments occur from HEAD instead of specific revisions or tags.

When adding a new configuration, it is done to the files in the `current` directory.

When creating a new release, the `current` directory is copied to create a new directory associate with the new release.

When performing a bugfix change to a release, the pull request must modify the YAML file in all the relevant release directories. This is a slightly better experience than the cherry-picking approach described earlier, since it is clear in a single change request that all of the relevant versions are being updated with the same change, instead of requiring a cherry-pick per version.

## Structuring Your Application for Development, Testing and Deployment

### Goals

Each developer should be able to easily develop new features for the application. It is essential that developers be able to work in their own environment, yet with all services available.

The ability to easily and accurately test your application prior to deployment is essential to the ability to quickly roll out features while maintaining high reliability.

### Progression of a Release

The stages of a release are:

1. HEAD: The bleeding edge of the configuration; the latest changes.
2. Development: Largely stable, but not ready for deployment. Suitable for developers to use for buildling features.
3. Staging: The beginnings of testing, unlikely to change unless problems are found.
4. Canary: The first real release to users, used to test for problems with real-world traffic and likewise give users a change to test what is coming next.

**Introducing a Development Tag**

To introduce a development stag, a new `development` tag is added to the source control system and an automated process is used to move this tag forward.

On periodic cadence, HEAD is tested via automated integration testing. If these tests pass, the `development` tag is moved forward to HEAD.

Thus, developers can track close to the latest changes when deploying their own environments, but they can also be assured that the deployed configurations have at least passed a limited smoke test.

**Mapping Stages to Revisions**

Regardless of whether you are using the filesystem or source-control revisions to represent different configuration versions, it is easy to implement a map from stage to revision.

In the filesystem case, you can use symbolic links to map a stage name to a revision"

```yaml
frontend/
	canary/ -> v2/
	release/ -> v1/
	v1/
		frontend-deployment.yaml
```

In the case of version control, it is simply an additional tag at the same revision as the appropriate version.

In either case, the versioning of releases proceeds using the processes described previously, and separately the stages are moved forward to new versions as appropriate.

Effectively, this means there are two simultaneous processes, the first for cutting new release versions and the second for qualifying a release version for a particular stage in the application cycle.

## Parameterizing Your Application with Templates

It is important to strive for the Cartesian product of environments and stages to be as identical as possible. Variance and drift between different environments produces snowflakes and systems that are hard to reason about.

Parameterized environments use templates for the bulk of their configuration, but they mix in a limited set of parameters to produce the final configuration.

The parameterization is limited in scope and maintained in a small parameters file for easy visualization of the differences between environments.

### Parameterizing with Helm and Templates

The Helm template language uses the moustache syntax:

```yaml
metadata:
	name: {{ .Release.Name }}-deployment
```

To pass a parameter for this value you use a *values.yaml* file with contents like:

```yaml
Release:
	Name: my-release
```

### Filesystem Layout for Parameterization

Instead of treated each deployment lifecycle stage as a pointer to a version, each deployment lifecycle is a combination of a parameters file and a pointer to a specific version.

In the directory-based layout, this might look like:

```yaml
frontend/
	staging/
		templates -> ../v2
		staging-parameters.yaml
	production/
		templates -> ../v1
		production-parameters.yaml
	v1/
		frontend-deployment.yaml
		frontend-server.yaml
...
```

Doing this with version control looks similar, except that the parameters for each life-cycle stage are kept at the root of the configuration directory tree:

```yaml
frontend/
	staging-parameters.yaml
	templates:
		frontend-deployment.yaml
```

## Deploying You Application Around the World

### Architectures for Worldwide Deployment

Generally speaking, each cluster is intended to live in a single region, and each cluster is expected to contain a single, complete deployment of your application.

A worldwide deployment of an application consists of multiple different clusters, each with its own application configuration.

A particular region's configuration is conceptually the same as a stage in the deployment lifecycle. E.g.

- Development
- Staging
- Canary
- EastUS
- WestUS
- Europe
- Asia

### Implementing Worldwide Deployment

The key to a highly available system is limiting the effect or "blast radius" of any change that you might make. As you roll out a version across regions, it makes sense to move carefully from region to region in order to validate in one region before moving on to the next.

Rolling out software across the world generally looks more like a workflow than a single declarative update: you begin by updating the version in staging to the latest version and then proceed through all regions until it is rolled out everywhere.

To determine the length of time between rollouts to regions, you want to consider the "mean time to smoke" for your software. This is the time it takes on average after a new release is rolled out to a region for a problem to be discovered.

- Two to three times the mean time to smoke is a reasonable place to start.

To determine the order of regions, it is important to consider the characteristics of various regions. You are likely to have high-traffic and low-traffic regions, and features which are more popular in some regions than others.

You likely want to begin rolling out to a low-traffic region, the roll out to a high-traffic region once you've validated that your release works correctly with low traffic. Then you can safely roll out everywhere.

It is important to follow the release schedule completely for every release, no matter how big or how small. Many outages have been caused by people accelerating releases either to fix some other problem, or because they believed it to be safe.

### Dashboards and Monitoring for Worldwide Deployments

It is essential to develop dashboards that can tell you at a glance what version is running in which region, as well as alerting that will fire when too many different versions of your application are deployed.

Best practice is to limit the number of active versions to no more than three: one testing, one rolling out, and one being replaced by the rollout.