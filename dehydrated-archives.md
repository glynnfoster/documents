Unified Archives were first introduced in Oracle Solaris 11.2, enabling users to create a single archive of a system for fast cloning in a cloud environment, or for full backup in the case of a disaster recovery. The technology built on the foundations of Oracle Solaris, including IPS packaging, ZFS file system, and Zones virtualization. The really nice thing was that you could capture a complete bare-metal system, virtualized environments, or a combination of both. Through powerful image transforms, users could quickly deploy a virtualized environment onto a physical bare-metal system, or a bare metal-system into a virtualized environment - flexibility that was really useful in a development, test and production environment.

Needless to say, that Oracle engineers have continued to work on Unified Archives and they're still as important as ever to the Oracle Solaris lifecycle management story. In Oracle Solaris 11.4, they now allow you to create much smaller archives for your golden images using a process call dehydration. While it's certainly true that disk storage has gotten significantly cheaper over the years, but that's no excuse to optimized where you can, and thanks to IPS meta-data we can do exactly that, and remove all the packaged content that you could expect to download from an IPS package repository from the archive. This could be useful when you have a regular backup schedule and you are continuously creating backups on a periodic basis. So let's take a look at what's involved.

Let's look at both a native non-global zone and a kernel zone. Hopefully most administrators will be familiar with creating these - we'll use the most simple example:

```
# zonecfg -z native-zone create
# zoneadm -z native-zone install
# zoneadm -z native-zone boot
# zonecfg -z kernel-zone create -t SYSsolaris-kz
# zoneadm -z kernel-zone install
# zoneadm -z kernel-zone boot
```

We've avoided the output from the zone installation, but typically this involves fetching packages from the IPS package repository and installing them into the zone root filesystem (or copying from the global zone in the case of a pre-cached native non-global zone). In a matter of minutes, we have 2 zones running on our system:

```
# zoneadm list -cv
  ID NAME             STATUS      PATH                         BRAND      IP    
   0 global           running     /                            solaris    shared
  57 native-zone      running     /system/zones/native-zone    solaris    excl  
  58 kernel-zone      running     -                            solaris-kz excl  
```

Now let's start creating a few archives to compare the size differences between a dehydrated archive and a regular archive - in both cases we're using a clone archive, which means that we're only capturing the current boot environment, and that the archive itself is unconfigured in terms of its identity (so that we can generically deploying it with different system identities later).

Let's first capture a clone archive of our native non-global zone and look at the resulting file size:

```
# archiveadm create -z native-zone native-zone-full.uar
Logging to /system/volatile/archive_log.43997
  0% : Beginning archive creation...
  5% : Initializing Unified Archive creation resources...
  5% : Unified Archive initialized: /root/native-zone-full.uar
  6% : Executing dataset discovery...
 10% : Dataset discovery complete
 11% : Executing staging capacity check...
 12% : Staging capacity check complete
 15% : Creating zone media: UnifiedArchive [d5f99ac3-b446-4d66-b56c-8a9cf8e952df]
 52% : CreateZoneMedia: UnifiedArchive [d5f99ac3-b446-4d66-b56c-8a9cf8e952df] complete
 55% : Preparing archive image...
 73% : Archive image preparation complete
 75% : Beginning archive stream creation...
 93% : Archive stream creation complete
 93% : Beginning archive descriptor creation...
 94% : Archive descriptor creation complete
 95% : Beginning final archive assembly...
100% : Archive assembly complete
# du -sh native-zone-full.uar
 1.1G   native-zone-full.uar
```

So we have a pretty small archive - only 1.1Gb. A default native non-global zone uses the `solaris-small-server` package group profile, so it's a pretty small size generally. If we look at the contents of the archive, we'll also see that it includes the AI bootable media. This allows us to be able to standalone install a system with this archive at any stage in the future, which is a nice safety feature of archives (deploying them years after the software has moved on).

```
# archiveadm info -v native-zone-full.uar
Archive Information
          Creation Time:  2016-11-06T23:58:06Z
            Source Host:  oow-openstack-compute-2
           Architecture:  sparc
       Operating System:  Oracle Solaris 12.0 SPARC
  Dehydrated Publishers:  
       Recovery Archive:  No
              Unique ID:  d5f99ac3-b446-4d66-b56c-8a9cf8e952df
        Archive Version:  1.0
Deployable Systems
     'native-zone'
             OS Version:  5.12
              OS Branch:  5.12.0.0.0.106.3
              Active BE:  solaris
                  Brand:  solaris
                  Zones:  
         Installed Size:  1.3GB
              Unique ID:  09351dba-1422-457f-b428-f37175fe422c
               AI Media:  5.12.0_ai_sparc.iso
              Root-only:  Yes
```

Now let's use the `dehydrate` option to the same archive creation:

```
# archiveadm create -z native-zone --dehydrate native-zone-dehydrate.uar
Logging to /system/volatile/archive_log.42387
  0% : Beginning archive creation...
  5% : Initializing Unified Archive creation resources...
  5% : Unified Archive initialized: /root/native-zone-dehydrate.uar
  6% : Executing dataset discovery...
 10% : Dataset discovery complete
 11% : Executing staging capacity check...
 12% : Staging capacity check complete
 15% : Creating zone media: UnifiedArchive [32fe93b4-24ca-431e-ad36-cb1645eb1658]
 52% : CreateZoneMedia: UnifiedArchive [32fe93b4-24ca-431e-ad36-cb1645eb1658] complete
 55% : Preparing archive image...
 73% : Archive image preparation complete
 75% : Beginning archive stream creation...
 93% : Archive stream creation complete
 93% : Beginning archive descriptor creation...
 94% : Archive descriptor creation complete
 95% : Beginning final archive assembly...
100% : Archive assembly complete
# du -sh native-zone-dehydrate.uar
 719M   native-zone-dehydrate.uar
```

719MB, not bad! We've shaved off nearly 400MB using the dehydration. But we can do even better if we want to, by excluding the AI media (if we assume we have an AI server around to deploy this archive, or don't really require a virtual-to-physical transform):

```
# archiveadm create -z native-zone --dehydrate -e native-zone-dehydrate-no-media.uar
Logging to /system/volatile/archive_log.45536
  0% : Beginning archive creation...
  5% : Initializing Unified Archive creation resources...
  5% : Unified Archive initialized: /root/native-zone-dehydrate-no-media.uar
  6% : Executing dataset discovery...
 15% : Dataset discovery complete
 15% : Executing staging capacity check...
 16% : Staging capacity check complete
 20% : Preparing archive image...
 51% : Archive image preparation complete
 56% : Beginning archive stream creation...
 87% : Archive stream creation complete
 87% : Beginning archive descriptor creation...
 89% : Archive descriptor creation complete
 90% : Beginning final archive assembly...
100% : Archive assembly complete
# du -sh native-zone-dehydrate-no-media.uar
  86M   native-zone-dehydrate-no-media.uar
```

86MB, now that is small! Before we go any further, let's try the size experiments on our kernel zone (ignoring the output):

```
# archiveadm create -z kernel-zone kernel-zone-full.uar
# archiveadm create -z kernel-zone --dehydrate kernel-zone-dehydrate.uar
# archiveadm create -z kernel-zone --dehydrate -e kernel-zone-dehydrate-no-media.uar
# du -sh kernel-zone*
  78M   kernel-zone-dehydrate-no-media.uar
  711M  kernel-zone-dehydrate.uar
  1.3G  kernel-zone-full.uar
```

Wow! We can see that we're actually able to get a kernel zone archive down to only 78MB, possibly due to the fact that this is a physical system, and that we're relying a little less on preserving the global zone to native non-global zone relationship and thus being a little more efficient.

Now let's compare the 3 kernel zone archives we've created:

```
# archiveadm info kernel-zone-full.uar
Archive Information
          Creation Time:  2016-11-07T00:13:24Z
            Source Host:  myhost
           Architecture:  sparc
       Operating System:  Oracle Solaris 12.0 SPARC
  Dehydrated Publishers:  
     Deployable Systems:  kernel-zone
# archiveadm info kernel-zone-dehydrate.uar        
Archive Information
          Creation Time:  2016-11-07T00:32:12Z
            Source Host:  myhost
           Architecture:  sparc
       Operating System:  Oracle Solaris 12.0 SPARC
  Dehydrated Publishers:  solaris
     Deployable Systems:  kernel-zone
# archiveadm info kernel-zone-dehydrate-no-media.uar
Archive Information
          Creation Time:  2016-11-07T00:53:43Z
            Source Host:  myhost
           Architecture:  sparc
       Operating System:  Oracle Solaris 12.0 SPARC
  Dehydrated Publishers:  solaris
     Deployable Systems:  kernel-zone
```

From the above, we can see that our two dehydrated archives (one with and one without the AI bootable media embedded in it) rely on IPS publisher information - i.e., where the system can go to get the package content that's missing from the archive. Let's take a look inside the archive to see what we can see. Unified Archives are essentially a USTAR tar archive.

```
# tar -xf kernel-zone-dehydrate-no-media.uar
# ls -1
350e9f38-2f15-4ebf-8e53-39921fe67a1e.ovf
895d6772-e14e-4156-9780-e2c011df97a9-0.zfs.0
kernel-zone-dehydrate-no-media.uar
# cat 350e9f38-2f15-4ebf-8e53-39921fe67a1e.ovf
<?xml version="1.0" encoding="UTF-8"?>
<!-- UnifiedArchive -->
<ovf:Envelope xmlns:vssd="http://schemas.dmtf.org/wbem/wscim/1/cim-schema/2/CIM_VirtualSystemSettingData" xmlns:ua="http://schemas.oracle.com/solaris/unifiedarchive/1" xmlns:rasd="http://schemas.dmtf.org/wbem/wscim/1/cim-schema/2/CIM_ResourceAllocationSettingData" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:ovf="http://schemas.dmtf.org/ovf/envelope/1" xsi:schemaLocation="http://schemas.dmtf.org/ovf/envelope/1 dsp8023_1.1.xsd http://schemas.oracle.com/solaris/unifiedarchive/1 unifiedarchive.xsd http://schemas.dmtf.org/wbem/wscim/1/cim-schema/2/CIM_VirtualSystemSettingData CIM_VirtualSystemSettingData.xsd http://schemas.dmtf.org/wbem/wscim/1/cim-schema/2/CIM_ResourceAllocationSettingData CIM_ResourceAllocationSettingData.xsd">
  <ovf:References>
    <ovf:File ovf:id="rpool@895d6772-e14e-4156-9780-e2c011df97a9" ovf:href="895d6772-e14e-4156-9780-e2c011df97a9-0.zfs" ovf:size="81864093" ovf:chunkSize="8589934591" ovf:compression="gzip" ua:space="2399733861" ua:zpool="rpool" ua:zpoolVersion="39" ua:zfsMaxFilesystemVersion="6"/>
  </ovf:References>
  <ua:UnifiedArchive ovf:id="uainfo">
    <ovf:Info>Unified Archive Metadata</ovf:Info>
    <ua:Version>1.0</ua:Version>
    <ua:UUID>350e9f38-2f15-4ebf-8e53-39921fe67a1e</ua:UUID>
    <ua:ArchiveType>clone</ua:ArchiveType>
    <ua:CreationTime>2016-11-07T00:53:43Z</ua:CreationTime>
    <ua:CreationHost>oow-openstack-compute-2</ua:CreationHost>
    <ua:OriginTarget><target name="origin">
  <disk in_zpool="rpool" in_vdev="rpool-none" whole_disk="true">
    <disk_name ctd="c0t5000CCA02F187484d0" devpath="/scsi_vhci/disk@g5000cca02f187484" devid="id1,sd@n5000cca02f187484" receptacle="/SYS/DBP/HDD0" name="/SYS/DBP/HDD0" name_type="receptacle"/>
    <disk_prop dev_type="scsi" dev_vendor="HGST" dev_size="1172123568secs"/>
    <disk_keyword key="boot_disk"/>
    <gpt_partition name="0" action="create" force="false" part_type="solaris">
      <size val="1172106895secs" start_sector="256"/>
    </gpt_partition>
  </disk>
  <logical noswap="false" nodump="false">
    <zpool name="rpool" action="create" is_root="true" is_boot="false" mountpoint="/rpool">
      <vdev name="rpool-none" redundancy="none"/>
    </zpool>
  </logical>
</target>
</ua:OriginTarget>
    <ua:Dehydration>True</ua:Dehydration>
    <ua:DehydratedPublishers>solaris</ua:DehydratedPublishers>
    <ua:RehyStagingSize>6.39gb</ua:RehyStagingSize>
  </ua:UnifiedArchive>
  <ovf:VirtualSystemCollection ovf:id="ua1">
    <ovf:Info>Clone Archive</ovf:Info>
    <ovf:VirtualSystem ovf:id="kernel-zone" ua:uuid="895d6772-e14e-4156-9780-e2c011df97a9" ua:isa="sparc" ua:activeBE="solaris" ua:deployBE="fb22db79-661a-4753-980c-768e5b8126ed" ua:brand="solaris-kz" ua:trusted="False">
      <ovf:Info>global zone</ovf:Info>
      <ovf:Name>kernel-zone</ovf:Name>
      <ovf:VirtualHardwareSection ovf:id="kernel-zone" ovf:transport="ua">
        <ovf:Info>Hardware description of source system</ovf:Info>
        <ovf:System>
          <vssd:ElementName>Virtual System Type</vssd:ElementName>
          <vssd:InstanceID>0</vssd:InstanceID>
          <vssd:VirtualSystemType>kernel-zone</vssd:VirtualSystemType>
        </ovf:System>
        <ua:ZoneConfiguration ua:dtdName="-//Sun Microsystems Inc//DTD Zones//EN" ua:dtdLocation="file:///usr/share/lib/xml/dtd/zonecfg.dtd.1">
          <ovf:Info>Zone configuration data</ovf:Info>
          <zone name="kernel-zone" brand="solaris-kz" hostid="0x65fd0d61" autoboot="false" ip-type="exclusive" boot-priority="normal">
  <mcap physcap="4294967296" pagesize-policy="largest-available"/>
  <virtual-cpu ncpu_min="4" ncpu_max="4"/>
  <automatic-network configure-allowed-address="true" auto-mac-address="2:8:20:e7:b8:79" id="0" mac-address="auto" lower-link="auto" link-protection="mac-nospoof" iov="off" lro="auto" ring-group="auto"/>
  <device storage="dev:/dev/zvol/dsk/%{global-rootzpool}/VARSHARE/zones/%{zonename}/disk%{id}" allow-partition="false" allow-raw-io="false" bootpri="0" id="0"/>
</zone>
        </ua:ZoneConfiguration>
      </ovf:VirtualHardwareSection>
      <ovf:OperatingSystemSection ovf:id="29">
        <ovf:Info>Specifies the operating system installed</ovf:Info>
        <ovf:Description>Oracle Solaris 12.0 SPARC</ovf:Description>
      </ovf:OperatingSystemSection>
      <ua:ZFSPool ua:name="rpool" ua:bootPool="true">
        <ovf:Info>root zpool</ovf:Info>
        <ua:VDev ua:layout="none" ua:type="root">
          <ua:Device ua:name="/dev/dsk/c1d0s0"/>
        </ua:VDev>
        <ua:Property ua:name="version" ua:value="39"/>
        <ua:Property ua:name="size" ua:value="15.9G"/>
        <ua:Property ua:name="bootfs" ua:value="rpool/ROOT/solaris"/>
        <ua:ZFSStream ua:fileRef="895d6772-e14e-4156-9780-e2c011df97a9-0.zfs"/>
      </ua:ZFSPool>
      <ua:PkgImage ua:osversion="5.12" ua:osbranch="5.12.0.0.0.106.3">
        <ovf:Info>pkg(7) image</ovf:Info>
        <ua:PkgVariant ua:variant="variant.arch" ua:value="sparc"/>
        <ua:PkgVariant ua:variant="variant.opensolaris.zone" ua:value="global"/>
        <ua:PkgFacet ua:facet="facet.locale.*" ua:value="False"/>
        <ua:PkgFacet ua:facet="facet.locale.de" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.de_DE" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.en" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.en_US" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.es" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.es_ES" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.fr" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.fr_FR" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.it" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.it_IT" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.ja" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.ja_*" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.ko" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.ko_*" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.pt" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.pt_BR" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.zh" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.zh_CN" ua:value="True"/>
        <ua:PkgFacet ua:facet="facet.locale.zh_TW" ua:value="True"/>
        <ua:PkgPublisher ua:publisher="solaris" ua:sticky="true" ua:enabled="true">
          <ovf:Info>publisher origin</ovf:Info>
          <ua:PkgPublisherOrigin ua:uri="http://10.134.6.21/solaris12/dev/"/>
        </ua:PkgPublisher>
      </ua:PkgImage>
      <ua:EmbeddedZones>
        <ovf:Info>Zones that are configured or installed</ovf:Info>
      </ua:EmbeddedZones>
    </ovf:VirtualSystem>
  </ovf:VirtualSystemCollection>
</ovf:Envelope>
```

There's an awful lot to absorb in the above, but a Unified Archive contains an Open Virtualization Format descriptor (essentially detail about the archive), along with a ZFS stream. We can see in the descriptor that we have some IPS publisher information to be able to rehydrate the archive and the approximate size required to rehydrate. Let's take a closer look at the ZFS stream by receiving it into a dataset that we'll create.

```
# zfs create rpool/archive
# cat 895d6772-e14e-4156-9780-e2c011df97a9-0.zfs.0 | gunzip | zfs receive -F -u -d rpool/archive
# zfs list | grep archive
rpool/archive                                                      238M   150G  73.5K  /rpool/archive
rpool/archive/ROOT                                                 237M   150G    31K  none
rpool/archive/ROOT/fb22db79-661a-4753-980c-768e5b8126ed            237M   150G   136M  /
rpool/archive/ROOT/fb22db79-661a-4753-980c-768e5b8126ed/var        101M   150G   101M  /var
rpool/archive/export                                                64K   150G    32K  /export
rpool/archive/export/home                                           32K   150G    32K  /export/home
# zfs set -r mountpoint=/rpool/archive/ROOT rpool/archive/ROOT
# zfs mount rpool/archive/ROOT/fb22db79-661a-4753-980c-768e5b8126ed
# zfs mount rpool/archive/ROOT/fb22db79-661a-4753-980c-768e5b8126ed/var
# cd /rpool/archive/ROOT/fb22db79-661a-4753-980c-768e5b8126ed
```

Now that we've mounted the root filesystem or our archive, let's run a check on how many directories, files, hard and soft links there are:

```
# find . -type d -exec echo dirs \; -o -type l -exec echo symlinks \; -o -type f -links +1 -exec echo hardlinks \; -o -type f -exec echo files \; | sort | uniq -c
6740 dirs
6156 files
  20 hardlinks
6068 symlinks
```

Browsing into `usr/bin` on that mount we can see that it's entirely symlinks. Let's now compare this to a non-dehydrated archive (after clearing up the previous mounts):

```
# tar -xf kernel-zone-full.uar
# ls -1
40446eda-3586-4796-a78f-d99315e1a88b-0.zfs.0
5.12.0_ai_sparc.iso
dbdb1dd1-9a74-4bde-8ef5-1da465ce47d5.ovf
kernel-zone-full.uar
# cat dbdb1dd1-9a74-4bde-8ef5-1da465ce47d5.ovf | grep DehydrationFalse
# cat 40446eda-3586-4796-a78f-d99315e1a88b-0.zfs.0 | gunzip | zfs receive -F -u -d rpool/archive
# zfs set -r mountpoint=/rpool/archive/ROOT rpool/archive/ROOT
# zfs mount rpool/archive/ROOT/2727e87a-0ec4-4f2e-910a-94cbea497a13
# zfs mount rpool/archive/ROOT/2727e87a-0ec4-4f2e-910a-94cbea497a13/var
# cd /rpool/archive/ROOT/2727e87a-0ec4-4f2e-910a-94cbea497a13
# find . -type d -exec echo dirs \; -o -type l -exec echo symlinks \; -o -type f -links +1 -exec echo hardlinks \; -o -type f -exec echo files \; | sort | uniq -c
5717 dirs
72174 files
 782 hardlinks
6068 symlinks
```

We can immediately see a significant increase in terms of files included in this archive. We also notice that our achive also contained an ISO image for the AI bootable media when we unpacked it.

So what about deploying these dehydrated archives. Deploying a native non-global zone archive is pretty straightforward, assuming that the package versions included in the archive are consistent with that with the global zone - for this we use the `-a` option on the branded zone install options:

```
# zoneadm -z new-native-zone install -a native-zone-dehydrate-no-media.uar
The following ZFS file system(s) have been created:
    rpool/VARSHARE/zones/new-native-zone
Progress being logged to /var/log/zones/zoneadm.20161107T020406Z.new-native-zone.install
       Image: Preparing at /system/zones/new-native-zone/root.
 Install Log: /system/volatile/install.24607/install_log
 AI Manifest: /tmp/manifest.new-native-zone.JK0PKd.xml
  SC Profile: /usr/share/auto_install/sc_profiles/enable_sci.xml
    Zonename: new-native-zone
Installation: Starting ...
        Commencing transfer of stream: 822a138a-6493-43dc-874c-0b85f5e73743-0.zfs to rpool/VARSHARE/zones/new-native-zone/rpool
        Completed transfer of stream: '822a138a-6493-43dc-874c-0b85f5e73743-0.zfs' from file:///root/ARCHIVES/native-zone-dehydrate-no-media.uar
        Archive transfer completed
Installation: Succeeded
Updating image format
Image format already current.
Updating non-global zone: Linking to image /.
Updating non-global zone: Syncing packages.
No updates necessary for this image. (zone:new-native-zone)
Updating non-global zone: Zone updated.
Result: Attach Succeeded.
 done.
        Done: Installation completed in 381.257 seconds.
  Next Steps: Boot the zone, then log into the zone console (zlogin -C)
              to complete the configuration process.
Log saved in non-global zone as /system/zones/new-native-zone/root/var/log/zones/zoneadm.20161107T020406Z.new-native-zone.install
# zoneadm -z new-native-zone boot
```

And we're up and running almost immediately. We really didn't have to do much work since there's a linked image dependency between the global zone and non-global zone. Deploying into a kernel zone ends up being a little bit trickier, because from a direct installation point of view, a kernel zone requires an ISO image for the AI bootable media (if not installing using an AI server which we won't cover in this article).

```
# zoneadm -z newzone install -a ./native-zone-dehydrate-no-media.uar
Progress being logged to /var/log/zones/zoneadm.20161107T020305Z.newzone.install
ERROR: Archived zone native-zone has no AI media
zoneadm: zone 'newzone': ERROR: installation failed: zone switching to configured state
```

For this, we'll need to create bootable from the archive using the `create-media` option to `archiveadm`:

```
# archiveadm create-media -f iso kernel-zone-dehydrate-no-media.uar
Logging to /system/volatile/archive_log.31326
  0% : Beginning media creation...
  7% : Transferring AI source...
 25% : Transfer AI source complete
 29% : Adding archive content...
 66% : Add archive content complete
 69% : Creating ISO image...
100% : ISO image creation complete
# du -sh AI_Archive.iso
  1.3G   AI_Archive.iso
```

Once we have the bootable media, we can use the `-b` option to the kernel zone install options:

```
# zoneadm -z new-kernel-zone install -b ./AI_Archive.iso
Progress being logged to /var/log/zones/zoneadm.20161107T035901Z.new-kernel-zone.install
[Connected to zone 'new-kernel-zone' console]
NOTICE: Entering OpenBoot.
NOTICE: Fetching Guest MD from HV.
NOTICE: Starting additional cpus.
NOTICE: Initializing LDC services.
NOTICE: Probing PCI devices.
NOTICE: Finished PCI probing.
SPARC T7-2, No Keyboard
Copyright (c) 1998, 2015, Oracle and/or its affiliates. All rights reserved.
OpenBoot 4.38.2, 4.0000 GB memory available, Serial #1057722244.
Ethernet address 0:0:0:0:0:0, Host ID: 3f0b8f84.
Boot device: disk1  File and args: - install aimanifest=prompt
/
SunOS Release 5.12 Version s12_111 64-bit
Copyright (c) 1983, 2016, Oracle and/or its affiliates. All rights reserved.
Remounting root read/write
Probing for device nodes ...
Preparing image for use
Done mounting image
Configuring devices.
Hostname: solaris
Discovering automated install services...
No services discovered.
Enter AI manifest [], [d]efault, [r]estart discovery, or [?]:
d
Downloading file:///.cdrom/auto_install/default.xml
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2731  100  2731    0     0   463k      0 --:--:-- --:--:-- --:--:--  463k
solaris console login:  
Automated Installation started.
The progress of the Automated Installation will be output to the console.
Detailed logging is in the logfile at /system/volatile/install_log.
Press RETURN to get a login prompt at any time.
solaris console login: 
Automated Installation started.
The progress of the Automated Installation will be output to the console.
Detailed logging is in the logfile at /system/volatile/install_log.
Press RETURN to get a login prompt at any time.
06:05:55    Install Log: /system/volatile/install_log
06:05:55    Using XML Manifest: /system/volatile/ai.xml
06:05:55    Using profile specification: /system/volatile/profile
06:05:55    Starting installation.
06:05:55    0% Preparing for Installation
06:05:56    100% manifest-parser completed.
06:05:56    0% Preparing for Installation
06:05:56    1% Preparing for Installation
06:05:56    2% Preparing for Installation
06:05:57    3% Preparing for Installation
06:05:57    4% Preparing for Installation
06:05:57    7% install-env-configuration completed.
06:05:58    10% target-discovery completed.
06:06:01    Pre-validating manifest targets before actual target selection
06:06:01    Selected Disk(s) : c1d0
06:06:01    Pre-validation of manifest targets completed
06:06:01    Validating combined manifest and archive origin targets
06:06:01    Selected Disk(s) : c1d0
06:06:01    10% target-selection completed.
06:06:01    11% ai-configuration completed.
06:06:02    11% var-share-dataset completed.
06:06:05    11% target-instantiation completed.
06:06:06    11% Beginning archive transfer
06:06:06    Commencing transfer of stream: 895d6772-e14e-4156-9780-e2c011df97a9-0.zfs to rpool
06:06:10    15% Transferring contents
06:06:11    Completed transfer of stream: '895d6772-e14e-4156-9780-e2c011df97a9-0.zfs' from file:///.cdrom/archive.uar
06:06:12    19% Transferring contents
06:06:16    Archive transfer completed
06:06:17    90% generated-transfer-980-1 completed.
06:06:19    Rehydrating archive. This operation may take a while
06:10:03    Rehydration complete.
06:10:03    90% apply-pkg-variant completed.
06:10:05    Setting boot devices in firmware
06:10:05    Setting openprom boot-device
06:10:05    Openprom boot-device value set to /kz-devices@ff/disk@0:a disk1
06:10:05    91% boot-configuration completed.
06:10:05    91% update-dump-adm completed.
06:10:05    92% setup-swap completed.
06:10:06    92% device-config completed.
06:10:06    92% apply-sysconfig completed.
06:10:06    92% transfer-zpool-cache completed.
06:10:15    98% boot-archive completed.
06:10:21    98% update-filesystem-owner-group completed.
06:10:21    98% transfer-ai-files completed.
06:10:22    98% cleanup-archive-install completed.
06:10:22    100% create-snapshot completed.
06:10:22    Automated Installation succeeded.
06:10:22    You may wish to reboot the system at this time.
Automated Installation finished successfully
The system can be rebooted now.
Please refer to the /system/volatile/install_log file for details.
After reboot it will be located at /var/log/install/install_log
Reboot to start the installed system.
[NOTICE: Zone halted]
[Connection to zone 'new-kernel-zone' console closed]
        Done: Installation completed in 2650.332 seconds.
# zoneadm -z new-kernel-zone boot
```

Now there is a gotcha in the above, that the kernel zone needs to be appropriately networked and have access to an IPS package repository to be able to rehydrate the archive on deployment. For a default kernel zone, an automatic VNIC is created and looks for that address using DHCP. If you statically define your kernel zones, you may want to consider using the `allowed-address` zone configuration to define an IP during zone creation:

```
# zonecfg -z new-kernel-zone
zonecfg:new-kernel-zone> select anet 0
zonecfg:new-kernel-zone:anet> set allowed-address=10.1.1.8/24
zonecfg:new-kernel-zone:anet> end
zonecfg:new-kernel-zone> verify
zonecfg:new-kernel-zone> commit
zonecfg:new-kernel-zone> exit
```