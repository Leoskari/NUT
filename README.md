# Preparing Raspbian

###### The RPi will try to get an IP adress from your DHCP server by default, so either check your DHCP leases to get the IP or simply connect to the host by the hostname “raspberrypi” the default username/password is pi/raspberry.

ssh pi@raspberrypi

The first thing you should do is to change the default password, so enter the following command and follow the instructions:

passwd

The following step is not required and you will have to prefix most commands with “sudo” from now on if you skip it. The root account is disabled by default on Raspbian but I prefer to have it enabled, so:

sudo passwd root

We also have to make it possible to log in as root over ssh so edit /etc/ssh/sshd_config and search for PermitRootLogin and change it to yes. Then restart the ssh daemon using:

sudo service ssh restart

log out and log back in using:

ssh root@raspberrypi

 

It is time to apply any patches so type in:

apt update && apt upgrade -y

nano should be installed by default but I prefer to use vim to edit config files, so:

apt install vim

We want to access this server using a static IP. This is configured in /etc/dhcpcd.conf add a section like this at the bottom of the file:

interface eth0
  static ip_address=192.168.0.13/24
  static routers=192.168.0.1
  static domain_name_servers=192.168.0.1

Adjust the parameters according to your network and save the file

Now hook up the UPS to the RPi using the USB cable and reboot the RPi.

Install NUT

ssh as root to the RPi and install the NUT-server and NUT-client:

apt install nut

Verify that the UPS is visible on the USB interface using the command:

lsusb

This should return something like this:

Bus 001 Device 004: ID 051d:0003 American Power Conversion UPS
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp. SMSC9512/9514 Fast Ethernet Adapter
Bus 001 Device 002: ID 0424:9512 Standard Microsystems Corp. LAN9500 Ethernet 10/100 Adapter / SMSC9512/9514 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

Configure NUT

The first file to edit is /etc/nut/ups.conf Add the following section to the bottom:

[ups]
        driver = usbhid-ups
        port = auto
        desc = "APC Smart-Ups X 750i"

Within the bracket, you can set your UPS name (no space allowed) but keep the name “ups” for easier usage with Synology DSM.

Test the UPS driver by running:

upsdrvctl start

This will return something similar to the below depending on your UPS model and if not, a reboot usually does the trick to get the UPS to play along:

Network UPS Tools - UPS driver controller 2.7.2
Network UPS Tools - Generic HID driver 0.38 (2.7.2)
USB communication driver 0.32
Using subdriver: APC HID 0.95

Upsmon and upsd

The next step is to configure upsmon and upsd of which the later communicates with the UPS driver configured while upsmon monitors and communicates shutdown procedures to upsd. NUT allows multiple instances of upsmon to run on different machines while communicating with the same physical UPS.

For upsd to be accessible via the network we edit /etc/nut/upsd.conf

Uncomment the LISTEN directive for localhost (127.0.0.1) and add another LISTEN directive for the static IP we assigned to the RPi earlier.

LISTEN 127.0.0.1 3493
LISTEN 192.168.0.13 3493

We will also need to add some users to manage access to upsd by editing the upsd users config file /etc/nut/upsd.users and adding the following:

[admin]
        password = hunter2
        actions = SET
        instcmds = ALL
[upsmon_local]
        password  = hunter2
        upsmon master
[upsmon_remote]
        password  = hunter2
        upsmon slave
[monuser]        #This is what Synology DSM expects
        password  = secret   #Leave this here.
        upsmon slave

Then we edit /etc/nut/upsmon.conf and add the UPS to be monitored and user credentials for upsd in the MONITOR section:

MONITOR ups@localhost 1 upsmon_local hunter2 master

And finally edit /etc/nut/nut.conf and set the value for MODE equal to 'netserver' without any spaces before and after the = sign:

MODE=netserver

Verify the configuration

reboot the RPi and verify that the nut-server and local nut-client services are up

service nut-server status
service nut-client status

test the configuration using the following command:

upsc ups

This should produce something like this:

root@raspberrypi:~ # upsc ups
Init SSL without certificate database
battery.charge: 100
battery.charge.low: 10
battery.charge.warning: 50
battery.runtime: 33046
battery.runtime.low: 150
battery.type: PbAc
battery.voltage: 52.4
battery.voltage.nominal: 48.0
device.mfr: American Power Conversion
device.model: Smart-UPS X 750
device.serial: AS1035120444
device.type: ups
driver.flag.pollonly: enabled
driver.name: usbhid-ups
driver.parameter.pollfreq: 30
driver.parameter.pollinterval: 2
driver.parameter.port: auto
driver.version: 2.7.2
driver.version.data: APC HID 0.95
driver.version.internal: 0.38
ups.beeper.status: enabled
ups.delay.shutdown: 20
ups.firmware: COM 03.6 / UPS 03.6
ups.mfr: American Power Conversion
ups.mfr.date: 2010/08/24
ups.model: Smart-UPS X 750
ups.productid: 0003
ups.serial: AS1035120666
ups.status: OL
ups.timer.reboot: -1
ups.timer.shutdown: -1
ups.vendorid: 051d

Now you can continue adding NUT-clients on your network, and on the clients set nut.conf MODE=netclient and upsmon.conf to:

MONITOR ups@192.168.0.13 1 upsmon_remote hunter2 slave

Congratulations. Your NUT server is now officially running!

Web monitoring

You can optionally install a simple web GUI to monitor and control the UPS. This will require a web server on the RPi so we will begin by installing Apache but you can really use whatever cgi-capable web server you want. On the RPi:

apt install apache2

Install the nut-cgi package:

apt install nut-cgi

To monitor the UPS via the web CGI script, add the following line to /etc/nut/hosts.conf:

MONITOR ups@localhost "Local UPS"

Enable CGI support in apache:

a2enmod cgi

Restart Apache:

service apache2 restart

You can now access the web UI via: (http://192.168.0.13/cgi-bin/nut/upsstats.cgi)

Upsset will not run until you convince it that your CGI directory has been secured. You should therefore secure this directory according to the instructions in upsset.conf (outside of the scope of this tutorial) and when you’re done, uncomment the following line in /etc/nut/upsset.conf:

### I_HAVE_SECURED_MY_CGI_DIRECTORY

This will allow you to log in to http://192.168.0.13/cgi-bin/nut/upsset.cgi using the admin user/password we configured in /etc/nut/upsd.users. You will be able to view and set options on your UPS if this is supported by your ups.
