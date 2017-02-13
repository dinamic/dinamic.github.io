---
layout: post
title:  "How to install Plex Media Server on Turris Omnia!"
date:   2017-02-14 01:42:16 +0200
categories: turris-omnia plex-media-server
---

**TL;DR** Plex works via LXC on Ubuntu, but we need to find a way to disable transcoding completely. Meanwhile I would put my trust on DLNA.

## What is Turris Omnia?

[Turris Omnia](turris-omnia) is a powerful router and could be, as the manufacturer claims, *the open-source center of your home*. Often times I back different projects at Indiegogo and Kickstarter. While most of them turn to dissapointments, this one has not failed to impress.

My home network relies on [Mikrotik CRS125-24G-1S-2HnD-IN][mikrotik-router] which acts like a gateway and a wonderful 24 port switch should me and my friends decide to have a hackaton or lan party weekend. Needless to say it is configured already and I am not up for reconfiguring.

So, what would I use the Turris Omnia for?

I decided to set it up as an access point. But, since I am having it in the network anyway, why not make it my media center storage then? It already has DLNA support, so why not.

I hooked up an external disk drive with some video files on and configured the DLNA support.

First thing that shocked me was that I spend an awful lot of time struggling to configure the Omnia to act as an access point. This could have been avoided. Please, Turris, add a button and make our lifes easier!

Once I've got DLNA running it came to me that since it is Linux, why not try and put [Plex][plex] on it. The device will only be used as an AP and boasts more than twice as much processor power and memory than my Mikrotik.

I started Googling only to find out the guys are trying to make it works, but are not there yet, so decided to give it a go myself.

People in the forums were complaining `unzip` don't work well to unzip the archive on the router itself. So I have unzipped it on my macbook and uploaded it to the Omnia over scp.

{% highlight bash linenos %}
wget "https://downloads.plex.tv/plex-media-server/1.3.4.3285-b46e0ea/PlexMediaServer-1.3.4.3285-b46e0ea-arm7.spk"
mkdir plex
mv PlexMediaServer-1.3.4.3285-b46e0ea-arm7.spk plex/PlexMediaServer-1.3.4.3285-b46e0ea-arm7.tar
cd plex
tar fxv PlexMediaServer-1.3.4.3285-b46e0ea-arm7.tar
tar fxz package.tgz
rm package.tgz PlexMediaServer-1.3.4.3285-b46e0ea-arm7.tar
scp -r ../plex root@YOUR_OMNIA_IP:/root
{% endhighlight %}

## Running Plex

### Prerequisites

The Turris Omnia sports a Marvel Armada 385 1.6GHz dual-core ARMv7 compatible sprocessor. I had seen that in a few forum posts, people were reporting struggle to make the Plex build for Synology work. It would make sense to pick this one up, because the Synology has an ARMv7 compatible processor as well. I will be trying it on as well.

You can name your LXC whatever you would like. For the purposes of this how-to I will be using `plex`.

### First attempt - Native support

Trying native support didn't really ran well. Doing `ldd` on the binary lets you know how many libraries you are lacking behind. I was unable to find how to install those on the Omnia, so this was a dead end. I have spent a huge amount of time making it work to no avail.

I was thinking on how can I remediate this. If only there was the possibility of running a full fledged Linux on it.

There is! The Omnia has support for LXC, so this was the next thing I tried on.

### Secondary attempt - LXC via Alpine Linux

I have used docker before and knew one of the lightweightest Linux distributions one could wish for is [Alpine Linux][alpine-linux]. Let's try that..

You have already uploaded the files to the Omnia, but you can't access them in the LXC. You need to create a mount point and bind it to the device content.

{% highlight bash %}
mkdir -p /srv/lxc/plex/rootfs/mnt/sda1
mount -o bind /tmp/run/mountd/sda1 /srv/lxc/plex/rootfs/mnt/sda1
cp -rf plex /srv/lxc/plex/rootfs/root
{% endhighlight %}

Well, it didn't work. A lot of libraries were missing here as well. I managed to find a docker image with glibc support, so I extracted the directions on how to install it properly. Problem was there were still issues with other libraries and up to this point I was not so keen on investing time to make it work.

If you managed to get Plex working on LXC using Alpine Linux, let me know and I will link your blog post with instructions in here.

### Thirt attempt - LXC via Ubuntu (Yakkety Yak)

[Ubuntu Linux][ubuntu-linux] has become a big name among Linux distributions. Ten years ago the closest thing to a friendly Linux environment we had was Mandrake Linux, later called Mandriva Linux and it was not great. Ubuntu made the overall experience better and showed the world a company can have an open-source product and still be able to make a profit. Job well done, [Canonical][canonical]!

I managed to run Plex without problems on the Ubuntu LXC. Here is what I did and my train of thought.

Upon first start you will need to get the plex files copied over to the LXC. You may be familiar with these commands from my second attempt.

{% highlight bash %}
mkdir -p /srv/lxc/plex/rootfs/mnt/sda1
mount -o bind /tmp/run/mountd/sda1 /srv/lxc/plex/rootfs/mnt/sda1
cp -rf plex /srv/lxc/plex/rootfs/root
{% endhighlight %}

Let's first examine what's the situation out of the box. Let's see if the dynamic libraries are to be found.

{% highlight bash %}
lxc-attach -n plex
cd plex
./Plex\ Media\ Server
{% endhighlight %}

{% highlight bash %}
./Plex Media Server: error while loading shared libraries: libboost_atomic.so.1.59.0: cannot open shared object file: No such file or directory
{% endhighlight %}

Well, we have the missing library in this very folder.
Let's move the libraries to a more appropriate folder and instruct Linux where to find it.

{% highlight bash linenos %}
mkdir -p /usr/lib/plexmediacenter
mv -fi *.so* /usr/lib/plexmediacenter
echo "/usr/lib/plexmediacenter" >> /etc/ld.so.conf.d/plexmediacenter.conf
ldconfig
{% endhighlight %}

Let's see if everything is okay now. We will be using `strace`.

{% highlight bash linenos %}
apt-get update && apt-get install -y strace
strace ./Plex\ Media\ Server
{% endhighlight %}

Um.. seems like there are some hardcoded paths and the libraries that are distributed with Plex are not found either. Let's move the application code to the hardcoded location and add the libraries to the linker and see if that works.

Seems like the application need to be under `/root/Library/Application Support/Plex Media Server/`, so let's move it and see how it looks.

{% highlight bash linenos %}
mkdir -p "/root/Library/Application Support/"
mv -fi /root/plex/* "/root/Library/Application Support/Plex Media Server"
cd "/root/Library/Application Support/Plex Media Server"
strace ./Plex\ Media\ Server
{% endhighlight %}

This seem to have worked. I can also see the Plex server on my Android Plex client listed as `LXC_NAME`.

Now, if we quit the ssh session we will quit Plex as well. We need to find how to keep it working on the background.

I have decided to give it a go working as a upstart service. Unfortunately, this fails with `Segmentation fault` and I was unable to figure out why.

I have tried making it work with `screen` as well, but screen didn't seem operational at all.

Thinking in the same direction, `byobu` is a nice option and seemed to have worked fine!

## Conclusion

After being able to run DLNA and Plex on the Turris Omnia and given it a few days to test. The Turris Omnia is very powerful device for a router, but is a no match for transcoding content on the fly. I was unable to find how to disable the transcoding and always stream the original quality of the video.

I think I will be sticking to DLNA for now.



[plex]: https://www.plex.tv/
[mikrotik-router]: https://routerboard.com/CRS125-24G-1S-2HnD-IN
[turris-omnia]: https://omnia.turris.cz
[alpine-linux]: https://alpinelinux.org/
[ubuntu-linux]: https://www.ubuntu.com/
[canonical]: https://www.canonical.com/
