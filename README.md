# General
This how to shows the installation of a keyhelp environment on an arm64 based server/system. In this special case it is for the oracle cloud free arm64 server. A description how to create a free tier virtual machine can be found here: https://cohost.org/awakecoding/post/384627-free-arm-server-with

As my system is in german I translated some names into english for you. So if a name is not matching, it should at least be a little bit like it :-).

# Mailserver
Oracle cloud deactivated the smtp sending by default. I think this is a security feature or Oracle to prevent spamming and phishing with their ip addresses. So in general it's good, but if you want to use the mailserver it is bad. (source: https://docs.oracle.com/en-us/iaas/Content/Network/Troubleshoot/vcn_troubleshooting.htm)

To be able to have a mailserver running, you need to follow the steps below. But as they are not really easy and need a lot of contact with the customer support I recommend to spend some bucks on a hosted mailserver (or self hosting it). I personally can recommend Mailcow. It can be self hosted (not in Oracle free tier) or you can pay for hosting: https://mailcow.email/

## reserver ip address
1. go to "Compute" -> "Instances" and open the details of the server
2. on the left choose "Attached VNICs" and open the details of your VNIC
3. now choose on the left "IPv4 Address", this shows you now your IPv4 Address
4. klick on the "..." on the right of the IPv4 Addresses entriy
5. Choose "No public IP"
6. go to "Networking" -> "IP Management" -> "Reserved Public IPs" (open it in a new tab)
7. click on "Reseve Public IP Address"
    - Reserved Public IP Address Name: <choose anything you like>
    - Create in Compartment: <the one your servers uses, in most cases you only have one>
    - IP Address Source in ...: <leave empty>
    - click "Reserve Public IP Address"
8. go back to the tab of your servers IPv4 Addresses (or follow step 1-4 again)
9. Now choose:
    - Public IP Type: Reserved public IP
    - Check "Select Existing Reserved IP Address"
    - Check if the new IP Address is selected below

Now your server has a new public IP address which is not changing.

## reverse ip lookup (PTR)
To be able to be allowed to send mails, you need to create a MX entry in the DNS. This looks up your domain like "srv.domain.tld" to your IP address like "10.11.12.13". But it also needs to lookup the other way around. If you check your public IP Address "10.11.12.13", it needs to lookup to "srv.domain.tld". If not a lot or all mailservers are not accepting mails from you.

But you have no option to enable or set it on your own. For this you need:
1. login to your cloud account
2. On the top right click the question icon and choose "Request a limit increase"
3. Nicely ask the support staff to change the reverse lookup to the hostname you need

## enable smtp sending
As written in the top part, smtp sending is disabled by default. So if you try to send to someone, Oracle is blocking this connection. This results in my case in connection timeouts.

The only thing you can do is:
1. login to your cloud account
2. On the top right click the question icon and choose "Request a limit increase"
3. Nicely ask the support staff to enable smtp sending

# prepare your server for keyhelp
## create root password and login to root
```bash
sudo passwd
su -
```

## unminimize system to have all the default stuff (this takes some time)
```bash
unminimize
```
You need to unminimize which takes about 1gb more space. If you do not unminimize it, a lot of things are missing which are needed for keyhelp (from software, features till log - yes by default there is no /var/log ...). Installing with the oracle minimized system, keyhelp is not going to work. Even if you unminimize after the installation things are not working as expected.

## update server (and some optional installations)
### update everything
```bash
apt update
apt upgrade
```

### (optional) install aptitude
I personally like aptitude more than apt. This is how to install it:
```bash
apt install aptitude
aptitude update
aptitude safe-upgrade
```

### (optional) install additional tools
```bash
apt install multitail iptraf-ng
```

## reboot
```bash
shutdown -r now
```

## install keyhelp (see [https://www.keyhelp.de/](https://www.keyhelp.de/en/#install))
```bash
su -
cd
wget https://install.keyhelp.de/get_keyhelp.php -O install_keyhelp.sh ; bash install_keyhelp.sh ;
```

## check installation
Do this before you add or change anything in keyhelp. The main reason is that most errors can only be fixed by a reinstall which is clearing all keyhelp is knowing (but keeping stuff you already added). So if you added your domain and then reinstall keyhelp, keyhelp is complaining that it already exists. Then you need to remove it on your own to be able to add and manage it again with keyhelp (same for other parts like users).

The log file I think should be accessable using:
    - shell: `/var/log/keyhelp/`
    - keyhelp: simply login and it should display that there were errors, you should be able to view the log in the ui

After that you can simply re-run the installation script. It should be inside your `/root/` folder. But you can also simply rerun the step `install keyhelp` from above.

# (optional) install webmin
Some stuff was translated from german to english, so some names could be a little bit different.

## default installation
1. go to https://webmin.com/download.html and copy the Debian *.deb file download link
2. be root: `su -`
3. install requirements: `apt install libauthen-pam-perl libio-pty-perl`
4. download the file: `wget -O webmin.deb https://prdownloads.sourceforge.net/webadmin/webmin_2.010_all.deb` (replace with the copied download link)
5. install webmin: `dpkg -i webmin.deb`
6. if there are missing dependencies install them as in step 3
7. login to keyhelp and add your webmin port to the firewall, by default TCP port 10000

## valid ssl certificate (recommended)
In this case we use "webmin.domain.tld" as subdomain for the ssl certificate.

1. login to keyhelp and create a subdomain which your webmin account should use:
    - Allgemein:
        - Domain-Typ: Subdomain
        - Domain-Name: webmin.domain.tld
        - Domain-Destination: redirection (because webmin does not like a proxy in between)
        - Destination-Address: https://webmin.domain.tld (if you change the webmin port later, make sure to change the redirection)
    - Sicherheit:
        - SSL/TLS: Let's-Encrypt
        - Force secure connections: true
2. after the subdomain was created the certificates are ready and can also be used by webmin
3. login to the shell and become root: `su -`
4. stop webmin: `service webmin stop`
5. open the webmin server configuration in your favorite editor: `/etc/webmin/miniserv.conf`
6. make sure to have the following two config entries, also replace `webmin.domain.tld` with the created subdomain and `keyhelpuser` with the owner username: 
    - Before applying them make sure you have the correct path!
    - `keyfile=/etc/ssl/keyhelp/letsencrypt/keyhelpuser/webmin.domain.tld/private.pem`
    - `certfile=/etc/ssl/keyhelp/letsencrypt/keyhelpuser/webmin.domain.tld/cert.pem`
7. start webmin and test if the page is now working with a valid certificate by opening the subdomain in your browser.

If it is not working:
- check if the files you specified are really working, you can simply do that by copying the path and doing a cat, example: `cat /etc/ssl/keyhelp/letsencrypt/keyhelpuser/webmin.domain.tld/private.pem`
- check the journal: `journalctl -u webmin.service` (you can close it by pressing `q` or `:` and then enter a `q`)

## (optional but recommended) change port
Lets say we want to change it to port 23423
1. login to keyhelp and open port 23423 TCP
2. login to webmin: "Webmin" -> "Webmin-Configuration" and choose "Ports and Address"
    - in the list at the top change from 10000 to 23423
    - i also recommend to NOT watch for broadcasts

## (optional) language
1. go to "Webmin" -> "Webmin-Configuration" and choose "languages and display settings"
2. you may change the default language but you should enable the browser language detection, after you changed this a simple reload should show you your prefered language

# (optional) enable IPv6
Based on:
https://www.51sec.org/2021/09/20/enable-ipv6-on-oracle-cloud-infrastructure/

## login to your cloud account
https://cloud.oracle.com/

## enable IPv6
1. go to "Networking" -> "Virtual Cloud Networks" and open your network (details)
2. on the left choose "CIDR Blocks"
3. "Add CIDR Block/IPv6 Prefix" and check "Assign an Oracle allocated IPv6 /56 prefix" and click add

## create IPv6 Subnet
1. inside the "Virtual Cloud Network Details" click "Subnets" on the left and open the details of your Subnet
2. on the left choose "IPv6 Prefixes"
3. "Add IPv6 Prefix" and "Assign an Oracle allocated IPv6 /64 prefix"

## create route to gateway
1. inside the "Virtual Cloud Network Details" click "Route Tables" on the left and open the "Default Route Table for vnc-..."
2. "Add Route Rules"
    - Protocol Version: IPv6
    - Target Type: Internet Gateway
    - Destination CIDR Block: ::/0
    - Target Internet Gateway: Internet Gateway vnc-...

## firewall rules
1. inside the subnet of your "Virtual Cloud Network Details" click on the left on "Security Lists" and choose the "Default Security List for vnc-..."
2. add "Ingress Rules"
    - Source CIDR (allow all IPv6): ::/0
    - IP Protocol: All Protocols
3. add "Egress Rules"
    - Destination CIDR (allow all IPv6): ::/0
    - IP Protocol: All Protocols

## add IPv6 to virtual machine
1. go to "Compute" -> "Instances" and open your Instance
2. on the left choose "Attached VNICs" and then open your attached VNIC
3. on the left choose "IPv6 Addresses" and click "Assign IPv6 Address"
    - Prefix: your created IPv6 prefix /64
    - IPv6 address assignment: Manually assign IPv6 addresses from prefix (to have a static IPv6 which can be added in your DNS records)
    - IPv6 Address: <choose any prefix you want, you can assign the last 4 Blocks> (examples: "c0fe", "b00b", "beef" and also up to 4: "dead:beef:cafe", "337:c0de:4:11fe")
