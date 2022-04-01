# Custom CI/CD with GitHub and Docker

I have a lot of side projects. Most of them are never really finished, but nevertheless I like to publish them to show them to friends and family.
 In most cases these projects are not worthy of some professional platform where to host them, so I end up using some free service like Heroku or similar.
 Of course, the free tier for these kind of services usually has some strong limitations that can be annoying, so I&#39;m always looking for something better.

But what if I could host my projects myself?

What if I made this... my new side project?

Self-hosting my own apps would bring some nice advantages, right?

- Free hosting
- No limitations
- Full control over the hardware

But, of course, is not all peaches:

- Security
- Scalability
- Malicious attacks
- Hardware, energy and bandwidth costs

![](RackMultipart20220401-4-s383db_html_e1d96e71044f0c56.png)

So why do it?

For science of course!

In general, I like the idea of self-hosting my services, although I see the advantages of not having to care for the technicalities that come with handling the &quot;bare metal&quot;. But doing so, for some toy project without so much criticality, is a fantastic learning opportunity.

My usual go-to solution is to dockerize my service and run it in my RasperryPi.

This works very well in a big variety of situations, but having to always deploy my services manually, becomes boring very quickly.

Wouldn&#39;t it be better if I could have my code automatically deployed to my RaspberryPi without my intervention, like in any respectable CI/CD system? Of course it would! So, let&#39;s try to build one.

## The setup

- A GitHub repository
- A local clone of the repository
- A local Docker instance
- An instance of Docker, always available and accessible via SSH (As mentioned, mine is on my RasperryPi)
- A simple Node app to test the process

## Expected Result

After every push of my code to the master branch of my GitHub repository, a new version of my app will be deployed to my Docker instance, which runs in my Raspberry Pi

## Requirements

- Basic knowledge JavaScript and NodeJS for the app used as an example.
- Basic knowledge of Git and GitHub to be able to commit the code to GitHub.
- Basic knowledge of Bash to be able to understand the commands needed for the automations.
- Basic knowledge of YAML to be able to understand how to read the configuration of the GitHub pipeline.
- Basic knowledge of docker and the image/container system.

This article will focus on the main topic and provide links for some more complex topics that are referred in the chapters.

A complete example of the final project is provided at [https://github.com/Eux86/remote-docker-deploy-action-example](https://github.com/Eux86/remote-docker-deploy-action-example).

## Step 1: Demo App

_Note: from this point on I will assume that a new GitHub repository was already created and that it was already cloned on the local development environment_

Let&#39;s create a small NodeJS app that will help us testing the deploy process.

- Create project
  - npm init
- Create express example
  - npm i express
  - Create basic NodeJS code
  - Create start script
    - npm run start

Example code:

const express = require(&#39;express&#39;);

const app = express();

const port = 3000

app.get(&#39;/&#39;, (req, res) =\&gt; {
res.send(&#39;Hello World 1!&#39;)
 })

 app.listen(port, () =\&gt; {
console.log(`Example app listening at http://localhost:${port}`)
 })

After this we should be able to connect to http://localhost:3000 and see:

![Picture 3](RackMultipart20220401-4-s383db_html_55395bf333dc749a.gif)

## Step 2: Dockerize the app

Let&#39;s now create a basic Docker image that runs the app.

![Picture 2](RackMultipart20220401-4-s383db_html_cc58ca7460a7b2a3.gif)

Note that in my case I&#39;m using the _arm32v7_ version of the node:lts-alpine which is built to run on the RaspberryPI arm processor. A different base image should be used if a different architecture is used. A list of all the available targets is available at: [https://hub.docker.com/\_/node?tab=tags](https://hub.docker.com/_/node?tab=tags)

Let&#39;s build the image:

docker build . -t test-server

And run it:

docker run --name test-server -d -p 3000:3000 test-server

If everything goes as expected, now we will have a local docker container running our test app available navigating at [http://localhost:3000](http://localhost:3000/) with our browser.

Good, let&#39;s now make it more interesting.

## Step 3: Deploy on the remote Docker

First things first: if we don&#39;t have it already, we need to configure a SSH key-based authentication for our remote environment.

I&#39;m not going to go into details about this since it is thoroughly explained here: [https://www.ssh.com/academy/ssh/public-key-authentication#setting-up-public-key-authentication-for-ssh](https://www.ssh.com/academy/ssh/public-key-authentication#setting-up-public-key-authentication-for-ssh).

Long story short, this allows us to access a remote SSH environment without an authentication prompt and will allow us to automate this process into our ci/cd system.

Now that we have access to the remote system and we also tested that the docker container is working in our local environment, let&#39;s publish it to our remote docker instance. To do so we use the &quot;DOCKER\_HOST&quot; environment variable, which tells docker that every command will be redirected to the remote instance:

DOCKER\_HOST=ssh://user@remote.address docker run --name test-server -d -p 3000:3000 test-server

If everything goes as expected, we should be able to see our demo app when we access

[http://remote.address:3000](http://remote.address:3000/)

### Deploy commands

Great, we were able to manually deploy our container on our remote docker instance. Now we want to make this step as simple as possible, so that we will be able, later, to automate it.

Let&#39;s create a new _deploy_ command in our package.json file. This file will take care of all the needs for publishing the project to our remote docker instance. It should be able to:

- build a new updated image every time we change our code
- stop and remove an old container that was previously created
- run a new container with the new updated image

To do so, let&#39;s create a set of command that will deal with each of these tasks:

&quot;deploy:stop-container&quot;: &quot;docker stop test-server || true&quot;,

&quot;deploy:remove-container&quot;: &quot;docker container rm test-server || true&quot;,

&quot;deploy:start-container&quot;: &quot;docker run --name test-server -d -p 3000:3000 test-server&quot;,

&quot;deploy:docker-build&quot;: &quot;docker build . -t test-server&quot;,

&quot;deploy&quot;: &quot;npm run deploy:docker-build &amp;&amp; npm run deploy:stop-container &amp;&amp; npm run deploy:remove-container &amp;&amp; npm run deploy:start-container&quot;

&quot; **deploy&quot;:** only executes all the other commands in the right sequence.

&quot; **deploy:docker-build&quot;:** executed the build process and gives a name to the image (in this case test-server)

&quot; **deploy:start-container&quot;:** runs an image of our test-server image, names it (again using test-server, but could be anything) and configures it to expose the internal container&#39;s port 3000, to the external port 3000. More details on publishing ports in docker can be found [here](https://docs.docker.com/config/containers/container-networking/).

&quot; **deploy:remove-container&quot;:** removes a previously created container. The first time we run the deploy, no container would be there and the command would exit with an error code, breaking our process. To avoid this we add a &quot;|| true&quot; at the end, which bypasses the error.

&quot; **deploy:stop-container&quot;:** very similarly to the deploy:remove-container command, this one tries to stop a container and if no container is found it does not exit with an error code.

With this set of commands, we can easily deploy our project just running a single command:

DOCKER\_HOST=ssh://user@remote.address npm run deploy

Let&#39;s give it a try and, if everything is working on our machine, it&#39;s time to automate the same process on GitHub!

## Step 4: GitHub actions

Note: from this point on I will assume that a new GitHub repository was already created and that it was already cloned on the local development environment

Let&#39;s now put it all together in a GitHub Action configuration file which will be executed every time we publish something new on our repository.

This file must be placed in the folder:

project\_root/.github/workflows/anyname.yaml

name: Remote Deploy

on: [push]

jobs:
build:
runs-on: ubuntu-latest
steps:
 - uses: actions/checkout@v2
- name: Npm Install
run: npm i
- name: configure ssh connection
env:
SSH\_AUTH\_SOCK: /tmp/ssh\_agent.sock
run: |
 mkdir ~/.ssh
 ssh-keyscan -p ${{ secrets.SSH\_PORT }} -t rsa ${{ secrets.SSH\_HOST }} \&gt;\&gt; ~/.ssh/known\_hosts
 ssh-agent -a $SSH\_AUTH\_SOCK \&gt; /dev/null
 ssh-add - \&lt;\&lt;\&lt; &quot;${{ secrets.SSH\_KEY }}&quot;
- name: Deploy
run: npm run deploy
env:
DOCKER\_HOST: ssh://${{ secrets.SSH\_USER }}@${{ secrets.SSH\_HOST }}:${{ secrets.SSH\_PORT }}
SSH\_AUTH\_SOCK: /tmp/ssh\_agent.sock

At first, this can look very complicated, but if we analyse it step by step it will make a lot more sense:

**name:**
 the name of our pipeline. This will be shown in The GitHub interface later.

**on:**
 The git event that we want to be triggering our pipeline

**jobs:**
 Groups together different phases of our deploy pipeline. In this case I&#39;m on going to use a single job for simplicity.

**build** :
 The name we give to a job

**runs-on:**
 The base container on which we want to execute the commands that will build and deploy our app. In this case it&#39;s a container running the latest ubuntu version.

**Steps**
 Literally, the steps needed to build and publish our app.

**uses: actions/checkout@v2**
Uses a GitHub action already prepared for a certain Task. In this case this action will checkout our code into the container where we are running the build process, so that we can work on it. This action is publicly available on GitHub at [https://github.com/actions/checkout/tree/v2](https://github.com/actions/checkout/tree/v2)

**name:**
Similarly as before, but in this case the name of a step of our pipeline. This section will contain attributes such as

**run:**
 The command to run.

**env:**
 environment variables to be used in the environment where the script is going to run (with the run command).

### The Configure SSH connection step

This is the most cryptic part of the configuration and it&#39;s the one that contains all the magic that allows GitHub to &quot;talk&quot; with our remote docker instance using SSH. Let&#39;s analyse it line by line:

mkdir ~/.ssh

Creates a directory called _.ssh_ inside the current user&#39;s home directory. This is the place where SSH checks for all possible configuration and we make it available in case it&#39;s not there already.

ssh-keyscan -p ${{ secrets.SSH\_PORT }} -t rsa ${{ secrets.SSH\_HOST }} \&gt;\&gt; ~/.ssh/known\_hosts

The ssh-keyscan command collects a public key from a SSH server. This process is needed to let the SSH agent existing in the GitHub action container the public key of our remote system. Without this step, the SSH agent would refuse to communicate with. The two variables ${{secrets.SSH\_PORT}} and ${{secrets.SSH\_HOST}} contain, as the name suggests, the port and the host url of our remote server. The next chapter &quot;Secrets&quot; explains how these secrets are configured.

ssh-agent -a $SSH\_AUTH\_SOCK \&gt; /dev/null

This would be very long to explain here, but let&#39;s just say it&#39;s needed for the next step to be able to work properly. More information can be found here: [http://blog.joncairns.com/2013/12/understanding-ssh-agent-and-ssh-add/](http://blog.joncairns.com/2013/12/understanding-ssh-agent-and-ssh-add/)

ssh-add - \&lt;\&lt;\&lt; &quot;${{ secrets.SSH\_KEY }}&quot;

Adds a private key to the list of known private key. This is a fundamental step to allow our pipeline to access our remote server, using SSH without any prompt for username and password. Without it, the deploy action would stop for the SSH login, waiting for a user input.

### Secrets

Our configuration file contains several references to the _secrets_ variable. This is a feature of GitHub that allows us to add to our pipelines some values that need to be kept secret for security reason. In depth documentation about how this works can be found at: [https://docs.github.com/en/actions/security-guides/encrypted-secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets).

Let&#39;s consider these secrets as a sort of secret configuration to which only the owner of the repository have access. So, let&#39;s create the following secrets:

- SSH\_HOST: must contain the public internet IP of our remote server. It&#39;s the same IP that we use for the host parameter when using the ssh command. Making it a secret avoids having our server&#39;s IP directly available in our repository. This wouldn&#39;t be a security issue per-se but could be a starting point for someone with hill intention to start probing our server for weak points.
- SSH-KEY: must contain the private SSH key needed to access our server. It goes without saying that we don&#39;t want this value in any public place. We created it on the step 3 of this guide.
- SSH-PORT: must contain the port where our server&#39;s SSH daemon is listening for connections. Normally is the 22, but depending on the server&#39;s configuration, it could be different.
- SSH-USER: Must contain the username we use to access our SSH server.

![](RackMultipart20220401-4-s383db_html_ad23f314702ebd85.png)

## Step 5: Testing our pipeline

Everything is set up: it&#39;s finally the moment to test the system!

Let&#39;s make a small change to our app:

app.get(&#39;/&#39;, (req, res) =\&gt; {
res.send(&#39;Hello World 2!&#39;) // \&lt;-- Let&#39;s change the value to 2
})

Now let&#39;s commit and push it.

If everything goes according to plan, we should now see the GitHub pipeline kicking in and start the deploy process.

If all the steps go through without any error, our new app should be deployed to our remote Docker instance. Let&#39;s now access our remote server from the browser and notice that the content is changed:

![Picture 1](RackMultipart20220401-4-s383db_html_80f68c0e163c8617.gif)

Success!

## Conclusion

With this exercise we now have our own very basic but free CI/CD pipeline that deploys our content to our own remote server. From here on, we can build more and more complex scenarios with the help of GitHub Actions and docker, scaling the system more and more with anyone of our projects!

And if we end up deciding that after all, this system is too clunky and requires too much work, and that we would rather be using one of the easier, but more expensive solutions out there, at least we learned something new and gained some more insight about what are the basic steps for an automated CI/CD system.
