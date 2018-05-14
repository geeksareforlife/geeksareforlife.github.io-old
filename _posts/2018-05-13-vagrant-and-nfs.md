---
layout: post
title: "Vagrant and NFS"
date: 2018-05-13
thumb: "/assets/images/2018/05/vagrant-thumbnail.png"
hero: "/assets/images/2018/05/vagrant-hero.png"
excerpt: "NFS can make your Vagrant box perform a lot better, but will it take long to set up?"
categories: [programming]
---
You've probably used [Vagrant](https://www.vagrantup.com), and therefore know that is can make development in small teams a lot easier by simplifying the process of getting a development environment that is the same for all team members up and running.

You'll probably also know that Vagrant can be **slow!** A lot of the time this is down to the way that Vagrant and Virtualbox work together to share your files from the host (your computer) to the guest (your dev box).

Recently, I had been trying to improve the performance of the box used in a side project. I had already improved the `vagrant up` time by using a custom base box, and if you are not doing this yet, it is one sure fire way of making your life easier. I was inspired by a [project](https://github.com/nottinghack/hms2) I work on at my local [Hackspace](http://nottinghack.org.uk/), who use their own [custom box](https://github.com/NottingHack/vagrant-hms2).

Unfortunately, whilst this improved the `up` time, it did nothing to improve the responsiveness of the box whilst it was up, and some googleing made me realise that this is due to the shared folder setup that Vagrant/Virtualbox use by default. Switching to NFS seemed to be advice, but that doesn't work on Windows... or does it?

Enter [WinNFSd](https://github.com/winnfsd/vagrant-winnfsd). This vagrant plugin allows your windows developers to use NFS in your vagrant boxes with a simple install:

```
vagrant plugin install vagrant-winnfsd
```

Enabling NFS is as easy as pie too.  First, make sure that you are using a private network, then simply change the shared folder line to:

```
config.vm.synced_folder '.', '/vagrant', type: "nfs"
```

Destroy and recreate your box and you are running on NFS! You may notice that your `up` time is shorter than normal, and the webserver on your box is super responsive. But dig a little deeper and you may hit a snag or two.

You will notice that Vagrant will install some other packages in your development box - if you are using a custom box like I recommend above, now is the time to go back to it and install those packages so that your vagrant doesn't have to do it everytime!

If you use any sort of monitoring program for your development, such as a build tool that watches a directory for changes and then triggers a build, you might see delays of up to 10 seconds before changes are noticed. If you are trying to read files in your shared directory that have changed on the development box (such as log files) then you may be waiting up to 30 seconds to see a change. Surely we can do better than this?

We can. You can pass mount options to NFS to change the way that the shared folder is mounted. I found a lot of opinions on which were the best to use, and in the end tried four different combinations.  The first one, without any mount options, was by far the best webserver performance, but the worst build tool.

```
config.vm.synced_folder '.', '/vagrant', type: "nfs", mount_options: ["lookupcache=none"]
```

This one had the best build tool performance, but the webserver was almost as bad as before we started using NFS!

```
#config.vm.synced_folder '.', '/vagrant', type: "nfs", mount_options: ['actimeo=1']
```

The build tool was ok with this, with 1-3 seconds between a change and the build system noticing, but the webserver was still pretty bad in comparison to the non mount options mode.

```
config.vm.synced_folder '.', '/vagrant', type: "nfs",  mount_options: ['rw', 'vers=3', 'tcp', 'actimeo=2']
```

This is the one that we went with in the end. Build tool performance is <1 second, and the webserver, whilst not as good as NFS without mount options, was better than the last two examples.

Whilst I was in there, I also increased the number of processors that the VM had available, and increased the memory.  I used a script I found in an excellent blog post to make the box use 1/4 of the host machines total memory - you can see this below. But **caution** - if you use more than one vagrant box on your machine at the same time, don't give them all 1/4 of your system memory, or you own performance will suffer!

```
config.vm.provider "virtualbox" do |v|
  host = RbConfig::CONFIG['host_os']

  # Give VM 1/4 system memory 
  if host =~ /darwin/
    # sysctl returns Bytes and we need to convert to MB
    mem = `sysctl -n hw.memsize`.to_i / 1024
  elsif host =~ /linux/
    # meminfo shows KB and we need to convert to MB
    mem = `grep 'MemTotal' /proc/meminfo | sed -e 's/MemTotal://' -e 's/ kB//'`.to_i 
  elsif host =~ /mswin|mingw|cygwin/
    # Windows code via https://github.com/rdsubhas/vagrant-faster
    mem = `wmic computersystem Get TotalPhysicalMemory`.split[1].to_i / 1024
  end

  mem = mem / 1024 / 4
  v.customize ["modifyvm", :id, "--memory", mem]

  v.cpus = 2
end
```

Whilst Vagrant is still not as fast as running a development enviroment yourself, this makes it work a lot better. Combine that with the convenience of `vagrant up` and it is a no-brainer!

## Errors

But what about problems?

I had quite a few problems getting NFS to work at all. At least one of these was very much related to my personal setup, but I am going to document them here in the hopes that they will help someone someday! For reference, my guest machine is Debian 9 (stable) and my host machine is Debian 10 (testing)

### Problem Installing NFS on the Guest Machine

As I mentioned above, when you `vagrant up` a NFS box, Vagrant very helpfully installs the packages required to make this work on your guest machine. Unfortunately, this wouldn't work for me, I got the error below:

```
E: Could not get lock /var/lib/dpkg/lock - open (11: Resource temporarily unavailable)
E: Unable to lock the administration directory (/var/lib/dpkg/), is another process using it?
The following SSH command responded with a non-zero exit status.
Vagrant assumes that this means the command failed!                                                                                         

apt-get -yqq update                                                                                                                         
apt-get -yqq install nfs-common portmap                                                                                                     
exit $?   
```

This error seems to imply that an installer was already running, on a freshly booted up box! And in a way it was. In building my custom box, I was using the standard Debian 9 image as my base. This has a feature that checks the package lists every day to see if any updates need installing, and then notfies the user. Because the custom box was built well over a day ago, this was happening every time the box booted, but was finished by the time my own provisioning scripts started running!

The solution is either one of the following, but I ended up doing both:

1. Install the packages `nfs-common` and `portmap` on your custom box. This is a good idea anyway, as it means that they won't have to be installed every time you `up` the box.
2. Prevent the custom box from searching for updates all the time. You will have either pinned the versions of packages you want, or be running an update during your provisioning anyway, so this is fairly risk free.

You can prevent auto updating by simply creating a file in `/etc/apt/apt.conf.d/` such as `20auto-upgrades` with the following content:

```
APT::Periodic::Update-Package-Lists "0";
APT::Periodic::Unattended-Upgrade "0";
```

### Problem with Connection

When I rebooted my host machine (doesn't happen often!) I got this error the next time I created my vagrant box:

```
mount.nfs: requested NFS version or transport protocol is not supported
```

This took a while to track down, but it seemed that my NFS daemon on my host machine just wasn't running! The solution I am using is a hacky hack, and I don't recommend it at all, but it is working for me, and I needed to get back to actually coding!  I now have a script that runs the following command on startup:

```
service nfs-common start
```

For some unknown reason, setting this service to start on boot just didn't work!

### Network doesn't exist

This issue is definitely a problem with my setup - the private network that Virtualbox creates simply doesn't come up properly, and I get all sorts of errors, from the NFS not working, or the webserver not respondeding.

If you are not aware, when you set up private networking in your vagrantfile, Virtualbox creates a virtual network interface on your host machine that is assigned the gatewy address of the IP address you choose. For example, my vagrant box is set up to use `192.18.32.36` and Virtualbox creates an interface named `vboxnet0` with the address `192.168.32.1` (the interface's name may change depending on how many boxes you are using).

At least, that is what is meant to happen! This has worked for me for years, but something broke a few days before I migrated to NFS, which is why I include it here even though it is not really related. Even with this error, my vagrant machine when using NFS would `up` seemingly without an issue. It would only be when I tried to access the webserver that I would notice it. With HFS enabled, the `up` fails as it can't connect to NFS on the guest machine.

No amount of removing/upgrading Virtualbox or Vagrant seemed to fix this issue, and I was unable to find anything useful on the web. So unfortunately I have resorted to another hacky hack. Once Virtualbox has tried to create the interface, you will see it when running `ip addr` but it will show as `DOWN`:

```
vboxnet0: <BROADCAST,MULTICAST,DOWN> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
```

Additionally, it won't have an IP address. You can find out what IP address it should have in Virtualbox, under "Host Network Manager".

Then, run the following commands as root (changing `vboxnet0` and `192.168.32.1/24` to match your setup:

```
ip link set vboxnet0 up
ip addr add 192.168.32.1/24 dev vboxnet0
```

As this problem is intermittent, and probably related to the fact that I change physical networks frequently, I have a bash script set up with these commands in my home directory. I have found that I can run that whilst `vagrant up` is running and it fixes the issue.  The issue normally remains fixed until I reboot the machine.