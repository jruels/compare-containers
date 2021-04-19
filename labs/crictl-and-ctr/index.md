# Using container tools

# Overview 
This lab introduces alternative tools to debug and troubleshoot running pods and containers. 

## Using runc
`runc` can be used to interact with the underlying system responsible for all OCI compliant container runtimes. 

### Install runc
Start by installing `runc`
```
sudo apt install -y runc
```

### Create an OCI application bundle
Create a working directory:
```
mkdir my-bundle
cd $_

sudo runc spec
```

`runc` spec generates a dummy `config.json`. It already has a "process" section which specifies which process to run inside the container - even with a couple of environment variables.

```json
{
	"ociVersion": "1.0.1-dev",
	"process": {
		"terminal": false,
		"user": {
			"uid": 0,
			"gid": 0
		},
		"args": [
			"sleep",
			"infinite"
		],
		"env": [
			"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
			"TERM=xterm"
		],
```

It also defines where to look for the root filesystem:
```json
...

        "root": {
                "path": "rootfs",
                "readonly": true

        },
...
```

and multiple other things, including default mounts inside the container, capabilities, hostname etc. If you inspect this file, you will notice, that many sections are platform-agnostic and the ones that are specific to concrete OS are nested inside appropriate section. For example, you will notice there is a "linux" section with Linux specific options.

Run the bundle 
```
sudo runc run test
```

`runc` returns an error because it is missing the root filesystem for the container. 
```
rootfs (/home/ubuntu/my-bundle/rootfs) does not exist
```

Ok, create the directory to resolve the issue 
```
mkdir rootfs
runc run test
```

Output:
```
container_linux.go:349: starting container process caused "exec: \"sleep\": executable file not found in $PATH"
```

This makes total sense - empty folder is not really a useful root filesystem, our container has no chance of doing anything useful. We need to create a real Linux root filesystem.

We are going to use Skopeo and umoci to download an image and unpack the root filesystem to our `my-bundle` directory.

## Use Skopeo and umoci to get an OCI application bundle
Creating a root filesystem from scratch is a rather cumbersome experience, so instead let's use one of the existing minimal images - `busybox`

To pull the image, we first need to install `skopeo`. 
Skopeo is a command line utility that performs various operations on container images and image repositories.

Skopeo can copy images between different sources and destinations, inspect images and even delete them. Skopeo can not build images, it won't know what to do with your Containerfile. It's perfect for CI/CD pipelines that automate container image promotion.

### Install Skopeo
```
. /etc/os-release
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/Release.key | sudo apt-key add -
sudo apt-get update
sudo apt-get -y install skopeo
```

Copy the `busybox` image from Docker Hub to a local OCI compliant store. 
```
skopeo copy docker://busybox:latest oci:busybox:latest
```
There is no "pull" - we need to tell Skopeo both source and destination for the image. Skopeo supports almost a dozen different types of sources and destinations. Note that this command will create a new `busybox` folder, inside which you will find all of the OCI Image files, with different image layers, manifest etc.

What we copied is an OCI Image, but as we already know, `runc` needs an OCI Runtime Bundle. We need a tool that will convert an image to a bundle. This tool will be `umoci` - an openSUSE utility with the purpose of manipulating OCI images. Use `umoci unpack` to convert an OCI image a bundle.

### Install umoci
```
wget https://github.com/opencontainers/umoci/releases/download/v0.4.7/umoci.amd64 && sudo cp umoci.amd64 /usr/local/bin/umoci
```

### Convert image to bundle
```
umoci unpack --image busybox:latest bundle
```
Take a look at what's in side the `bundle` folder:
```
ls bundle

config.json
rootfs
sha256_73c6c5e21d7d3467437633012becf19e632b2589234d7c6d0560083e1c70cd23.mtree
umoci.json
```
We already have `config.json`, so let's just copy ``rootfs directory to previously created ``my-bundle directory. 

The `rootfs` directory containers a minimal `/` (root) filesystem, which you can see with `ls`.
```
ls rootfs

bin  dev  etc  home  proc  root  sys  tmp  usr  var
```

### Run OCI application bundle with runc
We are ready to run our application bundle as a container named `test`:
```
sudo runc run test
```

After running the above command a container is started with a new shell.

```
# runc run test
/ # ls
bin   dev   etc   home  proc  root  sys   tmp   usr   var
```

We run the previous container in a default `foreground` mode. In this mode, each container process becomes a child process of a long-running `runc` process:

```
6801   997  \_ sshd: root [priv]
6805  6801      \_ sshd: root@pts/1
6806  6805          \_ -bash
6825  6806              \_ zsh
7342  6825                  \_ runc run test
7360  7342                  |   \_ runc run test
```

Let's inspect this container a bit closer by replacing the `sh` command with `sleep infinite` and setting terminal option to "false" inside `config.json`. runc does not provide a whole lot of command line arguments. It has commands like `start`, `stop` and `run`, but the configuration of the container is always coming from the file, not from the command line:

Update `config.json`, and change `terminal` to `false` and `args` to `sleep infinite` as shown below.
```json
{
        "ociVersion": "1.0.1-dev",
        "process": {
                "terminal": false,
                "user": {
                        "uid": 0,
                        "gid": 0
                },
                "args": [
                        "sleep",
                        "infinite"
                ]
...
```

Run the container in `detached` mode. 
```
sudo runc run test --detach
```

To see running containers use `runc list`
```
sudo runc list

ID          PID         STATUS      BUNDLE            CREATED                          OWNER
test        4258        running     /root/my-bundle   2020-04-23T20:29:39.371137097Z   root
```
Confirm the state file for `runc` is present and populated
```
sudo apt install -y jq
sudo cat /run/runc/test/state.json | jq . | head
```

As we ran the container in detached mode, there is no relation between the original `runc run` command (there is no such process anymore) and this container process.




## Using ctr 
The `ctr` tool can be used to interact directly with `containerd`. It supports many features including: 
* Manage containers 
* Interact with container images 
* Work with `containerd` namespaces
* More! 

### Namespaces
Start by listing namespaces.
```
sudo ctr namespace list
```

Output: 
```
NAME   LABELS
k8s.io
moby
```

**Protip:** You can abbreviate many commands, use `--help` to see the different options.

The `moby` namespace is created when Docker is installed, and the `k8s.io` namespace is created when the Kubelet is updated to use `containerd`.

Create a new namespace 
```
sudo ctr ns c testns
```

Confirm new namespace was created using the `list` command from above. 

### Images
`containerd` can pull images from a container registry and supports tags. 

Use `ctr` to pull the `nginx` image: 
```
sudo ctr image pull docker.io/library/nginx:latest
```
To pull the image to a specific namespace pass `-n <namespace>`, for example
```
sudo ctr -n testns image pull docker.io/library/nginx:latest
```
Output:
```
docker.io/library/nginx:latest:                                                   resolved       |++++++++++++++++++++++++++++++++++++++|
index-sha256:75a55d33ecc73c2a242450a9f1cc858499d468f077ea942867e662c247b5e412:    done           |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:42bba58a1c5a6e2039af02302ba06ee66c446e9547cbfb0da33f4267638cdb53: done           |++++++++++++++++++++++++++++++++++++++|
...snip
elapsed: 4.0 s                                                                    total:  51.2 M (12.8 MiB/s)
unpacking linux/amd64 sha256:75a55d33ecc73c2a242450a9f1cc858499d468f077ea942867e662c247b5e412...
done
```

Confirm the image was pulled 
```
sudo ctr -n testns image ls
```

If you only want the image name add `-q`.

### Containers
Using `ctr` start up the `nginx` image. 
```
sudo ctr -n testns container create docker.io/library/nginx:latest webdemo
```

To confirm the container is running use `
```
sudo ctr container list 
```

What is the output? Is it expected? 

Why didn't the last command show the container? How can it be fixed so the container is listed? 


## Using crictl
The `crictl` tool can be used to interact with any CRI compliant container runtime.

`crictl` connects to `unix:///var/run/dockershim.sock` by default. For other runtimes, you can set the endpoint in multiple different ways:

* By setting flags `--runtime-endpoint` and `--image-endpoint`
* By setting environment variables `CONTAINER_RUNTIME_ENDPOINT` and `IMAGE_SERVICE_ENDPOINT`
* By setting the endpoint in the configuration file `--config=/etc/crictl.yaml`

You can also specify timeout values when connecting to the server and enable or disable debugging, by specifying timeout or debug values in the configuration file or using the `--timeout` and `--debug` command-line flags.

Run the following on a worker node.

`containerd` listens on a unix socket by default so you can configure `crictl` to connect like this:

```
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
EOF
```

### Images
Pull a `busybox` image
```
sudo crictl pull busybox
```
Output:
```
Image is up to date for sha256:388056c9a6838deea3792e8f00705b35b439cf57b3c9c2634fb4e95cfc896de6
```

Confirm image was pulled successfully and `crictl` can see it 
```
sudo crictl images
```

Output: 
```
IMAGE                             TAG                 IMAGE ID            SIZE
docker.io/library/busybox         latest              388056c9a6838       769kB
```

You can also list only the image ID
```
sudo crictl images -q
```

### Pods and containers
Kubernetes abstracts containers using pods, and `crictl` supports pods. To list pods run: 
```
sudo crictl pods 
```

Output: 
```
POD ID              CREATED             STATE               NAME                                NAMESPACE           ATTEMPT
e81896d9dc723       2 hours ago         Ready               nginx-deployment-66b6c48dd5-rzsr6   default             0
980ca31e50e54       2 hours ago         Ready               nginx-deployment-66b6c48dd5-4pk8t   default             0
```

You can also show the most recently created pod
```
sudo crictl pods -l
```

Create configuration files for a pod and container. 

pod-config.json: 
```json
{
    "metadata": {
        "name": "nginx-sandbox",
        "namespace": "default",
        "attempt": 1,
        "uid": "hdishd83djaidwnduwk28bcsb"
    },
    "log_directory": "/tmp",
    "linux": {
    }
}
```

container-config.json
```json
{
  "metadata": {
      "name": "busybox"
  },
  "image":{
      "image": "busybox"
  },
  "command": [
      "top"
  ],
  "log_path":"busybox.log",
  "linux": {
  }
}
```

Using `crictl` to run a pod sandbox is useful for debugging container runtimes. On a running Kubernetes cluster, the sandbox will eventually be stopped and deleted by the Kubelet. To avoid this for the lab stop the Kubelet
```
sudo systemctl stop kubelet
```

Create a sandbox pod.
```
sudo crictl runp pod-config.json
```

Confirm the pod was created: 
```
sudo crictl pods -l 
```

You should see the pod in a `Ready` state.

Output:
```
POD ID              CREATED             STATE               NAME                                NAMESPACE           ATTEMPT
baa5869d2c574       5 seconds ago       Ready               nginx-sandbox                       default             1
```

Now create the container passing the pod ID from the above command
```
sudo crictl create $(sudo crictl pods -lq) container-config.json pod-config.json
```

List all containers and verify that the newly-created container has its state set to `Created`.
```
sudo crictl ps -a 
```

Output: 
```
CONTAINER ID        IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID
9c1ef3b0bae4f       busybox             28 seconds ago      Created             busybox             0                   baa5869d2c574
```

To start the container, pass it's ID to `crictl start`:
```
sudo crictl start $(sudo crictl ps -alq)
```

Output: 
```
9c1ef3b0bae4f7c12cd469f48db5cafd2d1be151e19bbb241c855d40bb6272e3
```

Confirm the container is in a `Running` state:
```
sudo crictl ps
```

Output: 
```
CONTAINER ID        IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID
9c1ef3b0bae4f       busybox             12 minutes ago      Running             busybox             0                   baa5869d2c574
```

Now that the container is running execute a command inside it:
```
sudo crictl exec -i -t $(sudo crictl ps -lq) ls
```

Now look at the container's metadata:
```
sudo crictl inspect $(sudo crictl ps -lq)
```

# Congrats!



