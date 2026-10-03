1. I dowload the debian-12-turnkey-zoneminder_18.0-1_amd64.tar.gz container following the https://pve.proxmox.com/wiki/Linux_Container documentation and I make it run in the 102 container ID, i make the container with --rootfs to define where to storage it and how it's created.

2. pct create 102 local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst \
  --hostname debian13 \
  --rootfs local-lvm:8 \
  --cores 1 \
  --memory 1024 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1


Container running:
<img width="758" height="655" alt="image" src="https://github.com/user-attachments/assets/ce2351fd-2895-4502-8f5c-6f783b9238ba" />
