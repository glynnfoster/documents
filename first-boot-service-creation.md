With the introduction of the Automated Installer in Oracle Solaris 11, many administrators will have groaned at the lack of scripting support both in Automated Installer (AI) and IPS packages. This was a deliberate architectural decision from the very start to always ensure a consistent state of the system through use of <a href="">self-assembly techniques</a> using IPS actuators and SMF services. Before then, scripting could do anything to the system, entirely opaque to the underlying packaging system - not such a good thing, particularly when you're trying to upgrade the system to a new version.

When additional customization needs to be done that can't be applied through delivering changes in SMF system configuration profiles or custom IPS packages, typically first boot SMF services are used to kick off additional scripts that allow for changes to be made in a much more controlled environment. There's already some good <a href="">documentation available</a> for how to achieve this, along with a <a href="https://blogs.oracle.com/scottdickson/entry/finish_scripts_and_first_boot">full working example</a> written by Scott Dickson.

With Oracle Solaris 11.4, they've introduced a new utility `svc-create-first-boot` to make this process easier. This utility can be used to create a package that contains a first boot SMF service, that can be published to an existing repository or exported to an IPS p5p archive. Once created, this package can be used in an AI manifest for a given install service. The utility can be run interactively prompting the administrator for information about the first boot service, or can be given command line arguments. Let's work through a simple example of what you can do with this utility.

Let's say I want to set up my Oracle Solaris system to take advantage of <a href="https://www.chef.io">Chef</a>, the popular configuration management framework. Chef isn't yet available in the IPS repository, so let's use a first boot service to pull down Chef, install it, and run a very simple 'Hello World' recipe. Of course this can be extended to getting this system hooked up to a Chef server, but that's left to another blog that focuses directly on Chef.

We've made a very simple script to do this as follows:

```
# cat myscript.sh
#!/usr/bin/bash

# Bring down the Chef client package - happy to see it's already in IPS format!
wget https://packages.chef.io/stable/solaris2/5.11/chef-12.15.19-1.i386.p5p

# Let's now install it on Oracle Solaris
pkg install -g ./chef-12.15.19-1.i386.p5p chef

# As a very simple example, let's create a very simple Hello World recipe and
# then call it using chef-solo, which essentially allows us to run Chef recipes
# without a server being involved

echo '
file "/root/helloworld.txt" do
  owner "root"
  group "root"
  mode 00644
  action :create
  content "Hello World!"
end' > /root/helloworld.rb

# Finally, let's call it using chef-solo
/opt/chef/bin/chef-solo /root/helloworld.rb</pre>
```

We would like to run the above script as soon as the system has been installed using AI during the first boot. We will need an IPS package repository available so we can publish the resulting package, so let's quickly using the `pkgrepo` command to do that:

```
# pkgrepo create myrepo
```

Now let's run the `svc-create-first-boot` utility in interactive mode without any arguments:

```
# svc-create-first-boot
Enter the path to the script to be run at first boot: /root/myscript.sh
Enter the FMRI of the first boot service [svc:/site/first-boot-svc]: svc:/site/setup-chef
Customize dependencies? [yes/No]: no
Should your first boot script be rerun on cloned instances of the installed client? [Yes/no]: no
Enter the method script timeout in seconds [60]: 600
Enter the package FMRI [first-boot-svc]: setup-chef
Enter the publisher name [firstboot]: myorg
Enter the URI to an existing repository or the path to the generated p5p archive []: /root/myrepo
Package published to repo at: /root/myrepo
Publisher name: myorg
Package name: setup-chef
The following is an example xml that can be added to an AI manifest:
<software type="IPS">
  <source>
    <publisher name="solaris">
      <origin name="http://pkg.oracle.com/solaris/release"/>
    </publisher>
    <publisher name="myorg">
      <origin name="<URI OF THE REPOSITORY>"/>
    </publisher>
  </source>
  <software_data action="install">
    <name>pkg:/entire@5.12-5.12.0</name>
    <name>pkg:/group/system/solaris-large-server</name>
    <name>pkg:/setup-chef</name>
  </software_data>
</software>
Note: Please make sure the AI clients can access the URI for the repository
```

From the above output, we can see that we've made a few choices - the path to the script we would like to use, the name of the SMF service we would like, whether it should be run whenever we have created cloned instances using `archiveadm`, the name of the package, and finally the publisher name and location where the IPS package should be published to. You'll notice that we skipped specifying any additional dependencies - the default means that the script will run when the system reaches the `svc:/milestone/multiuser` milestone. You can also see that the script has provided the AI manifest snippet that we will need to use with the appropriate AI service (associated using the `installadm create-manifest` for new manifests, or `installadm update-manifest `commands to edit existing manifests on the AI server).

Now we can take a quick look at what has been published to our repository using the `pkgrepo` command:

```
# pkgrepo -s ./myrepo/ list</b>
PUBLISHER NAME                                          O VERSION
myorg     setup-chef                                      1.0-0:20161103T194510Z
```

We can look a little deeper at the package manifest:

```
# pkgrepo -s ./myrepo/ contents setup-chef
set name=pkg.fmri value=pkg://myorg/setup-chef@1.0,5.12-0:20161103T194510Z
dir group=bin mode=0755 owner=root path=lib
dir group=sys mode=0755 owner=root path=opt
dir group=bin mode=0755 owner=root path=lib/svc
dir group=bin mode=0755 owner=root path=lib/svc/method
dir group=sys mode=0755 owner=root path=lib/svc/manifest
dir group=sys mode=0755 owner=root path=lib/svc/method/site
file 6929819c80976a900e698c0246d484d6e7da01e0 chash=e666d77b7b6bf5bcfd66615c09520f072730352d group=sys mode=0555 owner=root path=lib/svc/method/site/setup-chef_default pkg.chash.sha512t_256=be5a4be67a7354927e444a0b3720a0bdd8c791da2a7f0a5529783d573fc5bf74 pkg.content-hash=file:sha512t_256:df1cceddf6d8ca456a19be757faaeb7547f6b7d71c2fc21793ef18e0223b3028 pkg.csize=1041 pkg.size=2205
dir group=sys mode=0755 owner=root path=lib/svc/manifest/site
file ffedd97dc97da25bf0a3521280248033d2d4b7a2 chash=1daa6c8fbf7cff5ddc4c191584bb0646361dc3e5 group=sys mode=0644 owner=root path=lib/svc/manifest/site/setup-chef_default.xml pkg.chash.sha512t_256=afbc4e1660c10c341a25fde6c846581f636953e21194693d9792c91c08018eda pkg.content-hash=file:sha512t_256:16eb90e4d0f921b87e65826a59827df9bd62b8997db29a64d9db8503f95030d8 pkg.csize=775 pkg.size=2273
dir group=sys mode=0755 owner=root path=opt/svc-create-first-boot
dir group=sys mode=0755 owner=root path=opt/svc-create-first-boot/setup-chef:default
dir group=sys mode=0755 owner=root path=opt/svc-create-first-boot/setup-chef:default/scripts
file da39a3ee5e6b4b0d3255bfef95601890afd80709 chash=89892054d65b8b0dd6a081b33a97b6f2bd1fa267 group=sys mode=0644 owner=root path=opt/svc-create-first-boot/setup-chef:default/svc_completed pkg.chash.sha512t_256=1b57e79330d686247085553792cd8e2fcf61b03edc7da56d03efdb1e1458644a pkg.content-hash=file:sha512t_256:c672b8d1ef56ed28ab87c3622c5114069bdd3ad7b8f9737498d0c01ecef0967a pkg.csize=20 pkg.size=0
file ff0c42d0004650da6541d1af7eccc11b8a0b0e6f chash=484de929efae0981bb2768d833b1ee18449e2631 group=sys mode=0755 owner=root path=opt/svc-create-first-boot/setup-chef:default/scripts/myscript.sh pkg.chash.sha512t_256=33f651c07a39bb5911b418599ba82d1a7d40003c5abd77bcb7487c9ad248705d pkg.content-hash=file:sha512t_256:ae6ccb4e9bf7b47bbed9a9d2607745e4e8c53ca3712225bfdc6cb6bf11ac8a95 pkg.csize=379 pkg.size=612
set name=pkg.summary value="smf(7) service to run myscript.sh script at first boot."
set name=org.opensolaris.smf.fmri value=svc:/site/setup-chef:default
```

We can see that a new SMF service has been created with an SMF manifest installed into `/lib/svc/method/site` that kicks off a method script located in `/opt/svc-create-first-boot/setup-chef:default/scripts/myscript.sh`.

To quickly check that the package does what it's supposed to do, let's quickly install it on this system outside the AI install environment:

```
# pkg install -g /root/myrepo setup-chef
# pkg install -g ./myrepo/ setup-chef
           Packages to install:  1
       Create boot environment: No
Create backup boot environment: No
DOWNLOAD                                PKGS         FILES    XFER (MB)   SPEED
Completed                                1/1           4/4      0.0/0.0      --
PHASE                                          ITEMS
Installing new actions                         17/17
Updating package state database                 Done
Updating package cache                           0/0
Updating image state                            Done
Creating fast lookup database                   Done
Updating package cache                           3/3 
```

Because we're not going through a typical installation, we'll need to do an additional step to associate our new SMF service with the system. We do this by restarting the `svc:/system/manifest-import:default` service instance. This would happen automatically on first boot.

```
# svcadm restart manifest-import
```

Now let's check on our service with `svcs`:

```
# svcs -a | grep site
offline*       13:19:05 svc:/site/setup-chef:default
```

From the above, we can see that the current state is `offline` but because it has an asterisk beside it, it means that the SMF service is transitioning to online - meaning that it is running it's start method script. We can check what this is doing by looking at the SMF logs for this service instance in `/var/svc/log`:

```
# cat /var/svc/log/site-setup-chef\:default.log
[ 2016 Nov  3 13:19:05 Enabled. ]
[ 2016 Nov  3 13:19:05 Executing start method ("/lib/svc/method/site/setup-chef_default"). ]
Copying original boot environment as solaris-4.orig
 Executing first boot script: /opt/svc-create-first-boot/setup-chef:default/scripts/myscript.sh
--2016-11-03 13:19:16--  https://packages.chef.io/stable/solaris2/5.11/chef-12.15.19-1.i386.p5p
Resolving packages.chef.io... 151.101.76.65
Connecting to packages.chef.io|151.101.76.65|:443... connected.
HTTP request sent, awaiting response... 301 Moved Permanently
Location: https://packages.chef.io/files/stable/chef/12.15.19/solaris2/5.11/chef-12.15.19-1.i386.p5p [following]
--2016-11-03 13:19:17--  https://packages.chef.io/files/stable/chef/12.15.19/solaris2/5.11/chef-12.15.19-1.i386.p5p
Reusing existing connection to packages.chef.io:443.
HTTP request sent, awaiting response... 200 OK
Length: 65259520 (62M) [application/octet-stream]
Saving to: "chef-12.15.19-1.i386.p5p"
     0K .......... .......... .......... .......... ..........  0%  501K 2m7s
    50K .......... .......... .......... .......... ..........  0% 1.41M 86s
   100K .......... .......... .......... .......... ..........  0% 2.64M 65s
   150K .......... .......... .......... .......... ..........  0% 1.76M 57s
   200K .......... .......... .......... .......... ..........  0% 2.72M 50s
   250K .......... .......... .......... .......... ..........  0% 2.48M 46s
   300K .......... .......... .......... .......... ..........  0% 3.76M 42s
[snip]
 63500K .......... .......... .......... .......... .......... 99% 1.97M 0s
 63550K .......... .......... .......... .......... .......... 99% 3.03M 0s
 63600K .......... .......... .......... .......... .......... 99% 2.08M 0s
 63650K .......... .......... .......... .......... .......... 99% 3.76M 0s
 63700K .......... .......... ..........                      100% 7.02M=22s
2016-11-03 13:19:39 (2.80 MB/s) - "chef-12.15.19-1.i386.p5p" saved [65259520/65259520]
 Startup: Refreshing catalog &#39;Omnibus&#39; ... Done
 Startup: Refreshing catalog &#39;solaris&#39; ... Done
 Startup: Refreshing catalog &#39;Omnibus&#39; ... Done
Planning: Solver setup ... Done
Planning: Running solver ... Done
Planning: Finding local manifests ... Done
Planning: Fetching manifests: 1/1  100% complete
Planning: Fetching manifests: 1/1  100% complete
Planning: Package planning ... Done
Planning: Merging actions ... Done
Planning: Checking for conflicting actions ... Done
Planning: Consolidating action changes ... Done
Planning: Evaluating mediators ... Done
Planning: Planning completed in 47.86 seconds
           Packages to install:  1
       Create boot environment: No
Create backup boot environment: No
Download:     0/12534 items   0.0/41.0MB  0% complete
Download:  7673/12534 items  23.6/41.0MB  57% complete (5.3M/s)
Download: Completed 40.99 MB in 7.94 seconds (5.2M/s)
 Actions:     1/14709 actions (Installing new actions)
 Actions: 10969/14709 actions (Installing new actions)
 Actions: Completed 14709 actions in 6.60 seconds.
 Done
 Done
 Done
 Done
[2016-11-03T13:21:59-07:00] WARN: *****************************************
[2016-11-03T13:21:59-07:00] WARN: Did not find config file: /etc/chef/solo.rb, using command line options.
[2016-11-03T13:21:59-07:00] WARN: *****************************************
[2016-11-03T13:21:59-07:00] WARN: No cookbooks directory found at or above current directory.  Assuming /var/chef.
[2016-11-03T13:21:59-07:00] WARN: *****************************************
[2016-11-03T13:21:59-07:00] WARN: Did not find config file: /etc/chef/client.rb, using command line options.
[2016-11-03T13:21:59-07:00] WARN: *****************************************
[2016-11-03T13:21:59-07:00] INFO: Started chef-zero at chefzero://localhost:8889 with repository at /var/chef
  One version per cookbook
[2016-11-03T13:21:59-07:00] INFO: Forking chef instance to converge...
[2016-11-03T13:21:59-07:00] INFO: *** Chef 12.15.19 ***
[2016-11-03T13:21:59-07:00] INFO: Platform: x86_64-solaris2.11
[2016-11-03T13:21:59-07:00] INFO: Chef-client pid: 9184
[2016-11-03T13:22:08-07:00] INFO: Run List is []
[2016-11-03T13:22:08-07:00] INFO: Run List expands to []
[2016-11-03T13:22:08-07:00] INFO: Starting Chef Run for solaris_instance
[2016-11-03T13:22:08-07:00] INFO: Running start handlers
[2016-11-03T13:22:08-07:00] INFO: Start handlers complete.
[2016-11-03T13:22:08-07:00] INFO: HTTP Request Returned 404 Not Found: Object not found:
[2016-11-03T13:22:08-07:00] INFO: Loading cookbooks []
[2016-11-03T13:22:08-07:00] WARN: Node solaris_instance has an empty run list.
[2016-11-03T13:22:08-07:00] INFO: Processing file[/root/helloworld.txt] action create (@recipe_files::/root/helloworld.rb line 2)
[2016-11-03T13:22:08-07:00] INFO: file[/root/helloworld.txt] created file /root/helloworld.txt
[2016-11-03T13:22:08-07:00] INFO: file[/root/helloworld.txt] updated file contents /root/helloworld.txt
[2016-11-03T13:22:08-07:00] INFO: file[/root/helloworld.txt] owner changed to 0
[2016-11-03T13:22:08-07:00] INFO: file[/root/helloworld.txt] group changed to 0
[2016-11-03T13:22:08-07:00] INFO: file[/root/helloworld.txt] mode changed to 644
[2016-11-03T13:22:08-07:00] INFO: Chef Run complete in 0.307054495 seconds
[2016-11-03T13:22:08-07:00] INFO: Running report handlers
[2016-11-03T13:22:08-07:00] INFO: Report handlers complete
[ 2016 Nov  3 13:22:09 Method "start" exited with status 101. ]
[ 2016 Nov  3 13:22:09 "start" method requested temporary disable: "First boot configuration completed"]
```

We can see that the script has successfully run, downloaded the Chef package, installed it, and ran the recipe. Once the script finishes, we temporarily disable the SMF service.

```
# cat /root/helloworld.txt
Hello World!
# svcs -a | grep site
disabled       13:22:09 svc:/site/setup-chef:default
```

We can look a little deeper into this SMF service by looking at the manifest:

```
#!/usr/bin/ksh93
#
# Copyright (c) 2015, Oracle and/or its affiliates. All rights reserved.
#
# Method script to execute all first boot scripts in a particular directory
# Load SMF shell support definitions
#
. /lib/svc/share/smf_include.sh
# Global variables
BEADM='/usr/sbin/beadm'
NAWK='/usr/bin/nawk'
TEE='/usr/bin/tee'
#
# First check whether the first boot script has run successfully in the prior
# boot by checking size of
# /opt/svc-create-first-boot//svc_completed file. If size of
# the svc_completed file is greater than zero then script was executed
# in previous boot, so exit the start method and temporarily disable
# the service.
#
service_name=${SMF_FMRI##*/}
flag_file=/opt/svc-create-first-boot/$service_name/svc_completed
[ -s $flag_file ] && smf_method_exit $SMF_EXIT_TEMP_DISABLE method_completed \
        "First boot configuration completed"
#
# Obtain the active BE name from beadm: The active BE on reboot has an R in
# the third column of 'beadm list' output. Its name is in column one.
#
bename=$($BEADM list -Hd|$NAWK -F ';' '$3 ~ /R/ {print $1}')
echo "Copying original boot environment as ${bename}.orig"
$BEADM create ${bename}.orig
#
# Execute the first boot scripts which are stored in
# /opt/svc-create-first-boot//scripts
#
path=/opt/svc-create-first-boot/$service_name/scripts/*
#
# Execute all those files which have the execute bit set. If the execution of a
# script fails, store the error in the logfile and exit the method.
#
for file in $path; do
        if [[ -f $file && -x $file ]]; then 
                echo " Executing first boot script: $file " 
                if ! $file; then 
                        echo "WARNING: Execution of first boot script failed" | $TEE /dev/msglog
                        exit $SMF_EXIT_ERR_FATAL
                fi
        fi
done
#
# Record that this script's work is done. This phase is reached only if
# there has been no error in the execution of the script.
#
echo "1" > $flag_file
#
# Send the exit SMF_EXIT_TEMP_DISABLE to the start method with method completed
# as the short reason and First boot configuration completed as the detailed
# description and temporarily disable the service.
#
smf_method_exit $SMF_EXIT_TEMP_DISABLE method_completed "First boot configuration completed"
```

From the above script, we can see that it checks to see if `/opt/svc-create-first-boot/setup-chef\:default/svc_completed` exists. If it doesn't, it will run our script in `/opt/svc-create-first-boot/setup-chef\:default/scripts`. If it does exist, the service will disable itself again.

So in summary, `svc-create-first-boot` is a useful little utility if you're unfamiliar with how to create IPS packages for your scripts and write SMF services. Longer term, I'd recommend all administrators to get super familiar with IPS and SMF because they are really important in the overall lifecycle management of Oracle Solaris, but as a quick short cut to get you up and running, you can't beat `svc-create-first-boot`. Enjoy!
