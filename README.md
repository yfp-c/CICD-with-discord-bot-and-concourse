- [CICD-with-discord-bot-and-concourse-on-oracle](#cicd-with-discord-bot-and-concourse-on-oracle)
  * [Useful commands]
  * [Set up concourse on a new ubuntu oracle server](#set-up-concourse-on-a-new-ubuntu-oracle-server)
  * [Setting up SSH connection between discord bot server and concourse ci server](#setting-up-ssh-connection-between-discord-bot-server-and-concourse-ci-server)
  * [Set up git(hub) on your concourse ci server](#set-up-git-hub--on-your-concourse-ci-server)
    + [Configure your GitHub name and email:](#configure-your-github-name-and-email-)
    + [Generate a SSH key-pair and add a comment so you can identify it](#generate-a-ssh-key-pair-and-add-a-comment-so-you-can-identify-it)
    + [Add ssh key to ssh-agent](#add-ssh-key-to-ssh-agent)
    + [Either cd into the .ssh directory or the below and copy the public key:](#either-cd-into-the-ssh-directory-or-the-below-and-copy-the-public-key-)
  * [Create concourse job to test connection between concourse and github](#create-concourse-job-to-test-connection-between-concourse-and-github)
    + [Make sure a few things are set up](#make-sure-a-few-things-are-set-up)
    + [set up yaml file](#set-up-yaml-file)
    + [Export private key into pipeline](#export-private-key-into-pipeline)
    + [Set pipeline](#set-pipeline)
    + [Trigger pipeline](#trigger-pipeline)
  * [Create concourse job to connect to discord server](#create-concourse-job-to-connect-to-discord-server)
    + [Modify yml file to connect concourse ci to discord bot server](#modify-yml-file-to-connect-concourse-ci-to-discord-bot-server)
    + [export new ssh private key into env variable](#export-new-ssh-private-key-into-env-variable)
    + [set pipeline and trigger job](#set-pipeline-and-trigger-job)
  * [Create a test concourse job to push to github](#create-a-test-concourse-job-to-push-to-github)
    + [Create new yml to deploy to a test github repo](#create-new-yml-to-deploy-to-a-test-github-repo)
    + [create new pipeline and add in variables etc](#create-new-pipeline-and-add-in-variables-etc)
  * [Create a concourse job to pull changes from the discord bot server and push changes to github](#create-a-concourse-job-to-pull-changes-from-the-discord-bot-server-and-push-changes-to-github)
    + [The yaml file](#the-yaml-file)

# CICD-with-discord-bot-and-concourse-on-oracle
~~I will attempt to do an end to end pipeline with the discord bot I have created using concourse as my pipeline~~

I have created a concourse job to pull changes from the discord bot server and push it to github. 

Future plans:

- Unit testing
- Automatically detect changes on discord bot server and push to github
- Update modules/cogs on the discord client without myself manually refreshing them

## Useful commands
```
fly -t ci login -c http://your-ip-address:8080

fly -t <target> set-pipeline -p <pipeline-name> -c job.yml
fly -t <target> unpause-pipeline -p <pipeline-name> 
fly -t <target> pause-pipeline -p <pipeline-name> 

export SSH_PRIVATE_KEY="$(cat /home/ubuntu/.ssh/id_rsa)"
fly -t <target> set-pipeline -p <pipeline-name> -c job.yml --var "SSH_PRIVATE_KEY=$(echo $SSH_PRIVATE_KEY)"

fly -t <target> trigger-job -j <pipeline-name>/job-name
fly -t <target-name> destroy-pipeline -p <pipeline-name>

```

## Set up concourse on a new ubuntu oracle server 
1. First thing to do is run a few commands to open ports so you can ssh in
```
sudo iptables -I INPUT -j ACCEPT
```
[Reddit post with this solution(https://www.reddit.com/r/oraclecloud/comments/r8lkf7/a_quick_tips_to_people_who_are_having_issue/)

2. 

We need to open ports to allow yourself or other users (not recommended) to login to the server and its ports. 

On oracle head to Networking > Virtual Cloud Network > Virtual Cloud Network Details > Subnet Details > Security List Details > Ingress Rules > Add Ingress Rules 

<img width="500" alt="Screenshot 2023-03-14 at 18 54 34" src="https://user-images.githubusercontent.com/98178943/225108814-d6884385-0f42-4f2a-bb42-dc7d2b7cd22a.png">

Use https://mxtoolbox.com/subnetcalculator.aspx to find your CIDR block

3. 

```
sudo apt update && sudo apt upgrade -y
sudo reboot
```

4. [DigitalOcean link to install docker and docker-compose](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-ubuntu-22-04)

5. [Official link to install concourse ci](https://concourse-ci.org/quick-start.html)

In the yaml file: change ```CONCOURSE_EXTERNAL_URL: http://localhost:8080``` to your http://your-ip:8080 from http://localhost:8080

Login to http://your-ip:8080 to see if concourse works. There should be a page like this:

<img width="400" alt="Screenshot 2023-03-14 at 19 22 57" src="https://user-images.githubusercontent.com/98178943/225114939-59919e78-7ed0-45ab-8bfb-62d2ca6a985b.png">

Congrats! Concourse Ci is set up. 

## Setting up SSH connection between discord bot server and concourse ci server

```
ssh-keygen

cat ~/.ssh/id_rsa.pub

echo public_key_string >> ~/.ssh/authorized_keys
```

replace public_key_string with the copied key from the 'cat ~/.ssh/id_rsa.pub' command and then login using the command below:

```
ssh ubuntu@your-ip-address
```

## Set up git(hub) on your concourse ci server

Install Git on the concourse ci server: 
```
sudo apt update -y
sudo apt install git
```

### Configure your GitHub name and email:
```
git config --global user.name "Actual Name"
git config --global user.email "github@example.com"
```
### Generate a SSH key-pair and add a comment so you can identify it
```
ssh-keygen -t ed25519 -c "github-email@email.com"
```
### Add ssh key to ssh-agent
```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```
### Either cd into the .ssh directory or the below and copy the public key: 
```
cat ~/.ssh/id_ed25519.pub
```
Add the SSH public key to your GitHub account at https://github.com/settings/keys
and then test the connection by doing
```
ssh -T git@github.com
```
and you should get a message saying "Hi github-user! You've successfully authenticated, but GitHub does not provide shell access."

## Create concourse job to test connection between concourse and github

### Make sure a few things are set up
- If you're running concourse on docker, ensure it is running the concourse image (if you've forgotten it is docker-compose up -d)
- Open ports for your ip for ports 80,8080 etc and open ports for your concourse server's ip to 8080

### set up yaml file
```
resources:
- name: sbot-repo
  type: git
  source:
    uri: <your-github-repo-can-be-found-on-your-repo-page-click-the-green-link-and-copy-url->
    branch: <enter-repo-branch>
    private_key: ((SSH_PRIVATE_KEY))

jobs:
- name: pull-repo
  public: true
  plan:
  - get: sbot-repo
    trigger: true
  - task: display-git-info
    config:
      platform: linux
      image_resource:
        type: registry-image
        source:
          repository: ubuntu
      inputs:
      - name: sbot-repo
      run:
        path: bash
        args:
        - -c
        - |
          apt-get update
          apt-get install -y git
          echo "Git repository information:"
          cd sbot-repo
		  git --no-pager log -1
```
### Export private key into pipeline
```
export SSH_PRIVATE_KEY="$(cat /home/ubuntu/.ssh/id_rsa)"
```
### Set pipeline
```
fly -t <target-name> set-pipeline -p <pipeline-name> -c <path-to.yml> --var "SSH_PRIVATE_KEY=$(echo $SSH_PRIVATE_KEY)"
e.g. fly -t ci set-pipeline -p sbot-pipeline -c sbot_pipeline.yml --var "SSH_PRIVATE_KEY=$(echo $SSH_PRIVATE_KEY)"
```
If this is your first time opening the concourse ci ui and logging into it, it will prompt you to enter a token. Open the link it gives and copy the token in to access the gui.

### Trigger pipeline
```
fly -t ci trigger-job -j sbot-pipeline/sync-and-push
```
If all is good, you might see something similar to this:

<img width="500" alt="Screenshot 2023-04-01 at 11 57 07" src="https://user-images.githubusercontent.com/98178943/229284782-d430181e-4e22-4d7a-9be4-a35e337ef2a1.png">

## Create concourse job to connect to discord server
HIGHLY RECOMMEND to create new ssh key-pair using something like:
```
ssh-keygen -t rsa -b 4096
```
Display the public key:
```
cat ~/.ssh/your_public_key.pub
```
Copy contents to ~/.ssh/authorized_keys and save.

It took me HOURS to figure out issues, hours wasted of googling and going through logs etc. My concourse job was connecting to my discordbot server but it wasn't going through at the last stage. All I had to do was make a new ssh key-pair!

### Modify yml file to connect concourse ci to discord bot server
```
resources:
- name: sbot-repo
  type: git
  source:
    uri: <your-github-repo-can-be-found-on-your-repo-page-click-the-green-link-and-copy-url->
    branch: <enter-repo-branch>
    private_key: ((SSH_PRIVATE_KEY))

jobs:
- name: pull-repo-and-test-ssh
  public: true
  plan:
  - get: sbot-repo
    trigger: true
  - task: display-git-info-and-test-ssh
    config:
      platform: linux
      image_resource:
        type: registry-image
        source:
          repository: ubuntu
      params:
        DISCORD_BOT_SERVER_IP: ((DISCORD_SERVER_IP))
        DISCORD_BOT_SERVER_SSH_KEY: ((DISCORD_BOT_SERVER_SSH_KEY))
      inputs:
      - name: sbot-repo
      run:
        path: bash
        args:
        - -c
        - |
          apt-get update
          apt-get install -y git openssh-client
          echo "Git repository information:"
          cd sbot-repo
          git --no-pager log -1

          echo "Testing SSH connection to the Discord sbot server"
          echo "$DISCORD_BOT_SERVER_SSH_KEY" > id_rsa_discord_bot
          chmod 600 id_rsa_discord_bot
          ssh-keyscan $DISCORD_BOT_SERVER_IP >> known_hosts
	  # adding to knownhosts to avoid host authentication but it is still secure compared to 'ssh stricthostkeychecking no'
          ssh -v -i id_rsa_discord_bot -o UserKnownHostsFile=known_hosts ubuntu@$DISCORD_BOT_SERVER_IP 'echo "Connected to the Discord bot server"'
	  # verbose option to troubleshoot if needed
```
### export new ssh private key into env variable
```
export DISCORD_BOT_SERVER_SSH_KEY="$(cat path-to-private-key)"
```
### set pipeline and trigger job
```
fly -t <target-name> set-pipeline -p <name-of-pipeline> -c path-to.yml -v SSH_PRIVATE_KEY="$(cat path-to-private-key)" -v DISCORD_BOT_SERVER_SSH_KEY="$(cat path-to-private-key)"

fly -t ci trigger-job -j sbot-pipeline/pull-repo-and-test-ssh
```
Hopefully you'll see a successful build such as the below

<img width="1085" alt="Screenshot 2023-04-01 at 15 27 27" src="https://user-images.githubusercontent.com/98178943/229295216-f33cf538-436a-46a7-8567-e0ecfe5579dc.png">

## Create a test concourse job to push to github
So, it's more of the same. Set up new yml, new pipeline etc.

### Create new yml to deploy to a test github repo
```
resources:
- name: some-repo
  type: git
  source:
    uri: git@github.com:yfp-c/test.git
    branch: main
    private_key: ((SSH_PRIVATE_KEY))

jobs:
- name: push-text-file
  plan:
  - get: some-repo
    trigger: true
  - task: update-text-file-and-push
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: debian
          tag: stable-slim
      inputs:
      - name: some-repo
      outputs:
      - name: some-modified-repo
      run:
        path: bash
        args:
        - -c
        - |
          set -eux
          apt-get update -y
          apt-get install -y git
          git clone some-repo some-modified-repo

          cd some-modified-repo
          echo "new line" >> text.txt

          git add .

          git config --global user.name "Name"
          git config --global user.email "email@hotmail.com"

          git commit -m "Changed text.txt"

  - put: some-repo
    params: {repository: some-modified-repo}
    
```
### create new pipeline and add in variables etc
```
$ fly -t ci set-pipeline -p github-test-pipeline -c test-push-to-github.yml -v SSH_PRIVATE_KEY="$(cat path-to-private-key)"
$ fly -t ci trigger-job -j github-test-pipeline/push-changes-to-repo
```

## Create a concourse job to pull changes from the discord bot server and push changes to github

### The yaml file
Again, it's more of the same. Set up new yml, new pipeline etc. Combining what we've done in the steps above with some extra steps.
Make sure to create a .gitignore file of the repo to where you'll be pushing to with the private key you'll be using to enter the discord bot server. In the example below, it would be 'id_rsa_discord_bot'.
```
resources:
- name: some-repo
  type: git
  source:
    uri: git@github.com:yfp-c/test.git
    branch: main
    private_key: ((SSH_PRIVATE_KEY))

jobs:
- name: push-text-file
  plan:
  - get: some-repo
    trigger: true
  - task: update-text-file-and-push
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: debian
          tag: stable-slim

      params:
        DISCORD_BOT_SERVER_IP: xx.xx.xx.xx.xx.xx
        DISCORD_BOT_SERVER_SSH_KEY: ((DISCORD_BOT_SERVER_SSH_KEY))

      inputs:
      - name: some-repo
      outputs:
      - name: some-modified-repo
      run:
        path: bash
        args:
        - -cex
        - |
          set -eux
          apt-get update -y
          apt-get install -y git rsync openssh-client
          git clone some-repo some-modified-repo

          cd some-modified-repo

          echo "Testing SSH connection to the Discord sbot server"
          echo "$DISCORD_BOT_SERVER_SSH_KEY" > id_rsa_discord_bot
          chmod 600 id_rsa_discord_bot
          mkdir -p ~/.ssh
          ssh-keyscan $DISCORD_BOT_SERVER_IP >> ~/.ssh/known_hosts
          rsync -avz -e "ssh -i id_rsa_discord_bot -o UserKnownHostsFile=~/.ssh/known_hosts" --exclude 'module.py' --exclude 'module.py' ubuntu@$DISCORD_BOT_SERVER_IP:/home/ubuntu/sbot/cogs/ cogs/

          rm id_rsa_discord_bot

          ls -a

          git add .

          git config --global user.name "NAME"
          git config --global user.email "email@hotmail.com"

          git commit -m "Added cogs test"


  - put: some-repo
    params: {repository: some-modified-repo}
```
