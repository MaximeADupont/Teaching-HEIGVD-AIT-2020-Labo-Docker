

## Task 0



1. #### **[M1]** Do you think we can use the current solution for a production environment? What are the main problems when deploying it in a production environment?

   > Our current solution is not suitable for real a production environnement: 
   >
   > We have seen, with the lab wih Galaxus and black friday, that the request on the servers are absolutely not constant. Sometime there are huge increase in connection and we need more servers to handle this load. But it is in some rare occasions, so if we build our servers for our maximum demand, most of the year they will be useless. It is very inefficient and could be very cost expensive for nothing.
   >
   > Thus, we need a flexible solution for our production environnement that can scale ans add server when in need and then scale down when the time allows it.
   >
   > But our current solution gives us a constant number of 2 servers at any time, which is not efficient and don't allow us to create new server when there are a lot of incomming request

2. #### **[M2]** Describe what you need to do to add new `webapp` container to the infrastructure. Give the exact steps of what you have to do without modifiying the way the things are done. Hint: You probably have to modify some configuration and script files in a Docker image.

   > To add a new `webapp` container to the infrastructure, we need to create a new environnement variable (WEBAPP_3 and give its name and IP) for it in the `.env` file:
   >
   > We follow the same notation as for the 2 previous webapps
   >
   > ```
   > ...
   > WEBAPP_1_NAME=s1
   > WEBAPP_2_NAME=s2
   > WEBAPP_3_NAME=s3
   > 
   > WEBAPP_1_IP=192.168.42.11
   > WEBAPP_2_IP=192.168.42.22
   > WEBAPP_3_IP=192.168.42.33
   > ...
   > ```
   >
   >  Then we go to the **docker compose file** (`docker-compose.yml`) and we add some lines to configure the `webapp3` (like for webapp 1 and 2)
   >
   > ```
   > webapp3:
   >        container_name: ${WEBAPP_3_NAME}
   >        build:
   >          context: ./webapp
   >          dockerfile: Dockerfile
   >        networks:
   >          heig:
   >            ipv4_address: ${WEBAPP_3_IP}
   >        ports:
   >          - "4002:3000"
   >        environment:
   >             - TAG=${WEBAPP_3_NAME}
   >             - SERVER_IP=${WEBAPP_3_IP}
   > ```
   >
   > In the same file, we also modify the `haproxy` to add this new environnement variable (so we add the last line here):
   >
   > ```
   > haproxy:
   >        container_name: ha
   >        build:
   >          context: ./ha
   >          dockerfile: Dockerfile
   >        ports:
   >          - 80:80
   >          - 1936:1936
   >          - 9999:9999
   >        expose:
   >          - 80
   >          - 1936
   >          - 9999
   >        networks:
   >          heig:
   >            ipv4_address: ${HA_PROXY_IP}
   >        environment:
   >             - WEBAPP_1_IP=${WEBAPP_1_IP}
   >             - WEBAPP_2_IP=${WEBAPP_2_IP}
   >             - WEBAPP_3_IP=${WEBAPP_3_IP}
   > ```
   >
   >  And then we modify the configuration file of haproxy `haproxy.cfg` to add the new environnement variable of the new server (in the `backend nodes` part):
   >
   > ```
   > [...]
   > # Define the list of nodes to be in the balancing mechanism
   > # http://cbonte.github.io/haproxy-dconv/2.2/configuration.html#4-server
   > server s1 ${WEBAPP_1_IP}:3000 check
   > server s2 ${WEBAPP_2_IP}:3000 check
   > server s3 ${WEBAPP_3_IP}:3000 check
   > ```
   >
   > 

3. #### **[M3]** Based on your previous answers, you have detected some issues in the current solution. Now propose a better approach at a high level.

   > In the current solution, it takes some work to add a new server, we have to modify small things in several configuration files. If we do it "by hand", like this, we won't be very reactive and it will take some time to scale up when the server are on a high demand period. We need to dynamically add/remove web servers.
   >
   > So a better high level approach would be to create some scripts to automaticlly create a new webapp server container, and update automaticaly the environnement variable and config files of the docker compose and haproxy. If we have a script (or a service) that can do the steps above we can just monitor the demand on the server and execute it automatcaly when the demand is starting to be too high (threshold), thus it will immediatly create a new webapp when needed.

4. #### **[M4]** You probably noticed that the list of web application nodes is hardcoded in the load balancer configuration. How can we manage the web app nodes in a more dynamic fashion?

   > To be more dynamic, we can think of an application , like a proxy, that will scan the webapps containers to see if one is added or remove one. Then it will tell the load balancer that some changes occurs to make him update his haproxy configuration dynamically and restart to handle the new situation.
   >
   > Another way of doing it would be to make the webapp container introduce themself to the loadbalancer when created, and when they leave they notify him to. This way he can update dynamicaly. It can be easier to implement but if the webapp crash, maybe the loadbalancer won't notice and won't be notified.

5. #### **[M5]** In the physical or virtual machines of a typical infrastructure we tend to have not only one main process (like the web server or the load balancer) running, but a few additional processes on the side to perform management tasks.

   For example to monitor the distributed system as a whole it is common to collect in one centralized place all the logs produced by the different machines. Therefore we need a process running on each machine that will forward the logs to the central place. (We could also imagine a central tool that reaches out to each machine to gather the logs. That's a push vs. pull problem.) It is quite common to see a push mechanism used for this kind of task.

   #### Do you think our current solution is able to run additional management processes beside the main web server / load balancer process in a container? If no, what is missing / required to reach the goal? If yes, how to proceed to run for example a log forwarding process?

   > Unfortunatly the answer is no, our current solution is not able to run an additionnal management process. This is due to our Docker policy that state that one container is a process, thus only one process (our webapp) can be running within our container
   >
   > To be able to fix it and reach the goal, we need to add a **process supervisor** to the container. This thing will run like a "main" in our container and will handle the multiples process here (haproxy, webapp, monitoring agent, etc).

6. #### **[M6]** In our current solution, although the load balancer configuration is changing dynamically, it doesn't follow dynamically the configuration of our distributed system when web servers are added or removed. If we take a closer look at the `run.sh` script, we see two calls to `sed` which will replace two lines in the `haproxy.cfg` configuration file just before we start `haproxy`. You clearly see that the configuration file has two lines and the script will replace these two lines.

   #### What happens if we add more web server nodes? Do you think it is really dynamic? It's far away from being a dynamic configuration. Can you propose a solution to solve this?

   > Currently, if we add more web server nodes, nothing happens with them: the haproxy doesn't add it to its configuration file. 
   >
   > The load balancer configuration may be changed dynamically to the existing web server, but it won't be able to update dynamically for an add/remove of a web server.
   >
   > To solve this, we need to do something as discussed in `M3` and `M4` to add/remove the server to the haproxy configuration file dynamically: having a webapp server template and a service/script that change the haproxy container configuration and restart it when an add/remove is detected in the server cluster.

