# Software

## Matlab installation (here 2026a)

he Softwarezentrale (TUM-CIT-ITO) provides the ISO images and File Installation Keys (FIK) through a personal Nextcloud share. Each admin receives their own link. Links are valid for one year and are renewed on request by the Softwarezentrale.

The share contains two keys per release. Use the key labelled **Campus**. The key labelled **Concurrent MDCS** is for MATLAB Parallel Server and is not needed for this setup.

!!! warning
    Do not put the share link into scripts, shell history or this documentation. It is tied to one person.

### Download the ISO on the server
 
The ISO can be downloaded directly on the lnode through the public WebDAV endpoint of the share. The share token is the last part of the share link and is passed as username with an empty password:

Example: 

```bash
curl -u "<share-token>:" -o R2026a_Linux.iso \
  "https://nextcloud.in.tum.de/public.php/webdav/R2026a/R2026a_Update_1_Linux.iso"
```

### Check the license server
 
The node must reach the license server themselves, not through the login node. Test:

```bash
srun -N1 -n1 nc -zv mlm5.rbg.tum.de 28000
```

The full license check at the end of this page is the reliable test, since FlexLM also uses a vendor daemon port.


### Prepare the installation

Mount the ISO and create the target directory:

```bash
sudo mkdir -p /mnt/matlab_iso /opt/software/matlab/R2026a
sudo mount -o loop R2026a_Linux.iso /mnt/matlab_iso
```

Create the license file `~/license.lic`:
 
```text
SERVER mlm5.rbg.tum.de 005056BA37BA 28000
USE_SERVER
```

Copy the installer template from the ISO and make it writable. Files copied from an ISO keep their read-only permissions.
 
```bash
cp /mnt/matlab_iso/installer_input.txt ~/installer_input.txt
sudo chown $USER: ~/installer_input.txt
chmod u+w ~/installer_input.txt
```
 
 Set the following lines in `~/installer_input.txt`:
 
```text
destinationFolder=<Installaton Path>
fileInstallationKey=<Campus key>
agreeToLicense=yes
outputFile=/tmp/matlab_install.log
licensePath=/home/<user>/license.lic
```

If all `product.` lines stay commented out, the installer installs every product covered by the campus license. To install a subset, uncomment the corresponding lines, for example `product.MATLAB` and `product.Parallel_Computing_Toolbox`. Product names occasionally change between releases, so always take them from the template of the release being installed.

### Run the installer
 
```bash
sudo /mnt/matlab_iso/install -inputFile /home/<user>/installer_input.txt
```