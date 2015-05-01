cloudify-nodecellar-docker-example
==================================

Sample application running inside docker containers.

Nodecellar is a nodejs frontend that uses a mongo database as its backend.

In this example each nodejs and mongo run inside of Docker containers.

## Requirements
 * Cloudify
 * Docker (on any virtual host that runs the Docker containers.)



## Step by step for Single host

Start out by installing Cloudify on your local machine.

You can install Cloudify on Windows, Mac, or Linux. Visit the instructions [here.](http://getcloudify.org/guide/3.2m8/installation-cli.html)

If you already have virtualenv and pip installed, you can install Cloudify in a virtual environment:

`virtualenv cloudify`

`source cloudify/bin/activate`

`pip install cloudify`


Now we need to pull the git repository:

`wget https://github.com/cloudify-cosmo/cloudify-nodecellar-docker-example/archive/3.2m8.zip`

Check out the singlehost blueprint: singlehost-blueprint.yaml.

First notice the files that we import:

    imports:
      - http://www.getcloudify.org/spec/cloudify/3.2m8/types.yaml
      - http://www.getcloudify.org/spec/docker-plugin/1.2m8/plugin.yaml


The nodes in the blueprint derive their base properties from these two files. They should match in the minor version. (3.2m8 and 1.m8, or 3.0 and 1.0, etc)


Notice the first node_template: host. This describes the host machine that the Docker containers will run on. In this case it will be your local computer. However, this could be adjusted to be a vagrant box or something else. 

If you decide to run the containers somewhere else, you will need to adjust the values in the inputs section or use an external inputs file.

    host:
      type: cloudify.nodes.Compute
      properties:
        install_agent: { get_input: install_agent }
        ip: { get_input: host_ip }
        cloudify_agent:
          user: { get_input: agent_user }
          key: { get_input: agent_private_key_path }


Now let's look at the Nodecellar Container.

In this example, we are going to have a container that runs the nodejs nodecellar application. The Docker plugin will pull the uric/nodecellar container from Docker Hub.

```

  nodecellar_container:
    type: cloudify.docker.Container
    properties:
      name: nodecellar
      image:
        repository: uric/nodecellar
    interfaces:
      cloudify.interfaces.lifecycle:
        create:
          implementation: docker.docker_plugin.tasks.create_container
          inputs:
            params:
              ports:
                - { get_input: web_port }
              stdin_open: true
              tty: true
              command: nodejs server.js
              environment:
                NODECELLAR_PORT: { get_input: web_port }
                MONGO_PORT: { get_input: mongo_port }
                MONGO_HOST: { get_property: [ mongod_container, name ] }
        start:
          implementation: docker.docker_plugin.tasks.start
          inputs:
            params:
              links:
                 mongod: { get_property: [ mongod_container, name ] }
              port_bindings: { get_input: nodecellar_container_port_bindings }
    relationships:
      - type: cloudify.relationships.contained_in
        target: host
      - type: cloudify.relationships.depends_on
        target: mongod_container

```        

The plugin configures the container to expose port 8080, with a pseudo TTY, and two environment variables NODECELLAR_PORT, and MONGO_PORT.

Notice that port 8080 is mapped to port 8080. You could change this to map port 80 to 8080 or any other combination.


Next is the MongoDB container. The plugin will pull the dockerfile/mongodb image from Dockerhub, and create a container that exposes ports 27017 and 28017.

```

  mongod_container:
    type: cloudify.docker.Container
    properties:
      name: mongod
      image:
        repository: dockerfile/mongodb
    interfaces:
      cloudify.interfaces.lifecycle:
        create:
          implementation: docker.docker_plugin.tasks.create_container
          inputs:
            params:
              ports:
                - { get_input: mongo_port }
                - { get_input: web_status_port }
              stdin_open: true
              tty: true
              command: mongod --rest --httpinterface --smallfiles
        start:
          implementation: docker.docker_plugin.tasks.start
          inputs:
            params:
              port_bindings: { get_input: mongo_container_port_bindings }
    relationships:
      - type: cloudify.relationships.contained_in
        target: host
```

The plugin then starts the container with a pseudo TTY and runs the command `mongod --rest --httpinterface --smallfiles`. Again the ports 27017 and 28017 are mapped to themselves, but you could change them to different mappings if you configured MongoDB to other ports.


## Execute the Operations

Now, let's get Cloudify setup so you can run the plugin.

Start by installing the plugins:

`cfy local init --install-plugins -p singlehost-blueprint.yaml`

Now you are ready to run the blueprint:

`cfy local execute -w install`

This might take a while to execute. It needs to install download the Docker images and then it will create the containers.

When it is finished, you can open a browser to 127.0.0.1:8080 and you will see the Nodecellar application.



# Running the example inside of a manager

## Vagrant

First download the Cloudify 3.1 Vagrantfile:

`wget http://gigaspaces-repository-eu.s3.amazonaws.com/org/cloudify3/3.2.0/m8-RELEASE/Vagrantfile`

When the download is finished, you can "up" the box:

`vagrant up`

Now, you can ssh into the box:

`vagrant ssh`

You need to install Docker, because the plugin's Docker installation script only works with Ubuntu Precise:

`curl -sSL https://get.docker.com/ubuntu/ | sudo sh`

Add the agent user to the docker group, as the plugin requires that the agent user is able to manager docker without sudo.

`sudo gpasswd -a ${USER} docker`

Now logout and log back in:

`exit`

`vagrant ssh`

Change into the blueprints directory and download this example:

`cd blueprints`


`git clone https://github.com/cloudify-cosmo/cloudify-nodecellar-docker-example.git`

And checkout the version:

`git checkout tags/3.2m8`

There are some minor changes to use the single host example in vagrant. So in that case you will use the vagrant_inputs.json file.

`cfy blueprints upload -b nodecellar -p singlehost-blueprint.yaml`

Create a deployment:

`cfy deployments create -b nodecellar -i cfy-vagrant-inputs.json -d nodecellar`

And execute the install workflow:

`cfy executions start -w install`

After the deployment runs, you'll be able to go to the following URL and visit the Nodecellar application: http://11.0.0.7:8080/.



## Openstack

This is an example running the Openstack blueprint.

The Openstack is slightly different than what we have described:

* Creates a virtual machine in Openstack.
* Installs Docker on the virtual machine. (However, this is not supported. There is POC workaround in the blueprint. But in principle, users should bring a image in Openstack with Docker preinstalled and the agent user added to the Docker group.)
* Pulls the respective images and runs the containers on the virtual machine.
* Creates two security groups: Nodecellar, and Mongo, and attaches them to the manager.
* There is a cloudify.relationships.connected_to relationship between the Nodecellar and Mongo security groups, which might strike you as odd. This is a temporary work-around for an Openstack issue where simultaneously connecting security groups to a server might fail without an error. This work-around creates a dependency between these security groups to make sure that one is attached first, and then the other.

Conversely, the singlehost:

* Runs locally.
* Does not install Docker.
* Pulls the respective images and runs the containers locally.
* There are no additional resources like the security group.

When it comes to deploying from a manager, the process is essentially the same between blueprints:

Start by initializing the environment.

`cfy init`

Now tell Cloudify, the URL of your manager:

`cfy use -t your-manager-ip`

Upload the blueprint:

`cfy blueprints upload -p openstack.yaml -b nodecellar-docker`

Create a deployment:

`cfy deployments create -b nodecellar-docker -d deploy-blueprint`

Now you can execute it:

`cfy executions start -w install -d deploy-blueprint`

Again this may take a few moments to run, but when you are finished, you can direct a browser to the virtual machine created by the execution and you will see Nodecellar there.
