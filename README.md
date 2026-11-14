# DNS Exercise with AWS EC2, NGINX, and Cloudflare

**Module** ·  Networking  
**Lab date** · 14 November 2026

I launched an Ubuntu instance on Amazon EC2, configured NGINX using EC2 user data, and used Cloudflare to connect a custom subdomain to the web server. During the exercise, EC2 Instance Connect reported that SSH was blocked and Cloudflare returned a 522 timeout. A later browser check displayed the NGINX welcome page.

## 1. Launch the EC2 instance

| Setting | Value captured in the lab |
| --- | --- |
| Instance name | `nginx-demo` |
| Operating system | Ubuntu |
| AWS Region | US East (N. Virginia), `us-east-1` |
| Network | Default VPC, `172.31.0.0/16` |
| Subnet and Availability Zone | No preference at launch |
| Auto-assign public IPv4 | Enabled |
| Security group | `launch-wizard-1` |
| Inbound web rule | TCP port `80` from `0.0.0.0/0` |

The source CIDR `0.0.0.0/0` allowed IPv4 clients to reach port 80 for the lab. The public IPv4 address shown in the setup was `54.242.5.145`. Use the current instance address if repeating the exercise.

Evidence: [Instance launch](Screenshot%202026-11-12%20210345.png) · [Network settings](Screenshot%202026-11-12%20210418.png) · [Inbound HTTP rule](Screenshot%202026-11-12%20210447.png)

## 2. Install and start NGINX

I entered this script into **Advanced details → User data** when launching the instance.

```bash
#!/bin/bash
set -e
apt-get update
apt-get install -y nginx
systemctl enable --now nginx
```

The script refreshes Ubuntu's package index, installs NGINX, and starts the service while enabling it at boot. The `set -e` line stops the script if a command fails.

Evidence: [EC2 user data](Screenshot%202026-11-12%20210602.png)

## 3. Configure the Cloudflare DNS record

| Field | Value shown in the Add record form |
| --- | --- |
| Zone | `chrisdevops.uk` |
| Record type | A |
| Name | `nginx-demo` |
| Fully qualified name | `ngnix.chrisdevops.uk` |
| Origin IPv4 | `54.242.5.145` |
| Proxy status | Proxied |
| TTL | Auto |

An A record associates a hostname with an IPv4 address. With proxying enabled, public DNS queries return Cloudflare IP addresses. The browser connects to Cloudflare, which forwards web requests to the configured EC2 origin. [Cloudflare proxy status documentation](https://developers.cloudflare.com/dns/proxy-status/)

**Hostname discrepancy.** The Add record form shows `nginx-demo.chrisdevops.uk`. Both browser screenshots show `nginx.chrisdevops.uk`. The screenshots leave the final saved record name unverified. When repeating the lab, use the same hostname in the saved Cloudflare record, DNS lookup, and browser test.

Evidence: [Blank record form](Screenshot%202026-11-12%20211456.png) · [Populated A record form](Screenshot%202026-11-12%20211747.png)

## 4. Troubleshoot the errors

### EC2 Instance Connect reported that SSH was blocked

The instance was **Running**, with **2/3 status checks passed** when the screenshot was taken. The connection screen showed the Ubuntu username `ubuntu` and warned that the security group did not authorize TCP port `22` for browser-based EC2 Instance Connect.

For a repeat of the lab, check the instance status and the security group attached to it. If you need browser-based SSH, allow inbound TCP port 22 from the regional EC2 Instance Connect managed prefix list, `com.amazonaws.us-east-1.ec2-instance-connect`. SSH is for administering the instance. Web requests in this lab use TCP port 80. The capture shows the warning, and SSH recovery was undocumented. [AWS EC2 Instance Connect prerequisites](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-connect-prerequisites.html)

Evidence: [EC2 Instance Connect warning](Screenshot%202026-11-12%20210936.png)

### Cloudflare returned Error 522

The error page for `nginx.chrisdevops.uk` showed **Browser Working**, **Cloudflare Working**, and **Host Error** at **21:52:58 UTC** on 12 November 2026. Error 522 means Cloudflare timed out contacting the origin. The page shows that the request reached Cloudflare. [Cloudflare Error 522 documentation](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-522/)

My EC2 origin was set up for HTTP. NGINX listened on TCP port 80, and the security group allowed inbound web traffic on port 80 only. With Cloudflare SSL/TLS enabled, an HTTPS request using **Full** or **Full (strict)** mode requires Cloudflare to connect to the origin over HTTPS on TCP port 443. That port was unavailable in my setup, which explains the 522 timeout. I switched Cloudflare SSL/TLS to **Off** for this HTTP lab, then loaded the subdomain over HTTP. The NGINX welcome page appeared. [Cloudflare Full mode documentation](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/full/)

If the timeout recurs, check the request path in this order.

1. Confirm the exact hostname exists in Cloudflare and the saved A record points to the instance's current public IPv4 address.
2. Check EC2 status checks, the subnet route to the internet gateway, the attached security group's inbound HTTP rule, and any host firewall.
3. From the instance, check whether user data completed, whether NGINX is running, and whether it responds locally.

   ```bash
   sudo cloud-init status --long
   sudo tail -n 50 /var/log/cloud-init-output.log
   sudo systemctl status nginx --no-pager
   curl -I http://127.0.0.1/
   ```

4. From Windows PowerShell, test the origin IP and then the hostname. Replace `CURRENT_EC2_PUBLIC_IP` and `LAB_HOSTNAME` with the values from your setup.

   ```powershell
   curl.exe -I http://CURRENT_EC2_PUBLIC_IP/
   nslookup -type=A LAB_HOSTNAME
   curl.exe -I http://LAB_HOSTNAME/
   ```

   If local HTTP works but the direct public-IP request times out, review the origin's network path. If the direct request works but the proxied hostname still returns 522, compare the Cloudflare origin address and port with the working direct request and check whether the origin permits Cloudflare traffic. An `nslookup` result confirms DNS resolution, while the HTTP request checks whether the web service responds.

Evidence: [Cloudflare 522 error](Error%20encounted/Screenshot%202026-11-12%20220037.png) · [SSL/TLS settings](Error%20encounted/Screenshot%202026-11-12%20215926.png)

## 5. Verify the result

A later browser screenshot shows **Welcome to nginx!** at `nginx.chrisdevops.uk`. This confirms that the web page loaded at that hostname during the lab. The page loaded after I set Cloudflare SSL/TLS to **Off** and opened the hostname over HTTP.

Evidence: [NGINX welcome page](Screenshot%202026-11-12%20220223.png)

## What I learned

- DNS resolution and web-server reachability are separate checks. A browser can reach Cloudflare while Cloudflare times out reaching the origin.
- A proxied A record can return Cloudflare IPs even when the configured origin is an EC2 public IP.
- EC2 security group rules are port-specific. The HTTP rule on TCP 80 and the SSH rule on TCP 22 serve different purposes.
- Testing the server locally, by public IP, and by hostname helps isolate NGINX, AWS networking, and Cloudflare configuration.

