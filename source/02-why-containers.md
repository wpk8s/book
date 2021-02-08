# Why containers?

This chapter's purpose is to understand what are containers and the benefits of running services isolated in them.

**Containers** are a simple native capability of the Linux kernel, concept that exists also in other operating systems, somehow different, but from abstract view, they offer same business value: isolation and resource limits.

Containers are an implementation of *OS-level virtualization*[^os-level-virtualization]
[^os-level-virtualization]: Read more about [OS-level virtualization](https://en.wikipedia.org/wiki/OS-level_virtualization)

In our case, main interest would be how WordPress could run this way and all required services.

First, we would have one or more containers that provide PHP and all required extensions that make WordPress run and allow different plugins to enhance the website. How it will run, it's up to how we prefer PHP to run, as there are multiple ways of running php, and there are different oppinions which is the best way.

PHP can run either as an Apache HTTPD module, a cgi process which a web server like Apache HTTPD or Nginx can use by fastcgi or old slow cgid mode, as PHP-FPM, the native process manager of PHP or by command line with it's native server capability.

From the above, running as PHP-FPM or Apache HTTPD module are the best modes for containerizing WordPress main process. Both are almost identical in regards of performance for response time, PHP-FPM winning in regards of lower memory usage, especially when we want to benefit easy horizontal scalling and some extra features that will be discussing later.

**Horizontal scaling**[^horizontal-scaling] - when our website would face high amount of traffic, for example an ecommerce website on Black Friday sales, we need to ensure that all services will have enough resource power to face the hits. We can either scale up the virtual server or the shared hosting package we use, if it is possible, or we could increase the amount of instances that run our website, and we balance the traffic to the instances that are easier to just increase their numbers more and more. So horizontal scalling is the technique through which we multiply the necesarry services without changing configuration and we consume more and more from the allowed infrastructure, in a maner that is more accurate and measurable and without facing any downtime, we can as well decrease, resulting in minimising the costs, while we continuous serve the visitors. **No downtime, no slowdown!**

[^horizontal-scaling]: Read more about [Scalability](https://en.wikipedia.org/wiki/Scalability#Horizontal_(scale_out)_and_vertical_scaling_(scale_up))

A PHP container image, would be constructed on top of a linux distribution minimal environment, by adding all required dependencies that it needs. But we do not have to do this ourselves, as prebuilt images exists, and even dedicated WordPress starting images have been built by the Docker community, which we can use to understand how it runs. As we advance through our workshop, we will start by getting the community's example image quickly up and running and we will continue by constructing our own extended images that we want total control over them.

Another container image would offer MySQL, MariaDB or Percona Server for MySQL. They do not limit in anyway the features and to run them, it is even more easier that setting up yourself, even if you'd be using an installer or linux distribution's package manager. The official community's container images, expect to set through environment variables the name of the database, user and password and would simply initiate and start the server. They are already prepared for running within containers and tweak configuration based on cpu or memory limits, allowing us also to override anything like we would do for a normal installation as well.

Running MySQL in containers offers two more benefits than running it classic.

Major one is that we are trully isolating data between websites, as each container would run an instance of the service and we have one database per instance. If we have multiple WordPress websites, each one could benefit of dedicated MySQL service, ensuring availability is as per defined limits. This might be something in the interest of websites that work with user and epayments data, and this level of isolation would be the most beneficial.

The other benefit is that you can run multiple versions of MySQL and later on we will see that upgrading the database for a website, is not putting in danger of taking down all websites if some would break on a more recent server version. This gives you a breath of air when doing upgrades and avoids the embaracement of sending dozens of appologies to custommers with explanations for the suffered downtime. And as we are in the world of containers, we will learn a technique that can provide no downtime upgrades even in case of a broken upgrade. I bet you really want to be able to do that. Don't worry, you will learn in this book the technique of zero downtime upgrades.

Next, we can add additional services to offer higher performace to WordPress and worry free when we use horizontal scale technique. With plugins that offer object caching, page caching, enhanced searching, all in the benefit of increased performance, adding this services to WordPress is, again, easy as official supported images with configuration through environment exists in the community.

Another benefit that I really want to point out is easier centralised logging and integrate with a system that could be run either external or in same cluster to monitor and including to alert when issues are instantly discovered. We will test out Elastic Search and Prometheus for this purpose, two solutions which are available as addons that are simple to enable with MicroK8s and instead of learning how to set them up, you simply learn how to use them, and in this book I will teach you the best way in regards to WordPress.

To point out about the communities around containers, it's made by a huge amount of people that constantly participate in keeping container images as perfect as could be, and making life really simple in using them, great benefit for everybody, either advanced or beginner in this.

The most known community is centralizing efforts on Docker Hub[^docker-hub], that offers automation for building container images based on Dockerfile recipes, hosts the generated image files, and has a big team of specialists, most of them paid by participating companies in this effort, for currating, maintaining and ensuring that official images will run as expected with almost no issues. This is the community I had trust since beginning and never failed for me.

[^docker-hub]: Docker Hub's official link [https://hub.docker.com](https://hub.docker.com)

Alternative comunities or commercial based image repositories exists, backed by RedHat, Microsoft's Github and many other, but their usage goes beyound our purpose, of running WordPress in containers, using one Kubernetes simplified platform, MicroK8s.

So in the end, why containers? Stability, simplicity, performance, observability, scalability and high availability. Did I also mention automation? Well, let's dive in!