# Lab 1: Manage Docker Networking

OBJECTIVE 1
Defining and Testing a Docker Bridge Network
You'll identify your current Docker network environment, create a new network, and test connectivity between networks.

Once available, click the Open environment button. It will be available in about one to three minutes.

At the Connections page, open Ubuntu Desktop One in a new browser tab to access a Desktop.

Leave the Connections page open as you'll need to later in the lab open the other connection.

Click the Terminal Emulator icon at the bottom of the Desktop to open a new terminal.

Enter status (at the terminal), and wait until it returns SYSTEM COMPLETE.

You'll then be all set to go. It will take about two to three minutes to become ready.


When everything is ready, list your default Docker networks by running:
```sh
sudo docker network ls
```

You'll see bridge, host, and none.

You can now create a new network:
```sh
sudo docker network create --driver bridge net1
```

This creates a net1 network. You could run:
```sh
sudo docker network create --help
```
if you wanted to get a quick look at some help documentation, showing you what command arguments are available.

Confirm the network now exists and view its configuration by running:
```sh
sudo docker network inspect net1
```

You can identify details like the IP address range, its Subnet of 172.18.0.0/16.

Now you can test out the way that new network will behave by launching three simple Alpine Linux containers: two of them will live in the net1 network, and a third in the default bridge network. Launch the first container, named con1, using the following command:
```sh
sudo docker run -dit --name con1 --network net1 alpine
```
Now launch the second container. This time you could just hit the up arrow to get the previous command, and change con1 to con2, or you can type in the complete command:
```sh
sudo docker run -dit --name con2 --network net1 alpine
```

The third container, which you will name con3, and will use the bridge network.

To make that happen, run:
```sh
sudo docker run -dit --name con3 --network bridge alpine
```
(Make sure you get the network name right in addition to the container name.)

You can confirm that everything worked by viewing how those containers exist on their networks. Run:
```sh
sudo docker network inspect net1
```
This will give you profiles of the first two containers, each identified by their DNS names con1 and con2. Their IP addresses will probably be 172.18.0.2 and 172.18.0.3 respectively.

Run the same inspect command against the bridge network to see the con3 container profile:
```sh
sudo docker network inspect bridge
```

To test connectivity, use the ping command from con1 to con2:
```sh
sudo docker exec -it con1 ping con2
```
Since they're both in the same network, the pings should work. After a few seconds, hit Control+C and confirm that all packets were received.

But if you run the same command against the same con2, but this time from con3–which is on a different network–it will fail. Run:


```sh
sudo docker exec -it con3 ping con2
```

You'll get a ping: bad address 'con2' response.

OBJECTIVE 2
Building and Deploying a Docker Swarm Across an Overlay Network
Learn to deploy Docker Swarms into appropriately segmented networks.


The machine you're on, Ubuntu Desktop One, will become your Docker Swarm manager; create the swarm and set a static IP address and port:
```sh
sudo docker swarm init --advertise-addr 172.31.24.30:2377
```

Copy the long docker swarm join command from the output of the init command you just ran.

You can copy from the Terminal by pressing Control+Shift+C.

Go back to the Connections page, and open up Ubuntu Desktop Two.

Open up a new Terminal Emulator, type in sudo (note the space), and then right after paste in the docker swarm join command you copied (Control+Shift+V).

Switch back to the manager connection (Ubuntu Desktop One), run:
```sh
sudo docker info
```
and verify the line Swarm: action is displayed (indicating the swarm is active).

List all the nodes currently available by running:
```sh
sudo docker node ls
```
There will be two nodes, the manager, ip-172-31-24-30, and the worker node, ip-172-31-24-31.

You will now create a new overlay network for your swarm services. Run the following command which will specify the overlay driver and name the network my_overlay_network:
```sh
sudo docker network create --driver overlay my_overlay_network
```

This next (long) command will launch a container with MySQL to provide a backend service for your swarm. Besides creating the necessary authentication parameters, the command will also attach the service to your new network. Run:
```sh
sudo docker service create --name backend-service --network my_overlay_network --env MYSQL_ROOT_PASSWORD=root_password --env MYSQL_DATABASE=my_database mysql:8.0
```
This will take a minute or so to complete.

You can now do the same type of thing to create a frontend service:
```sh
sudo docker service create --name frontend-service --network my_overlay_network --publish 8080:80 nginx:latest
```
This creates a web server run by nginx.

Check to make sure the two new services have been created:
```sh
sudo docker service ls
```
Finally, inspect the my_overlay_network to confirm that's where your two new services are running:
```sh
sudo docker network inspect my_overlay_network
```
The output will end with the two node's IPs. You've got a Docker Swarm successfully running inside a custom overlay network. In the next challenge, you'll test connectivity to ensure it matches your exact needs.


OBJECTIVE 3
Testing a Docker Network Environment for Segmentation
Working with containers and services, you'll be able to better understand how and why your overlay network is segmented.

(Do the following while still on Ubuntu Desktop One.) Launch a new container–calling it con4–and attempt to attach it to the my_overlay_network overlay network:


```sh
sudo docker run -dit --name con4 --network my_overlay_network alpine
```

Note: This operation will fail with an Error because, by default, Docker overlay networks are not attachable by standalone containers unless explicitly created with the --attachable option. However, the services you launched in the previous challenge did successfully attach to the network since overlay networks are built to work for services. So this is all working exactly the way it should.

You will now test connectivity between your two services. First, list the container that is running your backend service:
```sh
sudo docker ps --filter name=backend
```
Copy the CONTAINER ID (Control+Shift+C), which you will use in the next command

Execute a shell within the backend service's container by running:
```sh
sudo docker exec -it <CONTAINER ID> sh
```
Make sure you use the correct CONTAINER ID (you can use Control+Shift+V to paste), the one you copied in the previous task.

From inside the service, connect to the frontend using curl and the frontend's DNS name:
```sh
curl frontend-service
```
This will result in html from the web server, where one of the final lines is Thank you for using nginx.

When you're done with that, type exit to back out to the host command line.

You can inspect the metadata of your services in details using docker service:
```sh
sudo docker service inspect backend-service
```
Towards the top of the output, you can see the base image that is used, mysql:8.0.

Do the same for the frontend-service:
```sh
sudo docker service inspect frontend-service
```
Its image is nginx:latest.

Finally, you can list the Docker Swarm tasks using the following commands (A task here is a running container managed by Swarm, and part of a service.)
```sh
sudo docker service ps frontend-service
sudo docker service ps backend-service
```

Note: Each container is on a different node, one on the manager, the other on the worker. By default, this happened due to the order of the service creation commands: the mysql service was created on the manager, the nginx service on the worker. It's beyond the scope of this lab, but in the real-world, you'd want to better control where the services end up, often having them only on worker nodes.

At this point, you've seen how to create, configure, manage, segment, isolate, and inspect Docker services and networks. Now applying those skills to your projects will be your task!

