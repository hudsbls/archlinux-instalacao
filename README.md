# **Guia de Instalação do Arch Linux (UEFI)**

Este manual é um guia passo a passo completo para a instalação do Arch Linux, baseado em um conjunto de anotações específicas, cobrindo o sistema base, drivers NVIDIA, e configurações essenciais de rede e boot.

## **1. Preparação**

### **Conexão à Internet (Wi-Fi)**

Se precisar de uma conexão Wi-Fi, use o iwctl. Para listar os seus dispositivos de rede, use device list.

```bash
# Inicie o utilitário iwctl
iwctl

# Liste seus dispositivos de rede
device list

# <device_name> é geralmente wlan0 ou similar
station <device_name> scan
station <device_name> get-networks
station <device_name> connect <network_name>
````

### **Ajustar o Layout do Teclado**

Para definir o teclado para o padrão brasileiro:

```bash
loadkeys br-abnt2
```

---

## **2. Particionamento e Montagem**

Use o fdisk para particionar o disco. Substitua `<device_name>` pelo seu disco (ex: /dev/sda ou /dev/nvme0n1).

```bash
fdisk /dev/<device_name>
```

**Partições Recomendadas (Exemplo UEFI):**

* **/boot/efi:** (Ex: /dev/sdX1) - fat32 (para bootloader)
* **swap:** (Ex: /dev/sdX2) - linuxswap
* **/ (root):** (Ex: /dev/sdX3) - ext4
* **/home:** (Ex: /dev/sdX4) - ext4

### **Formatação das Partições**

```bash
mkfs.fat -F32 /dev/<particao_boot_efi>
mkswap /dev/<particao_swap>
mkfs.ext4 /dev/<particao_root>
mkfs.ext4 /dev/<particao_home>
```

### **Montagem das Partições**

```bash
mount /dev/<particao_root> /mnt
mkdir /mnt/home
mount /dev/<particao_home> /mnt/home
mkdir /mnt/boot
mkdir /mnt/boot/efi
mount /dev/<particao_boot_efi> /mnt/boot/efi
swapon /dev/<particao_swap>
```

---

## **3. Instalação do Sistema Base**

### **Configurar o Pacman**

1. **Instalar Reflector** para otimizar a lista de mirrors:

   ```bash
   pacman -S reflector
   ```

2. **Editar mirrorlist** para adicionar servidores e prioridades:

   ```bash
   nano /etc/pacman.d/mirrorlist
   ```

   Adicione (ou mova para o topo) os servidores do Brasil e osbeck:

   ```
   Server = https://mirror.ufam.edu.br/archlinux/$repo/os/$arch
   Server = https://mirror.osbeck.com/archlinux/$repo/os/$arch
   ```

   Ou use o reflector para gerar a lista (opcional):

   ```bash
   reflector --country Brazil --latest 20 --sort rate --verbose --save /etc/pacman.d/mirrorlist
   ```



### **Instalar Pacotes Essenciais**

```bash
pacstrap /mnt base base-devel linux linux-headers linux-firmware nano vim
```

---

## **3. Configuração do Sistema (Chroot)**

### **Gerar Fstab e Entrar em Chroot**

```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```
**Habilitar Multilib, Cor e Downloads Paralelos** no pacman.conf:

   ```bash
   nano /etc/pacman.conf
   ```

   Remova o `#` das linhas:

   ```
   #Color
   #ParallelDownloads = 5
   [multilib]
   Include = ...
   ```

### **Fuso Horário, Idioma e Teclado**

```bash
# Fuso horário e NTP
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
timedatectl set-ntp true
hwclock --systohc
```

```bash
# Idioma (descomente pt_BR.UTF-8)
nano /etc/locale.gen
locale-gen
```

```bash
# Criar locale.conf
echo "LANG=pt_BR.UTF-8" > /etc/locale.conf
```

```bash
# Configurar teclado do console
nano /etc/vconsole.conf
# Adicione:
# KEYMAP=br-abnt2
```

### **Rede e Nome do Host**

```bash
# Definir Hostname
echo "archlinux" > /etc/hostname
```

```bash
# Editar arquivo hosts
nano /etc/hosts
```

Substitua `archlinux` pelo hostname escolhido:

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   archlinux.localdomain   archlinux
```
```bash
# Definir senha de root
passwd
```

### **Usuários e Sudoers**

```bash
# Remover # de:
# %wheel ALL=(ALL:ALL) ALL em:
nano /etc/sudoers
```

```bash
# Criar novo usuário e definir senha
useradd -m -G wheel <nome_do_usuario>
passwd <nome_do_usuario>
```


---

## **4. Configuração de Gráficos (NVIDIA)**

Esta seção é para usuários de placas de vídeo NVIDIA que buscam aceleração por hardware.

```bash
# Instale os pacotes do driver
pacman -S nvidia nvidia-utils lib32-nvidia-utils
```

```bash
# Ative o DRM Kernel Mode Setting
nano /etc/mkinitcpio.conf
```

Adicione na linha MODULES:

```
MODULES=(... nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

```bash
# Regenere o initramfs
mkinitcpio -P
```

---

## **5. Bootloader e Rede**

### **Instalar pacotes e Configurar o GRUB**

```bash
# Instalar pacotes do bootloader
pacman -S dosfstools mtools os-prober efibootmgr grub networkmanager iwd amd-ucode
```


```bash
# Instalar o GRUB na partição EFI
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=archlinux --recheck
```

```bash
# Desabilitar OS Prober
nano /etc/default/grub
```

Procure a linha:

```
#GRUB_DISABLE_OS_PROBER=false
```

e remova o `#`.

```bash
# Gerar arquivo de configuração do GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

### **Configurar e Ativar a Rede**

```bash
# Criar e editar arquivo do NetworkManager
nano /etc/NetworkManager/conf.d/wifi_backend.conf
```

Conteúdo:

```
[device]
wifi.backend=iwd
```

```bash
# Habilitar serviço
systemctl enable NetworkManager
```

---

## **6. Reiniciar o Sistema**

```bash
exit
shutdown -r now
```

Após o reinício, remova o pendrive de instalação e o sistema deverá iniciar no Arch Linux que você instalou.

```
```
