Generic Kickstart File
======================

This is a generic kickstart file for a minimal install of RHEL 7+ and compatible, on both BIOS as well as UEFI machines.


What are Kickstart Installations?
---------------------------------

Kickstart files contain answers to all questions normally asked by the installation program, such as what time zone you want the system to use, how the drives should be partitioned, or which packages should be installed. Providing a prepared kickstart file when the installation begins therefore allows you to perform the installation automatically, without need for any intervention from the user. This is especially useful when deploying Red Hat Enterprise Linux (RHEL) on a large number of systems at once.

Kickstart files can be kept on a single server system and read by individual computers during the installation. This installation method can support the use of a single kickstart file to install RHEL and compatible on multiple machines, making it ideal for network and system administrators. (`Source <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/chap-kickstart-installations>`_)


How to use
----------

This kickstart file can be used by booting from an ISO file, then either

* BIOS: pressing ``ESC`` on the first screen and providing these cmdline arguments (line breaks are only for better readability):

.. code-block:: text

    boot: linux inst.ks=https://raw.githubusercontent.com/Linuxfabrik/kickstart/main/lf-rhel.cfg
        [lftype=cis|cloud|cloud-cis|minimal]
        [lfdisk=$DISK]
        [ip=[IPADDRESS]::GATEWAY:NETMASK:::none nameserver=NAMESERVER]
        [...]

* UEFI: pressing ``e`` on the "Install ..." entry and appending the following to the ``linuxefi`` cmdline (line breaks are only for better readability), then booting by pressing ``Ctrl-X``.

.. code-block:: text

    linuxefi ... inst.ks=https://raw.githubusercontent.com/Linuxfabrik/kickstart/main/lf-rhel.cfg
        [lftype=cis|cloud|cloud-cis|minimal]
        [lfdisk=$DISK]
        [ip=[IPADDRESS]::GATEWAY:NETMASK:::none nameserver=NAMESERVER]
        [...]

Note that ``ip=`` is an array (for providing multiple IP addresses), so the inner brackets are mandatory.


What this Kickstart File does
-----------------------------

* Supports RHEL 7+ and compatible (See: `Tests <Tests_>`_).
* Works on legacy BIOS as well as UEFI.
* The kickstart file is intended to provide a minimal installation, with ``firewalld`` disabled and SELinux in "Enforcing" mode.
* Can be installed on a user-defined disk by specifying the kernel cmdline argument ``lfdisk=$DISK``. If unset, it tries to find the first block device, in the order ``vda`` > ``sda`` > ``nvme0n1``, and fails otherwise.
* There are two users: ``linuxfabrik`` and ``root``. The root account is always locked by default. This means that the root user will not be able to log in from any console. ``root`` has no password and no SSH keys. Login with the ``linuxfabrik`` user, which is also part of the the ``wheel`` group. ``sudo`` is configured to gain root.

The kickstart file can be used to install different types of minimal installs by setting the kernel cmdline argument ``lftype=``:

.. csv-table::
    :header-rows: 1

    ``lftype=``,             Install Type,   Partitioning Scheme,   Password of User ``linuxfabrik``,   SSH Keys of User ``linuxfabrik``
    ``cis``,                 Minimal,        "CIS, LVM",            ``password``,                       Those of Linuxfabrik
    ``cloud``,               Minimal,        "One partition, LVM",  unset (inject via ``cloud-init``),  none (inject via ``cloud-init``)
    ``cloud-cis``,           Minimal,        "CIS, LVM",            unset (inject via ``cloud-init``),  none (inject via ``cloud-init``)
    ``minimal`` (default),   Minimal,        "One partition, LVM",  ``password``,                       Those of Linuxfabrik

See `Extending the kickstart file <Extending the kickstart file_>`_ on how to define a custom ``lftype``

Useful Kernel Cmdline Arguments
-------------------------------

RHEL:

* ``inst.loglevel=[debug|info]``
* ``inst.ks=[hd:<device>:<path>|[http,https,ftp]://<host>/<path>|nfs:[<options>:]<server>:/<path>`` (MANDATORY)
* ``inst.noverifyssl``: Prevents Anaconda from verifying the ssl certificate for all HTTPS connections ("insecure").
* ``inst.nosave=[<option1>,]<option2>`` (options: ``input_ks,output_ks,all_ks,logs,all``)
* ``inst.rescue``
* `more... <https://anaconda-installer.readthedocs.io/en/latest/boot-options.html>`_

Specific to this kickstart file:

* ``lfdisk=$DISK``: User-defined disk for installing the OS. Default: unset (so tries to find the first block device, in the order ``vda`` > ``sda`` > ``nvme0n1``, and fails otherwise).
* ``lftype``: See table above. Defaults to ``minimal``.


.. _Modifying this kickstart:
Modifying this Kickstart
------------------------

This kickstart includes an additional kickstart ``/tmp/dynamic.ks``. This ``/tmp/dynamic.ks`` file is generated in a kickstart pre-script.
At the beginning of the pre-script the available Python version is determined, the Python script is written to ``/tmp/pre-script.py`` and executed.
The Python script then generates the included ``/tmp/dynamic.ks`` kickstart file.
This Python script contains 5 dictonaries with ``<lftype>`` as key and the values as described below:


* ``Bootloader``: A string with a kickstarter ``bootloader`` command as described at Fedora Kickstart Syntax Reference: `bootloader <https://docs.fedoraproject.org/en-US/fedora/f36/install-guide/appendixes/Kickstart_Syntax_Reference/#sect-kickstart-commands-bootloader>`_)
* ``Packages``: An array with package names/groups to install or exclude. A group is preseded with a ``@`` an exclude is preseded with a ``-``.
* ``Part``: An array of kickstart ``logvol`` lines that define logical volumes for ``<lftype>`` as documented in Fedora Kickstart Syntax Reference: `logvol <https://docs.fedoraproject.org/en-US/fedora/f36/install-guide/appendixes/Kickstart_Syntax_Reference/#sect-kickstart-commands-logvol>`_
* ``Post``: A multiline string containing the postscript for ``<lftype>``. Will be executed by ``/bin/sh``.
* ``Users``: A string containing a ":" separated list in the form ``name="<username>":cmd="<kickstart command used for user creation>":keys="<array of ssh-keys to add>"`` Fedora Kickstart Syntax Reference: `user <https://docs.fedoraproject.org/en-US/fedora/f36/install-guide/appendixes/Kickstart_Syntax_Reference/#sect-kickstart-commands-user>`_, `rootpw <https://docs.fedoraproject.org/en-US/fedora/f36/install-guide/appendixes/Kickstart_Syntax_Reference/#sect-kickstart-commands-rootpw>`_)

Then there is one array containing SSH keys.

* ``lfkeys``: An array of SSH keys that will be installed for either the ``root`` or ``linuxfabrik`` user depending on ``lftype`` as documented above.

Extending the kickstart file
----------------------------

As described in `Modifying this Kickstart <Modifying this Kickstart_>`_ each of these dictonaries can be extended with a new ``<lftype>`` key where the value contains the new ``<lftype>`` ``bootloader`` string, ``package`` array, ``part`` array, ``post`` multi-line string and ``user`` string. Once all these dictonaries contain a valid entry for the new ``<lftype>``, we can issue this new installation using the kernel cmdline argument ``lftype=<lftype>``.

Known Limitations
-----------------

This kickstart file does not work for RHEL 6- (and compatible).

.. _Tests:
Tests
-----

Tested combinations:

* OS: AlmaLinux 8.8, CentOS 7(only BIOS), Fedora 35(only BIOS), 37, 38, RHEL 7(only BIOS), 8, 9, Rocky8, Rocky9
* Firmware: BIOS, UEFI
* Disk: vda, sda
* ``lftype``: ``cis``, ``cloud``, ``cloud-cis``, ``minimal``

What to test within the VM:

* Console login using "root" + "password": Should not work.
* Console login using "linuxfabrik" + "password": Should work on non-cloud. On cloud, password depends on cloud-init.
* ``ip a``: Should get an IP.
* ``ssh root@vm``: Should not work.
* ``ssh linuxfabrik@vm``: Should work on non-cloud. On cloud, it depends on cloud-init.
* ``sudo su -``: Should work.
* ``cat /etc/shadow``: Should show that root's password is locked.
* ``df -hT``: One partition on non-cis, 7 partitions on cis.
* ``lvs``: Should work.
* ``sudo dnf -y install nano``: Should work.
* ``systemctl status cloud-init``: Not found on non-cloud, should work on cloud.
* ``systemctl status firewalld``: Should work.
* ``ll /root``: Should list at least two Anaconda files.


Troubleshooting
---------------

* ``page_poison=1`` kernel cmdline option installed by bootloader cmd can leave the system unbootable due to a buggy UEFI firmware. This was observed with TianoCore firmware on qemu. Remove this option to boot. See https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/8.7_release_notes/known-issues.
* Fedora 38: We observed problems booting into the installer. Try ``inst.neednet=1 rd.debug`` to get to the installer.


References
----------

* `Fedora Kickstart Syntax <https://docs.fedoraproject.org/en-US/fedora/f36/install-guide/appendixes/Kickstart_Syntax_Reference/#sect-kickstart-commands-bootloader>`_
* `RHEL 7 Kickstart Syntax <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/sect-kickstart-syntax>`_
* `RHEL 8 Kickstart Syntax <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/system_design_guide/kickstart_references>`_
* `RHEL 9 Kickstart Syntax <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/performing_an_advanced_rhel_9_installation/kickstart_references>`_
* `Rocky 8 Generic Cloud LVM Kickstart <https://git.resf.org/sig_core/kickstarts/src/branch/r8/Rocky-8-GenericCloud-LVM.ks>`_
* `OpenStack Image Requirements <https://docs.openstack.org/image-guide/openstack-images.html>`_
