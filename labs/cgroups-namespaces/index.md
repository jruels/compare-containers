
## docker namespace exploration


## Explore pid namespace

Take a look at the current processes
```
ps aux
```

Create a new namespace and run bash:
```bash
sudo unshare --fork --pid --mount-proc bash
```

Take a look around:
```bash
ps aux
```

What is pid 1 in the new namespace?

Inside the new namespace, run a long lived process like `top`. Then in a separate ssh connection to the docker host look at the current processes
```bash
ps aux
```
Find the namespaced processes? What PIDs do they have?

Feel free to explore with other commands. When you're done exploring just `exit`

## Explore docker namespaces
Run any container, for example:
```bash
#sleep for 5 minutes and exit
docker run --detach alpine sleep 300
```

Copy the container id for your running container, this is $CID below

If you don't already have the CID, you can find it with `docker ps`

### docker exec
Use the `docker exec` utility to explore the inside of the container:
```bash
docker exec -it $CID /bin/sh
#Explore within the container, e.g.
ps aux
ls /
# ...
# when you are done just exit
exit
```

Now use the more general linux `nsenter` utility to enter the container's namespaces


```bash
PID=$(docker inspect --format {% raw %}{{.State.Pid}}{% endraw %} $CID)
sudo nsenter --target $PID --mount --uts --ipc --net --pid /bin/sh
```

Look around, and `exit` when done.

## Explore the memory cgroup limits


```bash
sudo cgcreate -a ubuntu -g memory:playground
```

See what's in it
```bash
ls -la /sys/fs/cgroup/memory/playground/
```

`memory.limit_in_bytes` looks useful, let's try tweaking that.

Let's set a 10MiB memory limit for the cgroup

```bash
echo 10485760 | tee /sys/fs/cgroup/memory/playground/memory.limit_in_bytes
```

Let's try running a memory hogging program:

```bash
#Will use 100MiB
memhog 104857600 1
```

Now let's join the cgroup and see what happens
```bash
sudo cgexec -g memory:playground bash

#Will use 10MiB
memhog 10485760 1
```

You should see this:
> allocating 10485760 bytes...
> writing 10485760 bytes...
> Killed

What is the exit code? What does this exit code signify?
```bash
echo $?
```

If you want to delete the cgroup you can run
```bash
sudo cgdelete memory:playground
```

## Explore the cgroup namespace

In addition to cgcreate, we can create cgroups by simply creating a folder (because in linux almost everything is a file).

In this example we'll be working in the freezer subsystem, which is used to schedule processes on computer. But we won't actually be using the freezer subsystem, other than for the cgroups.

```bash
#Create a cgroup
sudo mkdir -p /sys/fs/cgroup/freezer/mycgroup
```

Note that your new cgroup is automatically populated by the kernel
```bash
ls -la /sys/fs/cgroup/freezer/mycgroup
```

Get your shell PID
```bash
echo $$
# 942
```

Add your shell to the cgroup
```bash
echo 942 | sudo tee /sys/fs/cgroup/freezer/mycgroup/cgroup.procs
```

Verify your shell is now in the cgroup
```bash
grep freezer /proc/self/cgroup
# 6:freezer:/mycgroup
```

Now we'll use unshare again to create a new process running in a new cgroup and mount namespace

```bash
sudo unshare -C bash
```

We are now running bash inside of the new namespace. Let's take a look at our freezer cgroup now:

```bash
grep freezer /proc/self/cgroup
# 6:freezer:/
```

Note that you don't see `/mycgroup` because the parent has made the cgroup relative to its parent, which is `/mycgroup`. In other words, this process is shown to be the root `/` of the parent `/mycgroup`.

Now get the PID of your namespaced shell and keep it running.
```bash
echo $$
# 1032
```

Now from your new shell in just the default namespaces, run this setting the correct PID from above:
```bash
grep freezer /proc/1032/cgroup
# 6:freezer:/mycgroup
```

This verifies that your namespaced process is in the cgroup "mycgroup", but your namespaced process can't tell from looking at its cgroups.

### Cgroup Namespace Exercise

How can the namespaced shell above figure what cgroup it is in? (hint: despite the cgroup namespace with relative path, there is still at least one way)

## Docker and CPU limits

### How performant are these cpu bound processes?

Try opening 2 shells and starting these commands at the same time. Does one finish notably faster?

```bash
docker run --cpu-shares=2 alpine time dd if=/dev/urandom of=/dev/null bs=1M count=2000
```

```bash
docker run --cpu-shares=1024 alpine time dd if=/dev/urandom of=/dev/null bs=1M count=2000
```

