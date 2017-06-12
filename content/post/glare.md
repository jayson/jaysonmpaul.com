+++
date = "2017-06-09T20:06:32-04:00"
title = "Glare: Lightweight Replacement for Glance"
project_url = "https://github.com/jayson/glare.git"
tags = [
    "bash",
    "virualization",
    "openstack",
    "projects",
]
draft = false

+++

# Glance

About a year ago I was tasked with upgrading our [Virtual Machine tooling](https://jaysonmpaul.com/post/continuous-integration-at-etsy-2/) from CentOS 6 to Centos 7. We use KVM and QEMU for our developer VMs and had a very lightweight installation of [OpenStack Glance](https://www.openstack.org/software/releases/ocata/components/glance) for serving our base images used during creation and bootstrap. In true CentOS fashion, the version of glance that shipped with it was very outdated: [2012.1.1](https://github.com/openstack/glance/releases/tag/2012.1.1).

Not surprisingly, the version that came with CentOS 7 was much newer but still hopelessly out of date: [2015.1.0](https://github.com/openstack/glance/releases/tag/2015.1.0). This didn't stop the entire glance infrastructure from being entirely re-built. What was once just a single package install and a small MySQL database had ballooned into 15+ packages and there was no way to upgrade the configuration. Once I got glance installed that wasn't the end. It apparently required I set up half of OpenStack as well including [Keystone Auth](https://www.openstack.org/software/releases/ocata/components/keystone).

After spending a few days trying to get all the requisite packages installed and configured for what amounted to a few scripts to uploading images and an http hosted image repo, I thought to myself "Couldn't I just write this in bash?". At this point I was somewhat frustrated with glance so I made the name a play off the name. With that, [Glare](https://github.com/jayson/glare) was born. 

# Building Glare


I started off by creating a script for adding a new image to the repository. We didn't need the tons of metadata the new Glance supported, just the image, a message and max number of images of that type to store. Pair this with apache and wget and I should have a working image store and cache. Seems easy enough!

## Adding images

```bash
IMAGE_SHORTNAME=$(basename $IMAGE)

MD5=$(md5sum $IMAGE | cut -d' ' -f1)
echo "Uploading $IMAGE as $IMAGE_SHORTNAME-$MD5"
mkdir -p $ARCHIVE_DIR
pv $IMAGE > $ARCHIVE_DIR/$IMAGE_SHORTNAME-$MD5
echo "Setting current image to uploaded..."
ln -sf $DIRECTORY/archive/$IMAGE_SHORTNAME-$MD5 $DIRECTORY/$IMAGE_SHORTNAME
echo "Writing changelog..."
touch $DIRECTORY/$IMAGE_SHORTNAME.md
echo "# [$MD5] - $(date)"$'\n'"## $MESSAGE"$'\n\n'"$(cat $DIRECTORY/$IMAGE_SHORTNAME.md)" > $DIRECTORY/$IMAGE_SHORTNAME.md
echo "Success!"
```

This takes the name of the file you are uploading as the image name, adds a unique identifier and places it in your configured image root in an archive folder. It keeps a symlink of latest image available at the top level. It also creates a small markdown metadata file including the message specified.

I added some nice option and configuration parsing,  loading configurations first from /etc/glare.conf, then ~/.glare.conf and lastly passed from the command line. I also implemented a maximum number of old images to keep so I didn't fill the disk. [See full script](https://github.com/jayson/glare/blob/master/bin/glare-add-image).

## Building Index

We had previously written a small python script for querying the glance api and pulling down the latest images on each of our host machines for building new VMs locally. With glare I wanted to keep this as simple as possible so I used apache to serve up the image root directory, but really any flavor of web server could be used here.

Since I was just serving up a directory of files via HTTP, I needed a way to "query" what image types were available.

```bash
(cd $DIRECTORY ; find . -type l -exec basename {} \; > $DIRECTORY/images.txt)
```
Simple yet effective! Now I had an images.txt that I could curl in my caching script that looked like this

```
cent6_vm_template
cent6_vanilla_vm_template
cent7_vm_template
```

See [full script here](https://github.com/jayson/glare/blob/master/bin/glare-rebuild-index)

## Caching Images

Great! The server side of this was complete and now I just needed something that cached all available images on every host that we built VMs on. First grab a list of all available image types and then download the images to a local cache directory. I knew wget had mirroring capability so I opened up the man page to learn some of the more obscure wget params:

```bash
IMAGE_LIST=$(curl $SERVER/images.txt)
for image in $IMAGE_LIST; do (cd $DOWNLOAD_DIR && wget -nv -nH --cut-dirs=1 --unlink -N $SERVER/$image); done
```

1. ```-nv``` for non-verbose script output
2. ```-nH``` to remove glare.website.com from the directory stucture
3. ```--cut-dirs=1``` to cut the image name down down to the base name with no directories
4. ```-N``` to turn off timestamping and keep server timestamps

### What about ```--unlink```?

When implementing this solution, I found that if a wget was running when I tried to make a new VM, it would get created with a half downloaded image. I needed to get a little tricky to maintain [Atomicity](https://en.wikipedia.org/wiki/Atomicity_(programming)) of my downloads. Even if I downloaded the images to a download directory and moved them into place there is a possibility of a race condition with large files. Here is what I came up with:

1. Create a hard link for all files in the download directory into the cache directory. This causes there to only ever be a single image on disk if there are no updates to download.

```bash
for i in $(ls $TARGET_DIR); do ln -f $TARGET_DIR/$i $DOWNLOAD_DIR/$i; done
```

2. wget with --unlink parameter so that if a newer image exists on the remote server it will undo the hard link before replacing it, creating another copy so new VMs would get that while it's downloading
3. Update the hard links for all files in the download directory to the cache directory to instantly update images with a single disk operation

```bash
for i in $(ls $DOWNLOAD_DIR); do ln -f $DOWNLOAD_DIR/$i $TARGET_DIR/$i; done
```

With this neat trick I am also able to delete the download directory every sync removing clutter since it is recreated with hard links on every run (including timestamp so wget knows what is new).

```bash
# Create template directory
mkdir -p $TARGET_DIR
# Create temporary download dir
mkdir -p $DOWNLOAD_DIR
# Hard link all current copies of templates (used later with wget option --unlink to provide atomic downloads)
for i in $(ls $TARGET_DIR); do ln -f $TARGET_DIR/$i $DOWNLOAD_DIR/$i; done
IMAGE_LIST=$(curl $SERVER/images.txt)
# Download images that are out of date unlinking ones it clobbers without changing anything in live template directory
for image in $IMAGE_LIST; do (cd $DOWNLOAD_DIR && wget -nv -nH --cut-dirs=1 --unlink -N $SERVER/$image); done
# Atomically hard link all changed images
for i in $(ls $DOWNLOAD_DIR); do ln -f $DOWNLOAD_DIR/$i $TARGET_DIR/$i; done
# Clean up download dir
rm -rf $DOWNLOAD_DIR
```

See [full script with option parsing](https://github.com/jayson/glare/blob/master/bin/glare-sync)

## Command Line Usability

Having these three separate commands worked just fine but I wanted to emulate the behavior of git where ```git-anything``` can also be called by invoking ```git anything```. Git also doesn't care where the subcommands live as long as they are in your path. Git isn't written in bash so I came up with something myself:

```bash
GLARE_PATH="$(dirname $(readlink -f $0))/"

PATH_FOR_PLUGINS="$GLARE_PATH:$PATH"
SUB_COMMAND=$(PATH="$PATH_FOR_PLUGINS" which glare-$1 2>/dev/null)
RETURN_VAL=$?

if [ 0 -eq $RETURN_VAL ]; then
    shift
    $SUB_COMMAND "$@"
else
    SUB_CMDS=$(IFS=:; find $PATH_FOR_PLUGINS -maxdepth 1 -executable -type f -name 'glare-*' -printf '%f\n' 2>/dev/null | cut -d'-' -f2-1000 | sort | uniq)
    echo "Invalid glare command $1"
    echo "Valid subcommands:"
    for i in $SUB_CMDS; do 
        SHORT_USAGE=$(PATH=$PATH_FOR_PLUGINS glare $i -h 2>/dev/null | sed -e 's/Usage.*glare-/glare /')
        if [ 0 -eq $? -a -n "$SHORT_USAGE" ]; then
            echo $SHORT_USAGE;
        else
            echo "glare $i"
        fi
    done
    exit 1
fi
```

This takes the current path glare is being run from as the highest level path heirarchy and adds it to your ```$PATH```. Then it searches for all ```glare-*``` executables in this path and will ferry along all the parameters to the right executable. Now I was able to run ```glare sync``` or ```glare add-image``` instead of ```glare-add-image```. This wasn't a necessary feature, I just thought it added a nicer user experience and allows others to plug into glare for their own additions.

I also added a [Makefile](https://github.com/jayson/glare/blob/master/Makefile) for installing my scripts. It supports installing to /usr/local/ or any prefix you specify. I added a snarky [README](https://github.com/jayson/glare/blob/master/README.txt) about Keystone Auth and called it a day.
