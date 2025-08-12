# 🐧 GUIA DE INSTALAÇÃO ARCH LINUX

UM GUIA UTILITÁRIO E DETALHADO PARA A INSTALAÇÃO MANUAL.

**Agradecimento**: TuxStation e seu vídeo [tutorial](https://www.youtube.com/watch?v=QYaYUxtFMII&list=PLZftrOaRlDDuu6dWnmSp65MP-Lzmkg9FL) no youtube.

## Sumário

1. Download e Preparação
    
2. Configuração Inicial
    
3. Particionamento do Disco
    
4. Formatação e Montagem
    
5. Instalação do Sistema Base
    
6. Configuração (Chroot)
    
7. Bootloader (GRUB)
    
8. Drivers de Vídeo (NVIDIA)
    
9. Finalização
    

### Download e Preparação

Nesta fase inicial, o objetivo é obter a imagem de instalação oficial do Arch Linux e gravá-la num pen drive, tornando-o inicializável (bootável). Isto permite que o seu computador arranque a partir do pen drive e carregue o ambiente de instalação temporário do Arch.

1. **Baixe a ISO:** A imagem ISO é um arquivo único que contém todo o sistema de instalação. É fundamental baixá-la do [site oficial](https://archlinux.org/download/ "null") para garantir a sua integridade e segurança.
    
2. **Crie um Pen Drive Bootável:** Ferramentas como Balena Etcher, Ventoy ou Rufus descompactam a imagem ISO e preparam o pen drive para que a BIOS/UEFI do seu computador o reconheça como um dispositivo de arranque.
    
3. **Inicie pelo Pen Drive:** Ao selecionar `Arch Linux install medium`, está a instruir o computador a carregar o ambiente live, que corre inteiramente a partir da RAM, sem tocar ainda no seu disco rígido.
    

### Configuração Inicial

Antes de instalar, precisamos de configurar o ambiente live para que seja utilizável.

1. **Layout do Teclado (BR-ABNT2):** O sistema live vem por defeito com o layout de teclado americano. Este comando ajusta-o para o padrão brasileiro, permitindo o uso correto de acentos e caracteres como o "ç".
    
    ```
    loadkeys br-abnt2
    ```
    
2. **Conexão Wi-Fi:** O `iwctl` é uma ferramenta interativa para gerir redes sem fios. É essencial para se conectar à internet, o que é necessário para descarregar os pacotes do sistema.
    
    ```
    iwctl
    ```
    
3. **Verifique a conexão:** O comando `ping` envia um pequeno pacote de dados para um servidor na internet e espera por uma resposta. É a forma mais simples de confirmar que a sua conexão está ativa.
    
    ```
    ping archlinux.org
    ```
    

### Particionamento do Disco

Esta é uma das etapas mais críticas. Particionar é como dividir o seu disco rígido (ou SSD) em diferentes secções, onde cada uma terá um propósito específico.

1. **Identifique o modo de firmware:** Saber se o seu sistema usa **UEFI** (moderno) ou **BIOS** (antigo) determina como o disco deve ser preparado para o arranque. O comando verifica a existência de um diretório que só existe em sistemas UEFI.
    
    ```
    ls /sys/firmware/efi/efivars
    ```
    
2. **Particione com `cfdisk`:** `lsblk` lista os seus discos. O `cfdisk` é uma ferramenta visual para criar, apagar e modificar as partições (ex: `/dev/sda` para um disco SATA, `/dev/nvme0n1` para um SSD NVMe).
    
    ```
    lsblk
    cfdisk /dev/sda
    ```
    

### Formatação e Montagem

Depois de criar as partições, elas são como terrenos vazios. A formatação cria um sistema de ficheiros (como `ext4` ou `FAT32`), que é a estrutura que organiza como os dados são guardados. A montagem associa essas partições formatadas a diretórios do sistema.

1. **Formate as partições:**
    
    - `mkfs.fat`: Formata a partição EFI em `FAT32`, o padrão exigido pela especificação UEFI.
        
    - `mkfs.ext4`: Formata as partições do sistema (`/`) e dos seus ficheiros pessoais (`/home`) com `ext4`, um sistema de ficheiros robusto e popular para Linux.
        
    - `mkswap`: Prepara a partição de swap, que funciona como uma extensão da memória RAM.
        
    
    ```
    # UEFI
    mkfs.fat -F32 /dev/sda1
    # Raiz e Home
    mkfs.ext4 /dev/sda2
    # Swap
    mkswap /dev/sda4
    ```
    
2. **Monte as partições:**
    
    - `mount`: Associa a partição raiz (`/dev/sda2`) ao diretório `/mnt`. A partir de agora, tudo o que for instalado em `/mnt` será, na verdade, gravado no seu disco.
        
    - `swapon`: Ativa a partição de swap.
        
    
    ```
    mount /dev/sda2 /mnt
    mkdir -p /mnt/boot /mnt/home
    mount /dev/sda1 /mnt/boot
    mount /dev/sda3 /mnt/home
    swapon /dev/sda4
    ```
    

### Instalação do Sistema Base

Agora, com o disco preparado, vamos descarregar e instalar os pacotes essenciais que formam o núcleo do Arch Linux.

1. **Otimize os espelhos (opcional):** O `reflector` testa e seleciona os servidores de download (espelhos) mais rápidos para a sua localização, acelerando significativamente a instalação.
    
    ```
    pacman -Sy reflector
    reflector --country Brazil --latest 20 --sort rate --save /etc/pacman.d/mirrorlist
    ```
    
2. **Instale os pacotes base:** O `pacstrap` é um script que instala os pacotes diretamente no diretório montado (`/mnt`).
    
    - `base`: O conjunto mínimo de pacotes para um sistema funcional.
        
    - `base-devel`: Ferramentas de compilação, úteis para instalar software do AUR (Arch User Repository).
        
    - `linux`: O kernel do Linux, o coração do sistema operativo.
        
    - `linux-firmware`: Ficheiros de firmware necessários para que o kernel comunique com diversos hardwares (placas de rede, gráficas, etc.).
        
    - `nano`: Um editor de texto simples para a linha de comandos.
        
    - `networkmanager`: Um serviço essencial para gerir as conexões de rede após a instalação.
        
    
    ```
    pacstrap /mnt base base-devel linux linux-firmware nano networkmanager
    ```
    

### Configuração (Chroot)

Até agora, estávamos a operar a partir do ambiente live do pen drive. O `arch-chroot` permite-nos "entrar" no sistema que acabámos de instalar no disco rígido e configurá-lo por dentro, como se já tivéssemos arrancado a partir dele.

1. **Acesse o novo sistema:**
    
    - `genfstab`: Gera um ficheiro que diz ao sistema quais partições montar e onde, durante o arranque.
        
    - `arch-chroot`: Muda a raiz do ambiente atual para `/mnt`.
        
    
    ```
    genfstab -U /mnt >> /mnt/etc/fstab
    arch-chroot /mnt
    ```
    
2. **Configure sistema (local, tempo, teclado):** Estas configurações definem o fuso horário, o idioma e o layout de teclado padrão para o seu novo sistema.
    
    ```
    ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
    hwclock --systohc
    locale-gen
    echo "LANG=pt_BR.UTF-8" > /etc/locale.conf
    echo "KEYMAP=br-abnt2" > /etc/vconsole.conf
    ```
    
3. **Crie usuário e senhas:** Por segurança, não se deve usar o superutilizador (`root`) para tarefas diárias. Criamos um utilizador normal e definimos senhas para ambos. O grupo `wheel` é o grupo padrão que pode obter privilégios de administrador.
    
    ```
    passwd
    useradd -m -G wheel <seu_usuario>
    passwd <seu_usuario>
    ```
    
4. **Habilite o sudo:** O `sudo` permite que um utilizador normal execute comandos como `root` de forma segura. O `visudo` é a ferramenta segura para editar as suas permissões, e ao descomentar a linha `%wheel ALL=(ALL:ALL) ALL`, estamos a dar permissão de `sudo` a todos os utilizadores do grupo `wheel`.
    
    ```
    pacman -S sudo
    EDITOR=nano visudo
    ```
    

### Bootloader (GRUB)

O bootloader é o primeiro programa que corre quando liga o computador. A sua função é carregar o kernel do sistema operativo (neste caso, o Linux). O GRUB é um dos bootloaders mais populares e poderosos.

1. **Instale pacotes:**
    
    - `grub`: O bootloader em si.
        
    - `efibootmgr`: Ferramenta para gerir as entradas de arranque da UEFI.
        
    - `os-prober`: Deteta outros sistemas operativos (como o Windows) para adicionar ao menu de arranque (dual boot).
        
    
    ```
    pacman -S grub efibootmgr os-prober
    ```
    
2. **Instale o GRUB (UEFI):** Este comando instala os ficheiros do GRUB na partição EFI e cria uma entrada de arranque na UEFI do seu computador.
    
    ```
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Arch
    ```
    
3. **Gere a configuração:** O `grub-mkconfig` gera o ficheiro de configuração `grub.cfg`, que contém o menu que vê ao ligar o PC, com as opções para arrancar o Arch Linux.
    
    ```
    grub-mkconfig -o /boot/grub/grub.cfg
    ```
    

### Drivers de Vídeo (NVIDIA)

Esta etapa é crucial para quem tem uma placa gráfica NVIDIA, garantindo o desempenho gráfico e a aceleração por hardware.

1. **Instale os pacotes do driver:**
    
    - `nvidia`: O driver principal.
        
    - `nvidia-utils`: Ferramentas como `nvidia-smi` para monitorizar a placa.
        
    - `lib32-nvidia-utils`: Bibliotecas de 32 bits, necessárias para correr aplicações e jogos mais antigos (requer o repositório `multilib`).
        
    
    ```
    pacman -S nvidia nvidia-utils lib32-nvidia-utils
    ```
    
2. **`mkinitcpio`:** Este comando regenera o `initramfs`, um pequeno sistema de ficheiros inicial que carrega os módulos do kernel (incluindo o da NVIDIA) antes do sistema principal arrancar. Geralmente, é executado automaticamente pela instalação do driver.
    
    ```
    mkinitcpio -P
    ```
    
3. **Ative o DRM Kernel Mode Setting:** O `DRM (Direct Rendering Manager)` é um subsistema do kernel que gere a saída de vídeo. Ativar o `modeset` para a NVIDIA permite uma transição mais suave e estável do ecrã de arranque para o ambiente gráfico, evitando ecrãs pretos.
    
4. **Regenere a configuração do GRUB:** É necessário para que a opção `nvidia_drm.modeset=1` que adicionou seja aplicada no próximo arranque.
    
    ```
    grub-mkconfig -o /boot/grub/grub.cfg
    ```
    

### Finalização

Os últimos passos para garantir que o sistema arranque corretamente e com acesso à rede.

1. **Habilite a rede e saia:**
    
    - `systemctl enable NetworkManager`: Configura o serviço de rede para iniciar automaticamente com o sistema.
        
    - `exit`: Sai do ambiente `chroot`, voltando para o ambiente live do pen drive.
        
    
    ```
    systemctl enable NetworkManager
    exit
    ```
    
2. **Desmonte e reinicie:**
    
    - `umount -R /mnt`: Desmonta todas as partições de forma segura antes de reiniciar.
        
    - `shutdown -r now`: Reinicia o computador. Não se esqueça de remover o pen drive!


        
    
    ```
    umount -R /mnt
    shutdown -r now
    ```
