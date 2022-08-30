# Nyx VMs

This guide will introduce the option of using a full-blown virtual machine as a base image for fuzzing. In the past, we already used this mode to fuzz different parts of the Windows Kernel, Linux, macOS and FreeBSD. This mode also allowed us to get more complex targets running in a Nyx virtual machine, such as games (like Counter-Strike) or even web browsers like Firefox. Once the target is executed in this mode, you can use either source-level instrumentation or Intel PT for fuzzing.

## Nyx VM Running Modes

Nyx supports two running modes to provide a virtualized environment in which the target application is executed. The first and probably most commonly used mode is called "Kernel Running Mode" in Nyx. Both a Linux kernel bzImage and a small RAMFS image are used to boot a virtual machine in that mode. No virtual hard disks are emulated in this mode, and everything is executed from the initial boot RAM disk. This provides some performance benefits compared to the other running mode. Moreover, the packer repository already contains a pre-build busy box RAMFS image, which is suitable to run most Linux CLI programs. 

But at the same time, this mode has several limitations: first, since this is just a busy box image, more sophisticated subsystems are not present, which a target application might rely on. In particular, a program might expect several `systemd` components, an X-Server or any other UI environment, or even another operating system than Linux.

Another limitation is that everything is stored on a RAM disk, which means that huge applications like web browsers (especially when built with several debug options enabled) are not supported due to limited space on that RAM disk. For these cases, Nyx offers another running mode (called "Snapshot") which expects a typical KVM / QEMU virtual machine image and turns that into a base image for fuzzing. 

The snapshot running mode in Nyx is obviously more powerful because it is not limited to one operating system or a specific OS configuration (like subsystems, which must be present to get a target application running). But at the same time, this mode is more complex to set up, and due to the emulation of virtual hard drives, there will be an additional performance overhead.

## Terminology 

Nyx uses different types of snapshots. The following table briefly introduces the 3 most important ones:

| Nyx Snapshot Types   |                                                              |
| -------------------- | ------------------------------------------------------------ |
| Pre-Snapshot         | This snapshot stores the entire delta up from the beginning of the boot time until the snapshot is created. A Pre-Snapshot is usually created by the loader agent program. This snapshot type is only used in Snapshot running mode. In the Kernel running mode, the agent is part of the init script and is thus automatically executed during the boot process. But in this running mode, no pre-snapshot will be created. |
| Fuzzing-Snapshot     | A Fuzzing Snapshot is created with the first input request by the guest. Depending on various running options, the fuzzer will usually always jump back to this state after each execution has finished. |
| Incremental-Snapshot | The concept of incremental snapshots is described in detail in our Paper Nyx-Net. For now, this feature is only used by Nyx-Net, but it can easily be adapted to other fuzzers. This snapshot type stores another delta of the time between the Fuzzing Snapshot and the time the incremental snapshot is created (like all other snapshot types, the creation of this snapshot is initiated by the guest via hypercalls). |

Note that the old kAFL `savevm` and qcow2 overlay snapshot format is no longer supported. 


## Howto: Use Ubuntu 22-04 LTS as a Nyx VM image for fuzzing

In the following section, we will present how to install and prepare a QEMU / KVM virtual machine, which we will later use as a Nyx base image for fuzzing. Like with any other virtual machine, you need to manually install and prepare everything as you would normally do. This includes installing an OS and the subsequent setup of other components (like installing specific packages). This guide provides step-by-step instructions for Ubuntu 22-04 LTS. 

**Note**: If you want to use the packer utility to prepare the target, use the same Linux distribution (including the same version of it) as on the host system to avoid issues with different library versions. The following guide can also be applied to other Linux distributions.

First, we need to compile QEMU-Nyx and check out a compatible version of the Nyx packer utility:

```bash
git clone git@github.com:nyx-fuzz/QEMU-Nyx.git
git clone git@github.com:nyx-fuzz/packer.git

cd QEMU-Nyx
./compile_qemu_nyx.sh static
cd -

cd packer/packer
# run the following command once to create a nyx.ini config file
python3 python nyx_packer.py > /dev/null 
cd -

# run the following commands to compile the agent (more on that later)
cd packer/packer/linux_x86_64-userspace/
make
cd -
```

Please keep in mind that the Nyx snapshot file format might have changed and, therefore, be incompatible with the latest version. This is especially true for Nyx-Net, which still uses an old version of QEMU-Nyx. To avoid any issues, please make sure you are using a compatible version of QEMU-Nyx and the packer by checking out the same commit version as used by the Git submodules of Nyx-Net , kAFL or AFL++. 

### OS Install 

Next, we create and move into a dedicated directory containing the base image, the install image, and the Nyx snapshot folder. Here, we create a new VM image file (which represents the virtual hard disk of our fuzzing VM) and download the install image: 

```bash 
mkdir ubuntu
cd ubuntu

# create a VM image file (15GB in size)
../packer/qemu_tool.sh create_image ubuntu.img $((1024*15))

# download Ubuntu 22.04 LTS
wget https://releases.ubuntu.com/22.04/ubuntu-22.04.1-desktop-amd64.iso
```

You can use the following command to start the virtual machine and begin with the OS install. This command will also open a VNC port on port 5900. Note that this port is not limited to the loopback device.

```bash
../packer/qemu_tool.sh install ubuntu.img ubuntu-22.04.1-desktop-amd64.iso
```

Once the VM is launched, use a VNC viewer (`tigerVNC` works best for this purpose) to get through the OS install.

Once the installation process has finished, reboot the VM and install the following packages via `apt` in the VM. You can either use `install` or `post_install` mode if you need to restart the VM. 

```bash
sudo apt-get update && \
sudo apt-get install openssh-server vmtouch
```

OpenSSH will ease file transfer from the host to the guest. But you can also use any other tool to copy files into the virtual machine. Next, the utility program `vmtouch` allows it to prefetch files and entire directories, such that data is not fetched from the hard disk during fuzzing. Nyx supports snapshots with `read` and `write` accesses to emulated hard disks. Still, these accesses are notoriously slow (due to device emulation) -- something you wouldn't notice in normal use cases because the data is usually cached in RAM by the kernel after the first access. If data is fetched from emulated hard disks on each iteration of the fuzzing loop, it might have an impact on the overall performance. To avoid that, you can use `vmtouch` to prefetch all essential application files before the fuzzing snapshot is created.

Next, shutdown the VM gracefully and start the VM in `post_install'mode: 

```bash
../packer/qemu_tool.sh post_install ubuntu.img
```

The VM's port 22 (SSH) in this mode is redirected to the host's port 2222. You can use the following `scp` command to transfer files from the host into the VM. In our case, we copy the Nyx agent into the VM (depending on the chosen user name, the command might need to be adjusted accordingly):

```bash
scp -P 2222 packer/packer/linux_x86_64-userspace/bin64/loader user@localhost:/home/user/
```

Once the loader executable is copied into the VM, everything is ready at this point. You can now use SSH (`ssh -p 2222 user@localhost`) or VNC to shutdown the VM.


### Create a Nyx Pre-Snapshot

In the final step, we use the `create_snapshot` mode of the `qemu_tool.sh` utility program to create the pre-snapshot. No network devices are emulated in this mode, and write accesses to the hard disks are cached in memory and not stored in the image file (and later saved in a dedicated file as part of the snapshot). 

```bash
# start the VM with a RAM size of 2048 MB 
../packer/qemu_tool.sh create_snapshot ubuntu.img 2048 ./nyx_snapshot/
```

Connect via VNC and launch the agent program in the virtual machine (preferably as root):

```bash
sudo ./loader
```

At this point, the snapshot is created, and once that process is finished, the VM and QEMU-Nyx are automatically terminated. The snapshot folder should now contain the following files: 

```bash 
$ ls nyx_snapshot/
fast_snapshot.mem_dump fast_snapshot.qemu_state fs_drv_ubuntu.img.khash global.state ready.lock
fast_snapshot.mem_meta fs_cache.meta fs_drv_ubuntu.img.pcow INFO.txt
```

### Prepare a Nyx Fuzzer Configuration

To use our VM image and the pre-snapshot as a base image, we need to modify the `nyx.ini` file first and then generate a new configuration file. Set the following options in the `../packer/packer/nyx.ini` file and point both to the image file and snapshot folder: 

```bash
default_vm_hda = /home/user/ubuntu/ubuntu.img
default_vm_presnapshot = /home/user/ubuntu/nyx_snapshot/
```

Now we can simply generate a new configuration. Instead of the `Kernel` option, which we usually use, we chose the `Snapshot` option. Also, don't forget to specify the RAM size used (in our case 2048MB): 

```bash
python3 ../packer/packer/nyx_config_gen.py <target-shardir> Snapshot -m 2048
```

At this point, you can now use Nyx-Net or AFL++ to start fuzzing.


## Performance Considerations:

We strongly recommend prefetching as much application data as possible to avoid as many IOPS as possible (remember that these will re-occur after every execution due to snapshot resets). This is especially important for huge targets such as web browsers with multiple application files (fonts, images, etc.). To do so, you can simply use `vmtouch` to prefetch data between the time the pre-snapshot is loaded and the creation of the fuzzing snapshot. Then, simply add the following line to the `fuzz.sh` script (or `fuzz_no_pt.sh` if you are running Nyx without KVM-Nyx): 

`vmtouch -t <folder>`

Another simple yet effective optimization is to avoid input writes to the file system and instead use a RAM disk for that. So, instead of configuring the packer with the following options

```bash
python3 nyx_packer.py <...> -args `/tmp/input` -file `/tmp/input`
```

we can use the `/dev/shm` directory for that. By doing so, we can gain a significant performance boost. 
