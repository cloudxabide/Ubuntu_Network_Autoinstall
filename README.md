# Installing Ubuntu Server from Network

## Notes
You have to use the "Server" spin

My environment was built to "Kickstart" Red Hat Enterprise Linux nodes.  Therefore, you'll see some "carry over" from that, which likely does not apply to folks who *just* want to provide Ubuntu automated network install.  

This is *just* notes at this point.  If you already know what you are doing in regards tox.
* setting up tftp,dhcp,http,pxe
* install Ubuntu

Then you can probably take my notes and run with all this. 

I do intend to refactor this to be more of a "how-to", I just don't have time right now.

## Initial Install (manual) of Ubuntu
Similar to how I learned Kickstart/Jumpstart, I will first manually build a host - and then fetch the resulting "kickstart config"  

You will need to fetch 2 files from an initial build (user-data and shimx64.efi)
```
/var/log/installer/autoinstall-user-data
find /usr -name shimx64.efi -exec ls -l {} \;
```

## User-Data Web Location
```
mkdir -p /var/www/html/Kickstart/MORPHEUS/
cd /var/www/html/Kickstart/MORPHEUS/
cp autoinstall-user-data /var/www/html/Kickstart/MORPHEUS/user-data
touch meta-data
```

## Update WebServer

| URL | Server Mount Point | Purpose |
|:-----|:---------|:----------------|
| webserver/OS |  /var/www/OS | Allows you to browse the contents of the ISO file |
| webserver/ISOS | /var/www/ISOS | Allows you to see (and download) the *.iso | 
| websever/Kickstart | /var/www/html/Kickstart | A "base" directory to store all of my Kickstart Files |


```
cat << EOF > /etc/httpd/conf.d/OS.conf 
Alias "/OS" "/var/www/OS"
<Directory "/var/www/OS">
  Options FollowSymLinks Indexes
</Directory>
EOF

cat << EOF > /etc/httpd/conf.d/ISOS.conf 
Alias "/ISOS" "/var/www/ISOS"
<Directory "/var/www/ISOS">
  Options FollowSymLinks Indexes
</Directory>
EOF

systemctl restart httpd
```

## Installation Source (ISO)
```
mkdir /var/www/ISOS/; cd $_
wget https://mirror.us.leaseweb.net/ubuntu-releases/22.04.2/ubuntu-22.04.2-live-server-amd64.iso 
mkdir -p /var/www/OS/ubuntu-22.04.2-live-server-amd64 
echo "/var/www/ISOS/ubuntu-22.04.2-live-server-amd64.iso /var/www/OS/ubuntu-22.04.2-live-server-amd64 iso9660 defaults,nofail 0 0" >> /etc/fstab
systemctl daemon-reload; mount -a
```

## EFI Shim
```
mkdir -p /var/lib/tftpboot/efi/ubuntu-22.04.2-live-server-amd64/
cp shimx64.efi /var/lib/tftpboot/efi/ubuntu-22.04.2-live-server-amd64/
```

## DHCP foo
Add the following to /etc/dhcpd/dhcpd.conf (Notice the "filename" param matches the EFI Shim section
Again, since all of my systems are RHEL based, I had to deviate a bit to accommodate the Ubuntu box.  As such, I specify a shim file *just* for this DHCP client (which, actually is not a bad approach).

```
# MORPHEUS (intel NUC - i7 (dark color, older - blue front display))
host morpheus {
  option host-name "morpheus.matrix.lab";
  hardware ethernet 94:c6:91:1c:81:f6;
  fixed-address 10.10.10.13;
  filename "efi/ubuntu-22.04.2-live-server-amd64/shimx64.efi";
}
```


## Kernel (vmlinux) and Initialized Ram Disk (initrd)
```
cd /var/lib/tftpboot/
mkdir ubuntu-22.04.2-live-server-amd64
find /var/www/OS/ubuntu-22.04.2-live-server-amd64/ -name vmlinuz -exec cp {} /var/lib/tftpboot/ubuntu-22.04.2-live-server-amd64 \;
find /var/www/OS/ubuntu-22.04.2-live-server-amd64/ -name initrd -exec cp {} /var/lib/tftpboot/ubuntu-22.04.2-live-server-amd64 \;
```

## Grub.cfg (boot menu)
This is monumentall aggravating.  I don't see why Ubuntu decided to do kickstart differently, and the challenges that decision has created is a pain.

nocloud: https://cloudinit.readthedocs.io/en/17.2/topics/datasources/nocloud.html

* The format of the first line "shebang", a.k.a. #cloud-config matters. Do not change it or input extra spaces unless you know what you are doing.
* The double quotes matter when injecting user-data in the kernel parameters.
* The file name user-data is a must. Its the file name that cloud-init will look for.
* The file meta-data is also a must, even it is an empty file in our example.
* The trailing slash of the url serving user-data matters in the current autoinstall version.

Note, you will create the following file using your own MAC address.  But, notice there is a leading "-01-" before the MAC
```
cat << EOF > /var/lib/tftpboot/efi/grub.cfg-01-##-##-##-##-##-##
set timeout=30

loadfont unicode

set menu_color_normal=white/black
set menu_color_highlight=black/light-gray

menuentry "Install Ubuntu Desktop (22.04.2)" {
        set gfxpayload=keep
        linux   /ubuntu-22.04.2-live-server-amd64/vmlinuz url=http://10.10.10.10/ISOS/ubuntu-22.04.2-live-server-amd64.iso only-ubiquity ip=dhcp autoinstall "ds=nocloud-net;s=http://10.10.10.10/Kickstart/MORPHEUS/ cloud-config-url=http://10.10.10.10/Kickstart/MORPHEUS/user-data ---
        initrd  /ubuntu-22.04.2-live-server-amd64/initrd
}

menuentry "Try or Install Ubuntu" {
	set gfxpayload=keep
	linux	/ubuntu-22.04.2-live-server-amd64/vmlinuz url=http://10.10.10.10/ISOS/ubuntu-22.04.2-live-server-amd64.iso maybe-ubiquity quiet splash --- 
        initrd  /ubuntu-22.04.2-live-server-amd64/initrd
}
grub_platform
if [ "$grub_platform" = "efi" ]; then
menuentry 'Boot from next volume' {
	exit 1
}
menuentry 'UEFI Firmware Settings' {
	fwsetup
}
else
menuentry 'Test memory' {
	linux16 /boot/memtest86+.bin
}
fi
EOF
```

## Cleanup SELinux
```
restorecon -RFvv /var/lib/tftpboot /var/www
```


## Troubleshooting
```
tcpdump -i ens192 -vvv -s 1500 '((port 67 or port 68 or port 69))' 
```

```
tail -f /var/log/httpd/access_log
```

## References
https://github.com/dannf/ubuntu-server-netboot

