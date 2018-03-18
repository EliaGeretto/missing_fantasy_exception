---
title: "Reviving an ASUS Eee PC 1201N"
date: 2018-02-10T11:02:56+01:00
---

A week ago or so, I decided to finally send my main laptop, an ASUS N550JV, to
the repair shop. It has been having a screen flickering problem for a year and
I decided that I had waited long enough.

That said, I had to move all my files and tools into my desktop PC in order to
be able to continue my work. However, that PC did not allow me to fulfill an
important part of my workflow: navigating and watch my TV series on my bed or
on my couch. With that goal in mind, my only option was to try and revive the
netbook I was using something like five years ago, before buying my laptop, an
old ASUS Eee PC 1201N.


## Problem
Unfortunately, I have never been able to make Linux work decently on my old
netbook, which is the main reason why it has been sitting on a shelf for the
best part of the last five years. The problems I encountered were related
mainly to a mixture of old Nvidia proprietary drivers and BIOS incompatibility
problems.

The main issue was that, sometimes, the BIOS would reset without any apparent
reason. This, on the next boot, would make the laptop believe it had only 768
MB of RAM instead of the 3 GB it actually has. Some time ago, I did some
research and discovered that the problem had already been reported
[here][kernel-bugzilla], but it was not solved. A more complete description is
also included: a change in screen brightness and a subsequent reboot would
trigger the BIOS reset. After verifying that it was indeed my case, I decided
it was a kernel bug and I gave up.

[kernel-bugzilla]: https://bugzilla.kernel.org/show_bug.cgi?id=15554 


## DSDT Analysis
Leaving the RAM problem alone for a moment, another issue I had was that on
several distros, out of the box, the volume hotkeys would not work.
Fortunately, thanks to [this][arch-1201n] page on the Arch Wiki, I was able to
fix the problem long ago adding the `acpi_osi=Linux` boot parameter. However, I
never tried to find out what that string was actually doing and so, out of
curiosity, I started looking into it.

[arch-1201n]: https://wiki.archlinux.org/index.php/ASUS_Eee_PC_1201N#1201N_lspci_output

After a little research, I found the answer in a [post][askubuntu] where
basically it is said that the `acpi_osi` parameter forces the kernel to expose
the string `Linux` as part of the `_OSI` object while parsing the DSDT table.

[askubuntu]: https://askubuntu.com/questions/28848/what-does-the-kernel-boot-parameter-set-acpi-osi-linux-do

At this point, I started looking into the ACPI specification, which can be found
[here][acpi-spec], in order to get some more information about what this
actually meant. I discovered that the Differential System Description Table
(DSDT) gives information about the power events on a certain system; it is
implemented using the ACPI Source Language (ASL) by the manufacturer and is
interpreted by the ACPI Linux Subsystem. One of the predefined objects that are
set while parsing the table is the Operating System Interfaces (`_OSI`) object,
which is supposed to identify the operating system parsing the table.

[acpi-spec]: http://www.acpi.info/spec.htm

As described in [this][ubuntu-bios] page of the Ubuntu Wiki, a buggy DSDT can
lead to all sorts of problems, like my reboot and volume keys issues. Being
curious about which values made sense to be assigned to `acpi_osi` in my DSDT,
I decompiled it following the wiki and analyzed it. I was able to understand
that my DSDT checks against the following values for `_OSI` in various
conditional statements:

* Linux
* Windows 2001
* Windows 2001 SP1
* Windows 2001 SP2
* Windows 2006
* Windows 2009

I tried all of them, but the best result was indeed obtained exposing the
`Linux` string. The funny thing is that Linux does not expose the `Linux` string
by default since firmware writers do not test their software using Linux thus
they usually introduce bugs; as a consequence, it is way easier to just emulate
the behavior Windows has.

Even doing all these attempts though, I was not able to fix my BIOS reset
problem.

[ubuntu-bios]: https://wiki.ubuntu.com/BIOSandUbuntu

## BIOS Reset Fix
Scrolling down in the page of the Ubuntu Wiki referenced in the previous
section, I found a "Reboot Methods" section, which caught my attention. Reading
it, I discovered that there are five ways in which the Linux kernel can reboot
the system, switchable with the `reboot` boot option, and thus I wondered if
the one used by default for my system was simply the wrong one, thus causing
the BIOS reset.

As a consequence, I decided to look at the kernel code, in order to understand
which was the default reboot method for my platform. It was quite easy to find
the appropriate code in `arch/x86/kernel/reboot.c`.

Quite surprisingly, I found a series of functions somilar to the following one,
which are apparently meant to set a specific reboot method when called:
{{<highlight C>}}
/*
 * Some machines require the "reboot=a" commandline options
 */
static int __init set_acpi_reboot(const struct dmi_system_id *d)
{
	if (reboot_type != BOOT_ACPI) {
		reboot_type = BOOT_ACPI;
		pr_info("%s series board detected. Selecting %s-method for reboots.\n",
			d->ident, "ACPI");
	}
	return 0;
}
{{</highlight>}}

Looking at where they where used, I found an array containing several entries
which represent special cases for motherboards and laptops that require a
reboot method different from the default one. Going through it, I noticed the
following entries for other Eee PCs, which force the ACPI reboot method.
{{<highlight C>}}
/*
 * This is a single dmi_table handling all reboot quirks.
 */
static const struct dmi_system_id reboot_dmi_table[] __initconst = {
/* ... */
	{	/* Handle problems with rebooting on ASUS EeeBook X205TA */
		.callback = set_acpi_reboot,
		.ident = "ASUS EeeBook X205TA",
		.matches = {
			DMI_MATCH(DMI_SYS_VENDOR, "ASUSTeK COMPUTER INC."),
			DMI_MATCH(DMI_PRODUCT_NAME, "X205TA"),
		},
	},
	{	/* Handle problems with rebooting on ASUS EeeBook X205TAW */
		.callback = set_acpi_reboot,
		.ident = "ASUS EeeBook X205TAW",
		.matches = {
			DMI_MATCH(DMI_SYS_VENDOR, "ASUSTeK COMPUTER INC."),
			DMI_MATCH(DMI_PRODUCT_NAME, "X205TAW"),
		},
	},
/* ... */
	{ }
};
{{</highlight>}}

At this point, I tried to set the `reboot=a` boot option to check if that was
indeed my problem and, finally, my five years old issue was gone.

## Backlight Icon Fix
The second, less significant, fix I applied was introducing the
`acpi_backlight=vendor` boot parameter. This fixes the issue of the backlight
level window not showing up while changing brightness in Ubuntu MATE. I do not
know if this is useful in other desktop environments as well, but it seems to
behave correctly in MATE.

Summarizing, I altered my kernel command line adding the following boot
parameters:

	acpi_osi=Linux acpi_backlight=vendor reboot=a

I am also planning to submit a kernel patch in order to add the 1201N to the
exceptions table in `reboot.c`, so that the last parameter can be dropped.
