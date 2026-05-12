# VMware to Hyper-V VM Migration

A working method for moving a VM from VMware to Hyper-V, including the BitLocker workaround that gets you past the part that breaks most migration guides. Convert the VMDK to a VHDX with StarWind V2V Converter, create a fresh Hyper-V VM, attach the new disk, boot it, and deal with the post-migration cleanup.

The reason this is worth writing down is that 90% of guides stop at "import the disk and start the VM". The actual problem is what happens after that. BitLocker triggers because the hardware fingerprint changed. The Microsoft PIN stops working. Drivers are missing. The system reserved partition didn't come over cleanly. None of that gets covered. This does.

## The process

### Step 1: Convert the VMDK to VHDX

Make sure the VMware VM is powered off first. A live conversion of a running disk is asking for corruption.

Open StarWind V2V Converter (free, no licensing nonsense). Pick "Local file" as the source location, then point it at the main VMDK file inside the VMware VM folder. If the VM has multiple VMDKs, you want the main one with the OS, not the snapshot deltas.

<img width="560" height="637" alt="Selecting the VMDK in StarWind V2V Converter" src="https://github.com/user-attachments/assets/c5a0dd88-8f36-4fcd-99b8-ff6f2cf5fb5d">

Then pick "Local file" again for the destination, choose VHD/VHDX as the format, and select "VHDX growable image". Pick a destination folder that isn't a network share. Network shares slow the conversion down for no good reason and can fail mid-way if the connection drops.

### Step 2: Create a fresh Hyper-V VM

This is where most migrations go wrong. Don't reuse an existing Hyper-V VM. Create a new one from scratch and attach the converted VHDX. This one detail is what stops BitLocker from triggering most of the time. More on that in the Problems section below.

<img width="941" height="445" alt="Creating a new Hyper-V VM" src="https://github.com/user-attachments/assets/ce69765d-3afc-4eb0-8ca4-74081abdff79">

Generation matters and you have to get it right the first time, you can't change it after.

- **Generation 1** for BIOS-based VMs (older systems, Windows 7, some Linux distros)
- **Generation 2** for UEFI-based VMs (Windows 10/11, modern Linux, anything from the last 8 years or so)

Specify memory (4096 MB is fine for most workloads, bump up for heavier ones), connect to an existing virtual switch, and when you get to the disk step, choose "Use an existing virtual hard disk" and point it at the VHDX you just converted.

<img width="701" height="522" alt="Attaching the converted VHDX" src="https://github.com/user-attachments/assets/6c6a11f6-7dc8-41ea-9521-f5049d807f8d">

<img width="751" height="203" alt="VM summary before creation" src="https://github.com/user-attachments/assets/1b277b6c-0be7-4a58-934d-99bd36a5e609">

### Step 3: Boot and verify

Right-click the VM, Connect, then Start. Watch the boot sequence.

BIOS-based VMs usually boot first try. UEFI-based VMs sometimes need a hand. If you get an error or boot loop, power off the VM, go to Settings → Security, enable Secure Boot, and enable Trusted Platform Module. That fixes most Generation 2 boot problems on the first try.

Once it's up, confirm the OS loads, files are intact, applications open, and the system feels normal.

### Step 4: Post-migration cleanup

The VM is running but it's not done.

- **Uninstall VMware Tools** from inside the guest OS. They'll error and complain about missing devices forever if you leave them.
- **Check Device Manager** for missing drivers or unknown devices. Anything with a yellow triangle needs attention.
- **Install Hyper-V Integration Services** (or update them if they're already there from a previous Windows version). This is what gives you proper mouse, network, and time sync inside the guest.
- **Test network connectivity.** Network adapter changes are the most common silent failure after a migration.
- **Confirm all disks are visible** in Disk Management. If the VM had multiple disks, each one needs to be converted separately and attached.
- **Take a checkpoint** once the VM is confirmed working. Now you have a clean rollback point.

## Problems I had to solve

**BitLocker locking the drive after migration.** This is the big one. When a Windows machine moves between hypervisors, the hardware fingerprint changes (different motherboard, different TPM, different firmware). BitLocker sees a hardware change and locks the drive on next boot. If you don't have the recovery key, you're done. The drive is encrypted and there's no way back.

The trick I learned the hard way is to attach the converted VHDX to a brand new Hyper-V VM, not an existing one. Reusing an existing VM keeps old hardware definitions around that confuse the boot process and trigger BitLocker about 90% of the time. A fresh VM with the converted disk as its first and only disk avoids the trigger far more often. Not always, but enough that this should be the default approach.

**Microsoft account PIN stops working after boot.** First boot after migration and the PIN won't work. The PIN is tied to the TPM, and the TPM in the new VM is different from the old one. The fix is to sign in with the Microsoft account password (not the PIN), then go to Settings → Accounts → Sign-in options and set up a new PIN against the new TPM. If you can't sign in at all, boot into Safe Mode or use another admin account.

**VM blue screens on first boot.** This is usually a Generation mismatch. Created a Generation 2 VM but the source was BIOS-based, or the other way around. Hyper-V won't let you change Generation on an existing VM, so the fix is to delete the VM (not the VHDX, just the VM definition), make a new one with the right Generation, and attach the same VHDX.

**Boot loader errors after conversion.** Rare, but happens when the conversion misses the system reserved partition. The symptom is a boot loader error or a missing boot device on startup. The fix is to attach the VHDX to a recovery VM as a secondary disk, boot the recovery VM, and rebuild the boot configuration with `bcdedit` and `bootrec`. Worth checking the VMDK has all its associated files before converting (the .vmdk plus any -s001.vmdk, -s002.vmdk split files).

## What I learned

The biggest one is that hypervisor migration is not really a "disk migration". It's a "machine identity change", and Windows treats it like new hardware. Everything that's tied to the hardware fingerprint (BitLocker, TPM-backed PINs, hardware-locked licenses, Windows Hello) has to be dealt with on the other side.

Other things:

- Always have the BitLocker recovery key in hand before starting (If any). If you don't have it, get it from the user's Microsoft account or Azure AD before you touch the disk.
- Generation 2 / UEFI is the default for anything modern. Don't pick Generation 1 just because it boots faster.
- Don't run conversions over network shares. Local disk to local disk every time.
- Take a backup of the original VMware VM before doing anything. If the migration fails, you want to go back to a known good state, not a half-converted disk.

## Why I built it

I needed to migrate a VM and the existing guides got me 80% of the way there. The last 20% (BitLocker, PIN reset, the Generation 2 boot issues) was a rabbit hole I had to figure out on my own. Once I had it working, I wrote it down so the next migration takes 30 minutes instead of half a day.

