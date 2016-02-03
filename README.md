In this post, we will be building a Docker image and running it on AWS. However, 
if you are looking to do a really awesome tutorial on your local machine using Virtual Box, read [Awesome GET STARTED with Docker and Node.](https://airpair.com/node.js/posts/getting-started-with-docker-for-the-nodejs-dev "AirPair Posts")

## What to Expect and Why Am I Doing This 
This tutorial is for all those that want to host their app on AWS via Docker. 
We will be using a sample app (Angular, Mongo, Node) that enables rating an instructor by pressing LIKE (Heart icon). 
Using the app as our foundation, we will add DOCKER to the app and make it go live on AWS. 

## Background - Raw Steps 
    
### Live Demo – Before Beginning, Check It Out to Get Inspired
    http://ec2-54-86-155-250.compute-1.amazonaws.com
    
### Before Beginning

I'll be referring to commands executed in your terminal with:

```
    $osxterm: command
```

Commands inside a container with:
```
    $ root: command
```
				
Output after running a command (in container or terminal) with:
```
    %: output 
```

Output after running a command inside Mongo Shell with:
```
    > output 
```

### Local Host Test 
If you wish to run this app on your local machine, feel free to follow these steps:

You can fork this repo too :)
```
    $osxterm: git clone https://github.com/georgebatalinski/rate-instructor-demo-docker-aws.git
    $osxterm: npm install
    $osxterm: npm start
```

Your web browser: http://localhost:8080
    
    
    
## AWS Signup and the First Instance 
If you cloned the repo, you got the app now and, after npm install, you should be up and running on port: 8080. 
If not, make sure you complete -> ### Local Host Test Now 
Let’s move from our local environment to the cloud. We all love flying in the clouds. What better way then via Docker?


![Image showing our container in AWS](http://s21.postimg.org/3r96w9ocn/Screen_Shot_2016_02_02_at_8_22_25_AM.png)
Source: [Evan Brown (AWS)](https://youtu.be/OzLXj2W2Rss?list=PLVjIiWGLQYcp_BIIPAB1UuNWMwaL5mXtx)


### Signing Up for AWS and Configuration 

I. [Create an AWS account FREE tier.](https://aws.amazon.com/console/ "Amazon Console")
II. Note down your access ID, secret key
III. Go create a VPC /or use the default (in correct Availability zone)
![AWS VPCIP](https://www.evernote.com/l/AF3K_mYH59hD7IvA42598ZwyOYdSvfIOz9w)
IIII. Check if there is an existing security group, if not create one. 
NOTE: Security groups - allow you to setup rules around what/who can access your VPC, and control access from individual instances.
![Security Group](https://www.evernote.com/l/AF0buUB-AENF0aiRrOa0l7Uwxxemvl_9wXI)

NOTE: To get the VPC ID, click on the VPC icon in the Amazon Console Dashboard. 
NOTE #2: For a simple setup, use the "Start VPC Wizard." If you are not familiar why we need to setup a VPC: [Amazon Virtual Private Cloud (VPC)](https://aws.amazon.com/vpc/) 
NOTE: [AWS guide reference](https://docs.docker.com/machine/drivers/aws/)


### Creating a VM (or Docker Machine) on AWS 
The fun can now begin. We have the three required pieces of data for our AWS signup (access ID, secret key, and VPC ID). The cool part is we can now just use it. If you are going to be creating multiple Docker machines on AWS with the same credentials, you may want to export your credentials to your environment. 

```
export AWS_ACCESS_KEY_ID=<access-id>
export AWS_SECRET_ACCESS_KEY=<secret-key>
export AWS_VPC_ID=<VPC id> 
```

Next, we will create a VM. 'docker-machine create' will provision a 
host and create all the required keys for us to access the instance over SSH. 
Once the keys are issued, Docker daemon will be configured to accept remote connections.

If you would like to see where the keys are stored, go to the following directory. On my machine, it is here: 

```
 $osxterm: ~/.docker/machine/machines/<vm-name> 
```

```
$osxterm: docker-machine -D create \
    --driver amazonec2 \
    --amazonec2-access-key $AWS_ACCESS_KEY_ID \
    --amazonec2-secret-key $AWS_SECRET_ACCESS_KEY \
    --amazonec2-vpc-id $AWS_VPC_ID \
    --amazonec2-zone a \
    <your-vm-name>
```


Common Errors 

Error creating machine: Error with pre-create check: unable to find a subnet in the zone: us-east-1a
Check the zone - you are passing to the console - you may not have a subnet created 
```
--amazonec2-zone b \
```

Error creating machine: Error in driver during machine creation: Error launching instance: Problem with AWS API call: Non-200 API response: code=400 message=Security group sg-7d034304 and subnet subnet-6f778b45 belong to different networks.

```
Check your subnet and security group, make sure they are tied together.
```


The biggies: 

--Driver amazonec2. Some cloud providers have an official driver. In our case, we are using AWS, but you could also go with Microsoft Azure, Digital Ocean, etc. [(List of drivers)](https://docs.docker.com/machine/drivers)
     A driver is a set of defaults/requirements [If you can and are curious, read GO.](https://github.com/docker/machine/blob/master/drivers/amazonec2/amazonec2.go "AirPair Posts")
 
--amazonec2-zone. 
Note: defaultRegion = "us-east-1" If you wanted to specify a different region, use --amazonec2-region.

<your-vm-name> It is the name of the Docker machine. Name it what you like (e.g., {you-choose-name}). 



Using the following subshell method, our remote host behaves much like the local host:
```
    $osxterm: eval "$(docker-machine env <your-vm-name>)"
```

ERROR (if you are getting any of this)
```
Jerzys-MacBook-Air:rate-instructor-demo-docker-aws jerzybatalinski$ dm start putanga
Host does not exist: "putanga"
```

Make sure the machine exists 
```
ls -al ~/.docker/machine/machines
```




If you still are in awe (and cannot believe) that you created an instance on AWS, run the following simple tests to be triple sure: 
```
    $osxterm: docker-machine active 
    %: <your-vm-name>
    
    $osxterm: docker run busybox echo hello world
    %: hello world
```



## Adding Docker to the Mix 

### Dockerfile for Your Project 
A Dockerfile consists of instructions that will create an image for you to use with Docker. 
Its purpose is to automate the copying and installation of the system requirements for the environment you want. 
 
 
Creating a Docker file is easy; Using your favorite code editor, create a new file named 'Dockerfile', or via Terminal: 

```
   $osxterm: vim Dockerfile 
```

Here is a sample environment that we will be working with for our app. Feel free to copy this into your local file (For the most part, all we are doing here is creating an image and moving files from our local system to the remote.).

```
FROM centos:centos6

# Enable Extra Packages for Enterprise Linux (EPEL) for CentOS
RUN yum install -y epel-release
# Install Node.js and npm
RUN yum install -y nodejs npm

# Install app dependencies
COPY ./package.json /src/package.json
RUN cd /src; npm install --production

# Bundle app source
COPY . /src

EXPOSE  8080
CMD ["node", "/src/server.js"]

```

The biggies: 

FROM - centos. Any Dockerfile must start with an image. You can find images like MongoDB, Redis, etc. right here: [Images](https://hub.docker.com/)

EXPOSE 8080 - Internal port for the CONTAINER. This is what server.js is listening to. See app.listen(8000);

CMD ["node", "/src/server.js"] - You can only have 1 CMD per Dockerfile; if you have multiple CMDs, the last one is going to get executed. If you need to run any additional commands, before starting the server, use 'RUN' (see where we 'npm install').


### Docker-compose File 
We are setting up an application that will require multi-container env (a container for App, container for DB, container for Data storage). 
In order to create a multi-container environment, we have 'docker-compose up', which will use the above Dockerfile to build our containers from some fancy images and Dockerfiles.


Create and copy the contents of the following file: 

docker-compose.yml
```
app:
  build: .
  ports:
    - "80:8080"
  links:
    - db
    
db:
  image: mongo
  ports:
    - "27017:27017"
  volumes_from:
    - dbdata

dbdata:
  image: busybox
  volumes:
    - /usr/lib/mongodb
```

The biggies: 
Build context that is sent to the Docker daemon. Make sure you are not in your root directory when running 'docker-compose up' because the Docker build sends the whole directory (if you are in root, your whole filesystem) to the DEMON, which executes your build. Github issue.
 [Github issue](https://github.com/docker/docker/issues/2342 "AirPair Posts")

/usr/lib/mongodb. Docker volumes, by default, mount in read-write mode, but you can also set it to be mounted read-only (e.g., /usr/lib/mongodb:ro)

Why are volumes important? 
MongoDB stores its data in the data directory specified by dbpath. It uses a database format so it's not actual documents Therefore, we need to save it somewhere. The other added benefit is data volumes are designed to persist data (good for storing our DB data), independent of the container’s life cycle. Docker, therefore, never automatically deletes volumes when you remove a container, nor will it “garbage collect” volumes that are no longer referenced by a container. [Volumes](https://docs.docker.com/engine/userguide/dockervolumes/#adding-a-data-volume "AirPair Posts")



### Last Part: We are Done With Docker—Let’s Build on AWS 
All the setup is done. We now get the payoff:

```
    $osxterm: docker-compose up
```

Check Public DNS (from AWS) - you should be up and running.    
http://ec2-54-86-155-250.compute-1.amazonaws.com/


## Fill Your DB With Data
Let’s add some data to our MongoDB, so we can actually see something on the page, while our app is running. 

### Get Into Our OS on Docker
Let’s get into our Docker container and manually add data. You could also import it, if you have previously backed it up.
 
```
    $osxterm: docker exec -it <CONTAINERID running CENTOS image> /bin/bash
```

### Install Mongo Shell to Connect to Our DB
The shell will enable us to connect directly to a linked container: 
```
    $ root: vi /etc/yum.repos.d/mongodb-org-3.2.repo
    //Paste into above file
    [mongodb-org-3.2]
    name=MongoDB Repository
    baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.2/x86_64/
    gpgcheck=0
    enabled=1

    $ root: yum install -y mongodb-org 
```

Once Mongo Shell is installed, we need to find out the IP of our MongoDB container. Our MongoDB container is not exposed to the public DNS, so we cannot access it directly. We will have to access it via the main container that is exposed. Read this section to understand how our containers are talking to each other: [Connect with the linking system](https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/)

### Check the IP Address of the Mongod Process 
```
    $ root: cat /etc/hosts
    
    %: 172.17.0.2	db 729371a78b62 rateinstructordemodockeraws_db_1
       172.17.0.2	db_1 729371a78b62 rateinstructordemodockeraws_db_1
       172.17.0.2	rateinstructordemodockeraws_db_1 729371a78b62
```

Eureka! We now found the IP of the MongoDB container. Let’s go ahead and, using Mongo Shell, connect to that container.

### Connect to Mongod Using the Shell  
```
    $ root: mongo 172.17.0.2:27017
```

### You are Now Inside Mongod—Add Your Data

In our Single Page App—Remember rateYourInstructor.io (ofcourse io)—we will need the following data in order for the Angular template to show anything in the following places:

DB name: 'school' 
Collection Name: 'instructors'
Data fileds: { name : string, hoursflown: number, numberOfLikes: number } 


With all the automation we have around us, I wanted you to feel a sense of accomplishment by doing it the manual way, minus the one loop that does most of the work. :) 

```
> show dbs
    local  0.000GB
> use school
    switched to db school
> db.createCollection('instructors')
    { "ok" : 1 }
> for (var i = 1; i <= 25; i++) {
...    db.instructors.insert( { name : 'Frank ' + i, hoursflown: i * 100, numberOfLikes: 0 } )
... }

```


## Back Up Your Data
If you are a serious admin, you may want to back up the created database. So you can get a tar manually, in the following way: 
```
    $osxterm: docker exec -it 3fac66047f37 /bin/bash
    $ root: tar -cvpzf /dbbackup.tar.gz /data/db 
    $ root: exit 
    $osxterm: docker cp 3fac66047f37:/dbbackup.tar.gz /Users/jerzybatalinski/Desktop/ 
```



## Recommended Reading 
Read the article [Compose](https://docs.docker.com/compose/compose-file "AirPair Posts")

## Common Errors 

### Unable to Connect to MongoDB via Robomongo 
```
docker inspect -f '{{.NetworkSettings}}' db
map[27017/tcp:[{0.0.0.0 27017}]]
you may have no explicity exposed the port to the machine 
docker run -p 27017:27017 --name db -v /data/db -d mongo
```
