---
{"dg-publish":true,"permalink":"/self-hosting/creating-site-to-site-ip-sec-tunnel-using-strong-swan/","dgShowInlineTitle":true}
---


Folks at Oracle being nice bros offer you some very generous [Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm) including a 4-core ARM instance with 4Gbps network bandwidth. This is perfect for a VPN server. So let's do it.

### My server environment
>Oracle Cloud Infrastructure
>OS: Ubuntu-22.04 LTS



# Step 1 - Install everything we need
First update the repo
```bash
sudo apt update
```

Install `strongswan` and `certbot`
```bash
sudo apt install strongswan certbot libcharon-extra-plugins libcharon-extauth-plugins libstrongswan-extra-plugins
```

`strongswan` package is the strongswan software itself.
`certbot` is a tool for getting certificate from [Let's encrypt](https://letsencrypt.org/)
Other packages provide more authentication method and more encryption algorithm like elliptic curve cryptography

# Step 2 - Open ports on firewall
Open ports for http(80/tcp) and https(443/tcp) as well as 500, 4500/udp used by VPN in the firewall. The default firewall manager on Ubuntu is `ufw`

```bash
sudo ufw allow http
sudo ufw allow https
sudo ufw allow 500,4500/udp
sudo ufw reload
```

Normally for a cloud instance you also need to configure security rules for your cloud platform. On OCI this is done by [Security Group](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/networksecuritygroups.htm).

# Step 3 - Get certificate

## Get certificate using certbot
Simply run certbot using this command. It will spin up a temporary web server and handle certificate issuance automatically. (That why we need to open both 80 and 443 TCP port) 
```
sudo certbot certonly --rsa-key-size 4096 --standalone --agree-tos --no-eff-email --email <your-email@email.com> -d <your-domain.com>
```

I got the following output:

```bash
$ sudo certbot certonly --rsa-key-size 4096 --standalone --agree-tos --no-eff-email --email example@email.com -d vpn.nonaharry.com

Saving debug log to /var/log/letsencrypt/letsencrypt.log
Requesting a certificate for vpn.nonaharry.com

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/vpn.nonaharry.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/vpn.nonaharry.com/privkey.pem
This certificate expires on 2024-03-13.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

## Create a symlink for certificates
In Linux symlink is a reference to a file. 
Because these certificates need regularly updated, this make sure strongswan always use the latest certificate.
[More about certificate](https://eff-certbot.readthedocs.io/en/latest/using.html#where-are-my-certificates)
```bash
sudo ln -s /etc/letsencrypt/live/vpn.nonaharry.com/fullchain.pem /etc/ipsec.d/certs
sudo ln -s /etc/letsencrypt/live/vpn.nonaharry.com/chain.pem /etc/ipsec.d/cacerts
sudo ln -s /etc/letsencrypt/live/vpn.nonaharry.com/privkey.pem /etc/ipsec.d/private
```


# Step 4 Configuring StrongSwan

StrongSwan comes with a default configuration file containing some sample configurations. However, we'll need to configure most of it by ourselves. Start by backing up the existing configuration file for reference:

```bash
sudo mv /etc/ipsec.conf{,.original}
```

Next, create a new blank configuration file using your preferred text editor, such as `nano`:

```bash
sudo nano /etc/ipsec.conf
```

**Note:** In IPSec VPNs, "left" generally refers to the local system (here, the server) you're configuring, and "right" refers to remote clients, such as phones and computers.

Begin by configuring StrongSwan to log daemon statuses for debugging purposes and to allow duplicate connections. Insert the following lines into your configuration file:

```bash
config setup
    charondebug="ike 1, knl 1, cfg 0"
    uniqueids=no
```

Now, add a section for your VPN configuration. This section will set up IKEv2 VPN Tunnels and ensure StrongSwan loads this configuration on startup. Append these lines:

```bash
conn ikev2-vpn
    auto=add
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes
```

Include dead-peer detection to clear any unexpected disconnections. Add these settings:

```bash
dpdaction=clear
dpddelay=300s
rekey=no
```

Next, configure the server's "left" side IPSec parameters. These ensure the server accepts and correctly identifies incoming client connections. Add the following settings:

- `left=%any`: Uses the network interface where incoming connections are received.
- `leftid=@server_domain_or_IP`: The name presented to clients.
- `leftcert=server-cert.pem`: The path to the server's public certificate.
- `leftsendcert=always`: Ensures clients always receive the server's public certificate.
- `leftsubnet=0.0.0.0/0`: Tells clients the subnets reachable behind the server.

Include these in the file as shown:

```bash
left=%any
leftid=@server_domain_or_IP
leftcert=server-cert.pem
leftsendcert=always
leftsubnet=0.0.0.0/0
```

**Note:** For `leftid`, use `@` only if identifying the server by a domain name. Use the IP address directly if identifying by IP.

Configure the client's "right" side IPSec parameters:

- `right=%any`: Accepts connections from any remote client.
- `rightid=%any`: Accepts client identities before encrypted tunnel establishment.
- `rightauth=eap-mschapv2`: Authentication method for client-server authentication.
- `rightsourceip=10.10.10.0/24`: Assigns private IP addresses to clients from this pool.
- `rightdns=8.8.8.8,8.8.4.4`: Specifies DNS resolvers for clients.
- `rightsendcert=never`: Clients don't need to send a certificate.

Add these lines:

```bash
right=%any
rightid=%any
rightauth=eap-mschapv2
rightsourceip=10.10.10.0/24
rightdns=8.8.8.8,8.8.4.4
rightsendcert=never
```

Instruct StrongSwan to request user credentials from clients upon connection:

```bash
eap_identity=%identity
```

Finally, add support for various client types by specifying key exchange, hashing, authentication, and encryption algorithms:

```bash
ike=chacha20poly1305-sha512-curve25519-prfsha512,aes256gcm16-sha384-prfsha384-ecp384,aes256-sha1-modp1024,aes128-sha1-modp1024,3des-sha1-modp1024!
esp=chacha20poly1305-sha512,aes256gcm16-ecp384,aes256-sha256,aes256-sha1,3des-sha1!
```

After ensuring all lines are correctly added, save and close the file. In `nano`, do this with `CTRL + X`, `Y`, then `ENTER`.

The final config looks like this:
```
config setup
    charondebug="ike 1, knl 1, cfg 0"
    uniqueids=no

conn ikev2-vpn
    auto=add
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes
    dpdaction=clear
    dpddelay=300s
    rekey=no
    left=%any
    leftid=@server_domain_or_IP
    leftcert=server-cert.pem
    leftsendcert=always
    leftsubnet=0.0.0.0/0
    right=%any
    rightid=%any
    rightauth=eap-mschapv2
    rightsourceip=10.10.10.0/24
    rightdns=8.8.8.8,8.8.4.4
    rightsendcert=never
    eap_identity=%identity
    ike=chacha20poly1305-sha512-curve25519-prfsha512,aes256gcm16-sha384-prfsha384-ecp384,aes256-sha1-modp1024,aes128-sha1-modp1024,3des-sha1-modp1024!
    esp=chacha20poly1305-sha512,aes256gcm16-ecp384,aes256-sha256,aes256-sha1,3des-sha1!
```


# Configuring VPN Authentication
We need to indicate the private key for the server and create list of users and credentials.
This information are stored in `ipsec.secrets` file.

```
sudo nano /etc/ipsec.secrets
```
Add these two lines.
```
: RSA "server-key.pem"
your_username : EAP "your_password"
```


# Something I learned

The certificates generated by `Certbot` belong to user `root` by default which can not be accessed by other user. 

Running `ipsec` in the foreground reviews the problem.
```bash
ubuntu@owncloud:/var/log$ sudo ipsec stop
Stopping strongSwan IPsec...
ubuntu@owncloud:/var/log$ sudo ipsec start --nofork
Starting strongSwan 5.9.5 IPsec [starter]...
00[DMN] Starting IKE charon daemon (strongSwan 5.9.5, Linux 5.15.0-1049-oracle, aarch64)
00[LIB] providers loaded by OpenSSL: legacy default
00[NET] using forecast interface enp0s6
00[LIB]   opening '/etc/ipsec.d/private/privkey.pem' failed: Permission denied
00[LIB] building CRED_PRIVATE_KEY - RSA failed, tried 11 builders
00[LIB] loaded plugins: 
```

Solution is simple. Change the ownership of the file to the group the user is in, in my case `ubuntu`, and make is readable by users in the group.
```bash
sudo chgrp ubuntu /etc/letsencrypt/live/vpn.nonaharry.com/privkey.pem
sudo chmod 0640 /etc/letsencrypt/live/vpn.nonaharry.com/privkey.pem
```


# Reference 
Thanks to these awesome articles.
- [How to Set Up an IKEv2 VPN Server with StrongSwan on Ubuntu 22.04 | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-ikev2-vpn-server-with-strongswan-on-ubuntu-22-04#step-5-configuring-vpn-authentication)
- [How to Setup IKEv2 VPN Using Strongswan and Let's encrypt on CentOS 7 (howtoforge.com)](https://www.howtoforge.com/tutorial/how-to-setup-ikev2-vpn-using-strongswan-and-letsencrypt-on-centos-7)