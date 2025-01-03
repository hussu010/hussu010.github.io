---
layout: post
title: "Access Your Raspberry Pi Remotely: SSH Setup with FRP (No Port Forwarding Required)"
categories: devops
---

Set up your raspberry pi/ old laptop/ pc as a server and expose it to the internet for ssh access without port forwarding.

## Introduction

Many of my personal and work projects require dedicated development and testing environments. We often containerize these projects and deploy them on EC2 instances. However, as our EC2 bills started to grow, we decided to take a cost-effective approach by migrating some of these projects to our on-premises server.

In this post, I'll walk you through how we're leveraging a Raspberry Pi as a server and securely exposing it to the internet for SSH access.

## Prerequisites

- Raspberry Pi/ Old laptop/ PC with ubuntu installed
- Server with a static IP address (deployed on AWS/ GCP/ Azure/ Vultr/ DigitalOcean) to act as a relay server

## Step 1: Spin up a Relay Server

First, we need to set up a relay server that will act as a bridge between our Raspberry Pi and the internet. This server will be responsible for forwarding traffic to and from the Raspberry Pi. You can use any cloud provider to spin up this server. In this example, we'll use an AWS EC2 instance.

1. Launch an EC2 instance with Ubuntu 20.04 AMI.
2. Assign an Elastic IP to the EC2 instance to ensure a static IP address.
3. SSH into the EC2 instance and install the FRP server.

    ```bash
sudo apt update
sudo apt install wget unzip openssh-server -y
wget https://github.com/fatedier/frp/releases/download/v0.61.1/frp_0.61.1_linux_amd64.tar.gz
tar -xvf frp_0.61.1_linux_amd64.tar.gz
cd frp_0.61.1_linux_amd64
sudo cp frps /usr/local/bin
cd
sudo rm -rf frp_0.61.1_linux_amd64
sudo rm frp_0.61.1_linux_amd64.tar.gz
    ```

4. Create a configuration file for the FRP server.

    ```bash
sudo mkdir -p /etc/frp
sudo nano /etc/frp/frps.toml
    ```

5. Add the following configuration to the file.

    ```toml
bindPort = 7000
    ```

6. Start the FRP server.

    ```bash
frps -c /etc/frp/frps.toml
    ```

## Step 2: Set up FRP on Raspberry Pi

Next, we need to install the FRP client on the Raspberry Pi to establish a connection with the relay server.

1. SSH into the Raspberry Pi and install the FRP client.

    ```bash
sudo apt update
sudo apt install wget unzip openssh-server -y
wget https://github.com/fatedier/frp/releases/download/v0.61.1/frp_0.61.1_linux_amd64.tar.gz
tar -xvf frp_0.61.1_linux_amd64.tar.gz
cd frp_0.61.1_linux_amd64
sudo cp frpc /usr/local/bin
cd
sudo rm -rf frp_0.61.1_linux_amd64
sudo rm frp_0.61.1_linux_amd64.tar.gz
    ```

2. Create a configuration file for the FRP client.

    ```bash
sudo mkdir -p /etc/frp
sudo nano /etc/frp/frpc.toml
    ```

3. Add the following configuration to the file.

    ```bash
serverAddr = "x.x.x.x"
serverPort = 7000
[[proxies]]
name = "ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000
    ```

    Replace `x.x.x.x` with the Elastic IP/ dns name of the relay server.

4. Start the FRP client.

    ```bash
frpc -c /etc/frp/frpc.toml
    ```

## Step 3: Open port 6000 and 7000 on the Relay Server

You must have received an error message when you started the FRP client on the Raspberry Pi. This is because the relay server is not allowing traffic on ports 6000 and 7000 by default. Now, we need to open these ports on the relay server to establish a connection with the Raspberry Pi.

![Connection Error Client To Relay Server](/assets/images/connection-error-client-to-relay.png)
*Connection Error Client To Relay Server*

This can be done by updating the security group settings in the AWS console.

1. Go to the AWS console and navigate to the EC2 dashboard.
2. Select the security group associated with the relay server instance.
3. Click on the "Inbound rules" tab and add two new rules.

    - Type: Custom TCP Rule, Protocol: TCP, Port Range: 6000, Source: `0.0.0.0/0` (or restrict it to your IP address)
    - Type: Custom TCP Rule, Protocol: TCP, Port Range: 7000, Source: `0.0.0.0/0` (or restrict it to Raspberry Pi's IP address)

    ![Inbound Security Rules](/assets/images/inbound-security-rules.png)
    *Inbound Security Rules*

4. (Optional) If your using a firewall on the relay server, make sure to allow traffic on ports 6000 and 7000. For example, if you're using UFW, you can run the following commands.

    ```bash
sudo ufw allow 6000/tcp
sudo ufw allow 7000/tcp
    ```

5. Restart the FRP on the relay server.

    ```bash
frps -c /etc/frp/frps.toml
    ```

6. Restart the FRP on the Raspberry Pi.

    ```bash
frpc -c /etc/frp/frpc.toml
    ```

    ![Connection Success Client To Relay Server](/assets/images/connection-success-client-to-relay.png)
    *Connection Success Client To Relay Server*

## Step 4: SSH into the Raspberry Pi

Now that everything is set up, you can SSH into the Raspberry Pi from anywhere using the relay server. To do this, run the following command on your local machine.

```bash
ssh -p 6000 pi@x.x.x.x
```

Replace `pi` with the username of your Raspberry Pi and `x.x.x.x` with the Elastic IP of the relay server.

And that's it! You now have a Raspberry Pi accessible from anywhere on the internet without port forwarding. This setup is secure and cost-effective, making it ideal for personal projects, development environments, and testing setups.

## FAQs

### Can I use a different cloud provider for the relay server?

Yes, you can use any cloud provider that allows you to spin up a server with a static IP address. The steps may vary slightly depending on the provider, but the overall setup remains the same.

### Is this setup secure?

Encryption is off by default in FRP. You can enable encryption by adding the following configuration to the FRP server and client configuration files.

```toml
transport.useEncryption = true
```

I also recommend using SSH keys for authentication instead of passwords to enhance security. You can set up SSH keys by following the steps outlined in this <a href="https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04" target="_blank">tutorial</a>.

### Why not use a VPN instead?

While VPNs are a secure way to access your devices remotely, they can be complex to set up and maintain. FRP provides a simple and lightweight solution for exposing your Raspberry Pi to the internet without the need for port forwarding.

### Why not ngrok?

Ngrok is a popular tool for exposing local servers to the internet. However, it has limitations on the free tier, such as limited connections and tunnel duration. FRP, on the other hand, is open-source and allows you to set up your own relay server with no such limitations.

### Where can I learn more about FRP?

You can find more information about FRP on the <a href="https://github.com/fatedier/frp" target="_blank">official GitHub repository</a>.

## Conclusion

In this post, we talked about how to set up a Raspberry Pi as a server and expose it to the internet using FRP. This setup provides a cost-effective and secure way to access your Raspberry Pi remotely without port forwarding. In the next post, I'll show you how we are using cloudflare tunnel to expose docker containers to the internet.
