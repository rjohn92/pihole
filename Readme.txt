
Pi-hole Installation and Configuration Guide
Overview
Pi-hole is a network-wide ad blocker that acts as a DNS sinkhole, filtering unwanted ads and trackers at the DNS level. This guide walks you through setting up Pi-hole in a Docker container, resolving port conflicts (particularly for port 53), and configuring Pi-hole to use Cloudflare as the upstream DNS server instead of your router or ISP's DNS.

Prerequisites
A server running Docker (e.g., Ubuntu on a local machine or a Raspberry Pi)
Basic familiarity with command-line interface (CLI)
Network access to your router or devices for configuring DNS settings
systemd-resolved enabled (if applicable on your Linux system)
Steps
1. Installing Pi-hole in Docker
Start by creating a directory for your Pi-hole container:

bash
Copy code
mkdir -p ~/Containers/pihole
cd ~/Containers/pihole
Create the docker-compose.yml file:

yaml
Copy code
version: '3'
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    environment:
      TZ: 'America/New_York'  # Replace with your timezone
      WEBPASSWORD: 'your_password_here'  # Pi-hole web interface password
    volumes:
      - './etc-pihole/:/etc/pihole/'
      - './etc-dnsmasq.d/:/etc/dnsmasq.d/'
    ports:
      - "53:53/tcp"   # DNS port
      - "53:53/udp"   # DNS port
      - "80:80/tcp"   # Web interface
    restart: unless-stopped
    network_mode: "host"  # Allows Pi-hole to bind to port 53 on host
2. Handling Port Conflicts (Specifically Port 53)
Why Port 53 Is Special
Port 53 is used by the Domain Name System (DNS), which is crucial for resolving domain names into IP addresses. Since DNS is essential for internet connectivity, other services on your system (such as systemd-resolved) may already be using this port.

If something else is using port 53 (like systemd-resolved), Pi-hole won’t be able to bind to it. We need to free port 53 for Pi-hole.

Identifying Port Conflicts
To check if port 53 is in use:

bash
Copy code
sudo lsof -i :53
If you see something like systemd-resolve using port 53, you’ll need to change its configuration.

3. Disabling systemd-resolved or Configuring it to Free Port 53
Option 1: Disable systemd-resolved

If you don’t need systemd-resolved, you can disable it:

bash
Copy code
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
Then remove the symlink for /etc/resolv.conf:

bash
Copy code
sudo rm /etc/resolv.conf
Create a new resolv.conf pointing to Pi-hole’s IP:

bash
Copy code
sudo nano /etc/resolv.conf
Add the following content:

bash
Copy code
nameserver 10.0.0.146  # Replace with your Pi-hole’s IP address
Option 2: Modify systemd-resolved Configuration

If you prefer to keep systemd-resolved running, you can modify its configuration to free up port 53.

Open the systemd-resolved configuration file:

bash
Copy code
sudo nano /etc/systemd/resolved.conf
Find the line #DNSStubListener=yes and change it to:

bash
Copy code
DNSStubListener=no
This prevents systemd-resolved from binding to port 53.

Restart systemd-resolved:

bash
Copy code
sudo systemctl restart systemd-resolved
Verify port 53 is now free:

bash
Copy code
sudo lsof -i :53
You should no longer see systemd-resolved using port 53.

4. Running Pi-hole and Verifying Configuration
Start Pi-hole with Docker:

bash
Copy code
docker-compose up -d
Check if Pi-hole is running:

bash
Copy code
docker ps
Pi-hole should now be running, and port 53 should be successfully bound to the container.

5. Configuring Pi-hole to Use Cloudflare as Upstream DNS
By default, Pi-hole will use your router's DNS or Google's DNS. To enhance privacy and speed, it’s often a good idea to switch to Cloudflare DNS.

Steps to Use Cloudflare DNS in Pi-hole
Access the Pi-hole Admin Interface:

Open a web browser and navigate to http://<Pi-hole IP>/admin.
Log in using the password you set in the docker-compose.yml.
Navigate to Settings → DNS.

Under Upstream DNS Servers, deselect any other DNS options (e.g., Google).

Select Cloudflare (DNSSEC) under IPv4.

Scroll down and click Save.

6. Verifying Pi-hole is Using Cloudflare
After configuring Cloudflare as the upstream DNS provider, verify Pi-hole is correctly routing DNS queries through Cloudflare.

Go to Query Log in the Pi-hole admin interface.

Look for entries showing that DNS queries are being answered by one.one.one.one#53 (Cloudflare).

Alternatively, you can run an nslookup or dig command to verify:

bash
Copy code
nslookup google.com
The output should show Pi-hole’s IP as the DNS server and Cloudflare as the upstream resolver.

7. Configure Your Router or Devices to Use Pi-hole as DNS
For Pi-hole to block ads network-wide, you need to configure your router (or individual devices) to use Pi-hole as the DNS server.

Option 1: Configure Router DNS
Log into your router by entering its IP address in a browser (usually 192.168.1.1 or 10.0.0.1).

Navigate to DNS settings (this may be under LAN or DHCP settings).

Set your Pi-hole IP (e.g., 10.0.0.146) as the primary DNS server.

Save and reboot the router (if necessary).

Option 2: Configure Devices Manually
If your router doesn't allow DNS changes, you can configure each device individually:

Windows: Go to Control Panel → Network and Sharing Center → Change adapter settings, and set the DNS server manually to Pi-hole's IP.
Mac: Go to System Preferences → Network, select your connection, and manually configure the DNS server to Pi-hole's IP.
Android/iOS: Go to Wi-Fi settings for your network and set Pi-hole's IP as the DNS server.
8. Troubleshooting
Port 53 conflicts: Ensure that systemd-resolved or any other DNS service isn't using port 53. Check with sudo lsof -i :53.
DNS queries not blocked: Verify that devices on your network are using Pi-hole as the DNS server by running nslookup and checking the Query Log in Pi-hole's admin interface.
9. Conclusion
You’ve successfully installed Pi-hole in Docker, resolved any port conflicts, and configured it to use Cloudflare as the upstream DNS provider. Pi-hole is now ready to block ads and improve your privacy by filtering unwanted DNS requests.

