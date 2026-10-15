---
title: "How I built my home server with Pi-hole, Tailscale and Dokploy"
description: "How I moved my apps off Heroku and onto a Raspberry Pi 5 home server running Pi-hole, Tailscale and Dokploy."
pubDate: 'Oct 15 2026'
---

When Heroku ended the free tier I had to move all my applications off Heroku. After comparing all the cloud providers I found that Hetzner would be the cheapest but didn't want to be dependent on any cloud provider so I decided to host my applications at home in my own home server.

I was already hosting Pi-hole in a Raspberry Pi 3b and I had self hosted things on my home network before so after doing all the calculations I found that buying a Raspberry Pi 5 with 8 GB of ram and an SSD hat for the Raspberry Pi would pay for itself in 12 months compared to a similar machine in Hetzner so I went ahead and bought the Raspberry PI.

After some tries I ended up with a software setup that I like and that is closer to what cloud PaaS services provide. This setup is what I'm going to show in this article.

The server is composed of 3 software pieces:
- **Pi-hole**: Pi-hole is a network-wide DNS ad blocker. It also lets me define local DNS records for services I only want available at home.
- **Tailscale**: Tailscale is a mesh VPN. It lets me reach the server (and my whole home network) from anywhere, and use the server as an exit node.
- **Dokploy**: Dokploy is a self-hosted deployment platform. It runs on top of Docker and uses Traefik under the hood, which handles domains and HTTPS for each service I install.

## Installation steps

### 1. Install Ubuntu server
Starting with a fresh Ubuntu Server install we update and upgrade:

```sh
sudo apt-get update
sudo apt-get upgrade
```

### 2. Install Pi-hole

Use the following command to install Pi-hole:

```sh
curl -sSL https://install.pi-hole.net | sudo bash
```
	
> Note: Make sure to save the password provided by Pi-hole during installation.

Next, change the Pi-hole web admin port from 80 and 443 to 8080 and 4443, because it will conflict with the HTTP and HTTPS port used by Traefik (Dokploy's reverse proxy)

```bash
sudo nano /etc/Pi-hole/Pi-hole.toml
```

Change the line
```
port = "80o,443os,[::]:80o,[::]:443os"
```
To
```
port = "8080o,4443os,[::]:8080o,[::]:4443os"
```

Restart FTL to apply the changes

```bash
sudo systemctl restart Pi-hole-FTL
```
		
Finally, in the Pi-hole admin, go to the **Settings**/**DNS** tab, change to **Expert** mode, and under `interface settings`, change Pi-hole to **Permit all origins** and **Save & Apply**. This is needed so we can use Pi-hole as our DNS server in Tailscale.

### 3. Install Tailscale to access the server from anywhere
Use the Tailscale dashboard to generate your install script and run it on the server.

Then run the following command to bring Tailscale up without letting it override DNS, so Pi-hole stays in charge:

```bash
sudo Tailscale up --accept-dns=false --advertise-exit-node
```

Next, in the Tailscale dashboard:
1. Disable `key expiry` for your server.
2. Go to the `DNS` tab and in the `namespaces` section add your server IP address as the dns server. Make sure to activate `Override DNS servers`.

Then, in your server terminal, enable IP forwarding so the server can route traffic to your network:
	
```sh
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-Tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-Tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-Tailscale.conf
```
	
Re-enable Tailscale with subnet routing:

```sh
sudo Tailscale up --advertise-routes=[your network IP range]/24 --accept-dns=false --advertise-exit-node
```

> Replace the `[your network IP range]` with your own network IP range. Example: 192.168.1.0/24

Finally, approve the route in Tailscale admin by going to your machine in the subnet section

>More info: https://Tailscale.com/kb/1019/subnets?tab=linux#enable-ip-forwarding
		
### 4. Install Dokploy

Use the following script to install Dokploy:

```bash
curl -sSL https://dokploy.com/install.sh | sh
```
	
Then open the Dokploy web interface and add the domain you wish to access Dockploy.

### 5. Add a domain to Pi-hole using Traefik

If you want to access Pi-hole using a domain instead of the server IP, we can do this though Traefik. Create a dynamic config file:

```bash
sudo nano /etc/dokploy/Traefik/dynamic/Pi-hole.yml
```

```yaml
http:
  routers:
    Pi-hole:
      rule: "Host(`Pi-hole.example.com`)"
      entryPoints:
        - websecure
      service: Pi-hole
      tls:
        certResolver: letsencrypt   # Dokploy usually sets this automatically
      middlewares:
        - Pi-hole-redirect

  middlewares:
    Pi-hole-redirect:
      redirectRegex:
        regex: "^https?://Pi-hole\\.example\\.com/?$"
        replacement: "https://Pi-hole.example.com/admin"
        permanent: true

  services:
    Pi-hole:
      loadBalancer:
        servers:
          - url: "http://[your server IP address]:8080"
```
> Replace the `example` in the `regex` and `replacement` lines with your own domain. Also replace `[your server IP address]` with your server IP address.

### 6. Install your services
From here, install whatever services you want in Dokploy, add a domain to each one.

To access the service in your home network by it's domain, add the domain in the `Local DNS Records` tab in Pi-hole by giving Pi-hole the domain you chose for the service and the IP address of the server.

If you want to access the service from the internet you can add **Cloudflare tunnels** for that service.

## Wrapping up
With this setup, every device on my network gets ad blocking, I can reach my home network and use the server as an exit node from anywhere via Tailscale, and deploying a new self-hosted service is just a matter of installing it in Dokploy and giving it a domain.