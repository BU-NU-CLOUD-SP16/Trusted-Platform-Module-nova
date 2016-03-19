# Adding Trusted Platform Module Support to OpenStack
This project adds Xen VTPM support to OpenStack by contributing changes to Libvirt and Nova Compute Service which allow users to provision VTPMs for their compute instances.

## Setup
Disclaimer: This project requires setting up many different software components that each have their own special ways of breaking. We also incorporate some source code from MIT Lincoln Laboratory which has not been made public. This README is intended more to provide the high level steps to setting up this project and to provide some guidance on some problems faced during deployment.

### Pre-requisites
* Physical server capable of running the Xen Hypervisor under Ubuntu. Our deployment uses a Dell PowerEdge R620 blade server with a Xeon E5-2670.
* The server must also have TPM hardware installed.
* Make sure that the TPM is enabled in your BIOS settings.

### Xen Hypervisor
1. Start by installing Ubuntu onto the server. Our deployment uses Ubuntu 15.10.
2. Install the Xen Hypervisor packages through apt-get.
3. At this point you need to fetch your TPM's public endorsement key (EK) and put it in a safe place, because in the next step we will remove TPM support from the kernel.
3. Build and install a new Linux kernel for Ubuntu and make sure to remove TPM support when configuring your kernel. This is necessary so that the Xen hypervisor can take ownership of the TPM hardware instead of the Ubuntu OS.

### Libvirt
1. Clone our source for a version of Libvirt that supports Xen VTPMs.
2. Configure the build to replace the system binaries and also to build the Xen components.
3. Make and Make Install to deploy the binaries.
4. Set auth_unix_ro, auth_unix_rw, and auth_tls to 'none' in /etc/libvirt/libvirtd.conf to disable Polkit authorization which was giving Nova issues.
```
git clone https://github.com/BU-NU-CLOUD-SP16/Trusted-Platform-Module-libvirt.git
cd libvirt
./autogen.sh --system
make
sudo make install
sudo vi /etc/libvirt/libvirtd.conf
#Disable Polkit under the authorization section of the file
```

### OpenStack
1. Clone the DevStack source
2. Create a minimal local.conf configuration file. The important thing to configure here is to add a line with 'LIBVIRT_TYPE=xen' so that it doesn't use fully virtualized QEMU for a host.
2. Run stack.sh. Take a nap or grab a coffee, this will take 20-30 minutes to setup everything needed to run OpenStack.
3. Add our Nova repository as a remote to the repository at /opt/stack/nova.
4. Checkout and rebase our work onto the latest upstream branch. DevStack is only guaranteed to work completely if ALL of the components are up to date. This will not be deployed to the running instance until you unstack.sh and stack.sh again. Also note, for whatever reason the block storage driver can only be successfully be initialized once out of reset so reboot the machine before you "restack"

```
git clone https://git.openstack.org/openstack-dev/devstack
cd devstack
./stack.sh
sleep $[30*60]
#Git commands to rebase our work upstream
```

### VTPM UUID REST Server
1. Bring up the Xen VTPM Manager domain with the name vtpmmgr. The backend should point to the memory location of the TPM module. Below are some example configurations for these stub domains.
2. Bring up a Xen VTPM. Its backend parameter should be 'vtpmmgr' and its uuid parameter should be a UUID of your choosing.
3. Setup a new guest domain for the UUID Server. Its backend parameter should be the name of the VTPM domain setup in the prior step. At the very least this VM needs to support communicating with a TPM, be on the same subnet as Domain0, and can run our Python REST server.
4. Take ownership of the TPM and set a new SRK and owner password. There's a Perl script from IBM to do this, our mentor has written a Python module for it.
5. Create and activate a new TPM Group (Group1), store its AIK, and note its UUID. All VTPM UUIDs must be registered to Group1 and not Group0 because Group0 is 'special'. Don't create too many groups either because the TPM runs out of NVRAM quick.
2. Inside the /opt/stack/nova/ec500_project directory is a serve_uuid.py script. scp it over to the guest. Write a module to request UUIDs from the VTPM Manager and integrate it with the server.

VTPM Manager Configuration File
```
name="vtpmmgr"
#extra="tpmlocality=2"
extra="loglevel=debug"
kernel="/var/lib/xen/vtpm-kernel/vtpmmgr-46-stubdom.gz"
memory=8
disk=["file:/var/lib/xen/images/vtpmmgr.img,hda,w"]
iomem=["fed40,5"]
```
VTPM Configuration File
```
name="uuid-vtpm"
extra="loglevel=debug"
kernel="/var/lib/xen/vtpm-kernel/vtpm-stubdom.gz"
memory=8
disk=["file:/var/lib/xen/images/uuid-vtpm.img,hda,w"]
vtpm=["backend=vtpmmgr,uuid=698416c2-615f-4ffa-9a55-c04f76966b06"]
```
UUID Server Configuration File
```
name = "uuid_server"
vcpus = '4'
memory = 2048
disk = ['file:/var/lib/xen/images/uuid_server.img,xvda,w']
vif = ['bridge=virbr0 ']
kernel      = '/usr/lib/grub-xen/grub-x86_64-xen.bin'
vtpm        = ['backend=uuid-vtpm']
```

## Usage

### Nova Compute
Our changes make it such that for any compute instance you create with it, Nova will create a VTPM and attach it to your instance. Below are a few major assertions that have been made in our source that should be accounted for.
* The UUID REST server must be reachable at http://keylime_vm:8081
* /var/lib/xen/images/nova_vtpms exists and has rwx permissions for other
* The VTPM Manager domain name is 'vtpmmgr'

If everything has been setup correctly and is running as expected, creating VMs with VTPMs attached is as easy as requesting VMs through any OpenStack interface (Nova REST API, Nova Client, Horizon, etc.). VTPM domains can be observed on the xen layer using "sudo xl list".

### Libvirt XML device definition
The attributes defined in our Libvirt device correspond directly to the attributes defined by the Xen man pages for VTPMs. For VTPMs, the backend parameter is the domain name of the VTPM Manager and the UUID is the UUID that's been registered prior to spawning the VTPM. For VMs that use VTPMs the backend parameter is the domain name of the VTPM you intend to attach to the domain and the UUID should not be defined (It will be passed as a NULL pointer to LibXL). To spawn a VTPM the user must also create a 2MB backing image for the VTPMs emulated NVRAM and other backend things.

```
#Creates a blank 2MB image
dd if=/dev/zero of=vtpm.img bs=1024 count=$[2*1024]

#For a VTPM
<name>my_vtpm</name>
<devices>
  <vtpm backend='vtpmmgr' uuid='698416c2-615f-4ffa-9a55-c04f76966b06' />
</devices>

#For a Guest VM
<devices>
  <vtpm backend='my_vtpm' />
</devices>
```

### VTPM UUID REST Server
Currently, this component simply serves a UUID from the VTPM Manager to its root directory. Any other response hould be treated as an error.
```
GET /
```
Response Body: 
```
{"uuid": "7cacec46-ece6-4f03-a532-d75749e19c91"}
```

## Authors
Gerardo Ravago - gcravago@bu.edu

Yuxin Cao - tonycao@bu.edu

Nick Morrison - nmorrisn@bu.edu

Daniel Pereira - djp927@bu.edu

## Acknowledgements
Nabil Schear

Orran Krieger

Peter Desnoyers

Ata Turk

MOC Staff
