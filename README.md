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

## Configure your GitHub name and email:
```
git config --global user.name "Actual Name"
git config --global user.email "github@example.com"
```

