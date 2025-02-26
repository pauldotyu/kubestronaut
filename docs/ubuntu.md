## Download Ubuntu Server Image

First thing we need to do is download the Ubuntu Server image based on your machine's architecture:

- For amd64, go [here](https://releases.ubuntu.com/22.04/) and download the server install image
- For arm64, go [here](https://cdimage.ubuntu.com/releases/jammy/release/) and download the server install image

## Install Ubuntu Server on VMware Fusion

As mentioned in the previous section, I am using a M1 Macbook Pro, so I will be using VMware Fusion to create and run virtual machines on my laptop. If you are using VMware Workstation, the steps should be similar.

## Create a New Virtual Machine

Open VMware Fusion and double-click the **Install from disc or image** box in the center of the window.

![Install method](https://assets.paulyu.dev/articles/2024/vmware-fusion-install/6-install-method.png)

In the **Create a New Virtual Machine** window, drag and drop the Ubuntu Server image you downloaded into the **Drag a disc image here** section. Click **Continue**.

![New virtual machine](https://assets.paulyu.dev/articles/2024/vmware-fusion-install/7-new-vm.png)

In the **Finish** window, click the **Customize Settings** button to further configure the virtual machine settings, but for the purpose of our local Kubernetes cluster installation, the default settings should be sufficient. Click **Finish** and then give your VM a name and click **Save**. I named mine `k8s-control.vmwarevm` to distinguish it as the control plane node.

![Finish virtual machine](https://assets.paulyu.dev/articles/2024/vmware-fusion-install/8-finish.png)

## Ubuntu Server Installation

The virtual machine will start and you will be presented with the Ubuntu Server installer. Follow the on-screen instructions to install Ubuntu Server on the virtual machine.

![Installer option](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/1-install.png)

Let's go through the installation process step by step:

### Language

Select the language you want to use during the installation process and press the return key to proceed to the next step.

![Language option](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/2-language.png)

### Installer update

You will be presented with a notice that an installer update is available. We want to stick with the current installer so select **Continue without updating** and press the return key to proceed to the next step.

![Update option](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/8-update.png)

### Keyboard layout

Review your keyboard Layout and Variant, and press the return key to proceed to the next step.

![Keyboard option](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/3-keyboard.png)

### Choose type of install

Keep the options as they are and press the return key to proceed to the next step.

![Intall type option](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/4-type.png)

### Network connections

Wait for the network to configure itself and display an IP range for **DHCPv4** and press the return key to proceed to the next step.

![Network option](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/5-network.png)

### Configure proxy

Keep the proxy settings blank and press the return key to proceed to the next step.

![Network proxy option](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/6-proxy.png)

### Configure Ubuntu archive mirror

Wait for the installer to find the best mirror. Once you see "This mirror location passed tests." press **Enter**.

![Ubuntu mirror test](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/7-mirrors.png)

### Guided storage configuration

Keep the default settings then tab through to **Done** and press the return key to proceed to the next step.

![Storage options](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/9-storage.png)

### Storage configuration

Review the storage configuration and press the return key to proceed to the next step.

You will be warned about a "destructive action", tab through to **Continue** and press the return key to proceed to the next step.

![Storage configuration](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/10-storage-confirm.png)

### Profile setup

Enter your name, server name, username, and password then tab through to **Done** and press the return key to proceed to the next step.

![Profile](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/11-profile.png)

### Ubuntu to Ubuntu Pro

Keep the default settings and press the return key to proceed to the next step.

![Ubuntu Pro option](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/12-ubuntu-pro.png)

### SSH Setup

Press the space bar to select the **Install OpenSSH server** option then tab through to **Done** and press the return key to proceed to the next step.

![SSH Server option](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/13-ssh-server.png)

> ⚠️ This part is critical. We need this to be able to SSH into the virtual machine from the host machine (your laptop).

### Featured Server Snaps

Keep the default options and tab through to **Done** and press the return key to proceed to the next step.

![Server Snaps option](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/14-snaps.png)

### Installing system

You should see the installation logs as the system is installed. This will take several minutes to complete.

![Installer progress](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/15-install.png)

### Install complete

Once the installation is complete, tab through to **Reboot Now** and press the return key to reboot.

![Installer complete](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/16-reboot.png)

> You will be prompted to hit the **Enter** key to complete the reboot process.

### Post-installation

Once the system is rebooted, log into the virtual machine using the username and password you set up during the installation process.

You will see the IPv4 address that has been assigned to your virtual machine. Make a note of this as we will need it to SSH into the virtual machine from the host machine.

![Login to VM](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/18-login.png)

Using the IP address, SSH into the virtual machine from the host machine.

```bash
ssh paul@192.168.120.130
```

When prompted, type `yes` to add the virtual machine to the list of known hosts, enter your password, and you should be logged into the virtual machine.

![SSH login from host](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/19-ssh-login.png)

Congratulations! You now have an Ubuntu Server virtual machine running on VMware Fusion.

![Logged in using SSH](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/20-ssh-logged-in.png)

## Clone a worker node

Take a snapshot of the control node virtual machine so that you don't have to go through the installation process again. To take a snapshot, go to **Virtual Machine** > **Snapshots** > **Take Snapshot**. You should give your snapshot a name like "Before kubeadm init" so that you can easily identify it in the future.

![Take a snapshot](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/21-take-snapshot.png)

Shut down the virtual machine by clicking **Virtual Machine** > **Shut Down**.

![Shutdown the VM](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/23-shutdown.png)

View your snapshots by going to **Virtual Machine** > **Snapshots** > **Snapshots**.

![View your snapshots](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/22-view-snapshot.png)

Right click on the current state snapshot and click **Create Full Clone**.

![Clone the VM from a snapshot](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/24-clone.png)

Give your clone a new name. I named mine `k8s-worker`.

After the clone is complete, power on the k8s-control then the k8s-worker virtual machines and log in.

> The network uses DHCP, so make sure you power on k8s-control first so that it can be re-assigned the same IP. The other option is to set static IPs on your machines.

![Power on and log in](https://assets.paulyu.dev/articles/2024/ubuntu-server-install/25-worker.png)

You will notice the hostname is the same as the control node. We will need to change this with the following command.

```bash
sudo hostnamectl hostname worker
```

Verify the hostname has been changed by running the following command.

```bash
hostname
```

You can power down this virtual machine and take a snapshot so that you have a clean worker node to clone from in the future. Remember to give your snapshot a name like "Before kubeadm join" so that you can easily identify it in the future.
