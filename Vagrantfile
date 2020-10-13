# -*- mode: ruby -*-
# vim: set ft=ruby :
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
:pxe => {
        :box_name => "centos/8",
        :net => [
                   {ip: '192.168.199.2', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "pxe-net"},
                ]
  },
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
        box.vm.box = boxconfig[:box_name]
        box.vm.host_name = boxname.to_s
        boxconfig[:net].each do |ipconf|
          box.vm.network "private_network", ipconf
        end
        if boxconfig.key?(:public)
          box.vm.network "public_network", boxconfig[:public]
        end
        box.vm.provision "shell", inline: <<-SHELL
          mkdir -p ~root/.ssh
          cp ~vagrant/.ssh/auth* ~root/.ssh
        SHELL
        
        case boxname.to_s
        when "pxe"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
	sudo setenforce 0
	sudo dnf install dnsmasq nginx vsftpd syslinux wget -y
	sudo mv /etc/dnsmasq.conf  /etc/dnsmasq.conf.backup
	sudo touch /etc/dnsmasq.conf && cat > /etc/dnsmasq.conf <<EOF
# DHCP Server Configuration file.
#interface=eth1,lo
#bind-interfaces
domain=otus-linux
# DHCP range-leases
dhcp-range=eth1,192.168.199.3,192.168.199.5,255.255.255.0,1h
# PXE
dhcp-boot=pxelinux.0,pxeserver,192.168.199.2
# Gateway
dhcp-option=3,192.168.199.1
# DNS
dhcp-option=6,192.168.199.1
# Broadcast Address
dhcp-option=28,192.168.199.255
# NTP Server
dhcp-option=42,0.0.0.0
enable-tftp
tftp-root=/var/lib/tftpboot

EOF
	sudo mkdir -pv /var/lib/tftpboot/pxelinux.cfg
	sudo touch /var/lib/tftpboot/pxelinux.cfg/default && cat > /var/lib/tftpboot/pxelinux.cfg/default <<EOF
default menu.c32
prompt 0
timeout 111
#ONTIMEOUT local
menu title ########## PXE Boot Menu ##########
label 1
menu label ^1) Install CentOS 8 x64 with Local Repo
kernel centos8/vmlinuz
append initrd=centos8/initrd.img inst.repo=http://192.168.199.2/pub ks=http://192.168.199.2/pub/centos8.cfg devfs=nomount ip=dhcp
label 2
menu label ^2) Install CentOS 8 x64 with basic video driver
kernel centos8/vmlinuz
append initrd=centos8/initrd.img inst.xdriver=vesa nomodeset method=http://192.168.199.2/pub devfs=nomount ip=dhcp

label 3
menu label ^3) Boot from local drive
localboot 0xffff

EOF
	sudo cp -r /usr/share/syslinux/* /var/lib/tftpboot
	sudo wget http://ftp.mgts.by/pub/CentOS/8.2.2004/isos/x86_64/CentOS-8.2.2004-x86_64-minimal.iso
	#mount ISO
	sudo mount -o loop CentOS-8.2.2004-x86_64-minimal.iso /mnt/ && cd /mnt/
	sudo cp -av * /var/ftp/pub/
	sudo mkdir /var/lib/tftpboot/centos8
	sudo cp /mnt/images/pxeboot/vmlinuz /var/lib/tftpboot/centos8/
	sudo cp /mnt/images/pxeboot/initrd.img /var/lib/tftpboot/centos8/
	sudo cp -r /mnt/*  /var/ftp/pub/
	 
	chmod -R 755 /var/ftp/pub
	sed -i 's/anonymous_enable=NO/anonymous_enable=YES/g' /etc/vsftpd/vsftpd.conf
	mv /etc/nginx/nginx.conf /etc/nginx/nginx.confback && touch /etc/nginx/nginx.conf
	cat > /etc/nginx/nginx.conf <<EOF
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    #include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server;
        # listen       [::]:80 default_server;
        server_name  192.168.199.2;
        # root         /usr/share/nginx/html;
         root /var/ftp;

        # Load configuration files for the default server block.
        # include /etc/nginx/default.d/*.conf;
        #location / {}

        location /pub/ {
        #root /var/ftp/pub;
        autoindex on;
}

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
}
EOF
		
	systemctl start dnsmasq
	systemctl enable  dnsmasq
	systemctl start vsftpd
	systemctl enable vsftpd
	systemctl start firewalld
	systemctl enable firewalld
	firewall-cmd --add-service=ftp --permanent
	firewall-cmd --add-service=dns --permanent
	firewall-cmd --add-service=dhcp --permanent
	firewall-cmd --add-port=69/udp --permanent
	firewall-cmd --add-port=4011/udp --permanent
	firewall-cmd --add-service=http --permanent
	firewall-cmd --reload
	systemctl start nginx
	nginx -s reload
	systemctl enable nginx
	sudo touch /var/ftp/pub/centos8.cfg
	sudo cat >> /var/ftp/pub/centos8.cfg <<EOF
#version=RHEL8
# Firewall configuration
firewall --disabled
# Install OS instead of upgrade
install
# Use FTP installation media
url --url=http://192.168.199.2/pub/BaseOS
# Root password
rootpw --iscrypted $1$e2wrcGGX$tZPQKPsXVhNmbiGg53MN41
# System authorization information
# auth useshadow passalgo=sha512
# Use graphical install
text
firstboot --disable
# System keyboard
keyboard us
# System language
lang en_US
# SELinux configuration
selinux --disabled
# Installation logging level
#logging level=info
# System timezone
timezone Europe/Minsk
# automatically proceed for each steps
autostep
# Partition clearing information
clearpart --all --initlabel
ignoredisk --only-use=sda
autopart --type=lvm
reboot
%packages
@Core
%end
#%addon com_redhat_kdump --disable --reserve-mb='auto'
#%end

EOF
	sed -i 's/rootpw --iscrypted/rootpw --iscrypted $1$e2wrcGGX$tZPQKPsXVhNmbiGg53MN41/g' /var/ftp/pub/centos8.conf
	echo "Now, try to build a VM in VitrualBox and put properly network in LAN (pxe-net)"
SHELL
   end
  end
 end
end

