The ability to perform an ESXi Scripted Installation over the network has been a basic capability for non-Apple hardware customers since the initial release of classic ESX. However, for customers who run ESXi on Apple Mac Hardware (first introduced in vSphere 5.0), being able to remotely boot and install ESXi over the network has not been possible and customers could only dream of this capability which many of us have probably taken for granted.

Unlike traditional scripted network installations which commonly uses Preboot eXecution Environment (PXE), Apple Mac Hardware actually uses its own developed Boot Service Discover Protocol (BSDP) which ESXi and other OSses do not support. In addition, there are very few DHCP servers that even support BSDP (at least this may have been true 4 years ago when I had initially inquired about this topic). It was expected that if you were going to Netboot (equivalent of PXE/Kickstart in the Apple world) a server that you would be running a Mac OS X system. Even if you had set this up, a Netboot installation was wildly different from a traditional PXE installation and it would be pretty difficult to near impossible to get it working with an ESXi image. With no real viable solution over the years, it was believed that a Netboot installation of ESXi onto Mac Hardware just may not be possible.

tl;dr - If you are interested in the background to the eventual solution, continue reading. If not and you just want the goods, jump down a bit further. Though, I do think it is pretty interesting and worth getting the full context 🙂

It was only earlier this week in preparing for a customer call next week that I decided to re-open an email thread that I had with our Engineering folks which dates back almost 3 years ago. Although nothing had changed from the VMware side, this then lead me to check whether something had changed in the IT community via a Google search. To my surprise, I came across the following articlewhere the author claimed they were successful in Netbooting a Mac system using Linux. The biggest breakthrough for me in that article which also built upon the work from another individual here (sadly the link recently died) was the fact that they were able to get BSDP working with a standard ISC DHCP server. This was the first challenge in just getting a basic response from a Mac system, which the solution indeed provided.

My first attempt of booting ESXi onto my Mac Mini used the traditional PXE/TFTP combo and it was partially successful. It was a success in the fact that the ESXi bootloader (bootx64.efi) was actually loading, but it would consistently fail at downloading the 11th file with a Fatal 33 (Inconsistent Error). I had tried various TFTP boot options including the per_source option which supposedly lifted the default 11 file limitation, but there no change in behavior. The theory that I concluded with was that the TFTP client on the Mac system might have just been limited as I was able to successfully download the file from a Linux-based TFTP client. However, this basic test in itself was also a break through as it proved for the first time, that it might actually be possible to Netboot ESXi onto Mac hardware.

While sharing the updated news with a few of our Engineers, Tim Mann, one of our Sr. Staff Engineers at VMware provided a great pointer to the next step that I could try. Given the fact that we were able to serve the initial files via TFTP, what we could try is to boot up iPXE and then chain load that to the ESXi bootloader. A very useful resource for general PXE/UEFI installation of ESX is this whitepaper here which is also authored by Tim. This is definitely worth having around when working with scripted installs of ESXi. With a few additional tweaks to the DHCP configuration file, I was able to fully boot the ESXi installer over the network to my Mac Mini! I was literally jumping up and down in front of my wife when this happened 😀 I believe this might actually be the first time this has ever been done before! Very cool if you ask me.

I have verified that this solution works for ESXi 6.5 and ESXi 6.0 out of the box. This solution also does work for ESXi 5.5, but you will need to use a newer ESXi bootloader (bootx64.efi) from ESXi 6.0 or newer as it contains some additional fixes. I have also verified that this works with scripted installs (Kickstart) as you would expect. I think and hopefully you will agree with me that this is HUGE news for the Apple/VMware community. Historically, it has been very challenging to manage Apple Hardware as installations of ESXi were mostly done manually or at least requiring someone to physically plug in the media to the system. Not only will this benefit customers of all sizes whether you manage several dozen Mac machines to several hundred which we have quite a few customers doing, but it will actually enable customers to scale even higher and faster now. No longer is this an operational burden which is something that I hear quite frequently from customers in terms of ESXi lifecycle.

OK, so now that you have some background on how we arrived at the solution, below are the steps to set this up in your own environment.

Pre-requisite:

Ubuntu 16.0.4.1 already deployed
ESXi 6.5, 6.0 or 5.5 ISO
Apple Mac Hardware to test with. Although it didn't make difference, its worth checking that you are running latest EFI firmware on your Mac hardware.
Step 1 - Install the following packages by running the following command:

apt-get update && apt-get -y install isc-dhcp-server tftpd-hpa grub2-common grub-imageboot grub-pc-bin grub-efi wget net-tools build-essential liblzma-dev git apache2

Note: The initial reference article included several Docker Containers which I had started with to perform the initial test, but due to issues with journald not working within a Docker Container which made troubleshooting nearly impossible. I opted to just build this myself as it was pretty simple and allowed for easier debugging. I may consider building a specific Docker Container targeting the ESXi use case that folks can pull down and just use.

Step 2 - SCP the ESXi ISO to the Ubuntu system and run the following commands to extract the contents and place them in the webserver root directory:

mount -o loop VMware-VMvisor-Installer-6.5.0-4564106.x86_64.iso /mnt/
cp -rf /mnt /var/www/html/esxi65
umount /mnt
cp /var/www/html/esxi65/efi/boot/bootx64.efi /var/www/html/esxi65/mboot.efi

This will allow us to serve the ESXi bootloader from our webserver using http://[IP-OF-WEBSERVER]/esxi65/mboot.efi which will in turn load the ESXi installation files.

Step 3 - We need remove "/" (forward slash) character from ESXi's boot.cfg file by running the following command:

sed -i 's/\///g' /var/www/html/esxi65/boot.cfg

Step 4 - We need to add a prefix to the ESXi's boot.cfg to tell the Mac client where to download the ESXi installation files. To do so, add the following (replace with the IP Address of your Ubuntu system):

prefix=http://192.168.1.16/esxi65

Step 5 - When the ESXi bootloader runs, it will be looking for the boot.cfg in one of two locations. The first is in the root of your webserver /boot.cfg and the other is /[MAC-ADDRESS-OF-CLIENT/boot.cfg. The latter option allows you to control which version of ESXi is served to a particular Mac system. Assuming you would like that level of control, you will need to get the MAC Address of your Mac system prior to performing this step.

Create the following directory (replace the MAC Address with the Mac system):

mkdir -p /var/www/html/01-3c-07-54-77-af-54

Copy the boot.cfg from the specific version of ESXi that you wish to boot off of to directory you just created:

cp /var/www/html/esxi65/boot.cfg /var/www/html/01-3c-07-54-77-af-54/boot.cfg

Note: If you are lazy like me, you can skip this step and then just watch the apache logs for the failed request which will then give you the MAC Address of the system that you are trying to boot 😉

Step 6 - Next, we need to download and build iPXE from source. Run the following command to clone the ipxe Github repository:

git clone git://git.ipxe.org/ipxe.git

Step 7 - Change into the ipxe/src directory and then build the snponly.efi binary by running the following commands:

cd ipxe/src
make bin-x86_64-efi/snponly.efi

Step 8 - If you have successfully built iPXE, the following file should exists bin-x86_64-efi/snponly.efi and you will need to copy that over to your TFTP directory which by default is /var/lib/tftpboot, unless you have changed it.

cp bin-x86_64-efi/snponly.efi /var/lib/tftpboot/

Step 9 - We now need to edit our dhcpd.conf which is basically where all the magic happens. I have a fully functional version that you can use and modify or adjust your own DHCP server. To use the one I have, please run the following command to clone the repository:

git clone https://github.com/lamw/netboot-esxi

Step 10 - Copy the working sample-dhcpd.conf to DHCP configuration directory.

cp netboot-esxi/sample-dhcpd.conf /etc/dhcp/dhcpd.conf

You will still need to edit the file to adjust the networking to fit your environment starting with Line 1-2, 64-69. Line 106, 115 & 118 will also need to be updated to reflect the IP Address of your Ubuntu system. In the sample dhcpd.conf, the netboot class is what serves the initial iPXE image and then afterwards, the pxeclients class is what loads the ESXi bootloader.

Step 11 - Finally, the last step is to start both the DHCP and TFTP services by running the following commands. It is recommended that you run the status command to ensure there are no errors in your DHCP configuration file before proceeding to the TFTP service.

/etc/init.d/isc-dhcp-server start
/etc/init.d/isc-dhcp-server status
/etc/init.d/tftpd-hpa start

At this point, you are now ready to boot your Mac system! You will need a keyboard connected to your Mac as you will need to hold down the "n" character to perform a Netboot (not sure if this is the default behavior or a way to configure it to do this on-boot?). You should see a globe icon for a few seconds and then see iPXE getting booted if everything was configured correctly. This process will be quite fast, but if you look carefully, you should see iPXE now chain loading the ESXi bootloader as you can see from this semi-blurry screenshot (took me several attempts to capture this).


Once the ESXi bootloader is loaded, it will then look for the boot.cfg file that we had configured in Step 8. Once it successfully locates this file, it will know where to download the ESXi installation files which will now happen over HTTP, rather than the traditional TFTP.


This part of the boot was blazingly fast compared to a traditional PXE installation where the files were being downloaded and serve via TFTP. This definitely will speed up deployments and something to consider in general whether you are doing Netboot or PXE Boot of ESXi.

I never thought I would be so happy to see our ESXi installation screen on the Mac 😀There you have it, a successful Netboot of ESXi onto Apple Mac Hardware! The icing on the cake for me is that you do not need to have Mac OS X system to run the Netboot Server. This means for customers who are already doing PXE Installation of ESXi and other *nix OSes, they should be able to easily incorporate this into their existing infrastructure. One other thing I would like to try once I get some additional time is to see if this also works with vSphere Auto Deploy, which would also open another door for customers looking to leverage Auto Deploy and Host Profiles to easily manage and lifecycle their ESXi hosts running on both Apple and non-Apple hardware.

Scripted Installation via Kickstart
The instructions above only demonstrates how to boot an ESXi installer over the network to your Mac system. If you want to perform a scripted installation using Kickstart, you just need to simply place the Kickstart file on your webserver (I generally place them in the same path of the ESXi installation files) and replace the kerneloptsline in the boot.cfg with the path to the Kickstart file as shown below:

kernelopts=ks=http://192.168.1.16/esxi65/ks.cfg

I also have a very basic Kickstart script included in the Github repo that you had downloaded in Step 9 under netboot-esxi/sample-ks.cfg that you can use as a test.

