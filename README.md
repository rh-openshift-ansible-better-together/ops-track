# Better Together: OpenShift and Ansible Workshop - Ops Track

- [0: Preparation](#0-preparation)
- [1: Getting to know Ansible](#1-getting-to-know-ansible)
- [2: OpenShift Architecture](#2-openshift-architecture)
  * [2.1: The value of containers](#21-the-value-of-containers)
    + [2.1.1: Worst case scenario provisioning](#211-worst-case-scenario-provisioning)
    + [2.1.2: Comparing VM and container resource usage](#212-comparing-vm-and-container-resource-usage)
      - [2.1.2.1: Storage resource consumption](#2121-storage-resource-consumption)
      - [2.1.2.2: CPU and RAM resource consumption](#2122-cpu-and-ram-resource-consumption)
    + [2.1.3: Summary](#213-summary)
  * [2.2: How containers work](#22-how-containers-work)
    + [2.2.1: What exactly is a container?](#221-what-exactly-is-a-container)
    + [2.2.2: More effective process isolation](#222-more-effective-process-isolation)
    + [2.2.3: Isolation with kernel namespaces](#223-isolation-with-kernel-namespaces)
      - [2.2.3.1: The mount namespace](#2231-the-mount-namespace)
      - [2.2.3.2: The uts namespace](#2232-the-uts-namespace)
      - [2.2.3.3: The ipc namespace](#2233-the-ipc-namespace)
      - [2.2.3.4: The pid namespace](#2234-the-pid-namespace)
      - [2.2.3.5: The network namespace](#2235-the-network-namespace)
      - [2.2.3.6: The user namespace](#2236-the-user-namespace)
      - [2.2.3.7: Summary](#2237-summary)
    + [2.2.4: Quotas and Limits with kernel control groups](#224-quotas-and-limits-with-kernel-control-groups)
      - [2.2.4.1: Creating projects](#2241-creating-projects)
      - [2.2.4.2: Limits and requests](#2242-limits-and-requests)
      - [2.2.4.3: Creating limits and requests for a project](#2243-creating-limits-and-requests-for-a-project)
    + [2.2.5: Protection with SELinux](#225-protection-with-selinux)
    + [2.2.6: Summary](#226-summary)
  * [2.3: Applications in OpenShift](#23-applications-in-openshift)
    + [2.3.1: Deploying your first applications](#231-deploying-your-first-applications)
    + [2.3.2: Multiple ways to deploy applications](#232-multiple-ways-to-deploy-applications)
    + [2.3.3: Using the CLI](#233-using-the-cli)
    + [2.3.4: Scaling an application using the CLI](#234-scaling-an-application-using-the-cli)
    + [2.3.5: Using the web interface](#235-using-the-web-interface)
    + [2.3.6: OpenShift SDN](#236-openshift-sdn)
    + [2.3.7: Routing layer](#237-routing-layer)
  * [2.4: Summary](#24-summary)
- [3: OpenShift and Ansible integration points](#3-openshift-and-ansible-integration-points)
  * [3.1: Deploying OpenShift](#31-deploying-openshift)
  * [3.2: Modifying an OpenShift cluster](#32-modifying-an-openshift-cluster)
  * [3.3: Summary](#33-summary)
- [4: A real world CI/CD scenario](#4-a-real-world-cicd-scenario)
  * [4.1: Creating a CI/CD workflow manually](#41-creating-a-cicd-workflow-manually)
  * [4.2: Automating application deployment with Ansible](#42-automating-application-deployment-with-ansible)
- [5: Troubleshooting Applications](#5-troubleshooting-applications)
  * [5.1: Image Pull Failures](#51-image-pull-failures)
  * [5.2: Application Crashing](#52-application-crashing)
  * [5.3: Troubleshooting access to containers](#53-troubleshooting-access-to-containers)
- [6: Q&A and Wrap-Up](#6-qa-and-wrap-up)

### 0: Preparation

Each workshop participant is provisioned their own OpenShift Container Platform 3.11 cluster (using Ansible!). Each cluster contains one master node, one infrastructure node, one application node, and one bastion host. You will be working from the bastion host to complete today's lab. 

You'll need to claim your OpenShift cluster using our [cluster assignment tool](https://red.ht/2JK4yYh). Once you get to the cluster assignment tool, you'll need two pieces of information:

* Lab Code: Better Together (`<Insert City Name Here>`) - Ops Track
* Activation Key: `ansible+openshift`

Once you enter the information into the cluster assignment tool, you'll receive a "GUID" in the format `btws-<4_random_characters>` (for example, `btws-j1e2`). It is important to keep this GUID handy for the rest of the lab. 

To SSH to the bastion host, you'll need to download the private SSH key from the link provided in the cluster assignment tool. Select your operating system from the list and save the file to a location you know. 

For users who are running Linux, macOS, or Windows with the Windows Subsystem for Linux installed, open up a terminal and run the following commands:

```
chmod 600 /path/to/downloaded/key/ocp-workshop.pem
ssh -i /path/to/downloaded/key/ocp-workshop.pem ec2-user@bastion.<INSERT_GUID_HERE>.openshiftworkshop.com
```

For users who are running Windows and using PuTTY for SSH, follow the below directions:

1. Open PuTTY. 
2. In PuTTY, under Category on the left, navigate to Connection → SSH → Auth.
3. On the right under Authentication parameters, click Browse and locate the private key (ocp-workshop.ppk) you saved earlier.
4. On the left, navigate to Session.
5. On the right in the Host Name field, ec2-user@bastion.<INSERT_GUID_HERE>.openshiftworkshop.com
6. Click Open.
7. When prompted with the security alert, click Yes.

Once you've got an active SSH session, you'll need to change to be the root user by running the following command and then we'll need to set your user ID for the workshop.

```
$ sudo su -
$ export GUID=<INSERT_GUID_HERE>
```

You are now ready to start working through the workshop.

## 1: Getting to know Ansible

To get to know Ansible, we're going to rely on a [quick slideshow](http://www.ansible.red), along with some live examples in your OpenShift cluster. You'll be running these commands from your cluster's bastion host, which is already configured with Ansible and an inventory that references your OpenShift cluster.

## 2: OpenShift Architecture

In this section we'll discuss the fundamental components that make up OpenShift. Any ops-centric discussion of an application platform like OpenShift needs to start with containers; both technically and from a perspective of value. We'll start off talking about why containers are the best solution today to deliver your applications.

We promise to not beat you up with slides after this, but we do have a few more. Let's use the [OpenShift Technical Overview](https://docs.google.com/presentation/d/1o92SA69qPN1RwY102Rr7QtS8Bj1cUWWlfND8JrctmAw/preview) to become familiar with the core OpenShift concepts before we dive down into the fun details.

### 2.1: The value of containers

At their heart, containers are the next evolution in how we isolate processes on a Linux system. This evolution started when we created the first computers. They evolved from ENIAC, through mainframes, into the server revolution all the way through virtual machines (VMs) and now into containers.

![The evolution from ENIAC to Linux Containers](/images/evolution.png)

More efficient application isolation (we'll get into how that works in the next section) provides an Ops team a few key advantages that we'll discuss next.

#### 2.1.1: Worst case scenario provisioning

Think about your traditional virtualization platform, or your workflow to deploy instances to your public cloud of choice for a moment. How big is your default VMDK for your root OS disk? How much extra storage do you add to your EC2 instance, just to handle the 'unknown' situations? Is it 40GB? 60GB?

**This phenomenon is known as _Worst Case Scenario Provisioning_.** In the industry, we've done it for years because we consider each VM a unique creature that is hard to alter once created. Even with more optimized workflows in the public cloud, we hold on to IP addresses, and their associated resources, as if they're immutable once created. It's easier to overspend on compute resources for most of us than to change an IP address in our IPAM once we've deployed a system.

#### 2.1.2: Comparing VM and container resource usage

In this section we're going to investigate how containers use your datacenter resources more efficiently. First, we'll focus on storage.

##### 2.1.2.1: Storage resource consumption

Compared to a 40GB disk image, the [Red Hat Universal Base Image 7 container image](https://access.redhat.com/containers/#/registry.access.redhat.com/ubi7/ubi) is approximately 70MB. It's a widely accepted rule of thumb that container images shouldn't exceed 1GB in size. If your application takes more than 1GB of code and libraries to run, you likely need to re-think your plan of attack.

Instead of each new instance of an application consuming 40GB+ on your storage resources, they consume a couple hundred megabytes. Your next storage purchase just got a whole lot more interesting.

##### 2.1.2.2: CPU and RAM resource consumption

It's not just storage where containers help save resources. We'll analyze this in more depth in the next section, but we want to get the idea into your head for that part of our investigation now. Containers are smaller than a full VM because containers don't each run their own Linux kernel. All containers on a host share a single kernel. That means a container just needs the application it needs to execute and its dependencies. You can think of containers as a "portable userspace" for your applications.

> **Additional Concept:** Because each container doesn't require its own kernel, we also measure startup time in milliseconds! This gives us a whole new way to think about scalability and high availability!

In your cluster, log in as the admin user and navigate to the default project. Look at the resources your registry and other deployed applications consume. For example the `registry-console` application (a UI to see and manage the images in your OpenShift cluster) is consuming less than 2 MB of RAM!

![OpenShift Application Console focused on the registry-console Deployment Config](/images/metrics.png)

If we were deploying this application in a VM we would spend multiple gigabytes of our RAM just so we could give the application the handful of megabytes it needs to run properly.

The same is true for CPU consumption. In OpenShift, we measure and allocate CPUs by the _millicore_, or thousandth of a core. Instead of multiple CPUs, applications can be given the small fractions of a CPU they need to get their job done.

#### 2.1.3: Summary 

Containers aren't just a tool to help developers create applications more quickly. Containers take any application and deploy it more efficiently in your infrastructure. Instead of measuring each application in gigabytes used and vCPUs allocated, we measure containers using megabytes and millicores allocated.

OpenShift deployed into your existing datacenter gives you back resources. For customers deep into their transformation with OpenShfit, an exponential increase in resource density isn't uncommon.

### 2.2: How containers work

You can find five different container experts and ask them to define what a container is, and you're likely to get five different answers. The following are some of our personal favorites, all of which are correct from a certain perspective:

* A transportable unit to move applications around. This is a typical developer's answer.
* A fancy Linux process (one of our personal favorites)
* A more effective way to isolate processes on a Linux system. This is a more operations-centered answer.

#### 2.2.1: What exactly is a container?

There are t-shirts out there say "Containers are Linux". Well, they're not wrong. The components that isolate and protect applications running in containers are unique to the Linux kernel. Some of them have been around for years, or even decades. In this section we'll investigate them in more depth.

#### 2.2.2: More effective process isolation

We mentioned in the last section that containers utilize server resources more effectively than VMs (the previous most effective resource isolation method). The primary reason is because containers use different systems in the Linux kernel to isolate the processes inside them. These systems don't need to utilize a full virtualized kernel like a VM does.

![A diagram demonstrating the difference between virtual machines and containers](/images/vm_vs_container.png)

Let's investigate what makes a container a container.

#### 2.2.3: Isolation with kernel namespaces

The kernel component that makes the applications feel isolated are called namespaces. Namespaces are a lot like a two-way mirror or a paper wall inside Linux. Like a two-way mirror, from the host we can see inside the container. But from inside the container it can only see what's inside its namespace. And like a paper wall, namespaces provide sufficient isolation but they're lightweight to stand up and tear down.

SSH to your infrastructure node by running the following command:

```
ssh infranode1.$GUID.internal
```

Next, run the `sudo lsns` command. The output will be long, so let's use `grep` to filter it. 

```
$ sudo lsns | grep heapster
4026533532 mnt        1 43747 ec2-user   heapster --source=kubernetes.summary_api:${MASTER_URL}?useServiceAccount=true&kubeletHttps=true&kubeletPort=10250 --tls_cert=/heapster-certs/tls.crt --tls_key=/heapster-certs/tls.key --tls_client_ca=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt --allowed_users=system:master-proxy --metric_resolution=30s --sink=hawkular:https://hawkular-metrics:443?tenant=_system&labelToTenant=pod_namespace&labelNodeId=nodename&caCert=/hawkular-metrics-certs/tls.crt&user=hawkular&pass=$HEAPSTER_PASSWORD&filter=label(container_name:^system.slice.*|^user.slice)&concurrencyLimit=5
4026533536 uts        1 43747 ec2-user   heapster --source=kubernetes.summary_api:${MASTER_URL}?useServiceAccount=true&kubeletHttps=true&kubeletPort=10250 --tls_cert=/heapster-certs/tls.crt --tls_key=/heapster-certs/tls.key --tls_client_ca=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt --allowed_users=system:master-proxy --metric_resolution=30s --sink=hawkular:https://hawkular-metrics:443?tenant=_system&labelToTenant=pod_namespace&labelNodeId=nodename&caCert=/hawkular-metrics-certs/tls.crt&user=hawkular&pass=$HEAPSTER_PASSWORD&filter=label(container_name:^system.slice.*|^user.slice)&concurrencyLimit=5
4026533537 pid        1 43747 ec2-user   heapster --source=kubernetes.summary_api:${MASTER_URL}?useServiceAccount=true&kubeletHttps=true&kubeletPort=10250 --tls_cert=/heapster-certs/tls.crt --tls_key=/heapster-certs/tls.key --tls_client_ca=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt --allowed_users=system:master-proxy --metric_resolution=30s --sink=hawkular:https://hawkular-metrics:443?tenant=_system&labelToTenant=pod_namespace&labelNodeId=nodename&caCert=/hawkular-metrics-certs/tls.crt&user=hawkular&pass=$HEAPSTER_PASSWORD&filter=label(container_name:^system.slice.*|^user.slice)&concurrencyLimit=5
```

To limit this content to a single process, specify one of the PIDs on your system by using the `-p` parameter for `lsns`.

```
$ sudo lsns -p43747
        NS TYPE  NPROCS   PID USER     COMMAND
4026531837 user     354     1 root     /usr/lib/systemd/systemd --switched-root --system --deserialize 21
4026533532 mnt        1 43747 ec2-user heapster --source=kubernetes.summary_api:${MASTER_URL}?useServiceAccount=true&kubeletHttps=true&kubeletPort=10250 --tls_cert=/heapster-certs/
4026533536 uts        1 43747 ec2-user heapster --source=kubernetes.summary_api:${MASTER_URL}?useServiceAccount=true&kubeletHttps=true&kubeletPort=10250 --tls_cert=/heapster-certs/
4026533537 pid        1 43747 ec2-user heapster --source=kubernetes.summary_api:${MASTER_URL}?useServiceAccount=true&kubeletHttps=true&kubeletPort=10250 --tls_cert=/heapster-certs/
4026534019 ipc        2 43618 1001     /usr/bin/pod
4026534022 net        2 43618 1001     /usr/bin/pod
```

Let's discuss 5 of these namespaces.

##### 2.2.3.1: The mount namespace

The mount namespace is used to isolate filesystem resources inside containers. The files inside the base image used to deploy a container. From the point of view of the container, that's all that is available or visible.

If your container has host resources or persistent storage assigned to it, these are made available using a Linux bind mount. This means, not matter what you use for your persistent storage backed, your developer's applications only ever need to know how to access the correct directory.

We can see this using the `nsenter` command line utility on your infrastructure node. `Nsenter` is used to enter a single namespace that is associated with another PID. When debugging container environments, its value is massive. Here is the root filesystem listing from an infrastructure node.

```
$ sudo ls -al /
total 24
dr-xr-xr-x.  18 root root  236 Mar 23  2018 .
dr-xr-xr-x.  18 root root  236 Mar 23  2018 ..
lrwxrwxrwx.   1 root root    7 Mar 23  2018 bin -> usr/bin
dr-xr-xr-x.   5 root root 4096 Jul 15 18:35 boot
drwxr-xr-x.   2 root root    6 Mar 23  2018 data
drwxr-xr-x.  18 root root 2780 Jul 15 18:38 dev
drwxr-xr-x.  99 root root 8192 Jul 15 18:53 etc
drwxr-xr-x.   3 root root   22 Jul 15 18:26 home
lrwxrwxrwx.   1 root root    7 Mar 23  2018 lib -> usr/lib
lrwxrwxrwx.   1 root root    9 Mar 23  2018 lib64 -> usr/lib64
drwxr-xr-x.   2 root root    6 Dec 14  2017 media
drwxr-xr-x.   2 root root    6 Dec 14  2017 mnt
drwxr-xr-x.   3 root root   17 Jul 15 18:53 opt
dr-xr-xr-x. 366 root root    0 Jul 15 18:34 proc
dr-xr-x---.   6 root root  234 Jul 16 16:02 root
drwxr-xr-x.  38 root root 1140 Jul 15 18:53 run
lrwxrwxrwx.   1 root root    8 Mar 23  2018 sbin -> usr/sbin
drwxr-xr-x.   2 root root    6 Dec 14  2017 srv
dr-xr-xr-x.  13 root root    0 Jul 15 18:34 sys
drwxrwxrwt.  10 root root 4096 Jul 16 15:50 tmp
drwxr-xr-x.  13 root root  155 Mar 23  2018 usr
drwxr-xr-x.  20 root root  282 Jul 15 18:35 var
```

Let's use `nsenter` to enter the mount namespace for the heapster container using the PID we got from the `lsns` command above. Once you enter that namespace, you can look at the root filesystem to see that it is different than what we saw before.

```
$ sudo nsenter -m -t 43747
[root@infranode1 /]# ll
total 0
lrwxrwxrwx.   1 root root   7 Apr 16 15:25 bin -> usr/bin
dr-xr-xr-x.   2 root root   6 Dec 14  2017 boot
drwxr-xr-x.   5 root root 360 Jul 15 18:58 dev
drwxr-xr-x.   1 root root  66 Jul 15 18:58 etc
drwxrwxrwt.   3 root root 140 Jul 15 18:58 hawkular-account
drwxrwxrwt.   3 root root 160 Jul 15 18:58 hawkular-metrics-certs
drwxrwxrwt.   3 root root 120 Jul 15 18:58 heapster-certs
drwxr-xr-x.   1 root root  22 May 24 22:08 home
lrwxrwxrwx.   1 root root   7 Apr 16 15:25 lib -> usr/lib
lrwxrwxrwx.   1 root root   9 Apr 16 15:25 lib64 -> usr/lib64
drwxr-xr-x.   2 root root   6 Dec 14  2017 media
drwxr-xr-x.   2 root root   6 Dec 14  2017 mnt
drwxr-xr-x.   1 root root  62 May 24 22:08 opt
dr-xr-xr-x. 367 root root   0 Jul 15 18:58 proc
dr-xr-x---.   1 root root  27 Jul 16 16:01 root
drwxr-xr-x.   1 root root  18 May 24 22:08 run
lrwxrwxrwx.   1 root root   8 Apr 16 15:25 sbin -> usr/sbin
drwxrwxrwt.   3 root root 100 Jul 15 18:58 secrets
drwxr-xr-x.   2 root root   6 Dec 14  2017 srv
dr-xr-xr-x.  13 root root   0 Jul 15 18:34 sys
drwxrwxrwt.   1 root root   6 May 24 22:08 tmp
drwxr-xr-x.   1 root root  28 Apr 16 15:25 usr
drwxr-xr-x.   1 root root  52 Apr 16 15:25 var
```

The container image for heapster includes some of the filesystem like a normal server, but it also includes directories that are specific to the application.

##### 2.2.3.2: The uts namespace
UTS stands for "Unix Time Sharing". This is a concept that has been around since the 1970's when it was a novel idea to allow multiple users to log in to a system simultaneously. If you run the command `uname -a`, the information returned is the UTS data structure from the kernel.

```
$ uname -a
Linux infranode1.btws-6e50.internal 3.10.0-957.21.3.el7.x86_64 #1 SMP Fri Jun 14 02:54:29 EDT 2019 x86_64 x86_64 x86_64 GNU/Linux
``` 

Each container in OpenShift gets its own UTS namespace, which is equivalent to its own `uname -a` output. That means each container gets its own hostname and domain name. This is extremely useful in a large distributed application platform like OpenShift.

We can see this in action using `nsenter`.

```
$ hostname
infranode1.btws-6e50.internal
$ sudo nsenter -u -t 43747
[root@heapster-kclzv ec2-user]# hostname
heapster-kclzv
```

##### 2.2.3.3: The ipc namespace

The IPC (inter-process communication) namespace is dedicated to kernel objects that are used for processes to communicate with each other. Objects like named semaphores and shared memory segments are included. here. Each container can have its own set of named memory resources and it won't conflict with any other container or the host itself.

##### 2.2.3.4: The pid namespace

In the Linux world, PID 1 is an important concept. PID 1 is the process that starts all the other processes on your server. Inside a container, that is true, but it's not the PID 1 from your server. Each container has its own PID 1 thanks to the PID namespace. From our host, we see all of the processes we would expect on a Linux server using `pstree`.

> **Privileged containers:** Most of the containers that run on your infrastructure node run in privileged mode. That means these containers have access to all or some of the host's namespaces. This is a useful, but powerful tool reserved for applications that need to access a host's filesystem or network stack (or other namespaced components) directly. The example below is from an unprivileged container running an Apache web server.

```
# ps --ppid 4470
   PID TTY          TIME CMD
  4506 ?        00:00:00 cat
  4510 ?        00:00:01 cat
  4542 ?        00:02:55 httpd
  4544 ?        00:03:01 httpd
  4548 ?        00:03:01 httpd
  4565 ?        00:03:01 httpd
  4568 ?        00:03:01 httpd
  4571 ?        00:03:01 httpd
  4574 ?        00:03:00 httpd
  4577 ?        00:03:01 httpd
  6486 ?        00:03:01 httpd
  ```

When you execute the same command from inside the PID namespace, you see a different result. For this example, instead of using `nsenter`, we'll use the `oc exec` command from our control node. It does the same thing, with the primary difference being that we don't need to know the application node the container is deployed to, or its actual PID.

```
$ oc exec app-cli-4-18k2s ps
   PID TTY          TIME CMD
     1 ?        00:00:27 httpd
    18 ?        00:00:00 cat
    19 ?        00:00:01 cat
    20 ?        00:02:55 httpd
    22 ?        00:03:00 httpd
    26 ?        00:03:00 httpd
    43 ?        00:03:00 httpd
    46 ?        00:03:01 httpd
    49 ?        00:03:01 httpd
    52 ?        00:03:00 httpd
    55 ?        00:03:00 httpd
    60 ?        00:03:01 httpd
    83 ?        00:00:00 ps
```

From the point of view of the server, PID 4470 is an `httpd` process that has spawned several child processes. Inside the container, however, the same `httpd` process is PID 1, and its PID namespace has been inherited by its child processes.

PIDs are how we communicate with processes inside Linux. Each container having its own set of Process IDs is important for security as well as isolation.

##### 2.2.3.5: The network namespace

OpenShift relies on software-defined networking that we'll discuss more in an upcoming section. Because of this, as well as modern networking architectures, the networking configuration on an OpenShift node can become extremely complex. One of the over-arching goals of OpenShift is to make the developer's experience consistent no matter the underlying host's complexity. The network namespace helps with this. On your infrastructure node, there could be upwards of 20 defined interfaces.

```
$ ip a
1: lo:  mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0:  mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 0e:39:78:cc:a6:58 brd ff:ff:ff:ff:ff:ff
    inet 172.16.87.199/16 brd 172.16.255.255 scope global noprefixroute dynamic eth0
       valid_lft 3178sec preferred_lft 3178sec
    inet6 fe80::c39:78ff:fecc:a658/64 scope link
       valid_lft forever preferred_lft forever
3: docker0:  mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:36:9f:24:e7 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
4: ovs-system:  mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether f6:95:72:0e:09:4f brd ff:ff:ff:ff:ff:ff
5: br0:  mtu 8951 qdisc noop state DOWN group default qlen 1000
    link/ether be:47:c6:da:e5:48 brd ff:ff:ff:ff:ff:ff
6: vxlan_sys_4789:  mtu 65000 qdisc noqueue master ovs-system state UNKNOWN group default qlen 1000
    link/ether 7a:0b:31:e4:a4:eb brd ff:ff:ff:ff:ff:ff
    inet6 fe80::780b:31ff:fee4:a4eb/64 scope link
       valid_lft forever preferred_lft forever
```
                  
However, from within one of the containers on that node, you only see an `eth0` and `lo` interface.

```
$ sudo nsenter -n -t 43747 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if24: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8951 qdisc noqueue state UP group default
    link/ether 0a:58:0a:01:04:12 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.1.4.18/23 brd 10.1.5.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::fc45:7aff:fe43:e99d/64 scope link
       valid_lft forever preferred_lft forever
```

Each container's network namespace has a single outbound interface (`eth0`) and a loopback address (lots of applications like to use the loopback interface). We'll cover OpenShift SDN (the software-defined network configuration in OpenShift) and how traffic gets from the interface inside a container out to its destination in an upcoming section.

##### 2.2.3.6: The user namespace

Currently in OpenShift, all containers share a single user namespace. This is due to some lingering performance issues with the user namespace that prevent it from being capable of handling the enterpise scale that OpenShift is designed for. Don't worry, we're working on it.

##### 2.2.3.7: Summary

Linux kernel namespaces are used to isolate processes running inside containers. They're more lightweight than virtulization technologies and don't require an entire virtualized kernel to function properly. From inside a container, namespaced resources are fully isolated, but can still be viewed and accessed when needed from the host and from OpenShift.


#### 2.2.4: Quotas and Limits with kernel control groups

Kernel Control Groups are how containers (and other things like VMs and even clever sysadmins) limit the resources available to a given process. Nothing fixes bad code. But with control groups in place, it can become restarting a single service when it crashes instead of restarting the entire server.

In OpenShift, control groups are used to deploy resource limit and requests. Let's set up some limits for applications in a new project that we'll call `image-uploader`. Let's create a new project using our control node.

##### 2.2.4.1: Creating projects
Applications deployed in OpenShift are separated into projects. Projects are used not only as logical separators, but also as a reference for RBAC and networking policies that we'll discuss later. To create a new project, use the oc new-project command.

```
$ oc new-project image-uploader --display-name="Image Uploader Project"
Now using project "image-uploader" on server "https://master.btws-6e50.openshiftworkshop.com:443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python.
```

We'll use this project for multiple examples. Before we actually deploy an application into it, we want to set up project limits and requests.

##### 2.2.4.2: Limits and requests

OpenShift Limits are per-project maximums for various objects like number of containers, storage requests, etc. Requests for a project are default values for resource allocation if no other values are requested. We'll work with this more in a while, but in the meantime, think of Requests as a lower bound for resources in a project, while Limits are the upper bound.

##### 2.2.4.3: Creating limits and requests for a project

The first thing we'll create for the Image Uploader project is a collection of Limits. This is done, like most things in OpenShift, by creatign a YAML file and having OpenShift process it. On your control node, create a file named `/root/core-resource-limits.yaml`. It should contain the following content.

```
apiVersion: "v1"
kind: "LimitRange"
metadata:
  name: "core-resource-limits"
spec:
  limits:
    - type: "Pod"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "100m"
        memory: "4Mi"
    - type: "Container"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "100m"
        memory: "4Mi"
      default:
        cpu: "300m"
        memory: "200Mi"
      defaultRequest:
        cpu: "200m"
        memory: "100Mi"
      maxLimitRequestRatio:
        cpu: "10"
```        
                  
After your file is created, have it processed and added to the configuration for the `image-uploader` project.

```
$ oc create -f core-resource-limits.yaml -n image-uploader
limitrange "core-resource-limits" created
```

To confirm your limits have been applied, run the `oc get limitrange` command.

```
$ oc get limitrange
NAME                   AGE
core-resource-limits   2m
```

The limitrange you just created applies to any applications deployed in the `image-uploader` project. Next, you're going to create resource limits for the entire project. Create a file named `/root/compute-resources.yaml` on your control node. It should contain the following content.

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "3"
    limits.memory: 3Gi
  scopes:
    - NotTerminating
```
                  
Once it's created, apply the limits to the `image-uploader` project.

```
$ oc create -f compute-resources.yaml -n image-uploader
resourcequota "compute-resources" created
```
               
Next, confirm the limits were applied using `oc get`.

```
$ oc get resourcequota -n image-uploader
NAME                AGE
compute-resources   1m
```

We're almost done! So fare we've define resource limits for both apps and the entire `image-uploader` project. These are controlled under the convers by control groups in the Linux kernel. But to be safe, we also need to define limits to the numbers of kubernetes objects that can be deployed in the `image-uploader` project. To do this, create a file named `/root/core-object-counts.yaml` with the following content.

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: core-object-counts
spec:
  hard:
    configmaps: "10"
    persistentvolumeclaims: "5"
    resourcequotas: "5"
    replicationcontrollers: "20"
    secrets: "50"
    services: "10"
    openshift.io/imagestreams: "10"
```

Once created, apply these controls to your `image-uploader` project.

```
$ oc create -f core-object-counts.yaml -n image-uploader
resourcequota "core-object-counts" created
```

If you re-run `oc get resourcequota`, you'll see both quotas applied to your `image-uploader` project.

```
$ oc get resourcequota -n image-uploader
NAME                 AGE
compute-resources    9m
core-object-counts   1m
```

The resource guardrails provided by control groups inside OpenShift are invaluable to an Ops team. We can't run around looking at every container that comes or go. We have to be able to programatically define flexible quotas and requests for our developers. All of this information is available in the OpenShift web interface, so your devs have no excuse for not knowing what they're using and how much they have left.

#### 2.2.5: Protection with SELinux

SELinux has a polarizing reputation in the Linux community. Inside an OpenShift cluster, it is 100% required. We're not going to get into the specifics due to time constraints, but we wanted to carve out a time for you to ask any specific questions and to give a few highlights about SELinux in OpenShift.

1. SELinux must be in enforcing mode in OpenShift. This is non-negotiable. SELinux prevents containers from being able to communicate with the host in undesirable ways, as well as limiting cross-container resource access.
2. SELinux requires no configuration out of the box. You can customize SELinux in OpenShift, but it's not required at all.
3. By default, OpenShift acts at the project level. In order to provide easier communication between apps deployed in the same project, they share the same SELinux context by default. The assumption is that applications in the same project will be related and have a consistent need to share resources and easily communicate.

#### 2.2.6: Summary

Wow. OK. We know that's a lot of firehose to point at you. We hope that you now have a solid grasp of how containers are constructed on a Linux server. But containers on their own have a limitation. Every thing we just discussed, namespaces, cgroups and SELinux, are part of Linux kernel. That is great for speed and security. But none of those things have any way of knowing what's going on in the kernel on another server.

Real power comes when you can orchestrate containers for your applications across multiple servers. That's where OpenShift and Kubernetes step in.

### 2.3: Applications in OpenShift

In this section, we'll use both the CLI and web interface to deploy and examine applications in OpenShift. While this certainly isn't an Ops-specific domain, it's required knowledge for all OpenShift users.

#### 2.3.1: Deploying your first applications

We'll be using a [sample application](https://github.com/OpenShiftInAction/image-uploader) which is a simple PHP image uploader application. To build the application we'll be using [Source-to-image (S2I)](https://docs.openshift.com/container-platform/3.11/creating_images/s2i.html). S2I is a workflow built into OpenShift that lets developers specify a few options along with their source code repository, and a custom container image is generated.

S2I uses a developer's source code along with what we refer to as a base image. An S2I base image is like a normal container base image with a little bit of JSON added in. This JSON tells the S2I process where to place the application source code so it can be properly served, along with any other instructions about requirements to build and prepare the application. The only requirement for S2I is that the source code exists in a git repository, like a GitHub or GitLab repository.

#### 2.3.2: Multiple ways to deploy applications

There are other ways to deploy applications into OpenShift that we won't have time to cover today. In short, OpenShift can deploy applications using just about any container-based workflow you can imagine.

* External CI/CD workflows that create images that are pushed into the OpenShift registry
* OpenShift with an internal Jenkins server (coming up in section 4.0)
* Specifying a Dockerfile
* Just about anything you can think of that builds out a container

Enough talk, let's get an application deployed into OpenShift. First, we'll use the CLI.

#### 2.3.3: Using the CLI

Let's make double-sure that we're using our `image-uploader` project.

```
$ oc project image-uploader
```

Inside the `image-uploader` project you'll use the `oc new-app` command to deploy your new application using S2I.

```
$ oc new-app --image-stream=php --code=https://github.com/OpenShiftInAction/image-uploader.git --name=app-cli
--> Found image eb1fe84 (2 weeks old) in image stream "openshift/php" under tag "7.1" for "php"

    Apache 2.4 with PHP 7.1
    -----------------------
    PHP 7.1 available as container is a base platform for building and running various PHP 7.1 applications and frameworks. PHP is an HTML-embedded scripting language. PHP attempts to make it easy for developers to write dynamically generated web pages. PHP also offers built-in database integration for several commercial and non-commercial database management systems, so writing a database-enabled webpage with PHP is fairly simple. The most common use of PHP coding is probably as a replacement for CGI scripts.

    Tags: builder, php, php71, rh-php71

    * The source repository appears to match: php
    * A source build using source code from https://github.com/OpenShiftInAction/image-uploader.git will be created
      * The resulting image will be pushed to image stream tag "app-cli:latest"
      * Use 'oc start-build' to trigger a new build
    * This image will be deployed in deployment config "app-cli"
    * Ports 8080/tcp, 8443/tcp will be load balanced by service "app-cli"
      * Other containers can access this service through the hostname "app-cli"

--> Creating resources ...
    imagestream.image.openshift.io "app-cli" created
    buildconfig.build.openshift.io "app-cli" created
    deploymentconfig.apps.openshift.io "app-cli" created
    service "app-cli" created
--> Success
    Build scheduled, use 'oc logs -f bc/app-cli' to track its progress.
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/app-cli'
    Run 'oc status' to view your app.
```

The build process should only take a couple of minutes. Once the output of `oc get pods` shows your `app-cli` pod in a Ready and Running state, you're (almost) good to go.

```
$ oc get pods
NAME              READY   STATUS      RESTARTS   AGE
app-cli-1-build   0/1     Completed   0          1m
app-cli-1-rkxt4   1/1     Running     0          7s
```

We talked briefly about services and pods and all of the constructs inside OpenShift during the presentation part of this lab. You can see in the output above that you created a service as part of your new application, along with other needed objects. Let's take a look at the `app-cli` service using the command line.

```
$ oc describe svc/app-cli
Name:              app-cli
Namespace:         image-uploader
Labels:            app=app-cli
Annotations:       openshift.io/generated-by: OpenShiftNewApp
Selector:          app=app-cli,deploymentconfig=app-cli
Type:              ClusterIP
IP:                172.30.191.106
Port:              8080-tcp  8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.1.2.8:8080
Port:              8443-tcp  8443/TCP
TargetPort:        8443/TCP
Endpoints:         10.1.2.8:8443
Session Affinity:  None
Events:            <none>     
```

Before we can get to our new application, we have to expose the service externally. When you deploy an application using the CLI, an external route is not automatically created. To create a route, use the `oc expose` command.

```
$ oc expose svc/app-cli
route "app-cli" exposed
```

You may notice that we used `svc` instead of `service` in the above command. Because typing is hard, most objects in OpenShift have an abbreviated syntax you can use on the CLI. Services can also be described as `svc`, DeploymentConfigs are `dc`, Replication Controllers are `rc`, and Pods are `po`. Routes don't have an abbreviation. A full list is available in the [OpenShift documentation](https://docs.openshift.com/container-platform/3.11/cli_reference/basic_cli_operations.html#object-types).

To see and confirm our route, use the `oc get routes` command.

```
$ oc get routes
NAME      HOST/PORT                                                      PATH   SERVICES   PORT       TERMINATION   WILDCARD
app-cli   app-cli-image-uploader.apps.btws-6e50.openshiftworkshop.com          app-cli    8080-tcp                 None
```

If you browse to your newly created route, you should see the Image Uploader application, ready for use.

![Sample Application](/images/app-cli.png)

And that's it. Using OpenShift, we took nothing but a GitHub repository and turned it into a fully deployed application in just a handful of commands. Next, let's scale your application to make it more resilient to traffic spikes.

#### 2.3.4: Scaling an application using the CLI

Scaling your `app-cli` application is accomplished with a single `oc scale` command.

```
$ oc scale dc/app-cli --replicas=3
deploymentconfig.apps.openshift.io "app-cli" scaled
```

Because your second application node doesn't have the custom container image for `app-cli` already cached, it may take a few seconds for the initial pod to be created on that node. To confirm everything is running, use the `oc get pods` command. The additional `-o wide` provides additional output, including the internal IP address of the pod and the node where it's deployed.

```
$ oc get pods -o wide
NAME              READY   STATUS      RESTARTS   AGE   IP          NODE                        NOMINATED NODE
app-cli-1-6hlq6   1/1     Running     0          14s   10.1.2.9    node1.btws-6e50.internal   <none>
app-cli-1-build   0/1     Completed   0          3m    10.1.2.6    node1.btws-6e50.internal   <none>
app-cli-1-cdjd8   1/1     Running     0          14s   10.1.2.10   node1.btws-6e50.internal   <none>
app-cli-1-rkxt4   1/1     Running     0          2m    10.1.2.8    node1.btws-6e50.internal   <none>
```
                  
Using a single command, you just scaled your application from 1 instance to 3 instances on 2 servers in a matter of seconds. Compare that to what your application scaling process is using VMs or bare metal systems; or even things like Amazon ECS or just Docker. It's pretty amazing. Next, let's do the same thing using the web interface.

#### 2.3.5: Using the web interface

The web interface for OpenShift makes additional assumptions when its used. The biggest difference you'll notice compared to the CLI is that routes are automatically created when applications are deployed. This can be altered, but it is the default behavior. To get started, browse to the OpenShift console using HTTPS and log in using your admin username provided by your instructor.

![OpenShift Container Platform Login Page](/images/ocp_login.png)

On the right side, select the Image Uploader Project. You may need to click the _View All_ link to have it show up for the first time.

![OpenShift Container Platform Service Catalog](/images/ocp_project_list.png)

After clicking on the project, you'll notice the `app-cli` project we just deployed. If you click on its area, it will expand to show additional application details. These details include the exposed route, build information, and even resource metrics.

![Image Uploader Project Application Console focused on the app-cli Deployment Config](/images/app-cli_gui.png)

To deploy an application from the web interface, click the _Add To Project_ button in the top right corner, followed by _Browse Catalog_.

![Add to Project -> Browse Catalog Menu in OpenShift Container Platform](/images/ocp_add_to_project.png)

This button brings up the Service Catalog. The 100+ templates available in today's lab are just what's available out of the box in OpenShift. You and your developers can also create custom templates and add them to a single project or make them avaialable to your entire cluster. Other platforms can also be integrated into your OpenShift Catalog. Ansible, AWS, Azure, and other service brokers are available for integration with OpenShift today.

![OpenShift Service Catalog view inside of the Application Console](/images/ocp_service_catalog.png)

Using the _Search Catalog_ form, search for PHP, because the Image Uploader application is written using PHP. You'll get 3 search results back.

![PHP Search Results in the OpenShift Service Catalog](/images/ocp_php_results.png)

Image Uploader is a simple application that doesn't require a database. So we'll just select the PHP builder image, which is the same image we used when we deployed the same application from the command line. Selecting this option takes you to a simple wizard that helps deploy your application. Supply the same git repository you used for `app-cli` (https://github.com/OpenShiftInAction/image-uploader.git), give it the name `app-gui`, and click Create.

![OpenShift PHP Web Console Build Wizard](/images/ocp_app-gui_wizard.png)

You'll get a confirmation that the build has started. Click the _Continue to project overview_ link to return to the Image Uploader project. You'll notice that the `app-gui` build is progressing quickly.

![Image Uploader Project Application Console focused on the app-gui Deployment Config](/images/ocp_app-gui_build.png)

After the build completes, the deployment of the custom container image starts and quickly completes. A route is then created and automatically associated with `app-gui`. And just like that, you've deployed multiple instances of the same application with different URLs onto your OpenShift platform.

Next, let's take a quick look at what is going on with your newly deployed applications within the OpenShift cluster.

#### 2.3.6: OpenShift SDN

OpenShift uses a complex software-defined network solution using [Open vSwitch (OVS)](https://www.openvswitch.org/) that creates multiple interfaces for each container and routes traffic through VXLANs to other nodes in the cluster or through a TUN interface to route out of your cluster and into other networks.

At a fundamental level, OpenShift creates an OVS bridge and attaches a TUN and VXLAN interface. The VXLAN interface routes requests between nodes on the cluster, and the TUN interface routes traffic off of the cluster using the node's default gateway. Each container also cretes a `veth` interface that is linked to the `eth0` interface in a specific container using [kernel interface linking](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-class-net). You can see this on your nodes by running the `ovs-vsct list-br` command.

```
$ ovs-vsctl list-br
br0
```

This lists the OVS bridges on the host. To see the interfaces within the bridge, run the following command. Here you can see the `vxlan`, `tun`, and `veth` interfaces within the bridge.

```
$ ovs-vsctl list-ifaces br0
tun0
veth1903c0e4
veth2daf2599
veth6afcc070
veth77b9379f
veth9406531e
veth97389395
vxlan0
```

Logically, the networking configuration on an OpenShift node looks like the graphic below.

![OpenShift Networking Diagram](/images/ocp_networking_node.png)

When networking isn't working as you expect, and you've already ruled out DNS (for the 5th time), keep this architecture in mind as you are troubleshooting your cluster. There's no magic invovled; only technologies that have proven themselves for years in production.

#### 2.3.7: Routing layer

The routing layer inside OpenShift uses HAProxy by default. It's function is to map the publicly available route you assign to an application and map it back to the corresponding pods in your cluster. Each time an application or route is updated (created, retired, scaled up or down), the configuration in HAProxy is updated by OpenShift. HAProxy runs in a pod in the default project on your infrastructure node.

```
$ oc project default
Now using project "default" on server "https://master.btws-6e50.openshiftworkshop.com:443".
$ oc get pods
NAME                          READY   STATUS    RESTARTS   AGE
docker-registry-1-j7hxb       1/1     Running   0          20h
logging-eventrouter-1-plbqk   1/1     Running   0          20h
registry-console-1-59clb      1/1     Running   0          20h
router-2-2zh8h                1/1     Running   0          20h
```

If you know the name of a pod, you can us `oc rsh` to connect to it remotely. This is doing some fun magic using `ssh` and `nsenter` under the covers to provide a connection to the proper node inside the proper namespaces for the pod. Looking in the `haproxy.config` file for references to `app-cli` gives displays your router configuration for that application. Ctrl-D will exit out of your `rsh` session.

```
$ oc rsh router-2-2zh8h
sh-4.2$ grep app-cli haproxy.config
backend be_http:image-uploader:app-cli
  server pod:app-cli-1-cdjd8:app-cli:10.1.2.10:8080 10.1.2.10:8080 cookie 3da20208fff607747e80c32e4e9a1eb4 weight 256 check inter 5000ms
  server pod:app-cli-1-rkxt4:app-cli:10.1.2.8:8080 10.1.2.8:8080 cookie 1927a94d903b1671af33f2daaad0d86e weight 256 check inter 5000ms
  server pod:app-cli-1-6hlq6:app-cli:10.1.2.9:8080 10.1.2.9:8080 cookie 3259a6df68f9840717af88209f7650e6 weight 256 check inter 5000ms
sh-4.2$ exit
```

If you use the `oc get pods` command for the Image Uploader project and limit its output for the `app-cli` application, you can see the IP addresses in HAProxy match the pods for the application.

```
$ oc get pods -l app=app-cli -n image-uploader -o wide
NAME              READY   STATUS    RESTARTS   AGE   IP          NODE                        NOMINATED NODE
app-cli-1-6hlq6   1/1     Running   0          12m   10.1.2.9    node1.btws-6e50.internal   <none>
app-cli-1-cdjd8   1/1     Running   0          12m   10.1.2.10   node1.btws-6e50.internal   <none>
app-cli-1-rkxt4   1/1     Running   0          14m   10.1.2.8    node1.btws-6e50.internal   <none>
```

To confirm HAProxy is automatically updated, let's scale `app-cli` back down to 1 pod and re-check the router configuration.

```
$ oc scale dc/app-cli -n image-uploader --replicas=1
deploymentconfig.apps.openshift.io "app-cli" scaled

$ oc get pods -l app=app-cli -n image-uploader -o wide
NAME              READY   STATUS    RESTARTS   AGE   IP         NODE                        NOMINATED NODE
app-cli-1-rkxt4   1/1     Running   0          15m   10.1.2.8   node1.btws-6e50.internal   <none>

$ oc exec router-2-2zh8h grep app-cli haproxy.config
backend be_http:image-uploader:app-cli
  server pod:app-cli-1-rkxt4:app-cli:10.1.2.8:8080 10.1.2.8:8080 cookie 1927a94d903b1671af33f2daaad0d86e weight 256 check inter 5000ms
```

We were able to confirm that our HAProxy configuration updates automatically when applications are updated.

### 2.4: Summary

We know this is a mountain of content. Our goal is to present you with information that will be helpful as you sink your teeth into your own OpenShift clusters in your own infrastructure. These are some of the fundamental tools and tricks that are going to be helpful as you begin this journey.

## 3: OpenShift and Ansible integration points

Deploying and managing an OpenShift cluster is controlled by Ansible. The [openshift-ansible](https://github.com/openshift/openshift-ansible) project is used to deploy and scale OpenShift clusters, as well as enable new features like [OpenShift Container Storage](https://www.openshift.com/products/container-storage/).

### 3.1: Deploying OpenShift

Your entire OpenShift cluster was deployed using Ansible. The inventory used to deploy your cluster is on your bastion host at the default inventory location for Ansible, /etc/ansible/hosts. To deploy an OpenShift cluster on RHEL 7, after registering it and subscribing it to the proper repositories two Ansible playbooks need to be run:

```
ansible-playook /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml
ansible-playook /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml
```

The deployment process takes 30-40 minutes to complete, depending on the size of your cluster. To save that time, we've got you covered and have already deployed your OpenShift cluster using Ansible.

### 3.2: Modifying an OpenShift cluster

In additon to deploying OpenShift, Ansible is used to modify your existing cluster. These playbooks are also located in `/usr/share/ansible/openshift-ansible/`. They can do things such as:

* Adding a node to your cluster
* Deploying OpenShift Container Storage (OCS)
* Other operations like re-deploying encryption certificates

Taking a look at the playbook options available from openshift-ansible, you see:

```
$ ls /usr/share/ansible/openshift-ansible/playbooks/
adhoc/		    deploy_cluster.yml	 openshift-cluster-autoscaler/	openshift-logging/		 openshift-nfs/			   ovirt/
aws/		    gcp/		 openshift-console/		openshift-management/		 openshift-node/		   prerequisites.yml
azure/		    init/		 openshift-descheduler/		openshift-master/		 openshift-node-problem-detector/  README.md
byo/		    metrics-server/	 openshift-etcd/		openshift-metering/		 openshift-provisioners/	   redeploy-certificates.yml
cluster-operator/   olm/		 openshift-glusterfs/		openshift-metrics/		 openshift-service-catalog/	   roles@
common/		    openshift-autoheal/  openshift-hosted/		openshift-monitor-availability/  openshift-web-console/		   updates/
container-runtime/  openshift-checks/	 openshift-loadbalancer/	openshift-monitoring/		 openstack/
```

### 3.3: Summary

In section 1, we talked about Ansible fundamentals. In section 2, we worked through the OpenShift architecture and how your cluster can be managed using Ansible.

This section has been about the deeper relationship between OpenShift and Ansible. All major lifecycle events are handled using Ansible.
Next, we'll take a look at an OpenShift deployment that provides everything you need to create and test a full CI/CD workflow in OpenShift.

## 4: A real world CI/CD scenario

In the final section of our workshop, we'll take everything we've been discussing and put it into practice with a large, complex, real-work workflow. In your cluster, you'll create multiple projects and use a Jenkins pipeline workflow that:

* Checks out source code from a git server within Openshift
* Builds a java application and archives it in Nexus
* Runs unit tests for the application
* Runs code analysis using Sonarqube
* Builds a Wildfly container image
* Deplous the app into a dev project and runs integration tests
* Builds a human break into the OpenShift UI to confirm before it promotes the application to the stage project

This is a complete analog to a modern CI/CD workflow, implemented 100% within OpenShift. First, we'll need to create some projects for your CI/CD workflow to use. The content can be found on GitHub at https://github.com/siamaksade/openshift-cd-demo. Let's clone this to your bastion host and change to that directory. To do so, let's run the following commands:

```
cd ~
git clone --branch ocp-3.11 https://github.com/siamaksade/openshift-cd-demo.git 
cd openshift-cd-demo
```

### 4.1: Creating a CI/CD workflow manually

Now, let's start creating the CI/CD workflow manually by running the following commands:
```
oc new-project dev --display-name="Tasks - Dev"
oc new-project stage --display-name="Tasks - Stage"
oc new-project cicd --display-name="CI/CD"
```

This will create three projects in your OpenShift cluster.
* Tasks - Dev: This will house your dev team's development deployment
* Tasks - Stage: This will be your dev team's Stage deployment
* CI/CD: This projects houses all of your CI/CD tools

Next, you need to give the CI/CD project permission to exexute tasks in the Dev and Stage projects.

```
oc policy add-role-to-group edit system:serviceaccounts:cicd -n dev
oc policy add-role-to-group edit system:serviceaccounts:cicd -n stage
```

With your projects created, you're ready to deploy the demo and trigger the workflow

```
oc new-app -n cicd -f cicd-template.yaml --param=DEPLOY_CHE=true
```

This process doesn't take much time for a single application, but it doesn't scale well and it relies on the person executing it knowing the commands, as well as specific information about the situation. In the next section, we'll accomplish the same thing with a simple Ansible playbook, executed from your bastion host.

### 4.2: Automating application deployment with Ansible

There are modules in Ansible for OpenShift, specifically the `k8s` module (also known as `k8s_raw` and `openshift_raw`). However, lots of interactions with OpenShift that you'll find in OpenShift playbooks still use the `command` module. There is minimal risk in doing so because the `oc` command itself is idempotent. 

A playbook that would create the entire CI/CD workflow could look as follows:

```
---
name: Deploy OpenShift CI/CD Project and begin workflow
hosts: masters
become: yes

tasks:
- name: Create Tasks project
  command: oc new-project dev --display-name="Tasks - Dev"
- name: Create Stage project
  command: oc new-project stage --display-name="Tasks - Stage"
- name: Create CI/CD project
  command: oc new-project cicd --display-name="CI/CD"
- name: Set serviceaccount status for CI/CD project for dev and stage projects
  command: oc policy add-role-to-group edit system:serviceaccounts:cicd -n {{ item }}
  with_items:
  - dev
  - stage
- name: Start application deployment to trigger CI/CD workflow
  command: oc new-app -n cicd -f cicd-template.yaml --param=DEPLOY_CHE=true
```

This playbook is relatively simple, with a single `with_items` loop. What sort of additional enhancements can you think of to make this playbook more powerful to deploy workflows inside OpenShift?

## 5: Troubleshooting Applications

Below are common application troubleshooting techniques to use when things aren't working quite right in your cluster. 

### 5.1: Image Pull Failures

When an image pull fails, it is important to consider the various reasons as to why an image pull may fail. Examples include: 

* The image tag is incorrect
* The image doesn’t exist (or is stored in a different registry)
* OpenShift doesn’t have the proper permissions to pull that image

To demonstrate this, let's first create a project to play with by running the following command:

```
$ oc new-project troubleshooting
Now using project "troubleshooting" on server "https://master.btatl-6e50.openshiftworkshop.com:443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example
```

Next, let's attempt to deploy a container image with a typo to demonstrate this issue. 

```
$ oc run fail --image=rh/fail-exit:latest
kubectl run --generator=deploymentconfig/v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deploymentconfig.apps.openshift.io/fail created
```

Ignore the deprecated message and let's check the status of the pod:

```
$ oc get pods
NAME            READY   STATUS             RESTARTS   AGE
fail-1-deploy   1/1     Running            0          1m
fail-1-gd5pt    0/1     ImagePullBackOff   0          1m
```

Now, let's look at the events for the pod that is stuck in ImagePullBackOff by running oc describe pod:

```
$ oc describe pod/fail-1-gd5pt
```

As we can see under the _Events_ section, the pod failed because it could not find the image.

```
Name:               fail-1-gd5pt
Namespace:          troubleshooting
Priority:           0
PriorityClassName:  <none>
Node:               node1.btatl-6e50.internal/192.168.0.71
Start Time:         Wed, 17 Jul 2019 12:24:13 -0400
Labels:             deployment=fail-1
                    deploymentconfig=fail
                    run=fail
Annotations:        kubernetes.io/limit-ranger: LimitRanger plugin set: cpu, memory request for container fail; cpu, memory limit for container fail
                    openshift.io/deployment-config.latest-version: 1
                    openshift.io/deployment-config.name: fail
                    openshift.io/deployment.name: fail-1
                    openshift.io/scc: restricted
Status:             Pending
IP:
Controlled By:      ReplicationController/fail-1
Containers:
  fail:
    Container ID:
    Image:          rh/fail-exit:latest
    Image ID:
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  1536Mi
    Requests:
      cpu:        50m
      memory:     256Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-wrx8q (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  default-token-wrx8q:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-wrx8q
    Optional:    false
QoS Class:       Burstable
Node-Selectors:  node-role.kubernetes.io/compute=true
Tolerations:     node.kubernetes.io/memory-pressure:NoSchedule
Events:
  Type     Reason          Age                    From                                Message
  ----     ------          ----                   ----                                -------
  Normal   Scheduled       2m29s                  default-scheduler                   Successfully assigned troubleshooting/fail-1-gd5pt to node1.btatl-6e50.internal
  Normal   Pulling         2m13s (x2 over 2m26s)  kubelet, node1.btatl-6e50.internal  pulling image "rh/fail-exit:latest"
  Warning  Failed          2m11s (x2 over 2m24s)  kubelet, node1.btatl-6e50.internal  Failed to pull image "rh/fail-exit:latest": rpc error: code = Unknown desc = repository docker.io/rh/fail-exit not found: does not exist or no pull access
  Warning  Failed          2m11s (x2 over 2m24s)  kubelet, node1.btatl-6e50.internal  Error: ErrImagePull
  Normal   SandboxChanged  2m5s (x7 over 2m23s)   kubelet, node1.btatl-6e50.internal  Pod sandbox changed, it will be killed and re-created.
  Normal   BackOff         2m3s (x6 over 2m22s)   kubelet, node1.btatl-6e50.internal  Back-off pulling image "rh/fail-exit:latest"
  Warning  Failed          2m3s (x6 over 2m22s)   kubelet, node1.btatl-6e50.internal  Error: ImagePullBackOff
```

Now that we've seen that it is broken, let's fix it. The proper image is located at quay.io/redhat/fail-exit. Let's patch the deployment config with the updated image:

```
$ oc patch dc/fail --patch='{"spec":{"template":{"spec":{"containers":[{"name": "fail", "image":"quay.io/redhat/fail-exit:latest"}]}}}}'
```

Now let's check the status of the pod again:

```
NAME            READY   STATUS             RESTARTS   AGE
fail-1-deploy   0/1     Error              0          20m
fail-2-deploy   1/1     Running            0          1m
fail-2-ssh7k    0/1     CrashLoopBackOff   3          1m
```

Well, we've fixed the image pull error, but now we've got a new error. Move on to the next section to troubleshoot this issue.

### 5.2: Application Crashing

Now that we have a pod that is in a `CrashLoopBackOff` state, we need to figure out why the container is failing:

```
$ oc get pods
NAME            READY   STATUS             RESTARTS   AGE
fail-1-deploy   0/1     Error              0          20m
fail-2-deploy   1/1     Running            0          1m
fail-2-ssh7k    0/1     CrashLoopBackOff   3          1m
```

Now, let's take another look at the detailed status of the pod to see what is going on. Specifically, we want to investigate the `Reason` under the `State` property, so let's use grep to limit our output:

```
$ oc describe pod/fail-2-ssh7k | grep -E "State:|Reason:"
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
```

We can also take a look at the crash data from the _Events_ tab in the crash looping pod.

![The events of a crash looping pod](/images/failing_events.png)

Now that we've found that the container is exiting with a status of 1, I can let you in on the secret: This container image's command is "exit 1". You can check out the dockerfile [here](/fail.Dockerfile).

Now that we've gotten here, let's go ahead and delete this deployment config and move on to some other troubleshooting exercies. 

```
$ oc delete dc/fail
```

### 5.3: Troubleshooting access to containers  

Next, let's talk about how to troubleshoot access to your pods and containers from external endpoints and internal endpoints.

First, a few important pieces of information: 
* [Troubleshooting OpenShift SDN](https://docs.openshift.com/container-platform/3.11/admin_guide/sdn_troubleshooting.html#overview)
* [List of HTTP status codes](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)  
  - 1xx (Informational): The request was received, continuing process
  - 2xx (Successful): The request was successfully received, understood, and accepted
  - 3xx (Redirection): Further action needs to be taken in order to complete the request
  - 4xx (Client Error): The request contains bad syntax or cannot be fulfilled  
  - 5xx (Server Error): The server failed to fulfill an apparently valid request

Next, let's deploy a quick sample application to work with:

```
$ oc new-app --name=network-example https://github.com/jboss-openshift/openshift-quickstarts --context-dir=undertow-servlet -i java:latest
```

Now let's expose the service externally to the internet and store that hostname in a variable:

```
$ oc expose svc/network-example
$ ENDPOINT=http://$(oc get route | grep network-example | awk '{print $2}')
```

Now, let's debug external access to the service:

```
$ curl -kv $ENDPOINT
* Rebuilt URL to: http://network-example-troubleshooting.apps.btatl-6e50.openshiftworkshop.com/
*   Trying 3.218.186.47...
* TCP_NODELAY set
* Connected to network-example-troubleshooting.apps.btatl-6e50.openshiftworkshop.com (3.218.186.47) port 80 (#0)
> GET / HTTP/1.1
> Host: network-example-troubleshooting.apps.btatl-6e50.openshiftworkshop.com
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Length: 11
< Date: Wed, 17 Jul 2019 17:13:45 GMT
< Set-Cookie: bdf8c5d069e23263d0c1d884fb88fb93=475e6f2c7603354e481c3a155c346c41; path=/; HttpOnly
< Cache-control: private
<
* Connection #0 to host network-example-troubleshooting.apps.btatl-6e50.openshiftworkshop.com left intact
```

Note that the HTTP response code is 200, so all appears to be well with this application. 

Let's do a few other troubleshooting steps while we're at it, just to make sure all is well. First, let's check to make sure that the DNS properly resolves: 

```
$ dig +short $ENDPOINT
3.218.186.47
```

Now let's take that IP address and verify that you can access port 80:

```
$ telnet 3.218.186.47 80
Trying 3.218.186.47...
Connected to ec2-3-218-186-47.compute-1.amazonaws.com.
Escape character is '^]'.
^]
telnet> quit
Connection closed.
```

All looks well with this external route, let's move on!

## 6: Q&A and Wrap-Up
