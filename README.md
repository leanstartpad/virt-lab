# Virtual Linux Lab

Install virtual machines on a local KVM hypervisor with cloud-init images, and
then optionally run Ansible [playbooks](playbooks/README.md) to configure the
new guests. Guest sets, called **labs**, are defined in an INI-style
configuration file.  MAC addresses assigned to guests are saved and reused for
the next generation.

`virt-lab` can be useful for spinning up clusters of guests on your local linux
system for testing or development.

`virt-lab` is a wrapper of the very nice [`kvm-install-vm`][1] shell script by
Giovanni Torres to download and install cloud-init images. `kvm-install-vm` is
included in this repo within a git submodule to make installation slightly
easier.

## Usage

    usage: virt-lab <command> [<name>] [<options>]
    
    Commands:
      create <name>     create guests (alias: new)
      destroy <name>    destroy guests (alias: rm)
      start <name>      start guests
      stop <name>       shutdown guests
      info <name>       show lab and guest status (alias: show)
      inventory <name>  generate ansible dynamic inventory
      list              list labs (alias: ls)
      help              show help
      version           show version
    
    Install and remove guests on a local kvm hypervisor using cloud-init images.

## KVM Setup

A local KVM hypervisor must be installed and running along with `libvirt`
before `virt-lib` can be run. An Ansible playbook to install the local KVM
hypervisor and the required libvirt tools is provided in the `kvm` directory.
Be sure you are able to connect to the locally running libvirt daemon, e.g.,
`virsh list --all`, before proceeding.

It is also possible to use Xen as a local hypervisor for virt-lab images. See
the "Advanced" `kvm-install-vm` configuration options for more information.

## Installation

Install `virt-lab` with the following commands, running as a non-root user:

    $ make update
    $ make install

### Ubuntu notes

You may encounter an error message on Ubuntu when attempting create a guest
with a custom disk size:

    - Resizing the disk to 20G ... ERR: Could not resize disk.

This happens because current versions of [Ubuntu][2] distribute linux images as
unreadable by regular users, which breaks the ability of libguestfs to modify
guest images.  This can be fixed by overriding the file permission settings of
the linux kernel images on Ubuntu.

Use the following `dpkg-statoverride` command to make your linux images
readable by non-root users, allowing you to modify your guest images:

    $ for image in /boot/vmlinu*; do sudo dpkg-statoverride --update --add root root 0644 $image || true; done

You will need to override new kernel images *everytime* the kernel is upgraded.

## Configuration

The guest sets, called **labs**, are defined in INI-style configuration file.
`virt-lab` will attempt read the `virt-lab.cfg` file in the current directory,
and if not found, will atempt to read the `virt-lab.cfg` file in the user's
home directory.  Use the `--config` option to specify an alternative
configuration file.

The section names specify the **lab** names available to the `virt-lab`
commands.  Lab names are limited to ASCII alphanumeric characters and the
underscore (`'_'`) character. The options for a given section specify the guest
configuration for each guest in the **lab**.  It is possible to override values
for a specific guest by providing a subsections in the INI file in the form
`<lab>.<number>`. Guest numbers start with 1.

Option names are:

| Option     | Description               | Default |
| ---------- | ------------------------- | ------- |
| desc       | lab description           | none |
| autostart  | enable guest autostart    | true |
| bridge     | bridge interface          | virbr0 |
| cpus       | number of cpus            | 1 |
| disksize   | disk size (GB)            | 10 |
| distro     | distro name               | centos7 |
| domain     | DNS domain                | example.com |
| feature    | cpu model / feature       | host |
| gateway    | virtual network gateway   | 192.168.122.1 |
| graphics   | graphics type             | spice |
| image      | custom image (qcow2)      | none |
| images     | number and type of guests | 3 |
| key        | ssh public key            | `$HOME/.ssh/id_rsa.pub` |
| mac        | mac address               | previous value if one, auto-assigned otherwise |
| memory     | memory (MB)               | 1024 |
| namefmt    | guest name format         | `{lab}{guest:02d}` |
| port       | console port              | auto-assigned |
| postcreate | command run after lab is created | none |
| scriptname | command run after guest is created | none |
| timezone   | timezone name             | US/Eastern |
| user       | additional username       | current login username |
| imagedir   | image directory           | `$HOME/virt/images` |
| vardir     | guest data directory      | `$HOME/virt/var` |
| vmdir      | data directory            | `$HOME/virt/vms` |
| scriptdir  | custom script directory   | `$HOME/virt/scripts` |
| playbookdir | Ansible playbooks        | `$HOME/virt/playbooks` |


## Distributions

The distro option specifies which cload-init image will be downloaded and
installed for a guest.  The following are currently supported:

| Name            | Description                         | Login    |
| --------------- | ----------------------------------- | -------- |
| amazon2         | Amazon Linux 2                      | ec2-user |
| centos7         | CentOS 7                            | centos   |
| centos7-atomic  | CentOS 7 Atomic Host                | centos   |
| centos6         | CentOS 6                            | centos   |
| debian9         | Debian 9 (Stretch)                  | debian   |
| fedora29        | Fedora 29                           | fedora   |
| fedora29-atomic | Fedora 29 Atomic Host               | fedora   |
| fedora30        | Fedora 30                           | fedora   |
| ubuntu1604      | Ubuntu 16.04 LTS (Xenial Xerus)     | ubuntu   |
| ubuntu1804      | Ubuntu 18.04 LTS (Bionic Beaver)    | ubuntu   |

# Playbooks

See the [playbooks](playbooks/README.md) for information on how to run one or
more Ansible playbooks on the set of newly created guests.

# Examples

Here is an example `virt-lab.cfg` configuration file.

    $ cat $HOME/virt-lab.cfg
    [test]
    guests = 6
    desc = My test environment
    distro = centos7
    key =  ~/.ssh/mykey.pub
    domain = example.com
    bridge = vlbr0
    gateway = 192.168.123.1
    # Setup local name resolution for the new guests. (Assumes you have systemd-resolve)
    postcreate = systemd-resolve --interface {bridge} --set-dns {gateway} --set-domain {domain}

    [dev]
    guests = 3
    desc = Guests for development
    distro = debian
    postcreate = ansible-playbook -i {scriptdir}/inventory.sh -i myvars myplaybook.yaml

    [dev.1]
    distro = centos6
    disksize = 20

List the configured labs.

    $ virt-lab list
    - name: test
      desc: My test environment
      guests:
      - test01: centos7, undefined
      - test02: centos7, undefined
      - test03: centos7, undefined
      - test04: centos7, undefined
      - test05: centos7, undefined

    - name: dev
      desc: Guests for development
      guests:
      - dev01: centos6, undefined
      - dev02: debian, undefined
      - dev03: debian, undefined

Create the "test" lab.

    $ virt-lab create test
     [info]: creating guest test01
    - Copying cloud image (CentOS-7-x86_64-GenericCloud.qcow2) ...
    ...
    - DONE
    [info]: creating guest test02
    ...
    - DONE
    [info]: creating guest test03
    ...
    - DONE
    [info]: creating guest test04
    ...
    - DONE
    [info]: creating guest test05
    ...
    - DONE

    $ virt lab list
    - name: test
      desc: My test environment
      guests:
      - test01: centos7, running
      - test02: centos7, running
      - test03: centos7, running
      - test04: centos7, running
      - test05: centos7, running

    - name: dev
      desc: Guests for development
      guests:
      - dev01: centos6, undefined
      - dev02: debian, undefined
      - dev03: debian, undefined


To destroy the guests after the tests have been run.

    $ virt-lab destroy test
    [info]: destroying guest test01
    - Destroying test01 domain ...
    ...
    [info]: destroying guest test05


[1]: https://github.com/giovtorres/kvm-install-vm
[2]: https://bugs.launchpad.net/ubuntu/+source/linux/+bug/759725
