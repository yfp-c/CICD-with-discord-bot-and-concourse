# CICD-with-discord-bot-and-concourse-on-oracle
I will attempt to do an end to end pipeline with the discord bot I have created using concourse as my pipeline

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

## Create pipeline build to test connection between concourse and github

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
