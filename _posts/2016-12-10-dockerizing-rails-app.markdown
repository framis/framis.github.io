---
layout: post
title:  "Migrating a Rails app to Kubernetes"
date:   2016-07-26 08:00:00 -0700
categories: ruby
---

In my current job, we moved a Ruby on Rails app from a classic Ubuntu hosting in a private instance to Docker on Kubernetes. We will see the steps in this article:

- Migrating the local environment from Vagrant to Docker
- Using Continuous Delivery with Jenkins
- Running the Docker image with Kubernetes

Migrating the local environment from Vagrant to Docker
----------------------------

We used [Vagrant](https://www.vagrantup.com/) locally in order to load the app with all its dependencies. 

### Vagrant

The main advantages of Vagrant is that a new developer working on the app can spawn a working app in a few minutes in a local Ubunutu Virtual Machine with all its dependencies regardless the dev environment he is using (MacOS like us, Linux or Windows). We used [VirtualBox](https://www.virtualbox.org/wiki/Downloads) to manage our VMs. The direct depedencies were:
- A specific Ruby version
- A specific Rails version
- MySQL
- RabbitMQ
- Other gems as mentionned in the Gemfile

Vagrant is especially useful to start the Database and RabbitMQ locally run functional tests.

The only problem with Vagrant is that a Virtual Machines are pretty heavy and slow to create. For that reason, we decided to move our local environment to Docker containers.

### Docker CLI

The move is not too hard. The first thing you want to do is download the Docker Toolbox or the new Docker for Mac if you are using Mac. The differences are explained [here](https://docs.docker.com/docker-for-mac/docker-toolbox/). When we first got started with Docker, docker was not running natively on Mac OS and we had to setup a VM to run docker with Boot2docker. Then, Boot2docker has been merged into docker-machine and packaged into Docker Toolbox. Now, Docker is running natively on Mac...

Follow the steps [here](https://docs.docker.com/docker-for-mac/) to setup Docker on your machine. Once your local docker CLI is working we can move to the next step. 


### Dockerfile

Docker uses a Configuration file called Dockerfile to provision your app. Think of it as the equivalent of Chef or Puppet. Dockerifle is a simple DSL (Domain Specific Language) that allows you to inherit from a base image and move files around to your docker container.

For your local setup, you do not need any web server such as [Phusion Passenger](https://www.phusionpassenger.com/) and the Webrick or Puma local server is enough. Below a simplified example of a Dockerfile for our app.

Dockerfile

```
FROM ruby:2.3

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app/
ADD . /usr/src/app

RUN bundle install

# Cleanup
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /var/tmp/* /tmp/* /usr/src/app/tmp
```

You see that we start from a ruby2.3 base image. This image is the Docker [official ruby image](https://hub.docker.com/_/ruby/), running on Debian Jessie. You can read the base image Dockerfile [here](https://github.com/docker-library/ruby/blob/master/2.3/Dockerfile).

Once your Dockerfile is there, you can run your app with the following command:

> docker run -e RAILS_ENV=development -p 3000 -c ["rails", "s", "-b", "0.0.0.0"]

### Docker-compose

Let's say you want to use a MySQL database in prod. Good practice suggest that you also use the same version of MySQL as your local database. Therefore, everytime you provision your app locally, you will need to start mysql and seed the data as well. 

You can start a mysql instance with a docker command but it becomes painful to manage. 

Therefore, we use docker-compose to manage our local containers depedencies. Docker-compose is a simple tool that enables to configure your containters in a yml file. It gives you a few simple heroku-like commands for managing your app such as up, scale, build and builds depedencies for you. It saves your from writing the docker run commands yourself. 

Below the docker-compose file from our current example, a rails 5 app with MySQL.

docker-compose.yml

```
version: '2'
services:
  mysql:  
    image: mysql:5.5
    expose:
      - "3306"
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: testdb
      MYSQL_PASSWORD: test
      MYSQL_ROOT_PASSWORD: test
      MYSQL_USER: test
  web:
    build: .
    volumes:
      - .:/usr/src/app
    ports:
      - "3000:3000"
    links:
      - mysql
    environment:
      RAILS_ENV: development
    command: ["rails", "s", "-b", "0.0.0.0"]
```

Basically, we create a first mysql container with a sample db and we link to our web container. The web app is exposed on port 3000 and accessible from http://your-docker-ip:3000/. Pay attention to the links: -mysql that will link your app to your running mysql container (by pointing mysql to the correct destination in the /etc/hosts of the web container).

The full example is available [here](https://github.com/francoismisslin/rails-docker/tree/master/rails-docker-simple).

### Using a Phusion Passenger server

Your production setup will hopefully use a ruby webserver such as [Phusion Passenger](https://www.phusionpassenger.com/). I suppose your are going to host your production application yourself (on a public or private cloud instance such as Amazon EC2 or on your own bare-metal machine) and not use a third party like [Heroku](https://dashboard.heroku.com/).

#### Before, we used Chef

Before migrating to Docker in production, we used [Chef](https://www.chef.io/) to provision our production VM. The Chef recipes were running every 30min and were responsible for the following things:
- install ubunutu to the VM and common security and logging modules to all Revinate's apps
- install ruby
- install linux depedencies with apt-get
- install ruby dependencies with bundle
- install phusion passenger with nginx
- configure Nginx
- pull code from Github
And if any code change was found:
- precompile the assets
- migrate the DB
- (re)start the webserver and all associated worker jobs

#### With Docker

Instead of using Ruby base image, we will now use a Docker image provided by Passenger. The documentation can be found here: https://github.com/phusion/passenger-docker

The Dockerfile now looks the following:

```
# (1) Using a different base-image
FROM phusion/passenger-ruby23:0.9.19

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app/
ADD . /usr/src/app

RUN bundle install

# (2) Start Nginx / Passenger
RUN rm -f /etc/service/nginx/down

# Remove the default site
RUN rm /etc/nginx/sites-enabled/default

# (3) Line used to start Passenger
CMD ["/sbin/my_init"]

# Cleanup
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /var/tmp/* /tmp/* /usr/src/app/tmp
```

You notice two differences with the previous Dockerfile.
(1) we now use a different base-image, provided by Phusion. I provided an explicit version (0.9.19), so that we do not pull the latest
(2) by default, Passenger advises you to run Nginx as well for serving your static assets (JS, images, CSS, html)
(3) The cmd used to start the server is different (and provided by the documentation). We will explore why later.

You can also change the command option in the docker-compose file to

```
command: ["/sbin/my_init"]
```



For production, you will use a different Nginx configuration.

You can override /etc/nginx/nginx.conf and /etc/nginx/sites-enabled/default.conf with you custom file. For instance, I have added a docker/myapp.conf file in my app and use the following line in the Dockerfile

```
ADD docker/myapp.conf /etc/nginx/sites-enabled/default.conf
```

docker/myapp.conf

```
server {
    listen 80;
    server_name "_";
    root /usr/src/app/public;

    passenger_enabled on;
    passenger_user root;
    passenger_min_instances 3;

    passenger_ruby /usr/bin/ruby2.3;
    
    location ~ ^/assets/ {
        root /usr/src/app/public;
        expires 1y;
        add_header Cache-Control public;
        add_header Last-Modified "";
        add_header ETag "";
        gzip_static on;
        break;
    }
}

```

