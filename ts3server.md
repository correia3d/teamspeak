#!/bin/bash
# Instalador Teamspeak Server

INSTALLER_VERSION="1.12"

function debug() {
    #echo "debug > ${@}"
    :
}

function warn() {
    echo -e "\\033[33;1m${@}\033[0m"
}

function error() {
    echo -e "\\033[31;1m${@}\033[0m"
}

function info() {
    echo -e "\\033[36;1m${@}\033[0m"
}

function green() {
    echo -e "\\033[32;1m${@}\033[0m"
}

function cyan() {
    echo -e "\\033[36;1m${@}\033[0m"
}

function red() {
    echo -e "\\033[31;1m${@}\033[0m"
}

function yellow() {
    echo -e "\\033[33;1m${@}\033[0m"
}

function option_quit_installer() {
    red "Instalador Teamspeak Server Fechado."
    exit 0
}

function invalid_option() {
    red "Opção inválida. Tente novamente."
}

function red_sleep() {
    red $1
    sleep 1
}

function green_sleep() {
    green $1
    sleep 1
}

function yellow_sleep() {
    yellow $1
    sleep 1
}

# Cheque se sudo está instalado.
if sudo -v >/dev/null 2>&1; then
    SUDO_PREFIX="sudo"
    debug "Sudo installed"
elif [ "`id -u`" != "0" ]; then
    error "Privilégios de root ou sudo são necessários para executar o script de instalação!"
    exit 1
fi

# Detectar arquitetura.
MACHINE_TYPE=`${SUDO_PREFIX} uname -m`
if [ ${MACHINE_TYPE} == "x86_64" ]; then
    ARCH="amd64"
else
    ARCH="x86"
fi

function detect_packet_manager() {
    PACKET_MANAGER_NAME=""
    PACKET_MANAGER_TEST=""
    PACKET_MANAGER_UPDATE=""
    PACKET_MANAGER_INSTALL=""

    SUPPORT_SCREEN=false
    SUPPORT_FFMPEG=false
    SUPPORT_YTDL=false
    SUPPORT_LIBNICE=false

    NEDDED_REPO=false
	
    CentOS_REPO="https://negativo17.org/repos/epel-multimedia.repo"
    CentOS_6_REPO="${CentOS_REPO}"
    CentOS_7_REPO="${CentOS_REPO}"
    RedHat_REPO="${CentOS_REPO}"
    Fedora_REPO="https://negativo17.org/repos/fedora-multimedia.repo"

    #<system name>| <packages>| <manager>| <repos>| <system version> (command seperated by :)
    PACKET_MANAGERS=(
        "Debian|screen:ffmpeg:youtube-dl:libnice10|apt:dpkg-query||"
        "Ubuntu|screen:ffmpeg:youtube-dl:libnice10|apt:dpkg-query||"
        "openSUSE|screen:ffmpeg:youtube-dl:libnice10|yzpper:rpm||"
        "CentOS|screen:ffmpeg:youtube-dl:libnice:yum-utils|yum:rpm|${CentOS_6_REPO}|6"
        "CentOS|screen:ffmpeg:youtube-dl:libnice:yum-utils|yum:rpm|${CentOS_7_REPO}|7"
        "RedHat|screen:ffmpeg:youtube-dl:libnice:yum-utils|yum:rpm|${RedHat_REPO}|"
        "Arch|screen:ffmpeg:youtube-dl:libnice|pacman||"
        "Fedora|screen:ffmpeg:youtube-dl:libnice|dnf:rpm|${Fedora_REPO}|"
        "Mint|screen:ffmpeg:youtube-dl:libnice10|apt:dpkg-query||"
    )
    PACKET_MANAGER_COMMANDS=(
        "apt:dpkg-query|dpkg-query -s %s|apt update -y && ${SUDO_PREFIX} apt upgrade -y|apt install -y %s"
        "yzpper:rpm|rpm -q %s|zypper ref && ${SUDO_PREFIX} zypper up|zypper install -y %s"
        "yum:rpm|rpm -q %s|yum update -y|yum install -y %s"
        "pacman|pacman -Qi %s|pacman -Syu --noconfirm|pacman -S --noconfirm %s"
        "dnf:rpm|rpm -q %s|dnf check-update -y && ${SUDO_PREFIX} dnf upgrade -y|dnf install -y %s"
    )
    SYSTEM_NAME=$(cat /etc/*release | grep ^NAME)
    SYSTEM_VERSION=$(cat /etc/*release | grep ^VERSION_ID | sed 's/[^0-9]*//g')
    SYSTEM_NAME_DETECTED="" #Give the system out own name :D

    for system in ${PACKET_MANAGERS[@]}
    do
        IFS='|' read -r -a data <<< $system
        debug "Testing ${system} => ${data[0]}"

        if echo "${SYSTEM_NAME}" | grep "${data[0]}" &>/dev/null && echo "${SYSTEM_VERSION}" | grep "${data[4]}" &>/dev/null; then
            SYSTEM_NAME_DETECTED="${data[0]}"
            SYSTEM_VERSION="${data[4]}"
            if [ "${data[4]}" == "" ]; then
            SYSTEM_VERSION=$(cat /etc/*release | grep ^VERSION_ID | sed 's/[^0-9]*//g')
            fi
        else
            continue
        fi

        cyan " "
        green "${SYSTEM_NAME_DETECTED} ${SYSTEM_VERSION} (${ARCH}) detected!"
        debug "Found system ${data[0]} ${SYSTEM_VERSION} (${ARCH})"
        for index in $(seq 2 ${#data[@]})
        do
            PACKET_MANAGER_NAME=${data[${index}]}

            debug "Testing commands ${PACKET_MANAGER_NAME}"
            for command in ${PACKET_MANAGER_NAME//:/ }
            do
                debug "Testing command ${command}"
                if ! [ $(command -v ${command}) >/dev/null 2>&1 ]; then
                    PACKET_MANAGER_NAME=""
                    break
                fi
            done
            if [ ${PACKET_MANAGER_NAME} != "" ]; then
                break
            fi
        done

        if ! [ "${PACKET_MANAGER_NAME}" == "" ]; then
            # Supported packages.
            echo "${data[1]}" | grep "screen" />/dev/null 2>&1
            if [ $? -ne 0 ]; then
                SUPPORT_SCREEN=true
            fi
			
            echo "${data[1]}" | grep "ffmpeg" />/dev/null 2>&1
            if [ $? -ne 0 ]; then
                SUPPORT_FFMPEG=true
            fi
			
            echo "${data[1]}" | grep "youtube-dl" />/dev/null 2>&1
            if [ $? -ne 0 ]; then
                SUPPORT_YTDL=true
            fi
			
            echo "${data[1]}" | grep "(libnice|libnice10)" />/dev/null 2>&1
            if [ $? -ne 0 ]; then
                SUPPORT_LIBNICE=true
            fi

            # Additional repository enabled.
            if [ "${data[3]}" != "" ]; then
                debug "Additional repo needed"
                NEDDED_REPO=true
            fi
            break
        fi
    done

    if [ "${PACKET_MANAGER_NAME}" == "" ]; then
        error "Failed to determine your system and the packet manager on it! (System: ${SYSTEM_NAME_DETECTED} ${SYSTEM_VERSION} ${ARCH})"
        return 1
    fi

    IFS='~'
    for manager_commands in ${PACKET_MANAGER_COMMANDS[@]}
    do
        IFS='|' read -r -a commands <<< $manager_commands

        if [ "${commands[0]}" == "${PACKET_MANAGER_NAME}" ]; then
            PACKET_MANAGER_INSTALL="${commands[3]}"
            PACKET_MANAGER_UPDATE="${commands[2]}"
            PACKET_MANAGER_TEST="${commands[1]}"
            break
        fi
    done

    if [ "${PACKET_MANAGER_INSTALL}" == "" ]; then
        error "Falha ao encontrar comandos do gerenciador de pacotes para o gerenciador (${PACKET_MANAGER_NAME})"
        return 1
    fi

    info "Comandos do gerenciador de pacotes:"
    info "Instalar                        : ${PACKET_MANAGER_INSTALL}"
    info "Atualizar                       : ${PACKET_MANAGER_UPDATE}"
    info "Testar                          : ${PACKET_MANAGER_TEST}"
    info "Suporte de tela de pacotes      : ${SUPPORT_SCREEN}"
    info "Suporte ao pacote ffmpeg        : ${SUPPORT_FFMPEG}"
    info "Pacote youtube-dl suporte       : ${SUPPORT_YTDL}"
    info "Suporte a pacote libnice        : ${SUPPORT_LIBNICE}"
    info "Repositório adicional           : ${NEDDED_REPO}"
    debug "${data[3]}"
    return 0
}

function test_installed() {
    local require_install=()

    for package in "${@}"
    do
        local command=$(printf ${PACKET_MANAGER_TEST} "${package}")
        debug ${command}
        eval "${command}" &>/dev/null
        if [ $? -ne 0 ]; then
            cyan " "
            warn "${package} é necessário, mas falta!"
            require_install+=(${package})
        fi
    done

    if [ ${#require_install[@]} -lt 1 ]; then
        echo ${#require_install[@]} &>/dev/null
        return 0
    fi

    packages=$(printf " %s" "${require_install[@]}")
    packages=${packages:1}

    packages_human=$(printf ", \"%s\"" "${require_install[@]}")
    packages_human=${packages_human:2}

    cyan "Devemos instalar "${packages_human}" para você? (Privilégios de root ou administrador necessários)"
    OPTIONS=("Sim" "Não" "Sair")
    select OPTION in "${OPTIONS[@]}"; do
        case "$REPLY" in
            1|2) break;;
            3 ) option_quit_installer;;
            *)  invalid_option; continue;;
        esac
    done

    if [ "$OPTION" == "${OPTIONS[0]}" ]; then
        green_sleep "# Instalando "${packages_human}" pacotes..."
        local command=$(printf ${PACKET_MANAGER_INSTALL} "${packages}")
        debug ${SUDO_PREFIX} ${command}
        eval "${SUDO_PREFIX} ${command}"
        green_sleep "Pronto!"
        if [ $? -ne 0 ]; then
            error "Falha ao instalar o pacote necessário!"
            exit 1
        fi
        return 0
    fi
    return 2
}

function updateScript() {
    INSTALLER_REPO_URL="https://api.github.com/repos/Sporesirius/TeaSpeak-Installer/releases/latest"
    INSTALLER_REPO_PACKAGE="https://github.com/Sporesirius/TeaSpeak-Installer/archive/%s.tar.gz"

    cyan " "
    cyan "Verificando a versão mais recente do instalador..."
    LATEST_VERSION=$(curl -s --connect-timeout 10 -S -L ${INSTALLER_REPO_URL} | grep -Po '(?<="tag_name": ")([0-9]\.[0-9]+)')
    if [ $? -ne 0 ]; then
        warn "Falha ao verificar se há atualizações para este script!"
        return 1
    fi

    if [ "`printf "${LATEST_VERSION}\n${INSTALLER_VERSION}" | sort -V | tail -n 1`" != "$INSTALLER_VERSION" ]; then
        red "New version ${LATEST_VERSION} available!"
        yellow "Você está usando a versão ${INSTALLER_VERSION}."
        cyan " "
        cyan "Deseja atualizar o script do instalador?"
        OPTIONS=("Download" "Pular" "Sair")
        select OPTION in "${OPTIONS[@]}"; do
            case "$REPLY" in
                1|2 ) break;;
                3 ) option_quit_installer;;
                *)  invalid_option; continue;;
            esac
        done

        if [ "$OPTION" == "${OPTIONS[0]}" ]; then
            cyan " "
            green_sleep "# Fazendo download da nova versão do instalador..."
            ${SUDO_PREFIX} curl -s --connect-timeout 10 -S -L `printf ${INSTALLER_REPO_PACKAGE} "${LATEST_VERSION}"` -o installer_latest.tar.gz
            if [ $? -ne 0 ]; then
                warn "Falha ao baixar a atualização. Atualização falhou!"
                return 1
            fi
            green_sleep "Pronto!"

            cyan " "
            green_sleep "# Desembalar o instalador e substituir o instalador antigo pelo novo."
            ${SUDO_PREFIX} tar -xzf installer_latest.tar.gz
            ${SUDO_PREFIX} rm installer_latest.tar.gz
            ${SUDO_PREFIX} cp TeaSpeak-Installer-*/teaspeak_install.sh teaspeak_install.sh
            ${SUDO_PREFIX} rm -r TeaSpeak-Installer-*
            green_sleep "Pronto!"

            cyan " "
            green_sleep "# Ajustando direitos de script para execução."
            ${SUDO_PREFIX} chmod 774 teaspeak_install.sh
            green_sleep "Pronto!"

            cyan " "
            green_sleep "# Restarting update script!"
            sleep 1
            clear
            ${SUDO_PREFIX} ./teaspeak_install.sh
            exit 0
        fi
        if [ $? -ne 0 ]; then
            yellow_sleep "New installer version skiped."
        fi
    else
        green_sleep "# Você está usando a versão ${INSTALLER_VERSION}."
    fi
}

function secure_user() {
    cyan " "
    cyan "Criar chave, definir senha ou não fazer login?"

    OPTIONS=("Criar chave" "Definir senha" "Sem login" "Sair")
    select OPTION in "${OPTIONS[@]}"; do
        case "$REPLY" in
            1|2|3 ) break;;
            4 ) option_quit_installer;;
            *) invalid_option;continue;;
        esac
    done

    if [ "$OPTION" == "${OPTIONS[0]}" ]; then
        if { command -v ssh-keygen; } >/dev/null 2>&1; then
            ${SUDO_PREFIX} groupadd $teaUser
            ${SUDO_PREFIX} mkdir -p /$teaPath
            ${SUDO_PREFIX} useradd -m -b /$teaPath -s /bin/bash -g $teaUser $teaUser
            if [ $? -ne 0 ]; then
                error "Falha ao criar o usuário do TeaSpeak!"
                exit 1
            fi

            if [ -d /$TEASPEAK_DIR/.ssh ]; then
                ${SUDO_PREFIX} rm -rf /$TEASPEAK_DIR/.ssh
            fi

            ${SUDO_PREFIX} mkdir -p /$TEASPEAK_DIR/.ssh
            ${SUDO_PREFIX} chown $teaUser:$teaUser /$TEASPEAK_DIR/.ssh
            cd /$TEASPEAK_DIR/.ssh

            cyan " "
            cyan "É recomendado, mas não é necessário definir uma senha."
            ${SUDO_PREFIX} su -c "ssh-keygen -t rsa" $teaUser
            if [ $? -ne 0 ]; then
                error "Falha ao criar uma chave SSH!"
                exit 1
            fi

            KEYNAME=`find -maxdepth 1 -name "*.pub" | head -n 1`

            if [ "$KEYNAME" != "" ]; then
                ${SUDO_PREFIX} su -c "cat $KEYNAME >> authorized_keys" $teaUser
                if [ $? -ne 0 ]; then
                    error "Não foi possível encontrar uma chave!"
                    exit 1
                fi
            fi
        fi
    elif [ "$OPTION" == "${OPTIONS[1]}" ]; then
        ${SUDO_PREFIX} groupadd $teaUser
        ${SUDO_PREFIX} mkdir -p /$teaPath
        ${SUDO_PREFIX} useradd -m -b /$teaPath -s /bin/bash -g $teaUser $teaUser
        if [ $? -ne 0 ]; then
            error "Falha ao criar o usuário do TeaSpeak! Talvez senha errada ou usuário existente?"
            exit 1
        fi
        ${SUDO_PREFIX} passwd $teaUser
    elif [ "$OPTION" == "${OPTIONS[2]}" ]; then
        ${SUDO_PREFIX} groupadd $teaUser
        ${SUDO_PREFIX} mkdir -p /$teaPath
        ${SUDO_PREFIX} useradd -m -b /$teaPath -s /usr/sbin/nologin -g $teaUser $teaUser
        if [ $? -ne 0 ]; then
            error "Falha ao criar o usuário do TeaSpeak!"
            exit 1
        fi
    fi
}

cyan " "
cyan " "
red "        Instalador Teamspeak Server"
cyan "        Espero que tenha sido útil! Deseja contribuir com tibia coin? envie para Du Artt"
cyan " "

yellow "NOTE: Você pode sair do script a qualquer momento com CTRL + C"
yellow "      mas nem sempre é recomendável!"

# Get packet manager commands.
detect_packet_manager
if [ $? -ne 0 ]; then
    error "Saindo do Instalador"
    unset IFS;
    exit 1
fi

# Update system packages.
cyan " "
cyan "Atualize a lista de pacotes para a versão mais recente?"
warn "AVISO: É recomendável atualizar o sistema antes de executar o instalador, caso contrário, as dependências podem travar ou o script pode não funcionar corretamente!"
OPTIONS=("Atualizar" "Pular (Não Recomendado)" "Sair")
select OPTION in "${OPTIONS[@]}"; do
    case "$REPLY" in
        1|2 ) break;;
        3 ) option_quit_installer;;
        *) invalid_option;continue;;
    esac
done

if [ "$OPTION" == "${OPTIONS[0]}" ]; then
    green_sleep "# Atualizando os pacotes do sistema..."
    eval "${SUDO_PREFIX} ${PACKET_MANAGER_UPDATE}"
    green_sleep "Pronto!"
elif [ "$OPTION" == "Skip" ]; then
    yellow_sleep "Atualização do sistema ignorada."
fi

# Install packages for the installer itself.
test_installed curl
test_installed tar
if [ $? -ne 0 ]; then
    error "Falha ao instalar os pacotes necessários para o instalador!"
    exit 1
fi

# Update installer script.
updateScript
if [ $? -ne 0 ]; then
    error "Falha ao atualizar o script!"
    exit 1
fi

# Check if repo is needed.
if [ "${NEDDED_REPO}" == "true" ] && ! yum -v repolist all 2>/dev/null | grep "epel-multimedia" &>/dev/null && ! dnf -v repolist all | grep "fedora-multimedia" &>/dev/null; then
    cyan " "
    warn "NOTE: This distribution (${SYSTEM_NAME_DETECTED} ${SYSTEM_VERSION} ${ARCH}) requires the "${data[3]}" repository to install ffmpeg!"
    cyan "Should we add "${data[3]}" to your repository list? (Required root or administrator privileges)"
    OPTIONS=("Sim" "Não" "Sair")
    select OPTION in "${OPTIONS[@]}"; do
        case "$REPLY" in
            1|2) break;;
            3 ) option_quit_installer;;
            *)  invalid_option; continue;;
        esac
    done

    if [ "$OPTION" == "${OPTIONS[0]}" ]; then
        green_sleep "# Adding "${data[3]}" repository..."
        for i in ${data[3]};
        do
            if [ "$SYSTEM_NAME_DETECTED" == "(CentOS|RedHat)" ]; then
                yum-config-manager --add-repo $i
			elif [ "$SYSTEM_NAME_DETECTED" == "Fedora" ]; then
                dnf config-manager --add-repo $i
            fi
            if [ $? -ne 0 ]; then
                error "Failed to add required repository for your distribution (${SYSTEM_NAME_DETECTED} ${SYSTEM_VERSION})!"
                exit 1
            fi
        done
    fi
fi

# Begin install needed TeaSpeak packages.
beginIFS=$endIFS
IFS=":"
split_data=${data[1]}
endIFS=$beginIFS
for split_package in ${split_data[@]}
do
    test_installed ${split_package}
    if [ $? -ne 0 ]; then
        error "Falha ao instalar os pacotes necessários para o TeamSpeak!"
        exit 1
    fi
done
unset IFS;

# Criar usuário, sim ou não?
cyan " "
cyan "Deseja criar um usuário do TeaSpeak?"
cyan "* Recomenda-se criar um usuário separado do TeaSpeak!"
OPTIONS=("Sim" "Não" "Sair")
select OPTION in "${OPTIONS[@]}"; do
    case "$REPLY" in
        1|2 ) break;;
        3 ) option_quit_installer;;
        *) invalid_option;continue;;
    esac
done

if [ "$OPTION" == ${OPTIONS[0]} ]; then
    cyan " "
    cyan "Por favor, digite o nome do usuário do TeaSpeak."
    read teaUser
    NO_USER=false
elif [ "$OPTION" == ${OPTIONS[1]} ]; then
    NO_USER=true
    yellow_sleep "A criação do usuário foi ignorada."
fi

# Caminho de instalação do TeamSpeak.
cyan " "
cyan "Por favor, digite o caminho da instalação do TeaSpeak."
cyan "Empty input = /home/ | Example input = /srv/"
read teaPath
if [ -z "$teaPath" ]; then
    teaPath='home'
fi

TEASPEAK_DIR="/${teaPath}/${teaUser}"

if [ "$NO_USER" == "false" ]; then
    secure_user
    if [ $? -ne 0 ]; then
        error "Falha ao definir a opção de segurança!"
        exit 1
    fi
fi

${SUDO_PREFIX} mkdir -p $TEASPEAK_DIR
cd $TEASPEAK_DIR
	
# Fazendo download e configurando o TeamSpeak.
cyan " "
cyan "Obtendo a versão do TeamSpeak..."
TEASPEAK_VERSION=$(curl -s --connect-timeout 10 -S -L -k https://repo.teaspeak.de/server/linux/${ARCH}/latest)
TEA_REQUEST_URL="https://repo.teaspeak.de/server/linux/${ARCH}/TeaSpeak-${TEASPEAK_VERSION}.tar.gz"
if [ $? -ne 0 ]; then
    error "Falha ao carregar a versão mais recente do TeaSpeak!"
    exit 1
fi
green_sleep "# A versão mais recente é ${TEASPEAK_VERSION}"

cyan " "
green_sleep "# Baixando ${TEA_REQUEST_URL} to ${TEASPEAK_DIR}"
${SUDO_PREFIX} curl -s --connect-timeout 10 -S -L "$TEA_REQUEST_URL" -o teaspeak_latest.tar.gz
if [ $? -ne 0 ]; then
    error "Falha ao baixar a versão mais recente do TeaSpeak!"
    exit 1
fi
sleep 1
green_sleep "# Descompactando e removendo .tar.gz"
${SUDO_PREFIX} tar -xzf teaspeak_latest.tar.gz
${SUDO_PREFIX} rm teaspeak_latest.tar.gz
green_sleep "Pronto!"

cyan " "
green_sleep "# Tornando scripts executáveis."
${SUDO_PREFIX} chown -R $teaUser:$teaUser /$teaPath/$teaUser/*
${SUDO_PREFIX} chmod 774 /$TEASPEAK_DIR/*.sh
green_sleep "DONE!"

cyan " "
green_sleep "Terminado, TeamSpeak ${TEASPEAK_VERSION} agora está instalado!"

# Inicie o TeaSpeak no modo mínimo.
cyan " "
cyan "Deseja iniciar o TeamSpeak?"
cyan "* Salve o logon Serverquery e o token Serveradmin na primeira vez em que você iniciar o TeaSpeak"
cyan "*CTRL+C = Sair"
OPTIONS=("Iniciar servidor" "Encerrar e sair")
select OPTION in "${OPTIONS[@]}"; do
    case "$REPLY" in
        1|2 ) break;;
        *) invalid_option;continue;;
    esac
done
	
if [ "$OPTION" == "${OPTIONS[0]}" ]; then
    cyan " "
    green_sleep "# Iniciando o Teamspeak..."
    cd $TEASPEAK_DIR; ${SUDO_PREFIX} LD_LIBRARY_PATH="$LD_LIBRARY_PATH;./libs/" ./TeaSpeakServer; stty cooked echo
	
    cyan " "
    green_sleep "# Tornando Executáveis os Novos Arquivos Criados."
    ${SUDO_PREFIX} chown -R $teaUser:$teaUser /$TEASPEAK_DIR/*
    ${SUDO_PREFIX} chmod 774 /$TEASPEAK_DIR/*.sh
    green_sleep "PRONTO!"
fi

cyan " "
yellow "	  NOTA: É recomendável iniciar o servidor TeaSpeak com o usuário criado!"
yellow "	  Para iniciar o TeaSpeak, você pode usar os seguintes scripts bash:"
yellow "      teastart.sh, teastart_minimal.sh, teastart_autorestart.sh and tealoop.sh."
green_sleep "Script concluído com sucesso."

exit 0
