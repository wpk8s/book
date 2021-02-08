# First experiment (local)

Let's try out a first experiment locally, no money spending on cloud resource, and also to learn how to work locally using MicroK8s.

For my local setup, I'm going to use a PC, with Windows 10 Home Edition[^win10homehyperv], VirtualBox 6.1.x and Vagrant 2.2.x, as this tools could be used identical on Windows, MacOS and Linux distributions. Alternative, any virtualization solution you enjoy can be used with the goal to get a virtual machine with some specific networking setup, which it's close to what is in public cloud.

[^win10homehyperv]: For Windows 10 users of WSL2 or Docker for Windows, make sure you have also Windows Hypervisor Platform or VirtualBox will not be able to start any virtual machine, as it is designed to use Hyper-V if Microsoft's virtualization solution is enabled for WSL2. Possible this will be already set right in the near future by Microsoft.

For MacOS and Linux users, there should not be any special things to consider about using VirtualBox and Vagrant, as they work out of the box, and do know how to autoconfigure anything needed.

In Visual Studio Code, use Open Folder, browse and create in a location of preference a folder named `tryMicroK8s` for example. Within it, let's create a file named `Vagrantfile`:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"

  config.vm.network "private_network",
    type: "dhcp"

  config.vm.provider "virtualbox" do |vb|
    vb.check_guest_additions = false
    vb.linked_clone = true
    vb.memory = "4096"
    vb.cpus = 2
  end
end
```

Next, press `` Ctrl+` `` (`` Cmd+` `` on Macos) and at the opened prompt, type

 `vagrant up`

and hit `Enter`.

If this is the first time it runs, it will download the official Ubuntu Linux image for Virtualbox and the normal process of creating and booting the machine will continue, taking probably a few minutes, depending on internet download speed and computing power of your device. If default virtual network did not exist before, vagrant will instruct VirtualBox to create one and you will see a request to allow this.

Tip: using the shortcut `` Ctrl+`` ` you can show and hide anytime the terminal in Visual Studio Code.

Once it finishes, run `vagrant ssh` and it will create a connection in the virtual machine, allowing us to run any commands to _handle the server_.

To get MicroK8s up and running, all we need is run this:

 `sudo snap install microk8s --classic`

This will download all it needs from the official online location that Canonical manages, install the necesarry services and run the Kubernetes cluster.

Run next `sudo microk8s status -w` and within a few seconds, you will see the ready status of the cluster.

We can enable now the minimal addons that we need to work with WordPress:

 `sudo microk8s enable dns storage ingress`.

**DNS** is the addon that will allow micro-services to be able to communicate in the internal networks the Kubernetes cluster will handle, by using hostnames. In our first example, `wordpress` service will be able to use `wordpress-mysql` database service, simply using the hostname with identical name, making us life easy when _baking recipes_ for our WordPress websites.

**STORAGE** is MicroK8s local storage addon, that simply enables creation of local volumes to hold permanent data. For single node, it is one quick way of getting the cluster with permanent storage for containers.

**INGRESS** is simply a service that runs Nginx, the minimal official Kubernetes supported ingress. It will work as a proxy balancer, that understands virtual hosting and allows to set many rules we would like to be handled outside of our WordPress services. As our WordPress main service could be scalled up to multiple instances to support high amount of traffic, the INGRESS service is a key component. Also, it allows us easy to run unlimited amount of WordPress websites in our cluster, depending on it's size.

In the Visual Studio Code's Workspace, create a folder, name it `wordpress` and within it create 3 files:
* `kustomization.yml`
* `mysql-statefulset.yml`
* `wordpress-statefulset.yml`

For each one, insert the next content:

Paste in `kustomization.yml` [gist link](https://j.mp/3q0UdLp):

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

Paste in `mysql-statefulset.yml` [gist link](https://j.mp/3cRFHSq):

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

Paste in `wordpress-statefulset.yml` [gist link](https://j.mp/2MJJMNZ):

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

Get back to the terminal (`` Ctrl+` ``), if you are not in the machine's shell, run `vagrant ssh`, and now change directory to our shared workspace, in the WordPress folder: `cd /vagrant/wordpress`. Run

 `sudo microk8s kubectl apply -k ./`

This command will instruct Kubernetes to load the _recipe_ and create all required resources, add a secret for mysql password, start the services and create an ingress entry to allow outside access to our WordPress website.

 `sudo microk8s kubectl get all`

With this command, you can check the status of all resources. When all is provisioned, you should see something like:

```
NAME                    READY   STATUS    RESTARTS   AGE
pod/wordpress-mysql-0   1/1     Running   0          11s
pod/wordpress-0         1/1     Running   0          11s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/kubernetes        ClusterIP   10.152.183.1    <none>        443/TCP    17m
service/wordpress-mysql   ClusterIP   None            <none>        3306/TCP   12s
service/wordpress         ClusterIP   10.152.183.94   <none>        80/TCP     12s

NAME                               READY   AGE
statefulset.apps/wordpress-mysql   1/1     12s
statefulset.apps/wordpress         1/1     12s
```

Next, let's discover the local ip assigned to our virtual machine, so we can put it in our host system's hosts file. Run `ip addr | grep enp0s` in the shell and look for an IP similar to `172.28.128.9`. Vagrant and VirtualBox will assign usually on default settings an ip within `172.28.128.x` range.

In Windows, press Start, type `cmd`, right click on the result and click `Run as administrator` and click `Yes` to allow higher privileges. In the opened prompt, run `cd drivers\etc` and than `notepad hosts`. This will open Notepad with the hosts file opened. Somewhere at the end of the file add next line, but **make sure to use the IP you identified previously**.

`172.28.128.9 wordpress.k8s`

Open a browser and load `http://wordpress.k8s`. You should be loading the WordPress installation page. Do the installation, post something, including some image and let's move to next step.

Let's do an experiment. Let's delete our WordPress installation.

 `sudo microk8s kubectl delete -k ./`

Try to reload the website in the browser. 504 or 404, eventually sticking to 404. As expected, we deleted the application. You can also use `sudo microk8s kubectl get all` to check how they get destroyed.

Let's bring it back:

 `sudo microk8s kubectl apply -k ./`
 `sudo microk8s kubectl get all`

When all are ready, faster than first time, in the browser, we can see the website back, as was before we deleted it.

The trick was in how we have declared volume allocation and type of resource are the applications. In terms of containers, our setup is composed from two applications until this moment, one the WordPress container that runs Apache and PHP embeded as a module, serving from a dedicated volume that will have WordPress installed when launched for the first time, and another application, our MySQL compatible service, running MariaDB, also having allocated a permanent volume.

We defined our applications as StatefulSets, to protect us in case of deleting them by mistake, allowing to recreate all in a matter of seconds.

Let's proceed to next chapter, where we will create our first MicroK8s Kubernetes cluster out in public cloud.
