---
layout: post
title: Headless Windows 10 VM and Guacamole
---

I had need for a Windows installation to process some old Outlook mail
archives. The archive is quite large, so instead of moving the archives to a VM
on my laptop, I decided to create a Windows 10 VM on the Ubuntu 18.04 server
where the archives reside. I wanted to try to install the machine headless to
VirtualBox and setup Guacamole to access it over a web server. This is the first
time I have installed an OS that require a GUI installation process to
VirtualBox without having direct access to the VirtualBox GUI.

### Preparing the VM

I start by installing VirtualBox. We also need the VirtualBox Extension Pack to
enable RDP access. And finally we need the VirtualBox Guest Additions ISO.
{% highlight bash %}
apt-get install virtualbox virtualbox-ext-pack virtualbox-guest-additions-iso
{% endhighlight %}

We [create](https://www.virtualbox.org/manual/ch08.html#vboxmanage-createvm) a
VM definition with a name and an ostype. We also
[register](https://www.virtualbox.org/manual/ch08.html#vboxmanage-registervm) it
with VirtualBox right away.

{% highlight bash %}
VBoxManage createvm \
--name "Windows 10 mail" \
--ostype Windows10_64 \
--register
{% endhighlight %}

We then set all the settings for the VM. Please refer to the [VirtualBox
manual](https://www.virtualbox.org/manual/ch08.html#vboxmanage-modifyvm) to get
a complete description of all the settings. Remember to update the '--memory'
and '--cpus'. Some especially interesting settings here are the network setup:
'--nic1 bridge' and '--bridgeadapter1 eno1' adds a bridged network to the hosts
network adapter 'eno1'. You'll likely need to change that to the correct name
for your network adapter. (Try running 'ip addr' to get a list of all network
adapters.)

{% highlight bash %}
VBoxManage modifyvm "Windows 10 mail" \
--memory 16384 \
--cpus 4 \
--acpi on \
--pae on \
--hwvirtex on \
--nestedpaging on \
--boot1 dvd \
--boot2 disk \
--nic1 bridged \
--bridgeadapter1 eno1
{% endhighlight %}

To access the VMs over RDP, we also enable the VirtualBox Remote Desktop
Extension with '--vrde on'. It needs some authentication backend. We use
'VBoxAuthSimple'. It is enabled with '--vrdeauthtype external --vrdeauthlibrary
VBoxAuthSimple'. 'VBoxAuthSimple' allow us to write RDP usernames and
password hashes to the 'extradata' section of each VM instead of relying on an
external service. See the [RDP
Authentication](https://www.virtualbox.org/manual/ch07.html#vbox-auth) section
of the VirtualBox manual for an in-depth explanation.

{% highlight bash %}
VBoxManage modifyvm "Windows 10 mail" \
--vrde on \
--vrdeauthtype external \
--vrdeauthlibrary VBoxAuthSimple
{% endhighlight %}

To generate a hash of the password, run the following, where you substitute
'secret' with a proper password. I generate a random password with 'pwgen'. Save
the password for later. It needs to be entered in the Guacamole GUI. Tip you'll
properly want to prefix the following two commands with a space to [prevent them
from being saved to you bash
history](https://stackoverflow.com/questions/6475524/how-do-i-prevent-commands-from-showing-up-in-bash-history).

{% highlight bash %}
VBoxManage internalcommands passwordhash "secret"
{% endhighlight %}

Take the output from the above command and replace the hash below. It will write
a key-value pair in 'extradata' of the username 'guacamole-service-account' and
password for 'VBoxAuthSimple' to read.

{% highlight bash %}
VBoxManage setextradata "Windows 10 mail" "VBoxAuthSimple/users/guacamole-service-account" \
2bb80d537b1da3e38bd30361aa855686bde0eacd7162fef6a25fe97bf527a25b
{% endhighlight %}

Create a storage controller, a virtual system disk and add the disk to the
controller. Remember to change the path and maybe adjust the size.

{% highlight bash %}
VBoxManage storagectl "Windows 10 mail" \
--name "SATA" \
--add sata
VBoxManage createhd \
--filename /my/path/windows-10-mail.vdi \
--size 200000
VBoxManage storageattach "Windows 10 mail" \
--storagectl "SATA" \
--port 0 \
--device 0 \
--type hdd \
--medium /my/path/windows-10-mail.vdi
{% endhighlight %}

[Download the Windows iso from
Microsoft](https://www.microsoft.com/software-download/windows10ISO) and place
it somewhere on the host. Attach it to the VM with the following.
{% highlight bash %}
VBoxManage storageattach "Windows 10 mail" \
--storagectl "SATA" \
--port 1 \
--device 0 \
--type dvddrive \
--medium /my/path/Win10_1909_EnglishInternational_x64.iso
{% endhighlight %}

Also attach the VirtualBox Guest Additions ISO.
{% highlight bash %}
VBoxManage storageattach "Windows 10 mail" \
--storagectl "SATA" \
--port 2 \
--device 0 \
--type dvddrive \
--medium /usr/share/virtualbox/VBoxGuestAdditions.iso
{% endhighlight %}

If you are not already doing so, you'll properly want to use 'screen' or 'tmux'
for the next part as the command keeps running in the foreground and you don't
want it to exit if you lose your 'ssh' connection.

Start the VM in the foreground, so we can see the output:

{% highlight bash %}
VBoxHeadless --startvm "Windows 10 mail"
{% endhighlight %}

If you see 'VRDE server is listening on port 3389.' in the output, it works as
expected.

### Installing Guacamole

I run Guacamole in Docker. Refer to the [Docker
documentation](https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-using-the-repository)
for an installation instruction.

Due to the [GUACAMOLE-962](https://issues.apache.org/jira/browse/GUACAMOLE-962)
bug in '1.1.0'. I run version '1.0.0'. It should be fixed in the upcoming
'1.2.0' release.

Create a 'docker-compose.yml' file somewhere on your system containing the
following:

{% highlight yaml %}
version: '3'

services:
guacamole-guacd:
image: guacamole/guacd:1.0.0
restart: unless-stopped

guacamole-db:
image: postgres:12
env_file:
- guacamole-db.env
restart: unless-stopped
volumes:
- ./10-init.sql:/docker-entrypoint-initdb.d/10-init.sql:ro
- guacamole-db:/var/lib/postgresql/data:rw

guacamole-app:
image: guacamole/guacamole:1.0.0
environment:
GUACD_HOSTNAME: guacamole-guacd
POSTGRES_HOSTNAME: guacamole-db
env_file:
- guacamole-db.env
depends_on:
- guacamole-guacd
- guacamole-db
# ports:
# - 8080:8080
restart: unless-stopped

volumes:
guacamole-db:
{% endhighlight %}

Unfortunately, the official Guacamole docker images does not support [Guacamoles
default
authentication](http://guacamole.apache.org/doc/gug/configuring-guacamole.html#basic-auth).
We are forced to use a database as the authentication backend. I use the
[PostgreSQL
authentication](http://guacamole.apache.org/doc/gug/guacamole-docker.html#guacamole-docker-postgresql).
To connect the application to the database make a 'guacamole-db.env' file with
the following content. You should update the password to something unique. Use
'pwgen' again.

{% highlight env %}
POSTGRES_USER=guacamole_user
POSTGRES_PASSWORD=secret
POSTGRES_DATABASE=guacamole_db
POSTGRES_DB=guacamole_db
{% endhighlight %}

'POSTGRES_{USER,PASSWORD,DATABASE}' is used by the guacamole docker image.
'POSTGRES_{USER,PASSWORD,DB}' is used by the [postgres docker
image](https://hub.docker.com/_/postgres). Beware that the role defined in
'POSTGRES_USER' created by the postgres docker image gets the 'SUPERUSER' [role
attribute](https://www.postgresql.org/docs/12/role-attributes.html). This is a
potential security issue. The proper way to do it is to create a non-'SUPERUSER'
role and grant it the privileges it needs.

The initialization of the database tables is a bit cumbersome. Unfortunately,
the Guacamole docker image does not do it by itself. You need to run the
following to get a script.
{% highlight bash %}
docker run --rm guacamole/guacamole:1.0.0 /opt/guacamole/bin/initdb.sh --postgres > 10-initdb.sql
{% endhighlight %}
This script can then be mounted into the 'postgres' container.

You'll likely want this behind a reverse proxy. Refer to the [Proxying
Guacamole](https://guacamole.apache.org/doc/gug/proxying-guacamole.html) section
in the Guacamole documentation. I use [Traefik](https://traefik.io/). You can
also uncomment 'ports: - 8080:8080' in 'docker-compose.yml' to bind it to port
8080 of the host.

When you have the parts in place, you can run
{% highlight bash %}
docker-compose up -d
{% endhighlight %}
to start the services.

You should not be able to reach Guacamole at
'http:<ip-of-your-server>:8080/guacamole' (depending on your reverse proxy
setup) and login with 'guacadmin:guacadmin'. See [Verifying the Guacamole
install(https://guacamole.apache.org/doc/gug/guacamole-docker.html#verifying-guacamole-docker).

You should then change the password.

### Connecting Guacamole to VirtualBox

In the Guacamole web UI, navigate to 'Settings' -> 'Connections' and press the
'New connection'-button. Add a name like "Windows 10 mail VirtualBox". Change
the protocol to 'RDP'.

Fill out the following fields under 'PARAMETERS':

* Network
* Hostname: the IP-address of your host machine where VirtualBox runs in the
'Hostname'-field. (You cannot put 'localhost' here as it refers to the
'guacd'-container.)
* Port: '3389'
* Authentication
* Username: 'guacamole-service-account'
* Password: The password you put into 'VBoxManage internalcommands
passwordhash' above.

Save the connection. Return to the Guacamole home screen and press the new
connection. The Windows installation should now appear on the screen.

### Install Windows

Complete the Windows installation. In this stage my mouse was not aligned
properly. You can use the keyboard or just keep an eye on the "Windows mouse".

When you are done with the installation, open the disk drive containing the
VirtualBox Guest Additions. Install it and reboot. The mouse should now align properly.

### Optional: Connecting Guacamole to Windows directly

While using the RDP from VitualBox works, you'll have a better experience by
connecting from Guacamole to the Windows RDP server directly. The Windows RDP
server is only in Windows 10 Pro, not in Windows 10 Home.

First make sure the user account have a password set.
Second you'll properly want to make sure the Windows VM have a static IP-address.

Third go to Settings->System->Remote Desktop and Enable Remote Desktop.

In Guacamole add a second connection. Add a name like "Windows 10 mail Windows". Change
the protocol to 'RDP'.

Fill out the following fields under 'PARAMETERS':

* Network
* Hostname: the IP-address of the Windows VM
* Port: '3389'
* Authentication
* Username: Name of your Windows account
* Password: The password of your Windows account
* Security mode: 'NLA'
* Ignore server certificate: Checked

Save the connection. Return to the Guacamole home screen and press the new
connection. This connection should be faster and stretch to fill the whole
window.

### Clean up

Finally, there are some clean up needed.

Start by shutting down the Windows machine. The 'VBoxHeadless' should now exit
in the terminal.

Remove the Windows install ISO and the VirtualBox Guest Additions ISO.
{% highlight bash %}
VBoxManage storageattach "Windows 10 mail" \
--storagectl "SATA" \
--port 1 \
--device 0 \
--medium none
VBoxManage storageattach "Windows 10 mail" \
--storagectl "SATA" \
--port 2 \
--device 0 \
--medium none
{% endhighlight %}

At last start the VM in the background this time:
{% highlight bash %}
VBoxManage startvm "Windows 10 mail" --type headless
{% endhighlight %}
