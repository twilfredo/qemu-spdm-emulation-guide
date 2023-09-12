# Testing SPDM over DOE on an Emulated NVME device with QEMU

This guide will walkthrough setting up a basic linux image, emulating an
NVMe device (PCIe) and testing SPDM over DOE all using QEMU. With everything setup,
we will also take a look at using the Kernel SPDM implementation (ongoing work)
to authenticate the NVMe device.

## Getting Started

We need:

- Linux Kernel
- Linux Userspace/FS
- QEMU
- SPDM-emu

Notes: There is an in progress effort to add an SPDM requester support to Linux,
it's at a stage where it can be used as a requester to authenticate a device.
Note that there maybe bugs as it's a work in progress. If you don't have a
userspace SPDM requester to test the NVMe responder with. Using `l1k`'s fork
with the kernel implementation is a good start.

Lets setup an isolated working directory:

```shell
$ mkdir qemu-spdm
$ cd qemu-spdm

# Linux Upstream
$ git clone https://github.com/torvalds/linux.git
# OR Linux Kernel with SPDM Requester Support (In development)
$ git clone https://github.com/l1k/linux.git
# We are using a fork of QEMU, but SPDM over DOE patches will be upstreamed soon
$ git clone https://github.com/twilfredo/qemu.git
$ git clone --recursive https://github.com/DMTF/spdm-emu.git
```

### Build Linux

We can start by compiling the kernel, we are going to be building an `x86-64`
basic kernel. But `aarch64` *should* also work (although not tested).

#### Building Upstream Kernel
```shell
$ cd linux
$ make defconfig
# Setup .config for QEMU
$ make kvm_guest.config
```

#### Building kernel SPDM support development branch
When building the kernel with SPDM support
```shell
$ cd linux
$ git checkout lukas/doe
$ make defconfig
# Setup .config for QEMU
$ make kvm_guest.config 
```

For both upstream and SPDM kernels:
In the .config file, we need to enable NVME and some debug settings (optional).

- CONFIG_BLK_DEV_NVME=y
- CONFIG_KCOV=y
- CONFIG_CONFIGFS_FS=y
- CONFIG_SECURITYFS=y

For the SPDM kernel to enable CMA/SPDM Requester set:
- CONFIG_PCI_CMA=y

```shell
$ make olddefconfig
```

To make things easier, you can use the attached `.config` (this is missing
`CONFIG_PCI_CMA=y`) file in this repo. Just copy and paste it in the
`linux` directory.

Now to build it with
```shell
$ make -j32 # Use appropriate number of threads
```

### Build QEMU

Now to build QEMU, from the `qemu-spdm` top directory

```shell
$ cd qemu
# Checkout the SPDM NVME Device support branch
$ git checkout wilfred/spdm-a
$ mkdir build; cd build
# We are using SLIRP for networking (you will need libslirp-devel installed)
$ ../configure --target-list=x86_64-softmmu --enable-slirp 
$ make -j32
```

### Setting up a Linux Userspace

We are going to setup a filesystem using, the [syskaller build script](https://github.com/google/syzkaller/blob/master/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md). It's a
convenient way to setup a userspace.

We need to have `debootstrap` installed

```shell
$ sudo dnf install debootstrap
```
Now to build the userspace/fs
```shell
$ cd qemu-spdm/linux
# You can make this elsewhere if preferred
$ mkdir image; cd image
# Get the build script
$ wget https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh -O create-image.sh
```
We are going to run this with some options

- distribution=bookworm  # Upto date glibc/pcilib maybe required for userspace DoE requesters (so use the latest debian image)
- feature=full           # More features
- seek=32768             # Image size (32GB)

```shell
$ chmod +x create-image.sh
$ ./create-image.sh --distribution bookworm --feature full --seek 32768
```
this will take awhile to cook. Go for a coffee run.

### Setting up SPDM-emu

The SPDM implementation depends on `spdm-emu` or known to QEMU as `OpenSPDM`.
Let's build and set this up.

```shell
$ cd qemu-spdm/spdm-emu
$ git submodule init; git submodule update --recursive
$ mkdir build; cd build
$ cmake -DARCH=x64 -DTOOLCHAIN=GCC -DTARGET=Debug -DCRYPTO=openssl ..
$ make -j32
$ make copy_sample_key # Build certificates, required for SPDM authentication.
```

### Now to setup an emulated NVME drive for QEMU

We need create a file that QEMU can use as the NVMe namespace for the emulated
NVMe device.

```shell
$ cd qemu-spdm/linux/image
$ dd if=/dev/zero of=blknvme bs=1M count=2096 # 2GB NNMe Drive
```

## Running everything through QEMU

We have everything setup. But QEMU depends on `spdm-emu` for it's SPDM
implementation. So let's start that first

```shell
$ cd qemu-spdm/spdm-emu/build/bin
# Must be ran from within `/bin` to keep the relative path to the certs accurate
$ ./spdm_responder_emu --trans PCI_DOE

spdm_responder_emu version 0.1
trans - 0x2
context_size - 0x2458
Platform server listening on port 2323
```

Now to run QEMU, create a `run.sh` for convenience

```shell
$ cd qemu-spdm/linux
$ vim run.sh
```
Copy the following in:
```bash
#!/bin/bash

echo "Starting QEMU..."

../qemu/build/qemu-system-x86_64 \
        -m 4G \
        -smp 8 \
        -kernel $1/arch/x86/boot/bzImage \
        -append "console=ttyS0 root=/dev/sda earlyprintk=serial net.ifnames=0 nokaslr" \
        -drive file=$2/bookworm.img,format=raw \
	-drive file=$2/blknvme,if=none,id=mynvme,format=raw \
	-device nvme,drive=mynvme,serial=deadbeef,spdm=2323 \
	-net user,host=10.0.2.10,hostfwd=tcp:127.0.0.1:10021-:22 \
        -net nic,model=e1000 \
        -enable-kvm \
        -nographic \
        -pidfile vm.pid \
	-machine q35 \
        2>&1 | tee vm.log
```

To summarise, we are virtualizing a `q35` machine (this is the only `x86-64`
machine type to support PCIe). The machine emulates an NVMe controller that
points `blknvme` (file we made) for it's namespace storage. The `spdm=2323`
option to the NVMe device tells us on which port to look for `spdm-emu-responder`.
`spdm-emu` currently uses port `2323`.

If `spdm-emu` is not started prior to QEMU, QEMU will fail to start with an
appropriate error.

Finally run:
```shell
$ chmod u+x run.sh
$ ./run.sh . image
...
....
....
Debian GNU/Linux 12 syzkaller ttyS0

syzkaller login:
```

You can login with `root` as the user.

### SSH to guest

From the host, you can SSH to the guest for a better console experience with
```shell
$ cd qemu-spdm/linux
$ ssh -i image/bookworm.id_rsa -p 10021 -o "StrictHostKeyChecking no" root@localhost
```

### Install tools

First install some tools

```shell
$ apt install nvme-cli pciutils
```

Check that the NVMe device is detected with
```shell
$ nvme list
Node                  Generic               SN                   Model                                    Namespace Usage                      Format          
--------------------- --------------------- -------------------- ---------------------------------------- --------- -------------------------- ----------------
/dev/nvme0n1          /dev/ng0n1            deadbeef             QEMU NVMe Ctrl                           1           2.20  GB /   2.20  GB    512   B +  0 B 
```

Check that the device shows DOE support with.

```shell
lspci -vvv | grep DOE
...
		DOECap: IntSup+
		DOECtl: IntEn-
		DOESta: Busy- IntSta- Error- ObjectReady-
```

That's it for setting up, we now have a functional NVMe drive that emulates
SPDM (Responder device). You can now start testing against the device. The next
section discusses using the kernel SPDM requester to test this. For this to work,
make sure you built and use the correct kernel from (`l1k`'s linux fork).

## Testing the Kernel SPDM Implementation

At boot, the kernel will attempt to authenticate the NVME responder, and likely
fail as we haven't specified root certificates for the slot0..N in the `_cma`
keyring. Verify this with

```shell
$ dmesg | grep SPDM
[    1.212647] pci 0000:00:03.0: SPDM: Root certificate for slot 0 not found in _cma keyring: DMTF libspdm ECP384 CA
[    1.397389] pci 0000:00:03.0: SPDM: Root certificate for slot 1 not found in _cma keyring: DMTF libspdm ECP384 CA
[    1.582627] pci 0000:00:03.0: SPDM: Root certificate for slot 2 not found in _cma keyring: DMTF libspdm ECP384 CA
```

Now, in user-space we can attempt to re-authenticate after adding
the required certificates to the `_cma` keyring.

You can copy across certificates from the host to the vm using scp, for example:

```shell
scp -i image/bookworm.id_rsa -P 10021 -o "StrictHostKeyChecking no" -rp /some/where/certs/ root@localhost:/root/
```
And now on the VM let's add a certificate to the `_cma` keyring. We are going to
use `keyctl` for this. Can be installed with `apt-get install keyutils` on the VM.

First find the keyring id/desc,
```shell
$ cat /proc/keys/
...
.
0f229c7d I------     1 perm 1f0f0000     0     0 keyring   _cma: empty
.
...
```
Now add the certificate to the keyring `0x0f229c7d` with.
```shell
$ keyctl padd asymmetric ""  0x0f229c7d < ca.cert.der
868775432

# See certs in the keyring with
$ keyctl show 0x0f229c7d
Keyring
 253926525 ---lswrv      0     0  keyring: _cma
 868775432 --als--v      0     0   \_ asymmetric: NAME: VAL
```

We can now attempt to re-authenticate with, ensure that you have the correct
path to the NVMe PCI device (`0000:00:03.0` in this case).

```shell
echo 1 > /sys/devices/pci0000\:00/0000\:00\:03.0/authenticated
```

This will start the authentication process.