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
    - For Windows use PuTTYGen to protect the key with password and use it as ssh key-base authentication
  - Configure the `Inbound Rules` in the `Security Group` used by the EC2 instance to allow Internet connections for these ports:
    - 80, required by letsencrypt.org
    - 443, will be used for normal web traffic and V2Ray websocket

## 
