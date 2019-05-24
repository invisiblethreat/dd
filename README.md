# Overview  of `dd`

*This is a forum post that I found 10-15 years ago. The content is not mine-
I've been carting it around in different formats formats for all this time. It
deserves to be seen by all. There are some edits for clarity and grammar, but
the substance remains the same.*

The basic command is structured as follows:

```
dd if=<source> of=<target> bs=<byte size> skip= seek= conv=<conversion>
#"USUALLY" some power of 2, not less than 512 bytes
# ie, 512, 1024, 2048, 4096, 8192, 16384, but can be ANY reasonable number
```

* Source is the data being read
* Target is where the data gets written.
* If you reverse the source and target, you can wipe out a lot of data. This feature has inspired the nickname `dd` "Data Destroyer".
* Caution should be observed when using `dd` to copy encrypted partitions.

## Make swap on a running system

How to make a swap file, or another swap file on a running system

```
dd if=/dev/zero of=/swapspace bs=4k count=250000
mkswap /swapspace
swapon /swapspace
```

This can mitigate out of memory issues due to memory leaks on servers that can
not easily be rebooted.

## Size and drives

How to pick proper block size

```
time `dd` if=/dev/zero bs=1024 count=1000000 of=/home/sam/1Gb.file
time `dd` if=/dev/zero bs=2048 count=500000 of=/home/sam/1Gb.file
time `dd` if=/dev/zero bs=4096 count=250000 of=/home/sam/1Gb.file
time `dd` if=/dev/zero bs=8192 count=125000 of=/home/sam/1Gb.file
```

This method can also be used as a drive benchmark, to find strengths and
weaknesses in hard drives

```
time `dd` if=/home/sam/1Gb.file bs=64k | `dd` of=/dev/null
```

```
time `dd` if=/dev/zero bs=1024 count=1000000 of=/home/sam/1Gb.file
```

The output looks like this

```
1000000+0 records in
1000000+0 records out

real    2m17.951s
user    0m0.930s
sys 0m25.160s
```

Your output may have different figures. The format is what I'm referring to. If
you play with `bs=` and `count=`, always having them multiply out to the same
figure, you can calculate bytes/second like this: 1Gb/total seconds = Gb/s. You
only use the 'real' timing figure from above. You can get more realistic results
using a 3Gb file.

## How to rejuvenate a hard drive - _NOT SSDs_

This will sometimes cure input/output errors experienced when using `dd`. Over
time the data on a drive, especially a drive that hasn't been used for a year or
two grows into larger magnetic flux points than were originally recorded. This
makes it hard for the drive heads to decipher these magnetic flux points. This
results in read/write errors. Also, sometimes sector 1 goes bad resulting in a
useless drive. What you need to do is

```
dd if=/dev/sda of=/dev/sda
```

to rejuvenate the drive. The process rewrites all the data on the drive in nice
tight magnetic patterns that can then be read properly. The procedure is
perfectly safe, and saves one a lot of money on HDDs.

## Copying disks and partitions

```
dd if=/dev/sda2 of=/dev/sdb2 bs=4096 conv=notrunc,noerror
```

`sda2` and `sdb2` are partitions. You want to copy `sda2` to `sdb2`. If `sdb2`
doesn't exist, `dd` will start at the beginning of the disk, and create it. Be
careful with order of if and of. You can write a blank disk to a good disk if
you get confused. The only difference between a big partition and a small
partition, besides size, is the partition table. If you are copying `sda` to
`sdb`, an entire drive with a single partition, `sdb` being smaller than `sda`,
then you have to

```
dd if=/dev/sda skip=2 of=/dev/sdb seek=2 bs=4k conv=noerror
```

`skip` skips input blocks at the beginning of the media (`sda`). Seek skips over so
many blocks on the output media before writing (`sdb`). By doing this, you leave
the first 4k bytes on each drive untouched. You don't want to tell a drive it
is bigger than it really is by writing a partition table from a larger drive to
a smaller drive. The first 63 sectors of a drive are empty, except sector 1,
the MBR. If you are copying a smaller partition to a larger one, the larger
partition will read the correct size with:

```
fdisk -l
```

```
df -h
```

This is because `fdisk` reads the partition table and df reads the format info.
If you `dd` a smaller partition to a larger one the larger one will now be
formatted the same as the smaller one and there won't be any space left on the
drive.The way around this is to build a partition image file.

```
dd if=smaller_partition of=/home/sam/smaller_partition.img
```

Mount the image like a drive using:

```
mount -o loop /home/sam/smaller_partition.img /mnt/directory
cd /mnt/directory
cp -r * /mnt/larger_partition_already_partitioned_and_formatted_to_the_size_you_want
```

CP has an -r switch for recursive copy. Now, if you are copying `sda3` to `sda2`
, this is different. What you want to do is this:

```
dd if=/dev/sda3 of=/dev/sda2 bs=4096 conv=notrunc,noerror
```

The very last part of the drive is usually zeroes. So, if you have room for the
data, and the zeroes get truncated that's ok. Make an iso image of a CD:

```
dd if=/dev/hdc of=/home/sam/mycd.iso bs=2048 conv=notrunc
```

This copies sector for sector. The result will be a hard disk image file of the
CD. You can mount the image with:

```
mkdir /mnt/mycd
```

this line in fstab:

```
/home/sam/mycd.iso /mnt/mycd iso9660 rw,user,noauto 0 0
```

Save fstab,

```
mount -o loop /mnt/mycd
```

Then the file system will be viewable as files and directories in the directory
```
/mnt/mycd
```

Copy a floppy disk:

```
dd if=/dev/fd0 of=/home/sam/floppy.image bs=2x80x18b conv=notrunc
```

or

```
dd if=/dev/fd0 of=/home/sam/floppy.image conv=notrunc
```

The 18b specifies 18 sectors of 512 bytes, the 2x multiplies the sector size by
the number of heads, and the 80x is for the cylinders--a total of 1474560 bytes.
This issues a single 1474560-byte read request to /dev/fd0 and a single 1474560
write request to

```
/home/sam/floppy.image
```

This makes a hard drive image of the floppy, with bootable info intact. The
second example uses default "bs=" of 512, which is the sector size of a floppy.

## Overwrite with /dev/urandom

If you're concerned about spies with superconducting quantum-interference
detectors you can always add a "for" loop for government level secure disk
erasure: copy and paste the following two lines into a text editor.
```
#!/bin/bash
for n in `seq 7`; do `dd` if=/dev/urandom of=/dev/sda bs=8b conv=notrunc; done
```

Now you have a shell script for seven passes of random characters over the
whole disk Do:
```
chmod a+x <shellscriptfile>
```

to make it executable.
To make a bootable USB thumb drive: Download 50 MB Debian based distro here:
http://sourceforge.net/projects/insert/
Plug in the thumb drive to a USB port. Do:
```
dmesg | tail
```

Look where the new drive is, sdb, or something similar. Do:
```
dd if=/home/sam/insert.iso of=/dev/sdb ibs=4b obs=1b conv=notrunc,noerror
```

Now set the BIOS to USB boot, plug in the thumb drive, and boot the machine.
Copy just the MBR and boot sector of a floppy to hard drive image:
```
dd if=/dev/fd0 of=/home/sam/MBRboot.image bs=512 count=2
```
This copies the first 2 sectors of the floppy.
Cloning an entire hard disk:
```
dd if=/dev/sda of=/dev/sdb conv=notrunc,noerror
```

In this example, sda is the source. Sdb is the target. Do not reverse the intended
source and target. Surprisingly many people do. Notrunc means 'do not truncate
the output file'. Noerror means to keep going if there is an error. Normally
`dd` stops at any error.
Copy MBR only of a hard drive:

```
dd if=/dev/sda of=/home/sam/MBR.image bs=446 count=1
```


This will copy the first 446 bytes of the hard drive to a file. If you haven't
already guessed, reversing the objects of if and of, in the `dd` command line
reverses the direction of the write.

## Helix and dcfldd

Wipe a hard drive of all data (you would want to boot from a cd to do this)
http://www.efense.com/helix
is a good boot cd. The helix boot environment contains the DoD version of `dd`
called `dcfldd`. It works the same way, but is has a progress bar.

```
dcfldd if=/dev/zero of=/dev/sda conv=notrunc
```

This is useful for making the drive almost like new. Most drive have `0x0000ffh`
written to every sector from the factory. Overwrite all the free space on a
partition (deleted files you don't want recovered)

```
dd if=/dev/urandom > fileconsumingallfreespace
```

When `dd` says no room left on device, all the free space has been overwritten with random characters. Then, delete the big file.

## Hexdumping for fun and profit

To view your virtual memory:
```
dd if=/proc/kcore | hexdump -C | less
```

use PgUp, PgDn, up arrow, down arrow to navigate in less. Less is my favorite
editor. Or I should say it would be my favorite editor if it allowed editing.
What filesystems are installed:

```
dd if=/proc/filesystems | hexdump -C | less
```

all loaded modules:

```
dd if=/proc/kallsyms | hexdump -C | less
```

interrupt table:

```
dd if=/proc/interrupts | hexdump -C | less
```

How many seconds has the system been up:

```
dd if=/proc/uptime | hexdump -C | less
```

partitions and sizes in kb:

```
dd if=/proc/partitions | hexdump -C | less
```

Memory stats:

```
dd if=/proc/meminfo | hexdump -C | less
```

## Cloning Drives

I put two identical drives in every one of my machines. Before I do anything
which might be disastrous, I do:

```
dcfldd if=/dev/sda of=/dev/sdb bs=4096 conv=notrunc,noerror
```

and copy my present working sda drive system to the sdb drive. If I wreck the
installation on sda, I just boot with the helix cd and:

```
dcfldd if=/dev/sdb of=/dev/sda bs=4096 conv=notrunc,noerror
```

and I get everything back exactly the same as before whatever boneheaded thing I
was trying to do didn't work. You can really, really learn linux this way,
because you absolutely can't wreck what you have an exact copy of. You also
might consider making the root partition separate from /home, and make /home
big enough to hold the root partition, plus more. Then you can do:

```
dd if=/dev/sda2 (root) of /home/sam/root.img bs=4096 conv=notrunc,noerror
```

To make a backup of root, and

```
dd if=/home/sam/root.img of=/dev/sda2 (root) bs=4096 conv=notrunc,noerror
```

To write the image of root back to the root partition if you messed up and can't
launch the X server anymore, or edited /etc/fstab and can't figure out what you
did wrong. It only takes a few minutes to restore a 15 GB root partition from an
image file.

To make a file of 100 random bytes:
```
dd if=/dev/urandom of=/home/sam/myrandom bs=100 count=1
```

/dev/random produces only as many random bits as the entropy pool contains. This
yields quality randomness for kryptographic keys. If more random bytes are
required, the process stops until the entropy pool is refilled (waggling your
mouse helps). /dev/urandom does not have this restriction. If the user demands
more bits than currently in the entropy pool, it produces them using a pseudo
random number generator. Here, /dev/urandom is the Linux random byte device.
myrandom is a file. Write random data over a file before deleting it: first do a

```
ls -l
```

to find filesize.
In this case it is 3769

```
ls -l afile -rw------- ... 3769 Nov 2 13:41 <filename>
```

```
dd if=/dev/urandom of=afile bs=3769 count=1 conv=notrunc
```

This will write random characters over the entire file.
Copy a disk partition to a file on a different partition.

Warning!! Do not copy a partition to the same partition.

```
dd if=/dev/sdb2 of=/home/sam/partition.image bs=4096 conv=notrunc,noerror
```

This will make a file that is an exact duplicate of the sdb2 partition. You can
substitue hdb, sda, hda, or whatever the disk is called. Or

```
dd if=/dev/sdb2 ibs=4096 | gzip > partition.image.gz conv=noerror
```

Makes a gzipped archive of the entire partition. To restore use:

```
 | gunzip >
```

for bzip2(slower,smaller), substitute bzip2 and bunzip2, and name the file

```
.bz2
```

Restore a disk partition from an image file.

```
dd if=/home/sam/partition.image of=/dev/sdb2 bs=4096 conv=notrunc,noerror
```

This way you can get a large hard drive and partition it so you can back up your root partition. If you mess up your root partition, you just boot from the helix cd and restore the image. To covert a file to all uppercase:

```
dd if=filename of=filename conv=ucase
```

Make a ramdrive:
The Linux kernel usually makes a number a ramdisks you can make into ramdrives.
You have to populate the drive with zeroes like so:

```
dd if=/dev/zero of=/dev/ram7 bs=1k count=16384
```

Makes a 16 MB ramdisk full of zeroes.
```
mke2fs -m0 /dev/ram7 4096
```

puts a file system on the ramdisk turning it into a ramdrive. Watch this puppy
smoke.

```
debian:/home/sam # hdparm -t /dev/ram7
/dev/ram7:
Timing buffered disk reads:   16 MB in  0.02 seconds = 913.92 MB/sec
```

You only to do the timing once because it's cool. Make the drive again because
`hdparm` is a little hard on ramdrives. You can mount the ramdrive with:

```
mkdir /mnt/mem
mount /dev/ram7 /mnt/mem
```

Now you can use the drive like a hard drive. This is particularly superb for
working on large documents or programming. You can copy the large file or
programming project to the ramdrive, which on my machine is at least 27 times
as fast as /dev/sda, and ever time you save the huge document, or need to do a
compile it's like your machine is running on nitromethane. The only thing is the
ramdrive is volatile. If you lose power, or lock up the data on the ramdrive is
lost. Use a reliable machine during clear skies if you use a ramdrive.
Copy ram memory to a file:

```
dd if=/dev/mem of=/home/sam/mem.bin bs=1024
```

The device

```
/dev/mem
```

is your system memory. You can actually copy any block or character device to a
file using dd. Memory capture on a fast system, with bs=1024 takes about 60
seconds, a 120 GB HDD about an hour, a CD to hard drive about 10 minutes, a
floppy to a hard drive about 2 minutes. With dd, your floppy drive images will
not change at all. If you have a bootable DOS diskette, and you save it to your
HDD as an image file, when you restore that image to another floppy it will be
bootable. `dd` will print to the terminal window if you omit the
```
of=/dev/output
```

part.
```
dd if=/home/sam/myfile
```

will print the file myfile to the terminal window.
To search the system memory:
```
dd if=/dev/mem | hexdump -C | grep 'some-string-of-words-in-the-file-you-forgot-to-save-before-the-power-failed'
```

If you need to cover your tracks quickly, put the following commands in a script to overwrite system ram with zeroes. Don't try this for fun.
```
mkdir /mnt/mem
```

```
mount -t ramfs /dev/mem /mnt/mem
```

```
dd if=/dev/zero > /mnt/mem/bigfile.file
```

This will overwrite all unprotected memory structures with zeroes, and freeze
the machine so you have to reboot (Caution, this also prevents committment of
the file system journal and could trash the file system).
If you are just curious about what might be on you disk drive, or what an MBR
looks like, or maybe what is at the very end of your disk:

```
dd if=/dev/sda count=1 | hexdump -C
```

Will show you sector 1, or the MBR. The bootstrap code and partition table are
in the MBR.

To see the end of the disk you have to know the total number of sectors for the
disk, and the disk has to be set up with Maximum Addressable Sector equal to
Maximum Native Address. The helix CD has a utility to set this correctly. In the
`dd` command your seek value will be one less than MNA of the disk. For a 120 GB
Seagate SATA drives

```
dd if=/dev/sda of=home/sam/myfile skip=234441646 default bs=512
```

So this reads sector for sector, and writes the last sector to myfile. Even with
LBA addressing Disks still secretly are read in sectors, cylinders, and heads.
There are 63 sectors per cylinder, and 255 heads per cylinder. Then there is a
total cylinder count for the disk. You multiply out 512x63x255=bytes per
cylinder. 63x255=sectors per cylinder. With 234441647 total sectors, and 16065
sectors per cylinder, you get some trailing sectors which do not make up an
entire cylinder, 14593.317584812. This leaves you with 5102 sectors which cannot
be partitioned because to be in a partition you have to be a whole cylinder.
It's like having part of a person. That doesn't really count as a person. So,
what happens to these sectors? They become surplus sectors after the last
partition. You can't ordinarily read in there with an operating system. But,
`dd` can. It is really a good idea to check for anything writing to surplus
sectors. For our Seagate 120 GB drive you subtract total
sectors(234441647)-(5102) which don't make up a whole cylinder=234436545
partitionable sectors.

```
dd if=/dev/sda of=/home/sam/myfile skip=234436545
```

This writes the last 5102 sectors to myfile. Launch midnight commander (mc) to
view the file. If there is something in there, you do not need it for anything.
In this case you would write over it with random characters:

```
dd if=/dev/urandom of=/dev/sda bs=512 seek=234436545
```

Will overwrite the 5102 surplus sectors on our 120 GB Seagate drive.

Block size:
One cylinder in LBA mode = 255 heads * 63 sectors per track = 16065 sectors =
16065 * 512 bytes = 16065b. The b means '* 512'. 32130b represents a two
cylinder block size. When using cylinder sized block sizes you never need to
worry about that last fraction of a block not being copied because partitions
are made of a whole number of cylinders. Partitions cannot contain partial
cylinders. One cylinder is 8,225,280 bytes.

If you want to check out some random area of the disk:
```
dd if=/dev/sda of=/home/sam/myfile bs=4096 skip=2000 count=1000
```

Will give you 8,000 sectors in myfile, after the first 16,000 sectors. You can
open that file with a hex editor, edit some of it, and write the edited part
back to disk:

```
dd if=/home/sam/myfile of=/dev/sda bs=4096 seek=2000 count=1000
```

So there you got yourself a disk editor. It's not the best, but it works.
You can make a boot floppy:With the

```
boot.img file
```

,
which is pretty easy to get. You just need a program that will literally start
writing at sector 1.
dd if=boot.img of=/dev/fd0 bs=1440k
This makes a bootable disk you can add stuff to. If you want to make a partition
image on another machine: on source machine:
Boot both machines with the helix CD just to be extra sure. Then,
```
dd if=/dev/hda bs=16065b | netcat targethost-IP 1234
```

On target machine:

```
netcat -l -p 1234 | `dd` of=/dev/hdc bs=16065b
```

Netcat is a program, available by default, on almost every linux installation.
It is like a swiss army knife of networking. In the preceding example netcat and
`dd` are piped to one another. One of the functions of the linux kernel is to
make pipes. The pipe character looks like two little lines on top of one
another, both vertical. Here is how this command behaves: This byte size is a
cylinder. bs=16065b equals one cylinder on an LBA drive. The `dd` command is
piped to netcat, which takes as its arguments the IP address of the target(like
192.168.0.1, or any IP address with an open port) and what port you want to
use(1234).

Alert!! Don't hit enter yet. Hit enter on the target machine, hit enter on the
source machine.

This is kind of how Norton Ghost works to image a drive to another machine.
Ok, say you want to find out if your girlfriend or wife is cheating on you,
having cyber whoopie, or just basically misbehaving with her computer. Even if
the computer is secured with a password you can boot with the:
http://www.efense.com/helix
CD and search the entire drive partition for text strings:

```
dd if=/dev/sda2 bs=16065 | hexdump -C | grep 'I really don't love him anymore.'
```

Will search the whole drive partition for the text string specified between the
single quotes. Searching an entire disk partition several times can be quite
tedious. This particular command string prints the search results to the screen,
with the offset where it is located in the partition. `dd` works in the decimal
system. Disk offsets work in hexadecimal. Say you found that text string in your
partition at offset `0x020d0d90h`. You convert that to decimal with one of the
many calculators found in Linux. This is decimal offset 34409872. Dividing by
512 per sector we get 67206.78125.

```
dd if=/dev/sda2 bs=16065 skip=2140 count=3 | less
```

This will print to the screen so you don't accidentally write a file over free
disk space, which may hold deleted files you want to search. With this method
you search all the deleted files, any chat activity, and emails. It works no
matter what security is being employed on the machine. It works with NTFS, ext2,
ext3, reiserfs, swap, and FAT partitions. But, it is illegal to use this method
on a computer you aren't authorized to search. People can be sued, or imprisoned
for unauthorized searches. On a related note, you can write the system memory to
a CD. This is useful for documenting memory contents without contaminating the
HDD. I recommend using a CD-RW so you can practice a little. This doesn't
involve `dd`, but it's cool.
```
cdrecord dev=ATAPI:0,1,0 -raw tsize=700000000 driveropts=burnfree /dev/mem
```

to find the cdwriter:
```
cdrecord -scanbus=ATAPI
```

`dd` will not copy or erase an HPA or host protected area. If used properly `dd` will erase a disk completely, but not as well as using the hardware secure erase, security erase unit command
This method records raw, so you have to do a:

```
dd if=/dev/hdd | less
```

to view the recorded memory. Searching the recorded memory is as above:

```
dd if=/dev/hdd | hexdump -C | grep 'string'
```

string is any ascii sequence, hex sequence (must be separated with a space: '
55<space>aa<space>09' searches for the hex string '55aa09'),

list:
```
'[[:alnum:]]' any alphanumeric characters
'[[:alpha:]]' any alpha character
'[[:digit:]]' any numeric character
'[[:blank:]]' tabs and spaces
'[[:lower:]]' any lower case alpha characters
'[[:upper:]]' any uppercase alpha character
'[[:cntrl:]]' ASCII characters 000 thru 037, and 177 octal
'[[:graph:]]' [:alnum:] and [:punct:]
'[[:punct:]]' any punctuation character ` ! ' # $ % ' ( ) * + - . / : ; < = > ? @ [ \ ] ^ _ { | } ~
'[[:space:]]' tab, newline, vertical tab, form feed, carriage return, and space '[[:xdigit:]]' any hex
digit ranges('[a-d]' = any, or all abcd, '[0-9]' = any, or all 0123456789)
```

You can back up your MBR:

```
dd if=/dev/sda of=mbr.bin count=1
```

Put this on a floppy you make with:

```
dd if=boot.img of=/dev/fd0
```

Along with `dd`. Boot from the floppy and

```
dd if=mbr.bin of=/dev/sda count=1
```

Will restore the MBR. I back up all my floppies to HDD. Floppies don't last
forever, so I do

Here is a command line to read your BIOS, and interfaces
```
dd if=/dev/mem bs=1k skip=768 count=256 2>/dev/null | strings -n 8
```

### `dd` Variants

There is a variation of `dd` for rescuing data off defective media, such as a
hard drive with some bad sectors. It is called dd_rescue. It is available here
  * http://www.garloff.de/kurt/linux/ddrescue/

The Department of Defense implementation of `dd` is called `dcfldd`, and has some
features like a progress monitor so you can time your trips to get coffee.
  * http://dcfldd.sourceforge.net/

Sdd is useful when input block size is different than output block size, and
will succeed in some instances `dd` fails.
  * http://linux.maruhn.com/sec/sdd.html

This is one of the best links on `dd` I haven't written
  * http://www.softpanorama.org/Tools/dd.shtml

### SIGUSR1 Notifications to STDERR

Note that sending a SIGUSR1 signal to a running 'dd' process makes it
print to standard error the number of records read and written so far,
then to resume copying.

```
$ `dd` if=/dev/zero of=/dev/null& pid=$!
              $ kill -USR1 $pid; sleep 1; kill $pid

              10899206+0 records in 10899206+0 records out
```


BLOCKS and BYTES may be followed by the following multiplicative suffixes: c 1, w 2, b 512, kB 1000, K 1024, MB 1000*1000, M 1024*1024, GB 1000*1000*1000, G 1024*1024*1024
So,
```
dd if=/dev/sda of=/dev/sdb bs=1GB
```
Will use one gigabyte block sizes.
bs=4b would give `dd` a block size of 4 disk sectors. 1 sector=512 bytes.
bs=4k would indicate `dd` use a 4 kilobyte block size. I have found bs=4k to be the fastest for copying disk drives on a modern machine.

===== Operands =====
Specifies the input path. Standard input is the default.
```
if=file
```

Specifies the output path.
Standard output is the default.
```
of=file
```

Skip this many blocks in the output file.
```
seek=blocks
```

Specifies the input block size in n bytes (default is 512).
```
ibs=n
```

Specifies the output block size in n bytes (default is 512).
```
obs=n
```

If no conversion other than is specified, each input block is copied to the output as a single block without aggregating short blocks.
```
sync, noerror, and, notrunc
```

Specifies the conversion block size for block and unblock in bytes by n (default is 0). If cbs= is omitted or given a value of 0, using the following produces unspecified results.
```
cbs=n
```

This option is used only if ASCII or EBCDIC conversion is specified.
```
block or unblock
```

operands, the input is handled as described for the unblock operand except that characters are converted to ASCII before the trailing SPACE characters are deleted.
```
ascii and asciib
```

operands, the input is handled as described for the block operand except that the characters are converted to EBCDIC or IBM EBCDIC after the trailing SPACE characters are added.
```
ebcdic, ebcdicb, ibm, and ibmb
```

Copies and concatenates n input files before terminating (makes sense only where input is a magnetic tape or similar device).
```
files=n
```

Skips n input blocks (using the specified input block size) before starting to copy. On seekable files, the implementation reads the blocks or seeks past them. On non-seekable files, the blocks are read and the data is discarded.
```
skip=n
```

Seeks n blocks from beginning of input file before copying (appropriate for disk files, where skip can be incredibly slow).
```
iseek=n
```

Seeks n blocks from beginning of output file before copying.
```
oseek=n
```

Skips n blocks (using the specified output block size) from beginning of output file before copying. On non-seekable files, existing blocks are read and space from the current end-of-file to the specified offset, if any, is filled with null bytes. On seekable files, the implementation seeks to the specified offset or reads the blocks as described for non-seekable files.
```
seek=n
```

Copies only n input blocks.
```
count=n
```

Copying options [,value. . . ] Where values are comma-separated symbols from the following list:
```
conv=value
```


Tells `dd` not to abbreviate blocks of all zero value, or multiple adjacent blocks of zeroes, with five asterisks (when you want to maintain size) Do not use notrunc for copying a larger volume to a smaller volume. Without

Do not truncate the output file.
```
conv=notrunc
```

Converts EBCDIC to ASCII.
```
ascii
```

Converts EBCDIC to ASCII using BSD-compatible character translations.
```
asciib
```

Converts ASCII to EBCDIC. If converting fixed-length ASCII records without NEWLINEs, sets up a pipeline with
```
ebcdic
```

===== break4 =====

Slightly different map of ASCII to EBCDIC using BSD-compatible character
translations. If converting fixed-length ASCII records without NEWLINEs, sets up
a pipeline with `dd conv=unblock` beforehand. The
ascii (or asciib), ebcdic (or ebcdicb), and ibm (or ibmb)
values are mutually exclusive. block Treats the input as a sequence of
NEWLINE-terminated or EOF-terminated variable-length records independent of the
input block boundaries. Each record is converted to a record with a fixed length
specified by the conversion block size. Any NEWLINE character is removed from the
input line. SPACE characters are appended to lines that are shorter than their
conversion block size to fill the block. Lines that are longer than the
conversion block size are truncated to the largest number of characters that
will fit into that size. The number of truncated lines is reported. unblock
Converts fixed-length records to variable length. Reads a number of bytes equal
to the conversion block size (or the number of bytes remaining in the input, if
less than the conversion block size), delete all trailing SPACE characters, and
append a NEWLINE character. The block and unblock values are mutually exclusive.

```
lcase
```

Maps upper-case characters specified by the LC_CTYPE keyword tolower to the
corresponding lower-case character. Characters for which no mapping is specified
are not modified by this conversion.

```
ucase
```

Maps lower-case characters specified by the LC_CTYPE keyword toupper to the
corresponding upper-case character. Characters for which no mapping is specified
are not modified by this conversion. The lcase and ucase symbols are mutually
exclusive.

```
swab
```

Swaps every pair of input bytes. If the current input record is an odd number of
bytes, the last byte in the input record is ignored.

```
noerror
```

Does not stop processing on an input error. When an input error occurs, a
diagnostic message is written on standard error, followed by the current input
and output block counts in the same format as used at completion. If the

```
sync
```

conversion is specified, the missing input is replaced with null bytes and
processed normally. Otherwise, the input block will be omitted from the output.
notrunc Does not truncate the output file. Preserves blocks in the output file
not explicitly written by this invocation of dd. (See also the preceding
operand.)

```
of=file
```

```
sync
```

Pads every input block to the size of the ibs= buffer, appending null bytes. (If
either block or unblock is also specified, appends SPACE characters, rather than
null bytes.)

ENVIRONMENT VARIABLES

The following environment variables affect the messages and errors messages of
`dd`:

```
LANG
```

Provide a default value for the internationalisation variables that are unset
or null. If

```
LANG
```
is unset or null, the corresponding value from the implementation-dependent
default locale will be used. If any of the internationalization variables
contains an invalid setting, the utility will behave as if none of the variables
had been defined.

```
LC_ALL
```

If set to a non-empty string value, override the values of all the other
internationalisation variables.

```
LC_CTYPE
```

Determine the locale for the interpretation of sequences of bytes of text data
as characters (for example, single- as opposed to multi-byte characters in
arguments and input files), the classification of characters as upper- or
lower-case, and the mapping of characters from one case to the other.

```
LC_MESSAGES
```

Determine the locale that should be used to affect the format and contents of
diagnostic messages written to standard error and informative messages written
to standard output.

```
NLSPATH
```

Determine the location of message catalogues for the processing of LC_MESSAGES.
it is acknowledged by the author that:

```
dd if=/dev/urandom bs=100 count=1
```

Is faster than

```
dd if=/dev/urandom bs=1 count=100
```

With urandom it is possible to read a byte size of 100. With `/dev/random` you
need to do `bs=1`. `urandom` does not have this restriction.

The `conv=notrunc` part of the `dd` command line is to prevent the output file
from being trucated. The way `dd` behaves is not intuitive. You would think if
you copy a hard drive that `dd` would copy the whole hard drive, but it doesn't.
It will stop when it gets to all unused sectors, and truncate the copy. For
forensic analysis this is unsuitable, as the MD5sum of the working image must be
the same as the original. `notrunc` is a good conversion for copying things.

Although they are part of the `dd` command, I didn't really cover skip or seek.
Seek means to skip that many ouput sized blocks before writing anything. Seek
means to skip that many input sized blocks before reading. For instance, if you
are copying a HDD, and the drives have different MBR's. Then you don't want to
read, or write on the first sector. bs defaults to 512 bytes.

512 bytes is the size of one sector on a hard drive. Partition tables are
written to the MBR. Different drives have different geometries. Modern drives
are all 63 sectors per cylinder with 255 heads. LBA addressing reads all drives
this way. Therefore, partition tables are interchangable between modern drives.
This is only true if the total cylinder count is less on the source than it is
on the target drive. If the source drive contains a partition which will make
the smaller target drive think it has more cylinders that it really does, this
will interfere with making a proper target image. This why, unless the target
drive is bigger than the source, I do not copy the entire drive. But, if you
wish to partition the target first, and skip the MBR on both drives, you can
still have a smaller target than source drive. Here is the command to skip the
MBR, thereby keeping intact different disk geometries.

```
dd if=/dev/sda of=/dev/sdb seek=1 skip=1 conv=notrunc,noerror.
```

When `dd` gets the MAS, or maximum addressable sector on the target, it will
stop. Whatever there was room for will be there.

It should further be noted that `dd` is a bitstream duplicator. It copies bit
for bit, regardless of file system, or anything else.