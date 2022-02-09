# zram-hibernate

Allows dynamic swap changes to activate disk-based storage as swap for hibernation support when a system typically uses only zram swap during normal operation.

Information about integration with systemd:

https://blog.christophersmart.com/2016/05/11/running-scripts-before-and-after-suspend-with-systemd/

Short summary: add script to /usr/lib/systemd/system-sleep/ that supports pre/post arguments.  More details available in systemd-suspend.service manpage.

There's likely a way to integrate with openrc as well, possibly using apcid or something like that, but I haven't needed to do so yet.
