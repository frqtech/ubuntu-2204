# Instalação de Ubuntu Server 22.04 LTS com padrão RNP/CAFe

## 1. Introdução

Este tutorial apresenta os passos necessários para se fazer a instalação de servidor baseado em [Linux Ubuntu 22.04 LTS](https://ubuntu.com/download/server) com o padrão de configurações RNP/CAFe.

> ATENÇÃO: Este tutorial assume a existência de um servidor Ubuntu Server 22.04 LTS recem instalado.

## 2. Configurações Básicas

2.1. Inicialmente será feira a configuração do hostname. Tal configuração deve ser feita através do comando `hostnamectl`. Um exemplo deste comando é exibido abaixo.

```
hostnamectl set-hostname "idp.instituicao.edu.br"
```

2.2. A configuração do arquivo de hosts é feita através do arquivo `/etc/hosts`. Um exemplo do conteúdo deste arquivo é exibido abaixo.

```
127.0.0.1    localhost
10.0.0.1     idp.instituicao.edu.br  idp
```

2.3. Considerando que esta máquina será utilizada como um Virtual Appliance recomenda-se que como configuração inicial e de caráter temporário, a máquina esteja configurada com DHCP. Isso permitirá que, havendo servidor DHCP, a máquina ingresse brevemente na rede da instituição. Ao colocar a máquina em ambiente de produção recomenda-se a utilização de IP estático.

A configuração deve ser feita no arquivo `/etc/netplan/00-installer-config.yaml` de acordo com o exemplo abaixo:

```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: true
```

2.4. As linhas a seguir tem por objetivo fazer a remoção de pacotes desnecessários, atualização da distribuição e ainda instalação de pacotes úteis.

```
apt update
apt remove --purge -y vim-tiny
apt dist-upgrade -y
apt install -y less vim bzip2 unzip ssh dialog ldap-utils build-essential net-tools
```

2.5. A fim de otimizar o espaço ocupado pelos arquivos de LOG, deverá ser habilitada a opção de compressão no arquivo `/etc/logrotate.conf`. Para tanto garanta que as linhas a seguir estejam presentes e descomentadas no referido arquivo.

```
compress
 
nodelaycompress
dateext
````

2.6. O firewall será composto por três arquivos:

```
/etc/default/firewall - arquivo com as regras de firewall
/etc/systemd/system/firewall.service - arquivo de configuração para o systemd
/opt/rnp/firewall/firewall.sh - script de manipulação do firewall
```

O bloco abaixo apresenta o conteúdo do arquivo `/etc/default/firewall`:

```
#!/bin/sh
 
# Atualizado em 31/08/2016 por Rui Ribeiro - rui.ribeiro@cafe.rnp.br
 
# Apaga todas as regras pré-existentes
iptables -F
# Politica padrão DROP
iptables -P INPUT DROP
 
iptables -A INPUT -i lo -j ACCEPT
 
# Só aceita ate 10 pacotes icmp echo-request por segundo. Previne flooding.
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 10/s -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
iptables -A INPUT -p icmp -j ACCEPT
 
# Libera conexões já estabelecidas
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
 
# Libera acesso SSH
iptables -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
 
# Libera acesso WEB
iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
 
# ntp - servidor de hora
# descomente a linha a seguir para habilitar o acesso a partir da rede local ao servidor em /etc/ntp.conf
#iptables -A INPUT -p udp -m udp --dport 123 -j ACCEPT
```

O arquivo acima descrito pode ser baixado através do seguinte comando:

```
wget https://raw.githubusercontent.com/frqtech/ubuntu-2204/main/firewall.rules -O /etc/default/firewall
```

2.7. O bloco abaixo apresenta o conteúdo do arquivo `/etc/systemd/system/firewall.service`:

```
[Unit]
Description=Firewall Basico - RNP/CAFe - v1.0
 
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/opt/rnp/firewall/firewall.sh start
ExecStop=/opt/rnp/firewall/firewall.sh stop
 
[Install]
WantedBy=multi-user.target
```

O arquivo acima descrito pode ser baixado através do seguinte comando:

```
wget https://raw.githubusercontent.com/frqtech/ubuntu-2204/main/firewall.service -O /etc/systemd/system/firewall.service
```

2.8. O bloco abaixo apresenta o conteúdo do arquivo `/opt/rnp/firewall/firewall.sh`:

```
#!/bin/bash
# Atualizado em 31/08/16 por Rui Ribeiro - rui.ribeiro@cafe.rnp.br
 
RULES_FILE="/etc/default/firewall"
 
RETVAL=0
 
# To start the firewall
start() {
 
    # Termina se nao existe iptables
    [ -x /sbin/iptables ] || exit 0
 
    # Arquivo com as regras propriamente ditas
    if [ -f "$RULES_FILE" ]; then
        echo "Carregando regras de firewall ..."
        . $RULES_FILE
    else
        echo "Arquivo de regras inexistente: $RULES_FILE"
        stop
        RETVAL=1
    fi
 
    RETVAL=0
 
}
 
# To stop the firewall
stop() {
 
    echo "Removendo todas as regras de firewall ..."
    iptables -P INPUT ACCEPT
    iptables -F
    iptables -X
    iptables -Z
    RETVAL=0
 
}
 
case $1 in
 
    start)
        start
        ;;
    stop)
        stop
        ;;
    restart)
        stop
        start
        ;;
    status)
        /sbin/iptables -L
        /sbin/iptables -t nat -L
        RETVAL=0
        ;;
    *)
        echo "Uso: $1 {start|stop|restart|status}"
        RETVAL=1;;
 
esac
 
exit $RETVAL
```

O arquivo acima descrito pode ser baixado através do seguinte comando:

```
mkdir -p /opt/rnp/firewall/
wget https://raw.githubusercontent.com/frqtech/ubuntu-2204/main/firewall.sh -O /opt/rnp/firewall/firewall.sh
```

2.9. Uma vez que os dois arquivos estejam nos locais apropriados, é necessário executar as seguintes linhas de comando:

```
chmod 755 /opt/rnp/firewall/firewall.sh
chmod 664 /etc/systemd/system/firewall.service
systemctl daemon-reload
systemctl enable firewall.service
```

Ao executar tais linhas serão atribuídas as devidas permissões aos arquivos e será configurado o firewall no systemd.

2.10. A sincronização do servidor será feita pelo ntp. Para tanto é necessário desabilitar tal funcionalidade no `timedatectl` e então fazer a instalação do pacote ntp:

```
timedatectl set-ntp no
apt install -y ntp
```

2.11. Após fazer a instalação é necessário modificar o arquivo `/etc/ntp.conf` que deverá ficar com o seguinte conteúdo:

```
# /etc/ntp.conf, configuracao do ntpd
 
# Atualizado em 01/04/2013 por Rui Ribeiro - rui.ribeiro@cafe.rnp.br
 
driftfile /var/lib/ntp/ntp.drift
statsdir /var/log/ntpstats/
 
statistics loopstats peerstats clockstats
filegen loopstats file loopstats type day enable
filegen peerstats file peerstats type day enable
filegen clockstats file clockstats type day enable
 
# Servidores ntp do nic.br
server a.ntp.br
server b.ntp.br
server c.ntp.br
 
# By default, exchange time with everybody, but don't allow configuration.
# See /usr/share/doc/ntp-doc/html/accopt.html for details.
restrict -4 default kod notrap nomodify nopeer noquery
restrict -6 default kod notrap nomodify nopeer noquery
 
# Local users may interrogate the ntp server more closely.
restrict 127.0.0.1
restrict ::1
 
# Para habilitar o servidor de hora para acesso a partir da rede
# local, altere a linha abaixo:
#broadcast 192.168.123.255
```

O arquivo acima descrito pode ser baixado através do seguinte comando:

```
wget https://raw.githubusercontent.com/frqtech/ubuntu-2204/main/ntp.conf -O /etc/ntp.conf
```