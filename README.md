# Nextcloud_Proxmox_TrueNAS


Guide: Deploying NextcloudPi on Proxmox with TrueNAS NFS Backend

This guide details the full process from storage creation to the final application configuration.

Phase 1: Configure Storage in TrueNAS

Our foundation is robust, redundant storage. We will create a dedicated space (dataset) and share it over the network.

1. Create a Dataset for Nextcloud

In the TrueNAS web interface, navigate to Storage > Pools.

Find your main storage pool, click the ⋮ (three-dot) menu, and select Add Dataset.

Name: Enter a clear name, for example, NextcloudData.

Leave other options at their default settings and click Submit.

2. Create and Configure the NFS Share

Navigate to Sharing > NFS Shares.

Click Add.

Path: Browse and select the dataset you just created (e.g., /mnt/YourPoolName/NextcloudData).

Click Advanced Options to reveal all settings.

Sync: This is critical for preventing data corruption during uploads. Change the value from "Default" to "Enabled".

Mapall User: This is critical for Proxmox integration. Set this to root.

Mapall Group: Set this to root (or wheel).

Authorized Networks: Enter the IP address of your Proxmox host or your entire network subnet (e.g., 192.168.1.0/24).

Click Submit. If prompted, enable the NFS service.

Phase 2: Integrate TrueNAS Storage into Proxmox

Proxmox needs to be aware of this new network storage location.

1. Add the NFS Share as Proxmox Storage

In the Proxmox web interface, go to the Datacenter view.

Select the Storage tab.

Click Add and choose NFS.

Fill in the details:

ID: Give it a descriptive name, like truenas_nfs_nextcloud.

Server: Enter the IP address of your TrueNAS VM.

Export: Enter the exact path of the share from TrueNAS (e.g., /mnt/YourPoolName/NextcloudData).

Content: Click the dropdown and select all available content types. This makes the storage versatile for backups, ISOs, disks, and containers.

Click Add. The new storage will now appear in the left-hand resource tree.

Phase 3: Create and Configure the NextcloudPi LXC Container

With storage ready, we can now build the application container.

1. Download the LXC Template

In Proxmox, select your server node, then the local storage pool.

Go to LXC Templates, click the Templates button, search for nextcloudpi, and download it.

2. Create the Container

Click the "Create CT" button in the top right.

General: Set a Hostname (nextcloudpi) and a strong root password.

Template: Select the local storage and the nextcloudpi template you downloaded.

Disks: Set the Root Disk size to 8GB on your local-lvm or local storage.

CPU: Assign at least 2 cores.

Memory: Assign at least 2048MB (2GB).

Network/DNS: Leave as defaults (DHCP).

Confirm: Review the summary and click Finish. Do not start the container yet.

3. Mount the NFS Storage into the Container

Select the newly created (and stopped) nextcloudpi container.

Go to the Resources tab.

Click Add and select Mount Point.

Configure the mount:

Storage: Select the truenas_nfs_nextcloud storage you added in Phase 2.

Path: This is the folder name where the storage will appear inside the container. A good, standard path is /mnt/ncdata.

Size: Leave blank to use the full share size.

Click Create.

Phase 4: Application Setup and Data Migration

This is the final phase where we configure the NextcloudPi software.

1. First Boot and Activation

Start the container.

Find its IP address from the container's Network tab in Proxmox.

In your browser, go to https://<Container-IP>:4443. Proceed past the security warning.

On the activation page, save the passwords shown, then click Activate.

Once complete, you can log in to the Nextcloud web interface at https://<Container-IP>. The application is now running, but its data is still on the small 8GB system disk.

2. Set Permissions on the Mount Point

In Proxmox, open the >_ Console for the nextcloudpi container.

Grant the Nextcloud web server user ownership of the mounted storage by running:

code
Bash
download
content_copy
expand_less
sudo chown -R www-data:www-data /mnt/ncdata

Note: You may see a harmless error: chown: cannot read directory '/mnt/ncdata/lost+found': Permission denied. This is expected and confirms the NFS mount is working correctly. You can safely ignore it.

3. Move the Nextcloud Data Directory

In the same console, launch the NextcloudPi configuration tool:

code
Bash
download
content_copy
expand_less
sudo ncp-config

Use the arrow keys to navigate to CONFIG and press Enter.

Select nc-datadir and press Enter.

A text box will appear. Delete any text inside it.

Carefully type the absolute path to your mount point:

code
Code
download
content_copy
expand_less
/mnt/ncdata

Press Enter and confirm Yes to start the move.

The script will move all data to your TrueNAS server. When it returns to the menu, the process is complete.

