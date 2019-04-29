# docker-tutorial

# Docker Tutorial:
A Docker container is like a **virtual machine** (and on windows it is) but on linux it is not VM, it just uses **namespace changes** to act like a VM. This means it runs quickly on your cpu, but it doesn’t see your normal file system, it sees its own view, and therefore you have to either copy files to its filesystem or else link directories that you want it to see. Similarly, it doesn’t see the normal network ports, so again, you need to link the ones you want to expose.

**!!!Important!!!** you may need to ```sudo``` to run any docker commands i.e. ```sudo docker run xyz```. You can use```sudo su``` to avoid having to type ```sudo``` constantly, but this is not necessarily a good practice.

##### Quick basic terminology:
A **Dockerfile** is used to **build** a docker **image**, which, when the image is being run, creates a **container** (i.e. virtual machine)

##### Dockerfile
The dockerfile gives the step by step instructions for building the image. 

**Fun Tip:** you can name the dockerfile anything you want but by convention it is named ```Dockerfile```

We will start with the a simple common docker file.
```
FROM python:3.7  #uses python:3.7 base dockerfile from dockerhub (like github for docker) 
WORKDIR /usr/src/app #change directory (like cd in bash)
ADD requirements.txt /usr/src/app/ #copy ./requirements.txt to from the host to /usr/src/app/ in the container.
RUN pip install -r /usr/src/app/requirements.txt  #pip install
```
This dockerfile basically just created an environment for us to run our python script, with all of our required modules installed. But note that we haven't even added the python script. 

**Fun Tip:** Dockerhub has millions of useful Dockerfiles we can base from, which means that we can use cool new tools, like Ubers horovod or Pyro withouth having to worry about installing them. In fact, we can even set up a Spark cluster in minutes, something some people get paid 300k per year to do.

We can run ```docker build -f Dockerfile -t danielsimage .``` to build it and give it the name of ```danielsimage```. Note ```-f Dockerfile``` gives the name of the docker file to build, ```-t danielsimage``` gives the name to tag it with, so we can reference the image later, and the ```.``` is the build context to use: i.e use the directory of ```.``` for any file copy in the docker (like when we said ```ADD requirements.txt``` then it will look in ```.``` for a file called ```requirements.txt``` to ADD to the image.)

Next, we can run ```docker run danielsimage``` which will start a container using the image we just built. 

We could cover docker commands, but in reality I rarely use them, so we will leave that as an appendix, since 99% of what we do uses a helpful tool called ```docker-compose```, which we cover next.

One last thing: here is the one Docker command we use on a daily basis:
```docker ps```
This just lists all running containers on a machine (note the similarity to the bash command ```ps``` which lists all running processes.)

##### Docker compose
Docker compose is great. 
It lets us start a group of networked containers simultaneously. 
Even better, it lets us specifiy all the docker options to use when starting a container, so we don't have to type out a complicated docker command each time we need to start our container.

To use ```docker-compose``` we just create a ```docker-compose.yml``` file (technically could be any name, but this is standard name), to specify all the docker options to use. Now we just use **3 key commands**:
1) ```docker-compose up --build -d``` 
which builds the images of any listed dockerfiles in the ```docker-compose.yaml```, and starts headless containers. The ```--build``` option says build image, and the ```-d``` says don't run the container in the terminal, run it in the background- this is super important in practice because without the ```-d``` the container would be running in our terminal and we couldn't do anything else until we hit ```ctrl-c```, but this would then kill the container.

2) ```docker-compose logs -f``` 
this shows the logs of the container, so we can see any logs or errors. the ```-f``` means follow, i.e. show new log messages in real time, and is optional, but I use it most of the time.

3) ```docker-compose down```
this stops all the containers referenced by our compose file.

**Rare 'gotcha' and solution:** ```docker-compose down``` takes down all the the containers that have the same name, which is the contcatentation of the directory and the tag name. What this means is that if I clone a repo into my home directory on a server, and do ```docker-compose up``` and then you clone the same repo into your directory and run ```docker-compose up``` you start new containers with the same name (since both have the same directory and same tag names). Now, if you or I do ```docker-compose down``` then **BOTH** of our containers go down. This rarely is an issue, but if it is the solution is easy- one of use just needs to change the name of the directory where we have the repo- i.e. if the repo is called ```dans_repo``` then when you clone it by default the directory ```./dans_repo``` is created and the repo is cloned into it, and all we need to do is have one of us change our directory to be called, for example, ```dans_repo2```.

#### Dockercompose files:
So docker compose commands are super easy. Literally there are 3 of them we use. They are easy because we made ```docker-compose.yml```.
Let's look at a simple but real example of a compose file:
```
version: '3'  # declare the version of dockercompose - 3 is the most recent.
services: #boilerplate- we have to say "services" before list all the containers we want to run
  web: #this is the name we will assign to this container
    restart: always #if the container stops for any reason just restart
    build: . #build the dockerfile that is in the directory "." (i.e. in the same directory)
    volumes: #volumes to mount
      - .:/usr/src/app #link the directory . on the host machine to /usr/src/app in the container
      - /data/repos/daniel/data:/usr/src/app/data #same
    env_file: env #add the variables from file ./env on the host to the bash environment of the container
    ports: # ports
      - 6543:8049 #map port 6543 on the host to 8049 in the container (our web app runs at port 8049, so this exposes it to the outside world at 6543)
    command: python /usr/src/app/app.py #Command to run when starting the container (in this case, just run our app)
```
There we have it, easy as pie. 
The first time we run this, we use ```docker-compose up -d --build``` to build the image, and then all subsequent times we can just do ```docker-compose up -d```, unless we have changed the dockerfile or the requirements file, in which case we will want to rebuild the image to incorporate these changes, and so we would use ```docker-compose up -d --build``` again.

### Advanced docker
To translate the above docker compose file into the docker command it is, it would be
```docker build -f Dockerfile -t exampleimage . && docker run --restart always -v .:/usr/src/app --env-file env -p 6543:8049 exampleimage /bin/sh -c "python /usr/src/app/app.py"```
So that is painful. You can construct the command just by looking at the docker documentation, but it is a real pain to type that, and even if you save this command to a file and then run that as a ```sh``` script it is still much harder to read then the compose file.

### Advanced compose
Networked containers: we could do this in pure docker, but at the point that you are networking containers then it is a huge hassle, and much nicer to do with ```docker-compose```. 
We will just give an example:

```
version: '3'
services:
  web:
    restart: always
    build: .
    volumes:
      - .:/usr/src/app
    env_file: env
    ports:
      - 6543:8049
    command: python /usr/src/app/app.py
  redis:
    restart: always
    expose:
      - 6379
    ports:
      - 6544:6379
    image: redis:latest
  postgres:
    restart: always
    expose:
     - 5432
    ports:
     - 6545:5432
    image: postgres:latest
    volumes:
      - postgres-data:/var/lib/postgresql/data
volumes:
    postgres-data:
```
This creates redis and postgres instances in addition to the web instance (redis is a key-value store service, and postgres is a SQL database). Redis is a super fast cache- given a function call with certain arguments that takes a long time to run we can store the result in redis, using those arguments as the key and the result as the value, and then in future if we can look up the result by those arguments. We can discuss this seperately, but suffice to say we use redis to speed up our web app, and postgres to store usage data and other statistics.
So, what is happening here?
First, because it is in one compose file a simple ```docker-compose up -d``` starts all the containers. 
Second, the containers are networked together. What this means is that processes running in the containers can communicate with each other without being exposed to the outside world. For example, the postgres database is linked to port 6545 by 
```
ports:
     - 6545:5432
``` 
so we can easily connect to it from any machine to get usage statistics. But by that same token anyone can connect to it and drop all the tables which would be bad. Since we are running this inside the CHR network the only people that could do this would be other CHR users, so we are not worried about that in this case, but for various preformance or security reasons that you can imagine we might want to network our containers without forwarding ports on all of them.
The bigger benefit of the networking are 1) we can use they container name as the hostname and the internal port number to communicate with the other containers. Consider
```    
REDIS_URL='redis://daniel123examplehost:6544'
```
vs
```
REDIS_URL='redis://redis:6379'
```
In the first case if we change the machine we run this on we would have to change the hostname from ```daniel123examplehost``` to the new host. Also, if we ever need to change the external port of the servie from 6544 to something else then we need to change the connection string. In the second case we will never have to change anything.

Second, transfering data between the 2 containers on the same host should be much faster than transfering data over the network, since the connection between two containers on the same host is virtual and limited only by software, whereas connection over the network is limited byt the physical network interface, which is much slower.

**Important point** The ```expose``` lines are what expose the ports to the container network, so don't forget them.

### Even more advanced stuff:
1) Link credfiles for easy passwords:
```    
env_file:
      - env
      - /passwords/supersecretpasswords.txt
```
This puts the ```supersecretpasswords.txt``` password file into the environment, so we don't have to include it in our code or github or do anything. If we have lots of machines we can use chef or puppet to put this file on all machines at that dir for this purpose.

2) If you change or delete the docker-compose file that you used to start a group of containers then ```docker-compose down``` won't take them down anymore. You will need to do ```docker ps```, which will return
```
7a4eaa19253b        XXXXXXXX  "gunicorn wsgi:app -…"   20 hours ago        Up 20 hours         0.0.0.0:8000->8000/tcp rasachat_rasa-app_1
1a292c34b762        XXXXXXXX    "python -m rasa_core…"   20 hours ago        Up 20 hours         0.0.0.0:5010->5010/tcp   rasachat_rasa-core-model_1
88577cd52c35        XXXXXXXX  "python -m rasa_nlu.…"   20 hours ago        Up 20 hours         0.0.0.0:5000->5000/tcp   rasachat_rasa-nlu-model_1
```
You can see that these containers are all part of an app called ```rasachat```, and were presumably started with a compose file. If we changed it or deleted it, then the alphanumeric string at the beginning of each line is the actual container id, which we will use to take them down (the other info is just the **image** name, but not the actual **container** id).
The commands then would be
```
docker stop 7a4eaa19253b && docker rm 7a4eaa19253b
docker stop 1a292c34b762  && docker rm 1a292c34b762 
docker stop 88577cd52c35   && docker rm 88577cd52c35  
``` 
Note that we are stopping and removing the containers. Stopping them but not removing them would allow us to restart them, but they would still be taking up memory, so it is much cleaner to remember to remove them.

3) You may see a lot of containers on a machine and wonder if there is enough ram and cpu left for your app. I would recommend first using ```htop``` to just see the total memory and cpu usage of the machine. We have huge machines and in most cases it is fine. If it isn't though, you can proceed to run ```docker stats $(docker ps --format={{.Names}})``` which will give the memory and cpu usage of all containers by name, rather than id. This is usually the most useful. In the rare case that there are multiple containers with the same name, and you need the specific id of the bad boy, then a simple ```docker stats``` will suffice.

4) Exploring a running container: 
```docker exec -it container-id bash```
Sometimes, to speed development, you may want to start a container, then enter it and do things (either to see what is happening, or to run commands that you want to save and later use in a Dockerfile- i.e. you may not know what you need to install to get something to run, so you may start a simple container, experiment, write down what you had to do to make it work, and then add those commands to the dockerfile for the future.) This is nice because now when we exit bash the container won't die. 
```docker attach container-id``` is similar, but I never use it because if you do ```ctrl-c``` to get out of it you kill the container.

5) If you are curious docker stores files that are not linked under ```/var/lib/docker/containers/[container-id]```
So when we talk about how docker is not VM, but rather uses namespaces to hide, this is what we mean. All of the files of a container are ```/var/lib/docker/containers/[container-id]```, but inside of the container it thinks that ```/var/lib/docker/containers/[container-id]``` is just ```/``` But ***please please please never play with the files here.*** It is very complex and anything you could ever want to do has an actual correct command to do it (the easiest way is just to link volumes, but if you can't do that there is ```docker copy``` for example, to copy files from container to host or vice versa.)

6) Applying changes to a single container:
There are two possibilities. In the first case, suppose we have changed something that will change the image. In this case, no need for ```docker-compose down```. Just ```docker-compose up -d --build``` will rebuild only the changed images and restart those containers. Alternatively, you may have changed something that is volume mounted in. In this case, you don't need to rebuild (and in fact if you try to rebuild it won't because there is no change). Instead, just do ```docker-compose restart [name]``` to restart that container.

7) Checking Docker Filesystem useage:
```docker system df```

8) Remove images without at least one container associated, to free up space: ```docker image prune --all```

