# Custom Builds of iPXE
For any more detailed instructions about iPXE, see http://ipxe.org   
These are custom builds of iPXE with some embebed ipxe scripts  
# Embebed Scripts
## tftpscript
Chain script from tftpf server, located at `http://${next-server}/pxe/${net0/mac}/ipxe-script`
```
#!ipxe
ifopen
isset ${net0/mac} && dhcp net0 || goto dhcpnet1
echo Received DHCP answer on interface net0 && goto proxycheck

:dhcpnet1
isset ${net1/mac} && dhcp net1 || goto dhcpnet2
echo Received DHCP answer on interface net1 && goto proxycheck

:dhcpnet2
isset ${net2/mac} && dhcp net2 || goto dhcpall
echo Received DHCP anser on infterface net2 && goto proxycheck

:dhcpall
dhcp && goto proxycheck || goto dhcperror

:dhcperror
prompt --key s --timeout 10000 DHCP failed, hit 's' for the iPXE shell; reboot in 10 seconds && shell || reboot

:proxycheck
isset ${proxydhcp/next-server} && set next-server ${proxydhcp/next-server} || goto nextservercheck

:nextservercheck
isset ${next-server} && goto netboot || goto setserv

:setserv
echo -n Please enter tftp server: && read next-server && goto netboot || goto setserv

:netboot
chain http://${next-server}/pxe/${net0/mac}/ipxe-script || prompt --key s --timeout 10000 Chainloading failed, hit 's' for the iPXE shell; reboot in 10 seconds && shell || reboot
```
## wanscript
Start Alpine Netboot located at: `http://dl-cdn.alpinelinux.org/alpine`
```
#!ipxe

set img_verify enabled ||
set console console=tty0 ||

set cmdline modules=loop,squashfs quiet nomodeset ip=dhcp ||
# set acpi acpi=force ||

set branch v3.12 ||
set version 3.12.0 ||
set flavor lts ||
cpuid --ext 29 && set arch x86_64 || set arch x86

# Set Urls
set repo-url http://dl-cdn.alpinelinux.org/alpine/${branch}/main
set mirror https://vanaware.github.io/ipxe/alpine
set img-url ${mirror}/netboot-${version}/${arch}
set modloop-url ${img-url}/modloop-${flavor}
set sig-url ${img-url}

set ssh_key ${mirror}/ssh_key ||
isset ${ssh_key} && set start_sshd yes || set start_sshd no
iseq ${start_sshd} yes && set ssh_key ssh_key=${ssh_key} || clear ssh_key

# Network
ifopen
isset ${net0/mac} && dhcp net0 || goto dhcpnet1
echo Received DHCP answer on interface net0 && goto netboot

:dhcpnet1
isset ${net1/mac} && dhcp net1 || goto dhcpnet2
echo Received DHCP answer on interface net1 && goto netboot

:dhcpnet2
isset ${net2/mac} && dhcp net2 || goto dhcpall
echo Received DHCP anser on infterface net2 && goto netboot

:dhcpall
dhcp && goto netboot || goto dhcperror

:dhcperror
prompt --key s --timeout 10000 DHCP failed, hit 's' for the iPXE shell; reboot in 10 seconds && shell || reboot
exit 0

:netboot
imgfree
kernel ${img-url}/vmlinuz-${flavor} ${cmdline} alpine_repo=${repo-url} modloop=${modloop-url} ${console} ${acpi} ${ssh_key}
initrd ${img-url}/initramfs-${flavor}
iseq ${img_verify} enabled && goto verify_img || goto no_img_verify

:verify_img
imgtrust
imgverify vmlinuz-${flavor} ${sig-url}/vmlinuz-${flavor}.sig || goto img_verify_error
imgverify initramfs-${flavor} ${sig-url}/initramfs-${flavor}.sig || goto img_verify_error
echo Alpine Image downloaded and verified && goto boot

:img_verify_error
imgstat
prompt --key s --timeout 10000 Alpine Image not trusted, hit 's' for the iPXE shell; reboot in 10 seconds && shell || reboot
exit 0

:no_img_verify
imgtrust --allow
echo Alpine Image downloaded && goto boot

:boot
boot || prompt --key s --timeout 10000 Booting failed, hit 's' for the iPXE shell; reboot in 10 seconds && shell || reboot
exit 0
```

## lanscript
Start Alpine Netboot located at: `http://192.168.1.1/pxe/${net0/mac}/alpine-${arch}`
```
#!ipxe

set img_verify disabled ||
set console console=tty0 ||

set cmdline modules=loop,squashfs quiet nomodeset ip=dhcp apkovl=apkovl ||
# set acpi acpi=force ||

cpuid --ext 29 && set arch x86_64 || set arch x86

# Set Urls
set server http://192.168.1.1
set local-url ${server}/pxe/${net0/mac}/alpine-${arch}

set ssh_key ${server}/pxe/${net0/mac}/ssh_key ||
isset ${ssh_key} && set start_sshd yes || set start_sshd no
iseq ${start_sshd} yes && set ssh_key ssh_key=${ssh_key} || clear ssh_key

# Network
ifopen
isset ${net0/mac} && dhcp net0 || goto dhcpnet1
echo Received DHCP answer on interface net0 && goto netboot

:dhcpnet1
isset ${net1/mac} && dhcp net1 || goto dhcpnet2
echo Received DHCP answer on interface net1 && goto netboot

:dhcpnet2
isset ${net2/mac} && dhcp net2 || goto dhcpall
echo Received DHCP anser on infterface net2 && goto netboot

:dhcpall
dhcp && goto netboot || goto dhcperror

:dhcperror
prompt --key s --timeout 10000 DHCP failed, hit 's' for the iPXE shell; reboot in 10 seconds && shell || reboot
exit 0

:netboot
imgfree
kernel ${local-url}/vmlinuz-lts ${cmdline} alpine_dev=nfs:${server}:/srv/pxe/${net0/mac} modloop=${local-url}/modloop-lts ${console} ${acpi} ${ssh_key}
initrd ${local-url}/initramfs-lts
iseq ${img_verify} enabled && goto verify_img || goto no_img_verify

:verify_img
imgtrust
imgverify vmlinuz-lts ${local-url}/vmlinuz-lts.sig || goto img_verify_error
imgverify initramfs-lts ${local-url}/initramfs-lts.sig || goto img_verify_error
echo Alpine Image downloaded and verified && goto boot

:img_verify_error
imgstat
prompt --key s --timeout 10000 Alpine Image not trusted, hit 's' for the iPXE shell; reboot in 10 seconds && shell || reboot
exit 0

:no_img_verify
imgtrust --allow
echo Alpine Image downloaded && goto boot

:boot
boot || prompt --key s --timeout 10000 Booting failed, hit 's' for the iPXE shell; reboot in 10 seconds && shell || reboot
exit 0
```
## githubscript
Chain another script located at https://vanaware.github.io/ipxe/alpine/boot.ipxe
```
#!ipxe

set alpine_loader true ||
ifopen
isset ${net0/mac} && dhcp net0 || goto dhcpnet1
echo Received DHCP answer on interface net0 && goto proxycheck

:dhcpnet1
isset ${net1/mac} && dhcp net1 || goto dhcpnet2
echo Received DHCP answer on interface net1 && goto proxycheck

:dhcpnet2
isset ${net2/mac} && dhcp net2 || goto dhcpall
echo Received DHCP anser on infterface net2 && goto proxycheck

:dhcpall
dhcp && goto proxycheck || goto dhcperror

:dhcperror
prompt --key s --timeout 10000 DHCP failed, hit 's' for the iPXE shell; reboot in 10 seconds && shell || reboot

:proxycheck
isset ${proxydhcp/next-server} && set next-server ${proxydhcp/next-server} || goto nextservercheck

:nextservercheck
isset ${next-server} && goto netboot || goto setserv

:setserv
echo -n Please enter tftp server: && read next-server && goto netboot || goto setserv

:netboot
set mirror https://vanaware.github.io/ipxe/alpine
set script-url ${mirror}/boot.ipxe
chain ${script-url} || prompt --key s --timeout 10000 Chainloading failed, hit 's' for the iPXE shell; reboot in 10 seconds && shell || reboot
```