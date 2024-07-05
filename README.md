This guide is based on SomeOrdinaryGamers video 
Link: https://www.youtube.com/watch?v=BUSrdUoedTo&t=1981s


To begin the gpu passthrough journey we first need to enable 
Intel: vt-d (vt-x) or for AMD svn

Remember if you have an intergrated gpu then you have to plugin your monitor into the montherboard otherwise it won't show anything after the next steps

Also if using only a single gpu make sure to either have a second display or you will have to switch between ports (hope I did not mess anything up since I have no experience with single gpu)

Now get your pci ids with lspci or 

`#!/bin/bash`

`for d in /sys/kernel/iommu_groups/*/devices/*; do`

  `n=${d#*/iommu_groups/*}; n=${n%%/*}`

  `printf 'IOMMU Group %s ' "$n"`

  `lspci -nns "${d##*/}"`

`done`

Make note of the gpu VIDEO and AUDIO ids something like
In my case with amd rx 6800: *1002:73bf, 1da2:e437, 1002:ab28* 


Next step is to edit the /etc/default/grub 

sudo nvim (text editor of choice) /etc/default/grub

![Pasted image 20240704212808](https://github.com/tlg-tg/Arch-Linux-Gpu-Passthrough/assets/61950743/764ee8d7-ece9-4a95-b27d-c63b83b52f2e)


after the loglevel3 or splash add the following (note if you have quiet then you can just remove the quiet line)

Intel: "rd.driver.pre=vfio-pci intel_iommu=on iommu=pt vfio pci.ids=1003:ab39,..."

Amd: "rd.driver.pre=vfio-pci amd_iommu=on iommu=pt video=efifb:off vfio pci.ids=1002:73bf,1da2:e437,1002:ab28"

Now we can update the grub config 

`sudo grub-mkconfig -o /boot/grub/grub.cfg`

after it completes reboot your pc.

Now we can go onto adding to our modprobe.d folder

`sudo nvim /etc/modprobe.d/vfio.conf`

and add the following

Nvidia users:

`options vfio-pci ids=...,...,...`
`softdep nvidia pre: vfio-pci`

Amd users:

`options vfio-pci ids=1002:73bf,1da2:e437,1002:ab28`
`softdep amdgpu pre: vfio-pci`


Now we have to edit the mkinitcpio.conf 

`sudo nvim /etc/mkinitcpio.conf`

And add the following in the modules 
MODULES=(vfio vfio_iommu_type1 vfio_pci)

![Pasted image 20240704214410](https://github.com/tlg-tg/Arch-Linux-Gpu-Passthrough/assets/61950743/7ebb8ccc-b7a1-4aba-b8e9-849e107cb61a)


It should look something like this.

Now we have to rebuild it so we can run 

`sudo mkinitcpio -p linux`

And we are at a point where we can install our virtualization software (virt-manager)

`sudo pacman -S qemu-full qemu-img libvirt virt-install virt-manager virt-viewer \`
`edk2-ovmf swtpm guestfs-tools libosinfo`

`sudo systemctl enable --now libvirtd`

Now let's configure the libvirtd.conf

`sudo nvim /etc/libvirt/libvirtd.conf`

Locate the unix_sock_group and remove the # do the same for unix_sock_rw_perms, 
after saving the configuration we can add our user to the libvirt group

`sudo usermod -a -G libvirt $(whoami)`

and restart libvirtd

`sudo systemctl restart libvirtd`

So now we can finally create a VM (the video also includes how to patch an nvidia card /change the rom so you can enable gpu passthrough since nvidia only allows this with newer card and enterprise cards)

I recommend following this video for vm creation at 17:40
https://www.youtube.com/watch?v=BUSrdUoedTo&t=1981s

After the vm starts and you finish installing windows just shutdown the vm.

Now we get into the fun part so if you want to use your keyboard and mouse in host and guest with a keybind follow this steps 

`git clone https://github.com/pavolelsig/evdev_helper.git`

make sure to unzip it

now we have to enable execution of run_ev_helper.sh

`sudo chmod +x run_ev_helper.sh`

and run it

`./run_ev_helper.sh`


After it finishes we now need to finally edit our xml of the vm (make sure xml editing is enabled)

First we need to change the domain to this:
`<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>`

and at the bottom right under `</devices>`  paste the following

 `<qemu:commandline>`
	`<qemu:arg value='-object'/>`
	`<qemu:arg value='input-linux,id=kbd1,evdev=/dev/input/by-id/usb-Logitech_G815_RGB_MECHANICAL_GAMING_KEYBOARD_177A39633338-event-kbd,grab_all=on,repeat=on'/>`
	`<qemu:arg value='-object'/>`
	`<qemu:arg value='input-linux,id=kbd2,evdev=/dev/input/by-id/usb-Logitech_G815_RGB_MECHANICAL_GAMING_KEYBOARD_177A39633338-if01-event-kbd,grab_all=on,repeat=on'/>`
	`<qemu:arg value='-object'/>`
	`<qemu:arg value='input-linux,id=mouse1,evdev=/dev/input/by-id/usb-Logitech_G815_RGB_MECHANICAL_GAMING_KEYBOARD_177A39633338-if01-event-mouse'/>`
	`<qemu:arg value='-object'/>`
	`<qemu:arg value='input-linux,id=mouse2,evdev=/dev/input/by-id/usb-Logitech_USB_Receiver-event-mouse'/>`
	`<qemu:arg value='-object'/>`
	`<qemu:arg value='input-linux,id=kbd3,evdev=/dev/input/by-id/usb-Logitech_USB_Receiver-if01-event-kbd,grab_all=on,repeat=on'/>`
  `</qemu:commandline>`

Click the apply button

in between tags `<hyprv>` and `</hyprv>` we have to add `<vendor_id state="on" value="anything"/>`

also after the closing tag `</hyprv>` we have to add
`<kvm>`
`<hidden state="on"/>`
`</kvm>`

and now we are finally done

For any user that wishes to use gpu again on the host after or before vm takes the gpu can follow the last steps

Nvidia user can follow this github tutorial for it to work:
https://github.com/bryansteiner/gpu-passthrough-tutorial

1.
`cd /etc/libvirt/`

`sudo mkdir hooks`

`cd` 

2.
`sudo wget 'https://raw.githubusercontent.com/PassthroughPOST/VFIO-Tools/master/libvirt_hooks/qemu' \`
     `-O /etc/libvirt/hooks/qemu`


`sudo chmod +x /etc/libvirt/hooks/qemu`

3.

`cd /etc/libvirt/hooks/`

`sudo nvim kvm.conf`

`VIRSH_GPU_VIDEO=pci_0000_03_00_0`
`VIRSH_GPU_AUDIO=pci_0000_03_00_1`


4.
`sudo mkdir qemu.d` 

`sudo mkdir qemu.d/win10`

`sudo mkdir qemu.d/win10/prepare`

`sudo mkdir qemu.d/win10/prepare/begin`

`sudo mkdir qemu.d/win10/release`

`sudo mkdir qemu.d/win10/release/end`



`cd /qemu.d/win10/prepare/begin`


`sudo nvim start.sh` 


`#!/bin/bash`

`set -x` 

`source "/etc/libvirt/hooks/kvm.conf"`

`echo 0 > /sys/class/vtconsole/vtcon0/bind`
`echo 0 > /sys/class/vtconsole/vtcon1/bind`

`sleep 2` 

`virsh nodedev-deatach $VIRSH_GPU_VIDEO`
`virsh nodedev-deatach $VIRSH_GPU_AUDIO`

`modprobe vfio`
`modprobe vfio_pci`
`modprobe vfio_iommu_type1`


`sudo chmod +x start.sh` 

5.

`cd into release/end`

`sudo nvim revert.sh` 


`#!/bin/bash`

`set -x` 

`source "/etc/libvirt/hooks/kvm.conf"`

`modprobe -r vfio_pci`
`modprobe -r vfio_iommu_type1`
`modprobe -r vfio`

`virsh nodedev-reattach $VIRSH_GPU_VIDEO`
`virsh nodedev-reattach $VIRSH_GPU_AUDIO`

`echo 1 > /sys/class/vtconsole/vtcon0/bind`
`echo 0 > /sys/class/vtconsole/vtcon1/bind`

`modprobe amdgpu`



And we are done now you can lunch your vm and it will take your gpu and after shutting down the vm we will have our gpu back for use in linux.
