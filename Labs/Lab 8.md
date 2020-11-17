# COSC 419: Topics in Computer Science
# Fall 2020 - Lab 8

In this lab, we'll be securing access to our servers and learning how to use monitoring software packages to help deal with bad actors automatically.

## Table of Contents
- [Changing Your SSH Port](#ssh-port)
- [Installing Fail2Ban](#fail2ban)
- [Intro to Mod-Security](#mod-security)

<a name="ssh-port"></a>
## Changing the SSH Port (2 marks)

Port 22 is considered the standard port for a Secure Shell (SSH) connection. Because of this, it is extremely common for automated brute forcing tools to simply default to a port 22 connection when trying to perform an SSH intrusion. By changing the port to a non-standard port number, we can reduce the number of automated intrusion attempts that we have to deal with.

First, edit your ```sshd``` configuration file, which is located at ```etc/ssh/sshd_config```. You should see a line that says ```#Port 22```. Uncomment the line by removing the '#' character, and change 22 to 888. Port 888 is dedicated to a couple different services, but we aren't using any of them, so this port should be okay to use. After you're done, save and close the file.

Now, we just need to inform the SELinux system that we have a new SSH port to manage:

	semanage port -a -t ssh_port_t -p tcp 888
	
Finally, we'll restart the sshd service to enable our changes: ```systemctl restart sshd```. Open a new PuTTY or SSH terminal. Try reconnecting via port 22; you should receive an immediate connection closure. Then, try connecting using port 888 - you should be able to log in as previously. Do NOT exit your first SSH session until you've confirmed you can connect on the new port.

<a name="fail2ban"></a>
## Installing Fail2Ban (6 marks)

While changing our SSH port will cut down on most automated intrusion attempts, it is still possible to identify the SSH port via a port scan, and then perform a bruteforce attack. For this reason, we'll install and configure Fail2Ban, a system that identifies and bans malicious IPs.

First, we'll install and enable the EPEL release package, if we don't already have it enabled:

	yum install epel-release
	
Then, we can go ahead and install the ```fail2ban``` package, and enable it, so that it'll start each time your server restarts.

	yum install fail2ban
	systemctl enable fail2ban
	
If you receive an error saying that the Fail2Ban package is not available, double check that you have properly installed ```epel-release``` package as noted above.

Now that Fail2Ban is installed, we can configure it. Referring back to the example slides from the lecture, create the configuration file ```/etc/fail2ban/jail.local``` to do the following:

* Ban malignant users for five minutes (300 seconds)
* Use a finding time frame of five minutes
* Allow up to five attempts before banning (Client should be banned on the fifth failed login)
* Use the iptables-multiport banaction
* Enable the [sshd] jail
* Use port 888 for the [sshd] jail

Once you're done with the configuration file, save and close it, then restart the Fail2Ban service. Fail2Ban should now enabled and enforcing. Test this by trying to log in to your server via SSH and entering an incorrect password five times. Any further connection attempts should result in being unable to connect to the server, until the 5 minute cooldown is finished.

While the 5 minute cooldown strikes a good balance between punishing an automatic bot while allowing a legitimate, but error-prone user to still access the system after a short while. However, we should also have a long-term banning option for users who prove to be a nuisance.

In order to combat this, we'll create a second Fail2Ban jail. This jail will track number of failed attempts over a longer period of time, and place a longer ban on users who are making excessive failed attempts throughout the day.

First, we'll re-open the ```/etc/fail2ban/jail.local``` file. Then, below the *[sshd]* entry in the file, add an entry called *[ssh-longterm]*. For this jail, assign the following values:

* Ban malignant users for 7 days
* Use a finding time frame of 24 hours
* Allow up to twenty attempts before banning
* Enable the [ssh-longterm] jail
* Use port 888 for the [ssh-longterm] jail

You'll also need to a couple additional entries to your *[ssh-longterm]* jail. First, add an entry: ```filter = sshd```. This enables a more aggressive filter for parsing the SSH logs for intrusion attempts. Then, two entries: ```logpath = %(sshd_log)s``` and ```backend = %(sshd_backend)s```. These entries help Fail2Ban identify the location of the SSH log files and backend for retrieving login data.

Now, restart your Fail2Ban service, and run the following command: ```fail2ban-client status```. If everything went correctly, it should return that you have two jails:

<img src="https://i.imgur.com/MpSSYxP.png" width="50%"/>

<a name="mod-security"></a>
## Getting Acquainted with Mod-Security (4 marks)

ModSecurity is an open-source software package that provides real-time web firewall protection. ModSecurity is a rules-based system that checks incoming web requests for potential suspicious activity by matching against a large set of rules.

In this exercise, we'll update our ModSecurity install with the latest release of the OWASP CRS ruleset, which is considered a standard for the ModSecurity package.

First, we move to our Apache ModSecurity configuration folder:

	cd /etc/httpd/modsecurity.d

Now, we can use Git to clone the latest rules into this folder:

	yum install git
	git clone https://github.com/SpiderLabs/owasp-modsecurity-crs.git
	
Rename the ```crs-setup.conf.example``` to ```crs-setup.conf```. Move this file to your Apache ```conf.d``` folder, which is located at ```/etc/httpd.conf.d```.

Then, move all the files from the ```/etc/httpd/modsecurity.d/owasp-modsecurity-crs/rules``` directory into the ```/etc/httpd/modsecurity.d/activated_rules``` folder.

At this point, we've loaded the OWASP CRS ruleset into our ```activated_rules``` folder, and we've moved our configuration file into the ```conf.d``` folder, so Apache should now be able to use our OWASP ruleset. Restart Apache now.

If you visit your website now, you'll probably notice that every link now takes you to an error page. This is because ModSecurity rules usually err on the side of being too strict, and something about our requests (quite possibly a session token) is triggering the OWASP rulesets. In order to figure out what's going on, we'll go to ```/etc/httpd/conf.d/mod_security.conf```.

Find the line that says ```SecRuleEngine On``` near the top of the file. Change this to ```SecRuleEngine DetectionOnly```. Now, ModSecurity will log when rules are broken, but not block or drop the connection. If you can't get an error, try making a GET request for a page, with the following GET query string:

	http://<my IP address>?q=' OR 1=1--

You can also try a XSS-style exploit:

	http://<my IP address>?q=<script>alert("hello")</script>
	
Both of these should result in a 403 Forbidden error message, as ModSecurity will catch the attempted SQLi/XSS attack, and return an error message instead.

Now, if we go to our log at ```/etc/httpd/logs/modsec_audit.log```, we should be able to scroll to the bottom of the file and find our rule that is causing issues. In my case, the rule is:

	REQUEST-920-PROTOCOL-ENFORCEMENT
	
Which appears to be tripping because we're asking for an IP address instead of a domain name. We'll disable it for now. Go to ```/etc/httpd/modsecurity.d/owasp-modsecurity-crs/rules```, and find the corresponding rule file:

	REQUEST-920-PROTOCOL-ENFORCEMENT.conf
	
Then move it to a new filename so that it isn't picked up by the Apache config:

	mv REQUEST-920-PROTOCOL-ENFORCEMENT.conf REQUEST-920-PROTOCOL-ENFORCEMENT.conf.disable
	
Now, go back to your ```/etc/httpd/conf.d/mod_security.conf```, change the SecRuleEngine back to ```On```, and then restart Apache. You should now be able to visit web pages again without getting an error.

Repeat the above until your application works properly and does not trigger any errors during normal use. You'll have to keep going back to the ```modsec_audit.log``` file to figure out what's breaking ModSecurity's rules.

