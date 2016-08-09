# QEMUQuickStart
A super short and sharp introduction to QEMU live migration.

QEMU/KVM is like docker for virtual machines. It uses the VT-x instruction, which means you probably want to have a CPU on this list:
- http://en.wikipedia.org/wiki/List_of_Intel_microprocessors and search for 'VT'
- http://en.wikipedia.org/wiki/List_of_AMD_microprocessors and search for 'AMD-V'
- Thanks to http://serverfault.com/a/24864

Libvirt is an interface to this and other hypervisors. It will not be used here, but I might share some other useful info about it.

## Intro
This morning I decided to learn about live migration. I had played with libvirt virsh a little, and spent some time in the AQEMU ui. I searched around for a bit, but couldn't really find a short and sharp guide to learn precisely what I wanted (as is often the case for a noob with highly specific interests). After much excitement, I decided to share my findings for those who wish to experiment, but do not wish to spend hours tracking down information.

## Setup
I am using Fedora 23 and 24 on my source and target machines respectively. 
I install qemu with the following:

``` 
  dnf install qemu kvm
  dnf isntall qemu-img #if you want to create virtual drives
  dnf install libguestfs libguestfs-tools #only if you want to configure drives for your vm
```
Of course, Ubuntu users should use apt-get instead of dnf/yum. Arch users should use pacman. Windows users should get linux :P

### To create a drive
```
qemu-img create -f qcow2 mydrive.qcow2 256M #Create a 256MB hard drive in qcow2 format.
#note that you can also do the following to make a snapshot (like a delta/diff filesystem)
qemu-img create -f qcow2 -b mydrive.qcow2 -o backing-fmt=qcow2 drive2.qcow2
#you can use the guestfish commands below to add files, then create snapshots on top of the new drive and so on..
```

https://kashyapc.com/2014/07/06/live-disk-migration-with-libvirt-blockcopy/
### To set up the drive (I didn't bother for this example, since we will be live booting)
```
guestfish -a mydrive.qcow2
  >run                        #initialize things
  >part-disk /dev/sda mbr     #set up your master boot record
  >mkfs ext4 /dev/sda1        #make the filesystem. You can >exit after this.
  >mount /dev/sda1 /          #mount the filesystem
  >touch /foo                 #make a file, so that we can compare drives later.
  >ll /
  >exit
```

https://kashyapc.com/2014/07/06/live-disk-migration-with-libvirt-blockcopy/
### At this point you can perform the following to define a virsh machine (not needed)
```
cat <<EOF > testvm.xml
<domain type='kvm'>
  <name>testvm</name>
  <memory unit='MiB'>256</memory>   
  <vcpu>1</vcpu>
  <os>
    <type arch='x86_64'>hvm</type>
  </os>
  <devices>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='mydrive.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>   
  </devices>
</domain>
EOF
virsh define testvm.xml
virsh start testvm
```

https://docs.fedoraproject.org/en-US//Fedora_Draft_Documentation/0.1/html/Virtualization_Deployment_and_Administration_Guide/App_Domain_Console.html
If you get "error: internal error cannot find character device (null)" you can add between </disk> and </devices>
```
<serial type='pty'>
  <target port='0'/>
</serial>
<console type='pty'>
  <target type='serial' port='0'/>
</console>
```
then
```
virsh destroy testvm
virsh undefine testvm
```

# The good stuff

Download an ISO. I chose DSL since it is quick and small. 
```
wget http://distro.ibiblio.org/damnsmall/current/dsl-4.4.10.iso
```

You can use the following command to boot up a vm. 
qemu-system-* is the architecture to use.
-hda is your first hard drive
-m is memory in MB. 
-soundhw is for some kind of audio driver, shown for completeness (I haven't experimented)
-cdrom mounts your dsl image
-boot sets the bootable drive
-machine is similar to the arch. It should be identical between source and target machine, so it is better to define it explicitly than default.

```
qemu-system-i386 -hda mydrive.qcow2 -m 1024 -soundhw es1370 -cdrom dsl-4.4.10.iso -boot d -machine pc-i440fx-2.4
```

You should then see a window with a menubar. Use the awful default release key Ctrl+Alt+G to unlock your mouse if need be. Under View you can switch between VGA and compatmonitor0 (in my case). The former is the default showing the output of the VM (a now-booting DSL instance). The latter will allow you to talk to the underlying qemu instance. Open up a paint app or something interesting, then go to your destination PC (or open another terminal) and enter the following. The Flags should be identical, except for drives.

```
qemu-system-i386 -incoming tcp:0:4444 -m 1024 -soundhw es1370 -boot d -machine pc-i440fx-2.4
```

Your target machine is now listening for a VM. Go back to your compatmonitor0 (qemu) interface and enter the following. The :0: should be replaced with the IP of your target host if you are not using a local terminal.

```
migrate tcp:0:4444
```

You should see the session copy over in a few seconds. Amazing.

If you get the following error, it may be because you tried to send a paused VM. Go to Machine in the menu and uncheck paused (turned on when you migrate). Since this is caused by uninitialised drives, you should check that both ends have the same boot flag, since excluding boot d will cause the target to assume that the hard drive should be initialised.
```
ERROR: invalid runstate transition: 'inmigrate' -> 'postmigrate'
```

These errors can be caused by having different flags on your commands (here for google links):
```
#If you try to use a different qemu-system-* on each end:
qemu-system-x86_64: Missing section footer for cpu
qemu-system-x86_64: load of migration failed: Invalid argument

#If you have a soundhw flag on one but not the other:
qemu-system-i386: Unknown savevm section or instance 'audio' 0
qemu-system-i386: load of migration failed: Invalid argument

```

## Sources 
http://grivon.apgrid.org/quick-kvm-migration
https://kashyapc.com/2014/07/06/live-disk-migration-with-libvirt-blockcopy/
http://blog.programster.org/kvm-cheatsheet/
https://docs.fedoraproject.org/en-US//Fedora_Draft_Documentation/0.1/html/Virtualization_Deployment_and_Administration_Guide/App_Domain_Console.html
http://www.suares.com/index.php?page_id=25&news_id=214
