# VMware to Hyper-V VM Migration

### Description

Step-by-step guide for migrating virtual machines from VMware to Microsoft Hyper-V using StarWind V2V Converter. This guide covers disk conversion, verification, and importing the VM into Hyper-V. Ideal for IT professionals and system administrators who need a smooth, reliable migration process.

### Requirements

- VMware virtual machine to migrate

- Hyper-V installed on the destination host

- StarWind V2V Converter installed (it's free)

- Basic knowledge of Hyper-V VM creation

## Best Practices  

- Always backup your VMware VM before converting

- Use a dedicated folder for your converted VHDX files

- Avoid running conversions over network shares for faster performance

- After migration, test the VM thoroughly before deploying it in production


### Migration Steps
### Step 1: Converting VMDK to VHDX

- Make sure the VM is off

- Open StarWind V2V Converter

- Select "Local file" for the location of the image

- Use the 3 dots to navigate to the VMware VM folder that you want to migrate 

- Select the main VMDK file (e.g., Your-VM-Name.vmdk)  
  <img width="560" height="637" alt="Screenshot 2025-11-13 200824" src="https://github.com/user-attachments/assets/9a1107d8-d543-4e49-9b7c-b7c0b0cc0d9c" />


- Select "Local file" for the location of the destination image

- Select "VHD/VHDX" for the destination image format

- Select "VHDX growable image"

- Choose an easily accessible destination folder, then click "Convert"

### Step 2: Create a New VM in Hyper-V

- Open Hyper-V Manager and select New → Virtual Machine <img width="941" height="445" alt="Screenshot 2025-11-13 193853" src="https://github.com/user-attachments/assets/9cf81b9a-83fc-448f-aa97-5ccf1b570bab" />

- Specify Name

- Choose Generation 1 for BIOS-based VMs

- Choose Generation 2 for UEFI-based VMs (most common)

- Specify memory (e.g., 4096 MB)

- Connect the VM to an existing Virtual Switch if available

- Choose "Use an existing virtual hard disk" and navigate to the new VHDX file that you created via StarWind V2V Converter <img width="701" height="522" alt="Screenshot 2025-11-13 193934" src="https://github.com/user-attachments/assets/e21b289b-6169-408b-baae-1c9e291636da" />  

<img width="751" height="203" alt="Screenshot 2025-11-13 194051" src="https://github.com/user-attachments/assets/40010459-82d3-4fbf-b80c-24447fd69c24" />

### Step 3: Start and Verify the Migrated VM

- Right-click the VM and select Connect → Start

- Watch the boot sequence.

  - If the VM was BIOS-based → it should boot normally

  - If UEFI → ensure secure boot settings match the OS type

- If you get an error, power off the VM → Settings... → Security → Enable Secure Boot → Enable Trusted Platform Module

- Verify migrated files and applications open normally

### Step 4: Post-Migration Checks  

- Uninstall VMware Tools from the guest OS (if installed)

- Check Device Manager for missing drivers or devices

- Test network connectivity and confirm all disks are visible

- Take a checkpoint/snapshot after confirming the VM runs properly

## Problems & Support

If you run into problems after migration I can try to help. Common issues and quick notes:

- Windows asks to reset your Microsoft PIN after boot

  - Quick hint: try signing in with your Microsoft account password first, then go to Settings → Accounts → Sign-in options to reset the PIN.

  - If you can’t sign in at all, boot into Safe Mode or use another admin account to reset the PIN.

- BitLocker triggered and encrypted drives

  - BitLocker will trigger 90% of the time when using existing Hyper-V VM to migrate
      
      - Try creating a completely new Hyper-V VM before migrating and only use the freshly created VHDX as the destination drive

  - BitLocker can lock the system when hardware/firmware changes. If you have the BitLocker recovery key, use it to unlock the drive

  - If you don’t have the key, do not enter random keys — stop and contact me; there are safer recovery steps to try

- Device/driver issues or missing network

  - Check Device Manager for missing drivers and install Integration Services (or latest guest additions)

  - Recreate or attach the correct virtual network switch in Hyper-V

- VM won’t boot or blue screens

  - Verify VM generation (1 vs 2) and secure boot settings; attach the VHDX to a recovery VM to inspect logs

  - If you see boot loader errors, the disk conversion may have missed the system reserved partition — I can advise recovery steps

## How I can help

If you want my assistance, contact me and I’ll try to help troubleshoot the issue. Please do not share passwords, private keys, or full BitLocker recovery keys in public posts

## Skills You Learned
- VMware/Hyper-V VM management

- Disk conversion (VMDK → VHDX)

- Hyper-V virtual machine setup (if new)

- Migration verification and troubleshooting

## License  
This project is licensed under the MIT License
