# TP2_2024_Network-boot
# TP2 - Boot rÃ©seau

Cette partie du projet utilisera le protocole PXE.
PXE est un protocole utilisÃ© pour lancer une VM sans disque dur via le rÃ©seau.

Pour ce faire, je vais d'abord lancer une VM :

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/rocky9"
  config.vm.hostname = "pxe.tp2.efrei"
  config.vm.network "private_network", ip: "10.1.1.2"
  config.vm.provider "virtualbox" do |vb|
    vb.name = "pxe.tp2.efrei"
  end
end
```

Et j'installerai quelques services comme :

## Installation du serveur DHCP

Le protocole DHCP sera utilisÃ© pour attribuer une adresse IP aux machines qui se connecteront au rÃ©seau :

```console
$ sudo dnf -y install dhcp-server
```

Le fichier de configuration (etc/dhcp/dhcpd.conf) ressemblera Ã  ceci :

```
default-lease-time 600;
max-lease-time 7200;
authoritative;

# PXE specifics
option space pxelinux;
option pxelinux.magic code 208 = string;
option pxelinux.configfile code 209 = text;
option pxelinux.pathprefix code 210 = text;
option pxelinux.reboottime code 211 = unsigned integer 32;
option architecture-type code 93 = unsigned integer 16;

subnet <NETWORK_ADDRESS> netmask 255.255.255.0 {
    range dynamic-bootp 10.1.1.5 10.1.1.10;
    
    # add follows
    class "pxeclients" {
        match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
        next-server 10.1.1.2;

        if option architecture-type = 00:07 {
            filename "BOOTX64.EFI";
        }
        else {
            filename "pxelinux.0";
        }
    }
}
```

Ensuite, je dÃ©marre mon serveur DHCP et j'ouvre les ports de pare-feu nÃ©cessaires :

```console
$ sudo systemctl enable dhcpd
$ sudo systemctl start dhcpd

$ sudo firewall-cmd --add-service=dhcp --permanent
$ sudo firewall-cmd --reload
```

## Installation du serveur TFTP

Le protocole TFTP sera utilisÃ© pour tÃ©lÃ©charger des donnÃ©es depuis le serveur :

```console
$ sudo dnf -y install tftp-server
$ sudo systemctl enable --now tftp.socket
$ sudo firewall-cmd --add-service=tftp --permanent
$ sudo firewall-cmd --reload
```

Ensuite, je rÃ©cupÃ¨re mon image ISO depuis mon ordinateur Ã  l'aide de la commande scp :

```console
$ scp fils@10.1.1.1:/home/t1110/Downloads/Rocky-9.3-x86_64-minimal.iso /home/vagrant/
```

Et je suivrai ces commandes :

```console
# prepare installation
$ sudo dnf -y install syslinux

# move the file pxelinux.0 to the folder that will be used by HTTP/TFTP server
$ sudo cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/

# prepare the environment
$ mkdir -p /var/pxe/rocky9
$ sudo mkdir /var/lib/tftpboot/rocky9

# adapt the path to the rocky iso on my VM
$ sudo systemctl daemon-relaod
$ sudo mount -t iso9660 -o loop,ro /home/vagrant/Rocky-9.3-x86_64-minimal.iso /var/pxe/rocky9

# take from rocky iso what's necessary
$ sudo cp /var/pxe/rocky9/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/rocky9/
$ sudo  cp /usr/share/syslinux/{menu.c32,vesamenu.c32,ldlinux.c32,libcom32.c32,libutil.c32} /var/lib/tftpboot/

# prepare boot options file
$ sudo mkdir /var/lib/tftpboot/pxelinux.cfg
```

Et je modifierai mon */var/lib/tftpboot/pxelinux.cfg/default*

```
default vesamenu.c32
prompt 1
timeout 60

display boot.msg

label linux
  menu label ^un manguito clasico tu sabe
  menu default
  kernel rocky9/vmlinuz
  append initrd=rocky9/initrd.img ip=dhcp inst.repo=http://10.0.1.2/rocky9
label rescue
  menu label ^Rescue installed system
  kernel rocky9/vmlinuz
  append initrd=rocky9/initrd.img rescue
label local
  menu label Boot from ^local drive
  localboot 0xffff
```

## Installation du serveur Apache

```console
$ sudo dnf -y install httpd
```

J'ai crÃ©Ã© */etc/httpd/conf.d/pxeboot.conf* :

```
Alias /rocky9 /var/pxe/rocky9
<Directory /var/pxe/rocky9>
    Options Indexes FollowSymLinks
    # access permission
    Require ip 127.0.0.1 10.1.1.0/24
</Directory>
```

J'ai dÃ©marrÃ© apache et j'ai ouvert les ports de pare-feu nÃ©cessaires :

```console
$ sudo firewall-cmd --add-port=80/tcp --permanent
$ sudo firewall-cmd --reload
```

## Test

Une fois toutes les Ã©tapes prÃ©cÃ©dentes terminÃ©es, je peux crÃ©er manuellement une VM qui prendra l'ISO de Rocky.

Cette machine devrait avoir la configuration suivante :
- Assez de mÃ©moire pour faire l'installation
- AmorÃ§age rÃ©seau activÃ©
- MÃªme interface rÃ©seau privÃ© que le serveur
- Mode promiscuitÃ© activÃ©

J'ai joint [ici](tcp_dump.md) une explication d'une capture TCPdump du dÃ©marrage d'une nouvelle VM avec PXE
