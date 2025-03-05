## Overview

This workshop uses VMware as a desktop hypervisor. I chose VMware for the simple fact that I've had some experience with VMware Player in the past. When I originally wrote my three-part blog series titled [Kubernetes on your Laptop](https://paulyu.dev/article/installing-vmware-fusion/), I used a M1 Macbook Pro, so VMware Fusion is used here. If you are using Windows or Linux machines, you can use VMware Workstation Pro.

With the Broadcom acquisition of VMware, the company has updated licensing for its products and both [VMware Fusion and VMware Workstation Pro are now free for personal use](https://blogs.vmware.com/workstation/2024/05/vmware-workstation-pro-now-available-free-for-personal-use.html). In this workshop, we're purely using the platform for study and learning purposes, and I am publishing this workshop as free and open-source content, so using the free version of VMware seems appropriate.

If you're opposed to using VMware products, you can use [VirtualBox]{:target="_blank"}, [Hyper-V]{:target="_blank"} or any other desktop hypervisor that you're comfortable with. The VM creation process will be similar, but the installation steps may vary slightly. The rest of the workshop will remain the same regardless of the hypervisor you choose.

## Download VMware products

Start by signing up for a free Broadcom Support account at [https://access.broadcom.com]{:target="_blank"} then log in and navigate to the [Free Downloads]{:target="_blank"} page.

If you are using a macOS machine with Apple silicon, download and install [VMware Fusion 13.6.2]{:target="_blank"}.

If you are using a Windows or Linux machine, download and install [VMware Workstation Pro 17.6.2]{:target="_blank"}.

!!! warning
    VMware Fusion 13.6.3 and VMware Workstation Pro 17.6.3 are the latest versions as of March 4, 2025. The instructions in this workshop are based on the versions listed above, so you should be using the same versions to avoid any discrepancies. 

## Install VMware products

Installing VMware Fusion or VMware Workstation Pro is straightforward. Simply double-click the downloaded file and follow the on-screen instructions.

!!! note
    Installing VMWare Workstation Pro 17.6.2 on Ubuntu Desktop can be a bit tricky, especially if you are using a UEFI-based system with secure boot enabled. I've ran into a few issues installing on my Ubuntu 24.10 machine and published a blog post titled, [Installing VMware Workstation Pro 17.6.2 on Ubuntu Desktop]{:target="_blank"} that you can follow.

[https://access.broadcom.com]: https://access.broadcom.com/
[Free Downloads]: https://support.broadcom.com/group/ecx/free-downloads
[VMware Fusion 13.6.2]: https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Fusion&freeDownloads=true
[VMware Workstation Pro 17.6.2]: https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Workstation%20Pro&freeDownloads=true
[VirtualBox]: https://www.oracle.com/virtualization/virtualbox/
[Hyper-V]: https://learn.microsoft.com/windows-server/virtualization/hyper-v/get-started/install-hyper-v?pivots=windows-server
[Installing VMware Workstation Pro 17.6.2 on Ubuntu Desktop]: https://paulyu.dev/article/installing-vmware-on-ubuntu-desktop/