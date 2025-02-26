This workshop uses VMware products for local virtualization. I chose VMware for the simple fact that I've had some experience with VMware Player in the past. When I originally wrote my three-part blog series titled [Kubernetes on your Laptop](https://paulyu.dev/article/installing-vmware-fusion/), I was on macOS so VMware Fusion is used here. If you are using Windows or Linux machines, you can use VMware Workstation Pro.

With the Broadcom acquisition of VMware, the company has updated licensing for its products and both VMware Fusion and VMware Workstation Pro are now free for personal use. In this workshop, we are purely using the platform for study and learning purposes, and this workshop and study guide are free and open-source, so the free version is sufficient. If you are using VMware products for commercial purposes, you will need to purchase a license.

If you're opposed to using VMware products, you can use [VirtualBox]{:target="_blank"}, [Hyper-V]{:target="_blank"} or any other virtualization platform that you're comfortable with. The steps will be similar, but the screenshots will be different.

## Download VMware products

Sign up for a free Broadcom Support account at [https://access.broadcom.com]{:target="_blank"} then log in and navigate to the [Free Downloads]{:target="_blank"} page.

If you are using a macOS machine, download and install [VMware Fusion 13.x]{:target="_blank"}.

If you are using a Windows or Linux machine, download and install [VMware Workstation Pro 17.x]{:target="_blank"}.

## Install VMware products

Installing VMware Fusion or VMware Workstation Pro is straightforward. Simply double-click the downloaded file and follow the on-screen instructions.

!!! warning
    Installing VMWare Workstation Pro on Ubuntu Desktop can be a bit tricky, especially if you are using a UEFI-based system with secure boot enabled. I've ran into a few issues installing on my Ubuntu 24.10 machine and published a blog post titled, [Installing VMware Workstation Pro on Ubuntu Desktop]{:target="_blank"} that you can follow.

[https://access.broadcom.com]: https://access.broadcom.com/
[Free Downloads]: https://support.broadcom.com/group/ecx/free-downloads
[VMware Fusion 13.x]: https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Fusion&freeDownloads=true
[VMware Workstation Pro 17.x]: https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Workstation%20Pro&freeDownloads=true
[VirtualBox]: https://www.oracle.com/virtualization/virtualbox/
[Hyper-V]: https://learn.microsoft.com/windows-server/virtualization/hyper-v/get-started/install-hyper-v?pivots=windows-server
[Installing VMware Workstation Pro on Ubuntu Desktop]: https://paulyu.dev/article/installing-vmware-on-ubuntu-desktop/