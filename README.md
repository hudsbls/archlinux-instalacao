Guia de Instalação do Arch Linux
Este manual é um guia passo a passo para a instalação do Arch Linux, baseado nas anotações de instalação.

1. Preparação
Conectar à internet
Se você precisar de uma conexão Wi-Fi, use os seguintes comandos:

Bash

iwctl
station <device_name> scan
station <device_name> get-networks
station <device_name> connect <network_name>
Para listar os dispositivos de rede, use device list.

Ajustar o Layout do Teclado
Para definir o teclado para o padrão brasileiro:

Bash

loadkeys br-abnt2
2. Particionamento do Disco
Use a ferramenta fdisk para particionar o disco. Certifique-se de substituir <device_name> pelo seu disco (ex: /dev/sda ou /dev/nvme0n1).

Bash

fdisk /dev/<device_name>
Crie as seguintes partições:

/ (root): ext4

/home: ext4

/boot/efi: fat32 (para sistemas UEFI)

swap: linuxswap

Após criar as partições, formate-as:

Bash

mkfs.ext4 /dev/<particao_root>
mkfs.ext4 /dev/<particao_home>
mkfs.fat -F32 /dev/<particao_boot_efi>
mkswap /dev/<particao_swap>
Montar as Partições
Monte as partições no diretório /mnt.

Bash

mount /dev/<particao_root> /mnt
mkdir /mnt/home
mount /dev/<particao_home> /mnt/home
mkdir /mnt/boot
mkdir /mnt/boot/efi
mount /dev/<particao_boot_efi> /mnt/boot/efi
swapon /dev/<particao_swap>
3. Instalação do Sistema Base
Configurar o Pacman
Instale o reflector para otimizar a lista de espelhos.

Bash

pacman -S reflector
Edite o arquivo de espelhos para adicionar os servidores de sua preferência.

Bash

nano /etc/pacman.d/mirrorlist
Adicione as seguintes linhas no topo do arquivo para priorizar os servidores do Brasil e o servidor osbeck:

Server = https://mirror.ufam.edu.br/archlinux/$repo/os/$arch
Server = https://mirror.osbeck.com/archlinux/$repo/os/$arch
Você também pode usar o reflector para gerar uma lista atualizada, baseada em velocidade e outros critérios:

Bash

reflector --country Brazil --latest 20 --sort rate --verbose --save /etc/pacman.d/mirrorlist
Adicionalmente, ative a cor e o repositório multilib no pacman para uma melhor visualização.

Bash

nano /etc/pacman.conf
Encontre as linhas #Color e #ParallelDownloads = 5 e remova o # para ativá-las. Salve o arquivo.

Em seguida, procure pela seção [multilib] e remova o # das duas linhas para habilitar o repositório:

[multilib]
Include = /etc/pacman.d/mirrorlist
Instalar o Sistema Base
Instale os pacotes essenciais, incluindo o linux-headers:

Bash

pacstrap /mnt base base-devel linux linux-headers linux-firmware nano vim
4. Configuração do Sistema
Gerar o Fstab
Gere o arquivo fstab para definir como as partições serão montadas na inicialização.

Bash

genfstab -U /mnt >> /mnt/fstab
Chroot e Configurações de Sistema
Entre no ambiente do sistema instalado com arch-chroot.

Bash

arch-chroot /mnt
Fuso Horário e Idioma
Defina o fuso horário e o idioma. Para o Brasil, use:

Bash

ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
timedatectl set-ntp true
hwclock --systohc
Edite o arquivo locale.gen e descomente o seu idioma (ex: pt_BR.UTF-8).

Bash

nano /etc/locale.gen
locale-gen
Crie o arquivo locale.conf:

Bash

echo "LANG=pt_BR.UTF-8" > /etc/locale.conf
Configure o layout do teclado para o console, para que ele persista após a reinicialização.

Bash

nano /etc/vconsole.conf
Adicione a seguinte linha:

KEYMAP=br-abnt2
Rede e Nome do Host
Defina o nome do seu computador (hostname).

Bash

echo "archlinux" > /etc/hostname
Nota sobre Hostname e Hosts: O hostname é o nome da sua máquina na rede. Ele é definido no arquivo /etc/hostname. Já o arquivo /etc/hosts faz o mapeamento do nome do host para um endereço IP. É crucial que o nome utilizado em ambos os arquivos seja o mesmo.

Edite o arquivo hosts:

Bash

nano /etc/hosts
Adicione as seguintes linhas, substituindo archlinux pelo nome que você escolheu:

127.0.0.1   localhost
::1         localhost
127.0.1.1   archlinux.localdomain   archlinux
Senha de Root e Usuário
Defina a senha para o usuário root:

Bash

passwd
Crie um novo usuário e adicione-o ao grupo wheel:

Bash

useradd -m -G wheel <nome_do_usuario>
passwd <nome_do_usuario>
Habilitar Sudoers:
Para dar ao seu novo usuário privilégios de administrador, edite o arquivo sudoers. É obrigatório usar o comando visudo para evitar erros de sintaxe.

Bash

EDITOR=nano visudo
Procure a linha que permite que o grupo wheel use o sudo e remova o # no início:

# %wheel ALL=(ALL:ALL) ALL
Salve o arquivo e saia.

5. Configuração de Gráficos (NVIDIA)
Esta seção é para usuários de placas de vídeo NVIDIA, garantindo o desempenho gráfico e a aceleração por hardware.

Instale os pacotes do driver:

nvidia: O driver principal.

nvidia-utils: Ferramentas como nvidia-smi para monitorar a placa.

lib32-nvidia-utils: Bibliotecas de 32 bits, necessárias para rodar aplicações e jogos mais antigos (requer o repositório multilib).

Bash

pacman -S nvidia nvidia-utils lib32-nvidia-utils
Regenere o initramfs:
Este comando regenera o initramfs, um pequeno sistema de ficheiros inicial que carrega os módulos do kernel (incluindo o da NVIDIA) antes do sistema principal arrancar.

Bash

mkinitcpio -P
Ative o DRM Kernel Mode Setting:
Edite o arquivo mkinitcpio.conf e adicione nvidia nvidia_modeset nvidia_uvm nvidia_drm na seção MODULES.

Bash

nano /etc/mkinitcpio.conf
A linha deve ficar parecida com a seguinte:

MODULES=(... nvidia nvidia_modeset nvidia_uvm nvidia_drm)
Após editar o arquivo, execute novamente o mkinitcpio -P para aplicar a mudança.

6. Bootloader e Finalização
Instalar e Configurar o GRUB
Instale os pacotes necessários para o bootloader:

Bash

pacman -S dosfstools mtools os-prober efibootmgr grub networkmanager iwd
Desabilite o OS Prober para evitar problemas.
Edite o arquivo de configuração do grub.

Bash

nano /etc/default/grub
Procure a linha #GRUB_DISABLE_OS_PROBER e remova o # para ativá-la.

Instale e Gere o GRUB:

Instale o GRUB na partição EFI:

Bash

grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=archlinux --recheck
Gere o arquivo de configuração do GRUB:

Bash

grub-mkconfig -o /boot/grub/grub.cfg
Configurar e Ativar a Rede
Para o NetworkManager usar o iwd como backend para o Wi-Fi, crie e edite o arquivo de configuração:

Bash

nano /etc/NetworkManager/conf.d/wifi_backend.conf
Adicione o seguinte conteúdo ao arquivo:

[device]
wifi.backend=iwd
Habilite o serviço para gerenciar a rede na próxima inicialização.

Bash

systemctl enable NetworkManager
Reiniciar
Saia do ambiente chroot e reinicie o sistema.

Bash

exit
shutdown -r now
Após o reinício, remova o pendrive de instalação e o sistema deve iniciar no Arch Linux que você instalou.
