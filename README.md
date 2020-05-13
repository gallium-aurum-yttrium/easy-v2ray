# easy-v2ray
This document includes instructions on how to setup an AWS EC2 server and AWS CloudFront CDN to set up a reliable proxy. Although these steps would work with other cloud providers, the integration of CloudFront and EC2 makes it very convenient.

# Overall architecture
 - Webserver (nginx) is used to serve real HTTPS requests and forward the special requests to V2Ray
 - HTTPS certificate provided by letsencrypt.org
 - V2Ray is configured to expect HTTPS connection through a websocket, which is indistinguishable from normal HTTPS websockets
 - The HTTPS connection will be exposed to Internet using AWS CloudFront CDN
 - V2Ray in the client is configured as a local Socks5 and http proxy, and redirects some traffic to the CDN point ultimately reaching the matching V2Ray in the server
 - Optionally use a dynamic DNS service to ensure your setup will not change if your AWS instance is shut down, as AWS will assign a new IP every time it is stopped.
 
# Server setup
## AWS EC2 Instance
  - Create an EC2 instance in AWS, very low resources are needed, the free tier instances work fine
  - Follow the instructions on screen to get the SSH key. Ensure this key is password-protected.
  - Configure ssh access from your coputer to the server (search online for details)
    - For Windows use PuTTYGen to protect the key with password and use it as ssh key-base authentication
    - For Linux
      - Use Openssh to encrypt the key with a password (for example using pcks8), delete the original
      - Copy the password-protected key to `~/.ssh/`
      - Add this code to `~/.ssh/config`, changing your-alias to a text you like, and the IP of your EC2
```
Host your-alias 123.123.123.123
    HostName ec2-123-123-123-123.us-east-2.compute.amazonaws.com
    IdentityFile ~/.ssh/your-ec2-encrypted-and-protected-key.pkcs8
    User ubuntu
```
  - Configure the `Inbound Rules` in the `Security Group` used by the EC2 instance to allow Internet connections for these ports:
    - 80, required by letsencrypt.org
    - 443, will be used for normal web traffic and V2Ray websocket

## Configure dynamic DNS
Using a dynamic DNS will ensure that your setup is stable even if the EC2 IP address changes, which happens when the EC2 instance is shut down.

Alternatively, you can set up an Elastic IP, but you will be charged if the instance is offline. See [pricing]https://aws.amazon.com/premiumsupport/knowledge-center/elastic-ip-charges/.

These steps use DNSExit.com as example, but other services [supported by ddupdate]https://github.com/leamas/ddupdate/tree/devel/plugins will do as well.
  - Create an account in dnsexit.com
  - Add one free DNS record
  - In your EC2 server, install ddupdate `sudo apt install ddupdate`
  - Configure `/etc/ddupdate.conf`
```
[update]
address-plugin = default-web-ip
service-plugin = dnsexit.com
hostname = the-hostname-you-chose-in-dnsexit.linkpc.net
ip-version = v4
loglevel = warning
```
  - Configure credentials in file `/etc/netrc`
```
machine update.dnsexit.com
login your-dnsexit-username
password your-dnsexit-password
```
  - Ensure this file is owned by root and only root can read and write it `sudo chown root:root /etc/netrc` and `sudo chmod 600 /etc/netrc`
  - Enable the service and the timer `sudo systemctl enable ddupdate.service` and `sudo systemctl enable ddupdate.timer`
  - Run the update once and check that dnsexit site has updated the IP `sudo systemctl start ddupdate.service`
  - Set the hostname in EC2 server `sudo hostnamectl set-hostname the-hostname-you-chose-in-dnsexit.linkpc.net`
  - Use the hostname to connect to your EC2 from Windows/Linux

## Configure web server
