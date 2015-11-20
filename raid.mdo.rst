
Creating a level 0 RAID from EBS volumes on an AWS EC2 instance
[ Snippets | aws, unix, linux, ubuntu ]

The size of single EBS volumes on Amazon's cloud offering AWS is limited to 1024 GB. To get an entity which acts like a single huge disk and exceeds that limitation, several volumes can be connected into a software RAID (redundant array of independent disks). In this example, we will create an array with a capacity of a little over 4 TB from four 1GB sized EBS volumes.

The exact kind of such an array depends on the application. Friends don't let friends use a level 0 RAID. These types of software RAIDs do not provide any data replication, so the failure of any single disk can be assumed to result in complete data corruption. Achtung, Danger! That said, in some cases it is a viable choice. If you just need a huge place to put data, which disposable and/or is replicated by different means. A good example would be the data of a single node in a larger database replica set. In this case a node could fail and lose all its data, but not affect the integrity or the functionality of the database setup as a whole.
AWS Prerequisites

Before getting started, you should make sure to be running an EC2 instance of your choice, the following will assume Ubuntu 14.04 LTS. Create four fresh volumes, and attach them to the instance. If you don't have good reasons, use magnetic volumes. They are cheaper and you will probably not feel a difference in performance, especially if you choose to use striping (see below). The following will assume, that the volumes are attached as /dev/sdf to sdi in the AWS EC2 console. Behind the scenes, Ubuntu will happily rename the devices into /dev/xvdf to xvdi. It's advisable to wait a bit until everything is started and attached.
Preparations

Install mdadm, a tool that to manage software RAID devices. The name can be interpreted as 'multiple device administration'

$ sudo apt-get install mdadm

A useful command to view block devices and their attributes is blkid. Try it and look at the output you get started, it may come in handy if you want to take a closer look at what is happening.

$ sudo blkid

Creating and mounting the RAID device

There are many choices to assemble a collection of volumes into a single array. We will take a look at the two most simple and irresponsible ones. Both require slightly different commands and have some distinct upsides and downsides while otherwise providing the same functionality. In each case, the result will be a single device. It will have a capacity which equals that of all single devices taken together. Neither type provice any redundancy, so strictly speaking the term RAID is slightly misapplied on them.

The first variant is striping, also known as level 0 RAID. Continuous storage space is split up into small chunks (the stripes), and subsequent stripes are placed on different devices. This has the benefit of effectively increasing the throughput of the resulting device. A downside of striping is, that adding devices to the array (growing) may take a pretty long time, depending on the size of the volumes. This is based on the moving and redistributing of existing stripes to the new array member.

The second variant of creating a James Dean style disk mashup, is just appending devices to each other in a linear fashion, which does not provide the speed benefits of striping but can be grown nearly instantaneous. With all the talk about enlarging the disks, it should be stated that removing devices once grown is not possible. A complete data wipe and recreation of the array with fewer devices is the only way to take disks away. While striping is level 0, linear is considered to be a non-RAID configuration at all.

To create an array, issue the following command substituting 'stripe' for 'linear' if you wish:

# the verbose parameter is only there to provide a little entertainment
$ sudo mdadm --create --verbose /dev/md/raid --level=stripe --raid-devices=4 /dev/xvd[fghi]

It will create a new device (it might be /dev/md0, this name varies), linked to from /dev/md/raid. To make the new device usable as any other disk, a filesystem needs to be created on it, XFS or an EXT version of your choice are good fits for most applications:

$ sudo mkfs.xfs /dev/md/raid

Finally, it's time to mount the device:

$ sudo mkdir /opt/raid
$ sudo mount /dev/md/raid /opt/raid

Hooray! A huge disk at our disposal! At this stage, we could start using it, but there's a bit more to make sure it will behave. By behave I mean survive an inevitable reboot.
Make the RAID survive reboots

Unfortunately, the array will not be reassembled under the same name upon a reboot without a few further steps. Run the following command, and copy the line corresponding to the newly created md device:

$ sudo mdadm --detail --scan

Add the extracted line to the bottom of the /etc/mdadm/mdadm.conf file. In addition, it might be a good idea to add the following line above it, which limits on where to look for eventual RAID components:

DEVICE /dev/xvd[fghi]

For the changes in this config file to take effect, it is necessary to update the initramfs. Otherwise setting will have noe effect, and the device might be reassembled but the naming will be something unexpected like /dev/md/0_0.

$ sudo update-initramfs -u

So far so good

These were all steps necessary to set up one of the two most basic (and dangerous) types of software RAIDs. If you want to look up some details or dive deeper into the topic, I would recommend to take a look at the following links:

    RAID docs in the excellent Arch Linux Wiki
    AWS docs on RAID creation

In the following subsections, some useful utility commands for handling md devices are listed, among them how to take the newly created device apart for good. I am sure you will find use in at least a few of them either for maintenance or while exploring the topic.
Temporary disassembling and reassembling RAID arrays

A RAID array can be temporary disassembled with:

$ sudo mdadm --stop /dev/md/raid

and subsequently put together automatically using:

# it's interesting to add --verbose and get more output here
$ sudo mdadm --assemble --scan

Mount a RAID array via fstab

Adding this line to fstab, will try to mount the device on boot but not wait for it neither complain if absent. Instead of the first field, you could provide a 'UUID=...'

/dev/md/raid       /opt/raid    xfs     defaults,nobootwait,auto      0       2

Growing

Oh no, the huge disk is not huge enough! Assuming you have created a RAID array with 4 devices, and want to add one more to have even more space, the array needs first to be grown to include the new device. Afterwards the filesystem needs to be expanded. Depending on the RAID type, the commands differ slightly and the required time differs significantly.

Once the appropriate growth process described below is complete, just expand the filesystem to match the new capacity. You have used some nice, growable filesystem like xfs, right?

$ sudo xfs_growfs /dev/md/raid

Add devices to a LINEAR-type array

As described earlier, growing a linear array works nearly instantly:

$ sudo mdadm --grow /dev/md/raid --add /dev/xvdj --backup-file=/tmp/raid.bak

That's it. Really.
Add devices to a STRIPE-type array

The command for this case looks slightly different, and takes time. Lots of time. Adding a 1TB disk takes a little over a day for my setup. The created backup file will be a few MB large. It contains critical sections of the RAID array, but then again you could just create it from scratch as the data is supposed to be disposable. Generally speaking, storing it in the tmp folder is a terrible idea as well if a reboot is at all possible.

$ sudo mdadm --grow /dev/md/raid --raid-devices=5 --add /dev/xvdj --backup-file=/tmp/raid.bak

To check the progress, run:

# for snapshot states
$ cat /proc/mdstat
# or for a fancy realtime view
$ watch -t 'cat /proc/mdstat'

Although this will be long and painful, the RAID device will remain usable in the meanwhile.
Destroying an array

To take devices out of a RAID completely and use it for something else, any traces of a previous array need to be removed. If a device is not zeroed and simply used as a common disk, the data on the device can (and probably will) be fubared upon a reboot by the mysterious ways of mdadm. This command is essential when trying out different configurations, but I hope it goes without saying that you need to use it with utmost caution.

$ sudo mdadm --zero-superblock /dev/xvd[fghi]

In conclusion
