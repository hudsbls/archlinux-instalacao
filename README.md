# **Guia de Instalação do Arch Linux (UEFI)**

Este manual é um guia passo a passo completo para a instalação do Arch Linux, baseado em um conjunto de anotações específicas, cobrindo o sistema base, drivers NVIDIA, e configurações essenciais de rede e boot.

## **1\. Preparação**

### **Conexão à Internet (Wi-Fi)**

Se precisar de uma conexão Wi-Fi, use o iwctl. Para listar os seus dispositivos de rede, use device list.

\# Inicie o utilitário iwctl  
iwctl   
\# Liste seus dispositivos de rede  
device list  
\# \<device\_name\> é geralmente wlan0 ou similar  
station \<device\_name\> scan  
station \<device\_name\> get-networks  
station \<device\_name\> connect \<network\_name\>

### **Ajustar o Layout do Teclado**

Para definir o teclado para o padrão brasileiro:

loadkeys br-abnt2

## **2\. Particionamento e Montagem**

Use o fdisk para particionar o disco. Substitua \<device\_name\> pelo seu disco (ex: /dev/sda ou /dev/nvme0n1).

fdisk /dev/\<device\_name\>

**Partições Recomendadas (Exemplo UEFI):**

* **/boot/efi:** (Ex: /dev/sdX1) \- fat32 (para bootloader)  
* **swap:** (Ex: /dev/sdX2) \- linuxswap  
* **/ (root):** (Ex: /dev/sdX3) \- ext4  
* **/home:** (Ex: /dev/sdX4) \- ext4

### **Formatação das Partições**

Formate as partições criadas:

mkfs.fat \-F32 /dev/\<particao\_boot\_efi\>  
mkswap /dev/\<particao\_swap\>  
mkfs.ext4 /dev/\<particao\_root\>  
mkfs.ext4 /dev/\<particao\_home\>

### **Montagem das Partições**

Monte as partições no diretório /mnt.

mount /dev/\<particao\_root\> /mnt  
mkdir /mnt/home  
mount /dev/\<particao\_home\> /mnt/home  
mkdir /mnt/boot  
mkdir /mnt/boot/efi  
mount /dev/\<particao\_boot\_efi\> /mnt/boot/efi  
swapon /dev/\<particao\_swap\>

## **3\. Instalação do Sistema Base**

### **Configurar o Pacman**

1. **Instalar Reflector** para otimizar a lista de mirrors:  
   pacman \-S reflector

2. **Editar mirrorlist** para adicionar servidores e prioridades:  
   nano /etc/pacman.d/mirrorlist

   Adicione (ou mova para o topo) os servidores do Brasil e osbeck:  
   Server \= https://mirror.ufam.edu.br/archlinux/$repo/os/$arch  
   Server \= https://mirror.osbeck.com/archlinux/$repo/os/$arch

   Ou use o reflector para gerar a lista (opcional):  
   reflector \--country Brazil \--latest 20 \--sort rate \--verbose \--save /etc/pacman.d/mirrorlist

3. **Habilitar Multilib, Cor e Downloads Paralelos** no pacman.conf:  
   nano /etc/pacman.conf

   Remova o \# das linhas:  
   * \#Color  
   * \#ParallelDownloads \= 5  
   * \[multilib\] e sua linha Include

### **Instalar Pacotes Essenciais**

Instale os pacotes básicos (incluindo linux-headers, essencial para drivers de placa de vídeo).

pacstrap /mnt base base-devel linux linux-headers linux-firmware nano vim

## **4\. Configuração do Sistema (Chroot)**

### **Gerar Fstab e Entrar em Chroot**

genfstab \-U /mnt \>\> /mnt/etc/fstab  
arch-chroot /mnt

### **Fuso Horário, Idioma e Teclado**

\# Fuso horário e NTP  
ln \-sf /usr/share/zoneinfo/America/Sao\_Paulo /etc/localtime  
timedatectl set-ntp true  
hwclock \--systohc

\# Idioma (descomente pt\_BR.UTF-8)  
nano /etc/locale.gen  
locale-gen

\# Criar locale.conf  
echo "LANG=pt\_BR.UTF-8" \> /etc/locale.conf

\# Configurar teclado do console  
nano /etc/vconsole.conf  
\# Adicione:  
\# KEYMAP=br-abnt2

### **Rede e Nome do Host**

Defina o nome da sua máquina e configure o mapeamento de hosts.

\# Definir Hostname  
echo "archlinux" \> /etc/hostname

\# Editar arquivo hosts  
nano /etc/hosts

Substitua archlinux pelo hostname escolhido:

127.0.0.1   localhost  
::1         localhost  
127.0.1.1   archlinux.localdomain   archlinux

### **Usuários e Sudoers**

1. **Definir senha de root:**  
   passwd

2. **Criar novo usuário e definir senha:**  
   useradd \-m \-G wheel \<nome\_do\_usuario\>  
   passwd \<nome\_do\_usuario\>

3. Habilitar Sudoers (acesso admin para o grupo wheel):  
   Use visudo com o editor nano (obrigatório para evitar erros de sintaxe).  
   EDITOR=nano visudo

   Descomente a linha:  
   \# %wheel ALL=(ALL:ALL) ALL

## **5\. Configuração de Gráficos (NVIDIA)**

Esta seção é para usuários de placas de vídeo NVIDIA que buscam aceleração por hardware.

1. **Instale os pacotes do driver:**  
   pacman \-S nvidia nvidia-utils lib32-nvidia-utils

2. Ative o DRM Kernel Mode Setting:  
   Edite o arquivo mkinitcpio.conf e adicione os módulos da NVIDIA na seção MODULES.  
   nano /etc/mkinitcpio.conf

   A linha deve ficar assim (adicionando os módulos nvidia e relacionados):  
   MODULES=(... nvidia nvidia\_modeset nvidia\_uvm nvidia\_drm)

3. **Regenere o initramfs** para carregar os novos módulos:  
   mkinitcpio \-P

## **6\. Bootloader e Rede**

### **Instalar e Configurar o GRUB**

1. **Instale os pacotes do bootloader:**  
   pacman \-S dosfstools mtools os-prober efibootmgr grub networkmanager iwd

2. **Desabilite o OS Prober** (recomendado para dual boot ou para um menu limpo):  
   nano /etc/default/grub

   Procure a linha **\#GRUB\_DISABLE\_OS\_PROBER=false** e remova o \# para desativar a função de procurar outros sistemas.  
3. **Instalar e Gerar o GRUB:**  
   \# Instale o GRUB na partição EFI  
   grub-install \--target=x86\_64-efi \--efi-directory=/boot/efi \--bootloader-id=archlinux \--recheck

   \# Gere o arquivo de configuração do GRUB  
   grub-mkconfig \-o /boot/grub/grub.cfg

### **Configurar e Ativar a Rede**

Para o NetworkManager usar o iwd como backend para o Wi-Fi:

1. **Crie e edite o arquivo de configuração do NetworkManager:**  
   nano /etc/NetworkManager/conf.d/wifi\_backend.conf

   Adicione o seguinte conteúdo:  
   \[device\]  
   wifi.backend=iwd

2. **Habilite o serviço:**  
   systemctl enable NetworkManager

## **7\. Reiniciar o Sistema**

Saia do ambiente chroot e reinicie a máquina.

exit  
shutdown \-r now

Após o reinício, remova o pendrive de instalação e o sistema deverá iniciar no Arch Linux que você instalou.
