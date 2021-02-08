# Cloud - first demo

In this chapter, we are going to try the same thing we did locally, but in the public cloud. We will cover single node, allowing to run MicroK8s with only it's official supported addons, and we will use from begining some configuration that will allow us to upgrade to multi-nodes in the feature.

To ensure upgrade to multiple nodes will be possible, some things need to be thought and decided.

Most of the online resources will point towards using Kubernetes multiple nodes on same network, that being set as in same VPC (Virtual Private Cloud) in Amazon AWS or other providers, to ensure you can use an internal private network and share easier certain resources, like networked file system, and focus to protect the cluster from one internet single entry point.

Let's create a private network in our project.

But let's focus on the simple implementation, just one node, MicroK8s with it's official addons, and make sure it can be extended.

I will be using [Hetzner](http://j.mp/3cLf8hH) for my public cloud provider. Using them, will require to go through the process to validate your identity, as they are very strict on security and avoiding fraud. You should use any prefered public cloud although, and adapt to their dashboard, as things are mostly common between clouds. I personally choosed Hetzner, as being the lowest in price, while having great performance and security, and suits my personal projects needs saving me a lot of money. I am not concerned about locations of servers, as on top I use Cloudflare which will ensure to deliver with lowest latency responses to the visitors - more in a later chapter about this and alternatives to Cloudflare.

If you want to read some comparison done by [Rasmus Lerdorf](http://j.mp/2YPYfua) on decent priced public cloud servers, it might help you in making a decision. I think he's be a better reference than me to influence you in your choice.

Let's start with one basic cloud server, sized at 1 core and 2GB of ram to experiment.

Hetzner supports the separation of resources by naming them Projects. Either use the default one or, better, create a new dedicated one to learn. Resources in one project are isolated from other projects. Projects also allow you to invite collaborators, and they will be limited at project level. A project can also be later transfered to a client.

![Create project](_static/images/hetzner-create-project.png)

Click on **+ NEW PROJECT**, give it a name, example `wpk8s` and click **ADD PROJEC**.

![Create network](_static/images/hetzner-create-network.png)

Next, on the left sidebar, click **Networks**, click **CREATE NETWORK**, name it for example `wpk8s`, and give it a range, like `172.16.0.0`. This would be full class B private network, Hetzner would create subnetworks inside of it, and that will give us 65534 auto assignable ip addresses that on any cloud server with this network attached will be preconfigured if selected on creation, or configurable manually with a very simple how linked from the dashboard. As a note why I use class B networks, as it is less used in local networks. Also avoid `` 10.0.0.0/8 `` as it is used by default with Kubernetes.

![Create ssh key](_static/images/create-ssh-key-win10.png)

If you have not used ssh, you need to generate one.

On Windows 10, if you have it recently installed, you should have ssh preinstalled (openssh). If not, open **Settings** > **Applications**, click **Optional features** link. If you see **OpenSSH client** you have it, if not, click on **Add feature**, search `openssh` and install the **client**; server is available also, so make sure you do not select that one! Now open **CMD** or **Windows Terminal** and type `ssh-keygen -b 4096 -C 'some-word-to-identify-this-computer'` replacing the _some-word-to-identify-this-computer_ with something representative for you. After a few **Enter** presses, when back on prompt, run `` type .ssh\\id_rsa.pub | clip ``. This will copy the public version of the key in the clipboard, and you can paste it now in Hetzner's panel.

On MacOS, open **Terminal** and run `ssh-keygen -b 4096 -C 'some-word-to-identify-this-computer'` replacing the _some-word-to-identify-this-computer_ with something representative for you. After a few **Enter** presses, when back on prompt, run `cat .ssh/id_rsa.pub | pbcopy`.

In Linux, like on MacOS, to copy in the clipboard run `cat .ssh/id_rsa.pub` and select and copy with the mouse. An alternative to _clip_ and _pbcopy_ does not come by deafault with most distributions and simple there are a lot of them installable, and different between X11 and Wayland.

![Add the ssh public key](_static/images/hetzner-add-ssh-publickey.png)

In the Hetzner's panel, in the sidebar, click **Security**, **ADD SSH KEY**, paste the key, it's name will be prefilled with the comment name you used when generated, but you may change it to identify it easy here in the panel.

**Create a new server.** Click on **Servers**, **ADD SERVER**

**Select a location**. For the future, using multiple locations is ideal for High Availability, something I will detail in different scenarios in the following chapters - worry free, no alarms at 3AM. I use servers in same location to do horizontal scalling and multiple locations to setup replications for high availability or disaster recovery, depends on how the clients choose.

**Select Ubuntu 20.04 or latest LTS version**. Setup can be done on other distributions, but for simplicity, I will demo using Ubuntu.

**Select Standard with Local storage type**. Possible to respond to why local and not **Network (CEPH)**, simply because IO (input/output) operations with the disk are close to native ssd/nvme on your local computer, and network is slow; when we will see how to do advanced setup for MariaDB (MySQL), you will understand the critical benefit achived and how easy we can set either replication or backups to recover very quickly or automatically from disasters, achiving maximum performance in read/write operations, most costly for any application with database access. If performance with the database is not critical in your situation and you intend to use a single node only, network storage type could help you to allow Hetzner to recover in a few minutes from a failure, as they move cloud servers to other hosts in the datacenter, and the volumes are already accesible from network, also being triple replicated and highly available for any situation.

**Select CX11**. When you will select for production and large setups, please note that switching between *CX?* and *CXP?* cloud servers is not possible, even with network storage (possible will change in the future for network storage type). I'm choosing _CX?_ (Intel) types in general, but _CXP?_ (AMD) types work well for having higher cpu cores ratio towards memory; _CXP?_ is good candidate to isolate php workers and web servers and _CX?_ prefered for database or cache services for more ram. Benefit of choosing is financial balance in the end.

Also a strong note, which applies to any public cloud, not Hetzner only, **Standard** or **Shared** (how Digital Ocean names) instance types are not intended for long continuous processing, so I strongly advise you to avoid using them for continuous processing like video conversion, optimizing images. If this is done ocassionally within processes of a few minutes, it is fine, but having them for hours, it's breaking the terms and conditions of usage in all public clouds. For this kind of almost non-stop processing, use **Dedicated** types.

**Select the network we created**. This will automatically setup the private network in the cloud server and doing this even for a single node, will help you for future expansion.

Click on user data and type or copy/paste the following, changing the user if you wish so:

{format: shell}
```
#!/bin/bash
set -euo pipefail

USERNAME=wpk8s # TODO: Customize the sudo non-root username here

# Create user and immediately expire password to force a change on login
useradd --create-home --shell "/bin/bash" --groups sudo "${USERNAME}"
passwd --delete "${USERNAME}"
chage --lastday 0 "${USERNAME}"

# Create SSH directory for sudo user and move keys over
home_directory="$(eval echo ~${USERNAME})"
mkdir --parents "${home_directory}/.ssh"
cp /root/.ssh/authorized_keys "${home_directory}/.ssh"
chmod 0700 "${home_directory}/.ssh"
chmod 0600 "${home_directory}/.ssh/authorized_keys"
chown --recursive "${USERNAME}":"${USERNAME}" "${home_directory}/.ssh"

# Disable root SSH login with password
sed --in-place 's/^PermitRootLogin.*/PermitRootLogin prohibit-password/g' /etc/ssh/sshd_config
if sshd -t -q; then systemctl restart sshd fi
```

[Gist link](https://j.mp/3p08OW2)

*original source: [Digital Ocean](http://j.mp/3pHxfZt)*

Select the SSH key you added.

Give it a __fully qualified domain name__ as name. I'll be using `vm1.hel1.wp.k8s`. Make sure you will use a proper **[fqdn](https://en.wikipedia.org/wiki/Fully_qualified_domain_name)**, as this is important when you expand to multi-nodes. The domain name does not need to be a real registred one. I use domains with DNS that does not exists, to avoid dns issues in case somebody registers it.

Hit the Create button and wait a few seconds, usually 15-30 seconds, and you can copy the server's ip, just a click on it is necesarry and back in your terminal ssh into it, example: `ssh wpk8s@135.181.93.201`. It will ask first time to give it a password. Type a decent strong password, hit enter, you will need to confirm it, and will exit. Now hit up key again and ssh again in it. It will login without asking for the password.

From now on, the password will be required only when we run commands using `sudo` or if you'll need to ssh in from another device.

Next we can do exactly the same scenario we did in the local experiment from the previous chapter, except that we need to install **snap** to be able to install MicroK8s. The following first commands should be run anyway in an online environment.

Run updates:

`sudo apt update`

(This will refresh system's package database).

If there are updates, you can list as suggested by running:

`sudo apt list --upgradable`

and IF within the list there are packages part of **focal-security**, I strongly recommend you run

`sudo apt upgrade -y`

If within the list there is the linux kernel also, starts with **linux-image**, please do reboot the virtual server after it finishes, and ssh in back after around 15-30 seconds.

Now install **snap** (make sure you type **snapd** for package installation, as snap package older and kept this namespace):

`sudo apt install -y snapd`

Next add MicroK8s:

`sudo snap install --classic microk8s`

It will download and install latest version of it, and we can do again our previous first WordPress deployment.

A tip for Visual Studio Code users. If you have installed the **[Remote SSH](http://j.mp/2MwT3ZO)**, follow this [help](http://j.mp/3tzUGGs) for making life easy when you will have to work with a lot of cloud servers.

To recap the identical scenario that we have experimented locally:

Enable DNS, Ingress and Storage addons.

`sudo microk8s enable dns ingress storage`

Create a folder named `wordpress` and in it, use **nano** or your favorite editor to create the same files we did previous.

For Visual Studio Code users, if you did follow my advice for **Remote SSH**, you should have discovered how quickly you can connect to the server and now is allowing you to edit files like you would be local and if you open a Visual Studio Code **Terminal** it is actually opening the prompt through ssh in the cloud server. Just enjoy simplicity.

I'll exemplify how to use **nano** to create this 3 files.

`mkdir wordpress; cd wordpress`

`nano kustomization.yml` and paste:

{format: yaml}
```
secretGenerator:
- name: mysql-pass
  literals:
  - password=password123
resources:
  - mysql-statefulset.yaml
  - wordpress-statefulset.yaml
```

Now press `CTRL+x`, hit `y` on **Save modified buffer?**, confirm name by hitting `Enter`.

`nano mysql-statefulset.yml` and paste:

{format: yaml}
```
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  serviceName: wordpress-mysql
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mariadb:10.5
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: wordpress-mysql
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: wordpress-mysql
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

Now press `CTRL+x`, hit `y` on **Save modified buffer?**, confirm name by hitting `Enter`.

Before you continue, if you want to use a real domain, replace **wordpress.k8s** with the exact domain and make sure to use it for the rest of instructions as well (hosts file, browser etc.).

`nano wordpress-statefulset.yml` and paste:

{format: yaml}
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  rules:
  - host: wordpress.k8s
    http:
      paths:
        - pathType: Prefix
          path: "/"
          backend:
            service:
              name: wordpress
              port:
                number: 80
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: wordpress
    tier: frontend
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  serviceName: wordpress
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      initContainers:
      - name: init-mysql
        image: busybox
        command: ['sh', '-c', 'until nslookup wordpress-mysql; do echo waiting for mysql; sleep 2; done;']
      containers:
      - image: wordpress:5.6
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress
          mountPath: /var/www/html
  volumeClaimTemplates:
  - metadata:
      name: wordpress
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

Now press `CTRL+x`, hit `y` on **Save modified buffer?**, confirm name by hitting `Enter`.

Run `sudo microk8s kubectl apply -k ./`, and while we wait 1-2 minutes to be 1st time deployed, let's put in our local hosts file the ip of the cloud server to test our online WordPress installation.

On Windows, right click on the **Start** icon and click on **Command Prompt (admin)** or **Powershell (admin)**, whichever is available. After confirming privileges elevation run `notepad drivers\\etc\\hosts` and add `172.16.0.1 wordpress.k8s` at the bottom of the file - **make sure to use the ip from the cloud server you have created instead of 172.16.0.1**.

On MacOS or Linux, open the terminal and run `sudo nano /etc/hosts` and add `172.16.0.1 wordpress.k8s` at the bottom of the file - **make sure to use the ip from the cloud server you have created instead of 172.16.0.1**. `Ctrl+x`, `y`, `Enter` will save and exit.

After you saved the hosts file, try loading http://wordpress.k8s in the browser.

Note: I did not setup yet anything for **HTTPS** as we are going to look in different ways we can setup that, as you might need to know each one, depending on how your clients need to have **HTTPS** setup. This also can be a benefit for increased security in one scenario I really enjoy using, and fully detailed in a future chapter.

OK, so, we did it, we have live online WordPress, running on MicroK8s. What is next?

In the following chapter we will concentrate on how to prepare _recipes_ to ensure the services use exact versions we need and we will experiment in upgrading them, as this is a critical requirement in a secured environment and also a benefit for avoiding bugs.

Also, we will see to setup multiple websites. We will detail running WordPress with self updates and than look into locking and managing WordPress like other modern php projects, by using [Composer](http://j.mp/3rt7ay4) and [Roots - Bedrock](http://j.mp/2LtvPmK) for modern WordPress development, that helps us to put WordPress in CI/CD - one dedicated chapter following to help you discover this DevOps style of working with WordPress, in an easy maner for everybody.

Will also extend our **WordPress** websites with extra services like ElasticSearch to have blazing speed in search, and instant suggestions, and add caching using Redis, add monitoring and observability, alerts and many other things. If I really captured your atention, I do hope you will enjoy the full book.