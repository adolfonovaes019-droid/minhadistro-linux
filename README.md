# MinhaDistro

Distribuição Linux personalizada baseada em **Debian Bookworm i386 (32-bit)**, construída do zero e otimizada para hardware legado. Sistema final em produção em um **HP Mini 110** (Intel Atom N270, 2GB RAM, SSD 120GB) de 2009.

Projeto pessoal de estudo de Linux e infraestrutura, desenvolvido como parte do portfólio acadêmico do curso de Análise e Desenvolvimento de Sistemas (UNIP Campinas).

## Origem

O projeto começou com um HP Mini 110 que estava parado com Windows 7, 1GB de RAM e HD de 80GB. A primeira tentativa foi um upgrade de hardware — RAM expandida para 2GB e substituição do HD por um SSD de 120GB — para ver se o desempenho melhorava o suficiente para uso prático. O ganho foi marginal.

A segunda tentativa foi instalar o antiX Linux, uma distribuição leve voltada para hardware antigo. O sistema funcionou, mas durante o processo de estudo do antiX surgiu uma pergunta: por que não construir uma distribuição própria do zero?

Foi a partir daí que o projeto MinhaDistro teve início — com um objetivo concreto além do estudo: hospedar o Aura System, um sistema de PDV desenvolvido em Python/Tkinter com SQLite, já em uso real no Magrão Lanches em Valinhos/SP. A ideia era ter uma distro enxuta, construída especificamente para rodar essa aplicação em autostart, transformando o netbook em um terminal de ponto de venda dedicado.

---

## Objetivo

Construir uma distribuição Linux funcional sem o uso de live-CDs prontos ou geradores automáticos de ISO, com os seguintes propósitos:

- Aprofundar o conhecimento sobre a arquitetura interna de sistemas Linux — sistema de arquivos, processo de inicialização, gerenciamento de serviços, servidor gráfico e drivers
- Validar a viabilidade técnica de hardware legado (Intel Atom, 2GB RAM) como plataforma para aplicações reais de pequeno porte
- Hospedar uma aplicação Python de ponto de venda em ambiente de produção sobre hardware de 2009
- Documentar cada etapa do processo, incluindo erros encontrados e suas respectivas soluções, como material técnico de referência

---

## Stack Técnica

| Camada | Componente |
|---|---|
| Sistema base | Debian 12 Bookworm i386 (32-bit, sem PAE) |
| Sistema de init | systemd |
| Bootloader | GRUB 2 (target i386-pc) |
| Servidor gráfico | Xorg |
| Gerenciador de janelas | IceWM |
| Gerenciador de login | LightDM |
| Gerenciador de arquivos | PCManFM |
| Terminal | LXTerminal |
| Navegador | Firefox-ESR |
| Gerenciador de rede | NetworkManager |
| Monitor de sistema | Conky |
| Aplicação hospedada | Aura System (PDV em Python/Tkinter + SQLite) |
| Hardware-alvo | HP Mini 110 — Intel Atom N270 @1.6GHz, 2GB DDR2, SSD 120GB, Wi-Fi Atheros AR9285, tela 1024×600 |

---

## Fases do Projeto

### Fase 1 — Sistema base
Instalação do Debian Bookworm i386 em máquina virtual VirtualBox, com configuração de rede, repositórios, criação de usuário com privilégios sudo e ajuste do layout de teclado ABNT2 no console TTY.

### Fase 2 — Ambiente gráfico
Instalação manual do servidor X (Xorg), avaliação inicial do Openbox como gerenciador de janelas (posteriormente substituído pelo IceWM por simplicidade de configuração), e configuração do LightDM como gerenciador de login, incluindo personalização do tema da tela de autenticação.

### Fase 3 — Aplicação e usabilidade
Integração da aplicação Aura System (sistema de PDV desenvolvido em Python com Tkinter e SQLite) ao ambiente, instalada em ambiente virtual Python (`venv`). Configuração de autostart no IceWM através do arquivo `~/.icewm/startup`, criação de item personalizado no menu do gerenciador de janelas e ajuste da resolução nativa do display do HP Mini (1024×600).

### Fase 4 — Implantação intermediária em pendrive
Clonagem do sistema da máquina virtual para um pendrive USB utilizando `rsync` com flags específicas para preservar permissões e atributos estendidos. Instalação do GRUB no dispositivo USB com `grub-install --target=i386-pc` para permitir boot independente. Primeira validação em hardware real (HP Mini 110).

### Fase 5 — Implantação definitiva em SSD interno
Formatação da partição interna do SSD em ext4, clonagem do sistema do pendrive para o SSD via `rsync`, instalação do GRUB no `/dev/sda`, ajuste dos UUIDs nos arquivos `/etc/fstab` e `/boot/grub/grub.cfg`, e validação completa do sistema com a aplicação Aura System em autostart funcional.

### Fase 6 — Rede e configurações finais
Configuração do Wi-Fi (placa Atheros AR9285, driver `ath9k` nativo no kernel), instalação e configuração do NetworkManager com reconexão automática no boot, fixação do DNS via NetworkManager para evitar sobrescrita pelo DHCP, correção do relógio do sistema e instalação do Conky como monitor de sistema em autostart.

---

## Desafios Técnicos Resolvidos

A seguir, os principais problemas enfrentados durante o desenvolvimento e suas respectivas soluções, documentados como referência técnica.

### 1. Layout de teclado ABNT2 incorreto no console TTY

**Sintoma:** Caracteres especiais como `/`, `?` e `;` produzindo saída incorreta no console de texto.

**Solução:** Reconfiguração do pacote de teclado com `sudo dpkg-reconfigure keyboard-configuration` seguido de `sudo setupcon`, ajustando o parâmetro `XKBMODEL=abnt2` em `/etc/default/keyboard`. Para correção em tempo real durante sessão X, `setxkbmap us` resolve imediatamente. Adicionado ao `~/.icewm/startup` para aplicação automática a cada inicialização.

### 2. Ordem de boot incorreta no VirtualBox

**Sintoma:** A máquina virtual ignorava o disco virtual configurado e tentava inicializar a partir de mídia inexistente.

**Solução:** Remoção do controlador IDE vazio que estava configurado como dispositivo prioritário na ordem de boot da VM.

### 3. Diretórios essenciais ausentes após `rsync`

**Sintoma:** Kernel panic durante o boot do sistema clonado, com mensagem `mounting /dev failed: No such file or directory`.

**Causa raiz:** A exclusão dos diretórios virtuais `/dev`, `/proc`, `/sys`, `/run` e `/tmp` no `rsync` é necessária e correta (esses diretórios são montados em tempo de execução pelo kernel), mas os mountpoints vazios precisam existir no destino para que o kernel possa montar os filesystems virtuais sobre eles.

**Solução:** Criação manual dos diretórios no destino com `sudo mkdir -p` seguido de `sudo chmod 1777` no diretório `/tmp` para aplicar o sticky bit obrigatório.

### 4. UUID inconsistente entre origem e destino

**Sintoma:** Boot do sistema clonado caía no shell de emergência do BusyBox com a mensagem `ALERT! UUID=... does not exist`.

**Causa raiz:** Os arquivos `/etc/fstab` e `/boot/grub/grub.cfg` no destino mantinham os UUIDs da partição de origem (VM), enquanto o kernel buscava montar a partição com o UUID real do dispositivo de destino.

**Solução:** Identificação do UUID correto via `sudo blkid /dev/sdX1`, seguido de substituição com `sed -i 's/UUID-ANTIGO/UUID-NOVO/g'` aplicado a ambos os arquivos.

### 5. Sistema de arquivos montado em modo somente-leitura

**Sintoma:** Após o sistema inicializar parcialmente, qualquer operação de escrita retornava `Sistema de arquivos somente para leitura`. O servidor X falhava ao tentar criar `/tmp/.tX0-lock` e `.Xauthority`.

**Causa raiz:** O UUID incorreto no fstab provocava falha de montagem, e a opção `errors=remount-ro` (padrão em instalações Debian) reagia remontando a raiz em modo somente-leitura como medida de proteção.

**Solução:** Verificação da integridade do sistema de arquivos com `sudo fsck -y /dev/sdX1` (confirmando que o disco estava saudável e descartando dano físico), seguida da correção do UUID.

### 6. Erro de digitação silencioso em UUID

**Sintoma:** Após a aplicação do `sed` para corrigir UUIDs no SSD, o boot continuava falhando com `UUID=95a27efi-... does not exist`.

**Causa raiz:** Erro de digitação no comando `sed` — o caractere `1` (número) foi substituído por `i` (letra) no UUID de destino. O comando foi executado sem aviso e propagou o valor incorreto em ambos os arquivos.

**Lição aprendida:** Sempre validar o UUID de destino com `blkid` imediatamente antes da execução de `sed`, evitando depender de memória ou cópias manuais.

**Solução:** Boot pelo pendrive de recuperação e nova execução do `sed`, substituindo `efi` por `ef1` nos dois arquivos do SSD.

### 7. Ausência de `grub-mkconfig` no sistema clonado

**Sintoma:** Comando `grub-mkconfig: comando não encontrado` ao tentar gerar o arquivo de configuração do GRUB para o SSD a partir do sistema vivo do pendrive.

**Solução:** Cópia direta do arquivo `grub.cfg` existente no pendrive para o SSD com `cp`, seguida de substituição do UUID no arquivo copiado via `sed`.

### 8. Interface de rede cabeada não inicializando automaticamente

**Sintoma:** Porta Ethernet conectada mas sem IP atribuído. `ip addr` mostrava interface `enp2s0` em estado DOWN.

**Solução:** Ativação manual da interface com `sudo ip link set enp2s0 up` e requisição de IP via `sudo dhclient enp2s0`. Para conexão permanente, configurar a interface no NetworkManager.

### 9. DNS sobrescrito pelo NetworkManager a cada reconexão

**Sintoma:** Após cada reconexão Wi-Fi, o arquivo `/etc/resolv.conf` era sobrescrito com DNS inválido, causando falha na resolução de nomes e impedindo o `apt` de baixar pacotes.

**Solução:** Configuração do DNS diretamente no perfil de conexão do NetworkManager:
```bash
sudo nmcli con mod antiX ipv4.dns "8.8.8.8 8.8.4.4"
sudo nmcli con mod antiX ipv4.ignore-auto-dns yes
```
Isso impede que o DHCP sobrescreva as configurações de DNS.

### 10. Relógio do sistema desatualizado bloqueando o `apt`

**Sintoma:** `apt update` retornava `Release file is not valid yet` e recusava todos os pacotes.

**Causa raiz:** O relógio do sistema estava configurado para 2001 (data padrão após formatação do SSD), e os certificados do Debian tinham datas futuras para o sistema.

**Solução:** Ajuste manual do relógio com `sudo date -s "AAAA-MM-DD HH:MM:SS"`. Para sincronização permanente, configurar o NTP após o Wi-Fi estar funcional.

### 11. Permissão de acesso à impressora USB no Linux
Sintoma: Ao tentar finalizar um pedido no Aura System, aparecia a mensagem Sem permissão em /dev/usb/lp0.
Causa raiz: No Linux, o dispositivo de impressora (/dev/usb/lp0) pertence ao grupo lp. O usuário não estava nesse grupo, portanto não tinha permissão de acesso ao dispositivo.

Solução:
bashsudo usermod -aG lp adolfo

Após reiniciar a sessão, o usuário passa a ter acesso ao dispositivo de impressora. O próprio Aura System detectou o ambiente Linux e exibiu o comando correto na mensagem de erro — resultado das adaptações de compatibilidade feitas durante a portabilidade do Windows para o Linux.

---

## Comandos Principais Utilizados

```bash
# Configuração de teclado
sudo dpkg-reconfigure keyboard-configuration
sudo setupcon
setxkbmap us                    # ajuste em tempo real na sessão X
echo "setxkbmap us &" >> ~/.icewm/startup   # permanente no autostart

# Instalação dos pacotes principais
sudo apt install xorg icewm lightdm pcmanfm lxterminal -y
sudo apt install firefox-esr mousepad galculator vlc htop -y
sudo apt install python3 python3-venv python3-pil -y
sudo apt install network-manager conky-all x11-xkb-utils -y

# Configuração de autostart (IceWM)
mkdir -p ~/.icewm
echo "python3 /home/adolfo/aura/main.py &" >> ~/.icewm/startup
echo "setxkbmap us &" >> ~/.icewm/startup
echo "conky &" >> ~/.icewm/startup
chmod +x ~/.icewm/startup

# Preparação e clonagem para dispositivo externo
sudo mkfs.ext4 /dev/sda1
sudo mount /dev/sda1 /mnt/ssd
sudo rsync -aAX --progress / /mnt/ssd/ \
  --exclude=/mnt --exclude=/proc --exclude=/sys \
  --exclude=/dev --exclude=/run --exclude=/tmp

# Criação dos mountpoints virtuais (etapa obrigatória)
sudo mkdir -p /mnt/ssd/proc /mnt/ssd/sys /mnt/ssd/dev /mnt/ssd/run /mnt/ssd/tmp
sudo chmod 1777 /mnt/ssd/tmp

# Instalação do bootloader
sudo grub-install --target=i386-pc --boot-directory=/mnt/ssd/boot /dev/sda

# Ajuste de UUIDs (sempre validar com blkid antes da substituição)
sudo blkid /dev/sda1
sudo sed -i 's/UUID-ANTIGO/UUID-NOVO/g' /mnt/ssd/etc/fstab
sudo sed -i 's/UUID-ANTIGO/UUID-NOVO/g' /mnt/ssd/boot/grub/grub.cfg

# Rede via cabo (manual)
sudo ip link set enp2s0 up
sudo dhclient enp2s0

# Wi-Fi via NetworkManager
nmcli dev wifi list
nmcli dev wifi connect NOME_REDE password SENHA
sudo nmcli con mod NOME_REDE ipv4.dns "8.8.8.8 8.8.4.4"
sudo nmcli con mod NOME_REDE ipv4.ignore-auto-dns yes

# Relógio do sistema
sudo date -s "2026-06-25 10:00:00"

# Diagnóstico geral
lsblk                           # listar dispositivos de bloco
ip addr                         # ver interfaces de rede e IPs
mount | grep ' / '              # verificar montagem da raiz
dmesg | tail -30                # consultar eventos recentes do kernel
sudo fsck -y /dev/sdXY          # reparar sistema de arquivos (desmontado)
```

---

## Progresso

- [x] Sistema base Debian i386 instalado e configurado
- [x] Layout de teclado ABNT2 funcional em console e em ambiente gráfico
- [x] Servidor X (Xorg), IceWM e LightDM operacionais em máquina virtual
- [x] Aplicação Aura System integrada com autostart configurado
- [x] Implantação intermediária validada em pendrive USB
- [x] Implantação definitiva no SSD interno do HP Mini 110
- [x] Wi-Fi configurado com reconexão automática no boot (Atheros AR9285)
- [x] DNS permanente configurado via NetworkManager
- [x] Conky instalado e em autostart com monitor de CPU, RAM, disco e rede
- [x] Layout de teclado `us` aplicado automaticamente no autostart do IceWM
- [ ] Sincronização de horário automática via NTP
- [ ] ISO instalável customizada (via Calamares ou live-build)

---

## Aprendizados Técnicos

Os conhecimentos consolidados ao longo deste projeto incluem:

- Estrutura interna de uma distribuição Linux, incluindo o diretório `/boot`, o papel do `vmlinuz`, do `initrd` e do GRUB
- Distinção entre filesystems persistentes (ext4 em partições reais) e filesystems virtuais montados em tempo de execução (`/proc`, `/sys`, `/dev`)
- Mecanismo de montagem da partição raiz pelo kernel através do UUID, e a criticidade desse identificador na cadeia de boot
- Funcionamento da opção `errors=remount-ro` no fstab, tanto como mecanismo de proteção contra corrupção quanto como possível causa de inacessibilidade do sistema
- Comportamento do `rsync` ao clonar sistemas em execução, com atenção aos excludes obrigatórios
- Processo pelo qual o GRUB localiza o kernel e a partição raiz através de `search --fs-uuid`
- Importância da validação de valores críticos (UUIDs, identificadores de dispositivos) antes da aplicação de comandos destrutivos
- Diferenças entre comportamento de sistemas em máquina virtual e em hardware real, particularmente em questões de boot e drivers
- Gerenciamento de interfaces de rede via NetworkManager e configuração de DNS permanente para evitar sobrescrita pelo DHCP
- Funcionamento do driver `ath9k` para placas Atheros e a distinção entre drivers nativos do kernel e firmware proprietário

---

## Autor

**Adolfo Novaes**
Técnico em TI Autônomo · Estudante de Análise e Desenvolvimento de Sistemas (UNIP Campinas)

- LinkedIn: [Adolfo Novaes](https://linkedin.com/in/adolfo-rodrigues10)
- Aura Tech — Serviços de TI em Valinhos/SP: [auratechvalinhos.vercel.app](https://auratechvalinhos.vercel.app)

---

## Licença

Projeto desenvolvido para fins educacionais e de portfólio. O conteúdo deste repositório pode ser livremente consultado, reproduzido e adaptado.
