# easy-v2ray
This document includes instructions on how to setup an AWS EC2 server and AWS CloudFront CDN to set up a reliable proxy. Although these steps would work with other cloud providers, the integration of CloudFront and EC2 makes it very convenient.

Note: this setup is quite stable, but more advanced detection techniques may still detect it in the future. Hopefully by using a CDN node this is mitigated, as the number of users using this is mostly limited to people with IT-background.

The latest version of this document can be found here: https://github.com/gallium-aurum-yttrium/easy-v2ray/

# Overall architecture
 - Webserver (nginx) is used to serve real HTTPS requests and forward the special requests to V2Ray
 - HTTPS certificate provided by letsencrypt.org
 - V2Ray is configured to expect HTTPS connection through a websocket, which is indistinguishable from normal HTTPS websockets
 - The HTTPS connection will be exposed to Internet using AWS CloudFront CDN
 - V2Ray in the client is configured as a local Socks5 and http proxy, and redirects some traffic to the CDN point ultimately reaching the matching V2Ray in the server
 - Optionally use a dynamic DNS service to ensure your setup will not change if your AWS instance is shut down, as AWS will assign a new IP every time it is stopped.

**_IMPORTANT:_** Follow all the steps in order, there are dependencies between them.

# Server setup
## AWS EC2 Instance
  - Create an EC2 instance in AWS, very low resources are needed, the free tier instances work fine
  - Follow the instructions on screen to get the SSH key. Ensure this key is password-protected.
    - Encrypt ssh key: `openssl pkcs8 -topk8 -v2 des3 -in YOUR_KEY.pem -out YOUR_KEY_ENCRYPTED.pkcs8`
    - Generate ssh key to use with PuTTY/WinSCP: `puttygen YOUR_KEY.pem -o YOUR_KEY_ENCRYPTED.ppk -O private -P`
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
  - After validating connectivity using your encrypted key, ensure you delete the original, non-ecrypted key! rm YOUR_KEY.pem
  - Configure the `Inbound Rules` in the `Security Group` used by the EC2 instance to allow Internet connections for these ports:
    - 80, required by letsencrypt.org
    - 443, will be used for normal web traffic and V2Ray websocket

**_NOTE:_** Update the packages database before proceeding `sudo apt update`

## Configure dynamic DNS
Using a dynamic DNS will ensure that your setup is stable even if the EC2 IP address changes, which happens when the EC2 instance is shut down.

Alternatively, you can set up an Elastic IP in AWS ECS and use it instead as URL. Then you can skip this step. The only benefit of using a dynamic DNS service is to have an easy to remember name, but not required at all.

These steps use DNSExit.com as example, but other services [supported by ddupdate](https://github.com/leamas/ddupdate/tree/devel/plugins) will do as well.

**_NOTE:_** ddupdate runs as a normal user, in the instructions below it uses user `ubuntu` which is the default in AWS ubuntu image.
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
  - Configure credentials in file `/home/ubuntu/.netrc`
```
machine update.dnsexit.com
login your-dnsexit-username
password your-dnsexit-password
```
  - Ensure this file is owned by you and only you can read and write it `chmod 600 /home/ubuntu/.netrc`
  - Enable the service and the timer `systemctl --user start ddupdate.timer` and `systemctl --user enable ddupdate.timer`
  - Enable start on system boot `sudo loginctl enable-linger ubuntu`
  - Start the service immediately, to make sure it updates the DNS and to validate the configuration `systemctl --user start ddupdate`
  - Set the hostname in EC2 server `sudo hostnamectl set-hostname the-hostname-you-chose-in-dnsexit.linkpc.net`
  - Use the hostname to connect to your EC2 from Windows/Linux

## Configure web server
This is needed because some firewalls monitor HTTPS trafic and establish an HTTPS connection to the webserver to ensure it is a real web server.
  - Install nginx and letsencrypt facilities `sudo apt install nginx-core python3-certbot-nginx`
  - Add V2Ray configuration to `/etc/nginx/sites-available/default`
```
At the end of section server {...

    # Redirect to V2Ray, this location will be set up in the client
    location /streaming-service/ {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:9999;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;
    }
```
  - Start nginx and set it to start by default `sudo systemctl enable nginx`, `sudo systemctl start nginx`
  - Configure letsencrypt certificate `sudo certbot --nginx -d the-hostname-you-chose-in-dnsexit.linkpc.net`
  - Enable automatic renewal `sudo systemctl enable certbot.timer`
  - Using a browser from your pc open URL "https://the-hostname-you-chose-in-dnsexit.linkpc.net/" to ensure it is working properly with https
  - Optional: replace the default nginx index.html with a custom page

## Set-up AWS CloudFront
AWS CloudFront is a service managed from its own console, it's independent of EC2. It's located under "Networking & Content Delivery".

  - Create a distribution point. Use these options for `Distribution Settings`
    - Price Class - Whatever zone you prefer
    - AWS WAF Web ACL	None
    - Alternate Domain Names: 
    - SSL Certificate: Default
    - Supported HTTP Versions: 
    - HTTP/2, HTTP/1.1, HTTP/1.0: 
    - Default Root Object: 
    - Logging: Off
    - Distribution State: Enabled

  - Create an Origin for that distribution point. Use these settings:
    - Origin Domain Name: Your dynamic dns hostname
    - Origin Path: 
    - Minimum Origin SSL Protocol: TLSv1.2
    - Origin Protocol Policy: HTTPS Only
    - Origin Response Timeout: 30
    - Origin Keep-alive Timeout: 5
    - HTTP Port: 80
    - HTTPS Port: 443
  
  - In the "Behaviors" tab of AWS, edit the default behavior and select `Use legacy cache settings` in "Cache and origin request settings". If this is not enabled, the websockets connection will fail
  
  - Use your CloudFront domain name from a browser to ensure that it is setup correctly, for example https://sdfgdfcg4352y.cloudfront.net

## Install V2Ray
  - Download latest v2ray-core for Linux 64 bit https://github.com/v2ray/v2ray-core/releases
  - Uncompress it to /usr/local/share/, for example, /usr/local/share/v2ray-4.20.0
  - Create a symbolic link to the directory, this will allow for easy updates, as we will use v2ray without the version in the configuration files `sudo ln -sf /usr/local/share/v2ray-4.20.0 /usr/local/share/v2ray`
  - Create link to the binary file so it is in the expected location `sudo ln -sf /usr/local/share/v2ray/v2ray /usr/local/bin/`
  - Copy systemd unit to /etc `sudo cp /usr/local/share/v2ray/systemd/system/v2ray* /etc/systemd/system/`
  - Edit `/etc/systemd/system/v2ray.service`, set up the correct path for the configuration file `ExecStart=/usr/local/bin/v2ray -config /etc/v2ray/config.json`
  - Create config dir `sudo mkdir /etc/v2ray`
  - Copy default config.json to /etc/v2ray `sudo cp /usr/local/share/v2ray/config.json /etc/v2ray/`
  - When updating V2Ray, just uncompress the new archive and force the link creation with the new version `sudo ln -sf /usr/local/share/v2ray-4.22.3 /usr/local/share/v2ray`
  - Start v2ray `sudo systemctl start v2ray`
  - Ensure service starts properly by checking `sudo systemctl status v2ray`
  - Enable v2ray when system starts `sudo systemctl enable v2ray`

## Configure V2Ray
Now that V2Ray is installed, it needs to be configured. It is a very versatile software that it is configured by setting up input points, output points and routing between. For example, in the server, it is configured with websocket input and just internet output. In the client, it is configured with socks proxy input and websocket output.

  - Create a new UUID number using `uuidgen` command line tool (check this [website](https://www.uuidgenerator.net/) if uuidgen not available) the output will be something like 7a2a08a1-78d1-4646-a0dc-6ac45855f14b (do not use this one)
  - Copy the file config.json from this [git repo](https://github.com/gallium-aurum-yttrium/easy-v2ray/) to /etc/v2ray/
  - Edit the config.json file, making sure that
    - id: put the UUID generated here
    - path: option matches the URL in nginx configuration `/streaming-service/`
    - port: option matches the port in nginx configuration `9999`
  - Restart v2ray with the new configuration `sudo systemctl restart v2ray`

# Configure clients
Clients use the same V2Ray software as he server does, except the configuration is different. You can either use directly V2Ray-core or any of the available GUI frontends for your platform (for example Windows and Android).

The only requirement for the client is to match the VMEss configuration defined in the server, including the custom ID, otherwise it will fail to connect to the server.

An example client configuration file with additional features is provided in this repository, with the configuration shown below:
|  | Server | Client |
|-|-|-|
| Input | VMess protocol encapsulated in websockets | Socks 5 proxy<br>HTTP proxy |
| Output | Internet | VMess protocol encapsulated in websockets<br>Internet<br>Discard |
| Rules Input -> Output | All traffic is forwarded from VMEss input to Internet | 1. Advertising domains are discarded<br>2. Chinese websites are routed to the Internet, unmodified traffic<br>3. Any other traffic is forwarded through the VMess output |

The provided configuration tries to restrict the number of websites that are automatically forwarded through the server, to save on bandwidth. Also, connecting to websites directly is faster than going through the proxy.

To further refine the list of websites forwarded through the proxy, you can configure a local PAC file, to be used in the Windows or Firefox prroxy configuration (a local PAC file is not supported by Chrome). A good list is the GFW list, predefined PAC files with this list can be found in GitHub.
