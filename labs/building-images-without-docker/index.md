# Building images without Docker

# Overview
This lab introduces alternatives to Docker for building images, creating containers, and managing them. 

## Buildah
Buildah is a CLI that facilitates building OCI images. A OCI Image is the open specification for the format that images should be to run on top of OCI compatiable Container Runtimes.

### Build an image using Buildah 
Clone the example application code from GitHub and enter the directory.
```
git clone https://github.com/katacoda/golang-http-server.git && cd golang-http-server
```

Buildah has built-in `Dockerfile` support, so it can be used to replace `docker build`. However, `buildah` also has many additional features. It supports building images one layer at a time, and has an interactive mode as well. 

Install `buildah`
```
. /etc/os-release
sudo sh -c "echo 'deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/x${ID^}_${VERSION_ID}/ /' > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list"
wget -nv https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/x${ID^}_${VERSION_ID}/Release.key -O Release.key
sudo apt-key add - < Release.key
sudo apt-get update -qq
sudo apt-get -qq -y install buildah
```

Use `buildah` to create an image from a `Dockerfile`
```
sudo buildah bud --format docker -t buildah-web:latest -f Dockerfile
```

When prompted which repository to use for`golang`, choose `docker.io`

Ouput:
```
Getting image source signatures
...snip
Writing manifest to image destination
Storing signatures
STEP 2: RUN mkdir /app
STEP 3: ADD . /app/
...
Writing manifest to image destination
Storing signatures
--> 6b1dc793a3b
6b1dc793a3b975c1d3aed3d6c03988b9410287b889f0ac066fd2e12ca98d2c0e
```

To confirm the image was build successfully run: 
```
sudo buildah images
```

Output: 
```
REPOSITORY                 TAG          IMAGE ID       CREATED         SIZE
localhost/buildah-web      latest       6b1dc793a3b9   2 minutes ago   298 MB
```

Inspect the image: 
```
sudo buildah inspect buildah-web:latest
```

Review the metadata available about the image. 


Buildah also supports creating images interactively. 

Create an environment variable to refer to your buildah working container
```
container=$(sudo buildah from alpine)
```
Output:
```
Resolved "alpine" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
Getting image source signatures
Copying blob 540db60ca938 done
Copying config 6dbb9cc540 done
Writing manifest to image destination
Storing signatures
```

Now echo the environment variable 
```
echo $container
```

Output:
```
alpine-working-container
```

Note that, by default, Buildah constructs the name of the container by appending `-working-container` to the name.

This can be overridden by passing `--name`
```
example_container=$(sudo buildah from --name "example-container" alpine)
```

Confirm the new name: 
```
echo $example_container
```

The Alpine Linux image you just pulled is only 5 MB in size and it lacks the basic utilities such as Bash. Run the following command to verify your new container image:
```
sudo buildah run $container bash 
```
The following output shows that the container image has been created, but `bash` is not yet installed:

```
ERRO[0000] container_linux.go:346: starting container process caused "exec: \"bash\": executable file not found in $PATH"
container_linux.go:346: starting container process caused "exec: \"bash\": executable file not found in $PATH"
error running container: error creating container for [bash]: : exit status 1
ERRO exit status 1
```

To install Bash, enter the `buildah run` command and specify:
* The name of the container ($container)
* Two dashes. The commands after `--` are passed directly to the container.
* The command you want to execute inside the container (apk add bash)

```
sudo buildah run $container -- apk add bash
```

Output:
```
fetch https://dl-cdn.alpinelinux.org/alpine/v3.13/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.13/community/x86_64/APKINDEX.tar.gz
(1/4) Installing ncurses-terminfo-base (6.2_p20210109-r0)
(2/4) Installing ncurses-libs (6.2_p20210109-r0)
(3/4) Installing readline (8.1.0-r0)
(4/4) Installing bash (5.1.0-r0)
Executing bash-5.1.0-r0.post-install
Executing busybox-1.32.1-r6.trigger
OK: 8 MiB in 18 packages
```

You can see that it added bash to the container.

Similarly to how you've installed `bash`, run the `buildah run` command to install `node` and `npm`:
```
sudo buildah run $container -- apk add --update nodejs npm
```

You can use the the `buildah config` command to set the image configuration values. The following command sets the working directory to `/usr/src/app/`:

```
sudo buildah config --workingdir /usr/src/app/ $container
```

To initialize a new JavaScript project, run the `npm init -y` command inside the container:

```
sudo buildah run $container -- npm init -y
```

Output:
```json
Wrote to /usr/src/app/package.json:

{
  "name": "app",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

Issue the following command to install Express.JS:
```
sudo buildah run $container -- npm install express --save
```

Output:
```
npm notice created a lockfile as package-lock.json. You should commit this file.
npm WARN app@1.0.0 No description
npm WARN app@1.0.0 No repository field.

+ express@4.17.1
added 50 packages from 37 contributors and audited 50 packages in 3.641s
found 0 vulnerabilities
```

Create a file named `HelloWorld.js` and copy in the following JavaScript source code:
```js
const express = require('express')
const app = express()
const port = 3000

app.get('/', (req, res) => res.send('Hello World!'))

app.listen(port, () => console.log(`Example app listening on port ${port}!`))
```

To copy the `HelloWorld.js` file to your container's working directory, enter the `buildah copy` command specifying:
* The name of the container ($container)
* The name of the file you want to copy (HelloWorld.js)

```
sudo buildah copy $container HelloWorld.js
```

Note: You can copy a file to a different container by passing the name of the destination directory as an argument. 

The following example command copies the `HelloWorld.js` to the `/tmp` directory:

To set the entry point for your container, enter the `buildah config` command with the `--entrypoint` argument:
```
sudo buildah config --entrypoint "node HelloWorld.js" $container
```
At this point, you're ready to write the new image using the `buildah commit` command. It takes two parameters:
* The  name of the container image (`$container`)
* The name of the new image (`buildah-hello-world`)
```
sudo buildah commit $container buildah-hello-world
```

Output:
```
Getting image source signatures
Copying blob b2d5eeeaba3a skipped: already exists
Copying blob f9564351b905 done
Copying config c78f3e6330 done
Writing manifest to image destination
Storing signatures
c78f3e6330a643131e290080d9da4a84d05d6d1aa66d2fa9c81ae5d3853be976
```

If the provided image name doesn't begin with a registry name, Buildah defaults to adding `localhost` to the name of the image.

The following command lists your Buildah images:
```
sudo buildah images
```

Output:
```
REPOSITORY                      TAG          IMAGE ID       CREATED              SIZE
localhost/buildah-hello-world   latest       c78f3e6330a6   About a minute ago   72.9 MB
```

## Podman
### Installing Podman
To install Podman on Ubuntu 20.04 we run the following commands:
```
. /etc/os-release
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/Release.key | sudo apt-key add -
sudo apt-get update
sudo apt-get install -y podman-rootless
```
NOTE: When Kubernetes is installed the `podman-rootless` package must be installed so there is no conflict with the `kubernetes-cni` package.

### Running a container using Podman
To run your image with Podman, you must first make sure your image is visible in Podman:
```
sudo podman images
```

Output:
```
REPOSITORY                     TAG         IMAGE ID      CREATED         SIZE
localhost/buildah-hello-world  latest      c78f3e6330a6  12 minutes ago  72.9 MB
```
Run the `buildah-hello-world` image by entering the `podman run` command with the following arguments:
* `dt` to specify that the container should be run in the background, and that Podman should allocate a `pseudo-TTY` for it.
* `-p` with the port on host (3000) that'll be forwarded to the container port (3000), separated by `:`.
* The name of your image (`buildah-hello-world`)

```
sudo podman run -dt -p 3000:3000 buildah-hello-world
```

In our environment because we already have Kubernetes handling container orchestration and traffic routing the application will not load. 

Remove the container:
```
sudo podman kill $(sudo podman ps -ql)
```

# Congrats!
