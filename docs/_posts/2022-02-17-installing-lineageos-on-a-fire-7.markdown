---
layout: post
title:  "Installing LineageOS on a Fire 7 Tablet"
date:   2022-02-17 16:43:00 +0000
categories: lineageos fire
---
When an old tablet is no longer up to the task for which it was originally designed, binning the thing seems a shame, but there is no guarantee that selling it is worthwhile. A recent [ebay search][ebay-fire] suggests that a Fire 7 tablet with 8GB storage could be worth as little as £2 or as much as £26 (if it's in decent condition, and comes with a child-friendly case). £26 is not to be sniffed at - if nothing else, and assuming you have bought into the Amazon ecosystem, that gets you pretty close to an Echo, or a Fire Stick. On the other hand, £2 is a pretty miserly return to be after for the effort of listing the item, dealing with questions, and posting it - assuming it sells at all.

I had a go at flashing my Kindle Fire 7 (7th Generation) tablet and then installing LineageOS. The aim was: 

a. Learn how to replace the Fire OS with a more generic flavour of Android;
b. Put together a tablet that I might be able to use as a hobby device for random bits and pieces - or even as a first "true Android" tablet for the kids;
c. Maybe even figure out how the tablet could be repurposed as a smart screen in the home; showing weather, news, or even a looped [video of a cat dressed as a shark chasing a duck around on a roomba][cat-duck-roomba]. Because, well, why not?

There are loads of guides online; the hardest part in all of this is working out which instructions are relevant to you. So if you aim to follow this guide, you'll need:

1. An Amazon Fire 7 Tablet (7th Generation). You can look at the [device specifications website][dev-spec] to work out what you have. If your Fire tablet has not been reset (i.e. you can still sign in to it), then you can also determine your hardware version by going to Settings > Device Settings > Device Model;
2. A micro-USB cable
3. Some means of opening the case of the tablet. If you have teh muscles, then you can try and pry the case apart by force. Otherwise, a plastic opening tool works well. You can use a small-sized slotted screwdriver, but you are liable to damage the case of your tablet, because in any battle of metal versus plastic (especially the plastic on a cheap tablet), the metal is likely to win;
4. A length of wire, with bare ends; each end should have about 1cm of exposed wiring; a paperclip works well here.
5. A small Phillips Head screwdriver (you'll need this to disconnect the Fire battery);
5. A computer running linux. I'm using Linux Mint;


### Step 1: Download the binaries you'll need onto your linux box

You need some components installed:

{% highlight bash %}
sudo apt update
sudo add-apt-repository universe
sudo apt install python3 python3-serial adb fastboot
{% endhighlight %}

You need to disable the Modem Manager:

{% highlight bash %}
sudo systemctl stop ModemManager
sudo systemctl disable ModemManager
{% endhighlight %}

You'll need the `amonet-austin` binaries, which have been helpfully put on a zip file [here][amonet-austin-zip]. You also want to download GApp binaries from [here][open-gapp]. I chose the arm 5.1 nano installation. Arm 5.1 is non-negotiable, and is required given the version of tablet I was installing on. The nano installation is much smaller than the stock one, and I wanted to install everything from the Fire Tablet's internal storage rather than relying on a microSD card (two cards have failed in my Fire Tablet in the past two years).

Finally, you need a `lineageOS` zip. I got mine [here][lineage].

1. Unzip the `amonet-austin` zipfile - we'll open a bash terminal in here to do the work. The other zips (for lineageOS and GApp) can be left as zips.

### Step 2: Prepare the Kindle

1. Reset the device, and unregister it from Amazon;
2. Switch it off, and remove the back cover. This works best if you start at the bottom end (the end opposite the camera), and work your way up.
3. Disconnect the battery. This involves unscrewing the two small screws that secure the connector to the main board, and then popping the cable off. You don't need to use a lot of force; once the screws are out, the cable should just "pop" free with some gentle encouragement.
4. Work out where the VDD connector is. This is the fiddliest part of the operation: you need to short one pin on the Fire PCB to allow the tablet to be flashed. [This picture][vdd] is a good guide. The picture shows two options, `VDD1` and `CMD`. I've not had a lot of luck with the `CMD` connector, so I tend to stick to the `VDD` one.
5. Push the wire under the shielding in order to contact the `VDD` connector, and ground it out. You can do this by connecting the other end of the wire to the shield.

### Step 3: Let the fun begin

From a terminal on your linux box, navigate into the folder into which you unzipped the amonet austin file. 

{% highlight bash %}
(base) x@x:~/work/fire_brick/amonet_austin$ sudo ./bootrom-step.sh 
[sudo] password for x:           
[2022-02-28 11:12:52.079500] Waiting for bootrom
{% endhighlight %}

At this point, connect the tablet to the linux box using a microUSB cable. If all goes to plan, you should see some nice stuff being displayed. If, however, you see this:

{% highlight bash %}
(base) x@x:~/work/fire_brick/amonet_austin$ sudo ./bootrom-step.sh 
[2022-02-28 11:18:08.283713] Waiting for bootrom
[2022-02-28 11:18:23.919498] Found port = /dev/ttyACM0
[2022-02-28 11:18:23.958787] Handshake
[2022-02-28 11:18:23.979722] Disable watchdog
Traceback (most recent call last):
  File "main.py", line 156, in <module>
    main()
  File "main.py", line 75, in main
    handshake(dev)
  File "/home/x/work/fire_brick/amonet_austin/modules/handshake.py", line 11, in handshake
    dev.write32(0x10007000, 0x22000000)
  File "/home/x/work/fire_brick/amonet_austin/modules/common.py", line 160, in write32
    self.check(self.dev.read(2), b'\x00\x01') # arg check
  File "/home/x/work/fire_brick/amonet_austin/modules/common.py", line 87, in check
    raise RuntimeError("ERROR: Serial protocol mismatch")
RuntimeError: ERROR: Serial protocol mismatch
{% endhighlight %}

the most likely issue is that your wire short between `VDD` and the shielding was not quite on point. You may need to disconnect the USB cable, adjust the wire, and try again.

If it works, the terminal will ask you to remove the shorting wire. At this point, toss the wire - you won't need it again - and hit `Enter`. The Fire tablet will reboot into an unlocked fastboot state. You can now run the fastboot step from your linux box.

{% highlight bash %}
sudo ./fastboot-step.sh
{% endhighlight %}

The tablet will reboot into TWRP.

### Step 4: Transfer and install `LineageOS` and `GApp`

1. Mount the tablet's storage device for data transfer.
2. From the linux box, use `adb push` to transfer the `lineageOS` and `GApp` zip files to the tablet;
3. Unmount the tablet as a storage device, and then use TWRP to install `lineageOS` and `GApp`.

The result should be a shiny lineageOS tablet.

[ebay-fire]: shorturl.at/hkyV0
[cat-duck-roomba]: https://www.youtube.com/watch?v=Of2HU3LGdbo
[dev-spec]: https://www.devicespecifications.com/en/model/94cf4357
[amonet-austin-zip]: https://forum.xda-developers.com/attachment.php?attachmentid=4732195&stc=1&d=1553717688
[open-gapp]: https://opengapps.org/
[lineage]: https://androidfilehost.com/?fid=1395089523397896035
[vdd]: https://forum.xda-developers.com/attachments/position-jpg.4771766/

