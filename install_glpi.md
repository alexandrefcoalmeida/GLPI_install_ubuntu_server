# GLPI_install_ubuntu_server
Este repositório irá conter os passos para instalar um GLPI 10.x em um Ubuntu server 22.x

## Passo 1: Após realizar o deploy do Ubuntu server 22.x é necessário realizar algumas configurações iniciais padrões no servidor:

## EXECUTE OS COMANDOS SEMPRE NA ORDEM INDICADA

1.0 - Defina um IP estático para o seu servidor: (Pode ser feito na instalação do Ubuntu)

- Abra o arquivo de configuração de rede em um editor de texto. Pode ser o VIM, por exemplo:

```
vim /etc/netplan/00-installer-config.yaml
```

- Aqui está um exemplo de como a configuração deve ficar:
```
# This is the network config written by 'subiquity'

network:
  ethernets:
    eno1:
      addresses:
      - 192.168.0.90/24
      nameservers:
        addresses:
        - 8.8.8.8
        - 8.8.4.4
        search: []
      routes:
      - to: default
        via: 192.168.0.1
    eno2:
      dhcp4: true
    eno3:
      dhcp4: true
    eno4:
      dhcp4: true
    enx0a94ef4f140b:
      dhcp4: true
  version: 2
```
- Saia do arquivo salvando e execute os comandos:

``` 
sudo netplan apply
sudo service networking restart
 ```

- E verifique se as alterações foram fixadas com o comando:

``` 
ip a 
```

1.1 - Certifique-se de que seu sistema está atualizado com os últimos patches de segurança e atualizações de software. Execute os seguintes comandos:
```
sudo apt update
sudo apt upgrade
```
1.2 - Configuração de Fuso Horário, verifique e configure o fuso horário do seu servidor:

```
timedatectl
timedatectl list-timezones # (Selecione o timezone de acordo com o seu)
sudo timedatectl set-timezone America/Sao_Paulo
```
- Execute novamente o comando timedatectl para garantir que o fuso horário tenha sido atualizado corretamente.

1.3 - Instale o pacote NTP:
```
sudo apt install ntp
sudo systemctl enable ntp
sudo systemctl start ntp
```
Especificar servidores NTP:

``` 
sudo vim /etc/ntp.conf 
# (Caso o vim não esteja instalado executar o comando sudo apt install vim)
```

- Dentro do arquivo, você verá linhas como:
```
pool 0.ubuntu.pool.ntp.org iburst
pool 1.ubuntu.pool.ntp.org iburst
pool 2.ubuntu.pool.ntp.org iburst
pool 3.ubuntu.pool.ntp.org iburst
```
- Adicione os servidores NTP acima das pools e comente as pools padrões:
```
server a.st1.ntp.br iburst
server b.st1.ntp.br iburst
server c.st1.ntp.br iburst
#pool 0.ubuntu.pool.ntp.org iburst
#pool 1.ubuntu.pool.ntp.org iburst
#pool 2.ubuntu.pool.ntp.org iburst
#pool 3.ubuntu.pool.ntp.org iburst
```
1.4 - Se necessário ative o firewall de sua preferência, nesse caso estaremos utilizando o UFW, e dê as permissões necessárias:
```
sudo ufw allow OpenSSH (Permitir Acesso SSH)
sudo ufw allow 161/udp (Permitir tráfego SNMP)
sudo ufw allow 162/udp (Permitir tráfego SNMP Traps)
sudo ufw allow 80/tcp  (Permitir tráfego HTTP, para o Apache)
sudo ufw enable
sudo systemctl restart ssh
```

- (Neste tutorial estaremos instalando o GLPI em HTTP, portanto liberamos somente HTTP no firewall)

1.5 - Monitoramento do Sistema, Isso facilitará a visualização do uso de recursos do sistema.
```
sudo apt install htop
```

1.6 - Habilitar o login SSH via root:

- Execute o comando:
```
vim /etc/ssh/sshd_config
```
- E altere a linha:
```
PermitRootLogin no 
```
- para:
```
PermitRootLogin yes
```
- Salve e saia do arquivo, e execute o comando:
```
sudo systemctl restart ssh
```
1.7 - Instalar o SNMP:
```
sudo apt update
sudo apt install snmp snmpd
```

- Após esses passos partiremos para a instalação das dependências que o GLPI necessita para ser instalado:

2.0 - Atualizar o Ubuntu:
```
sudo apt update -y
sudo apt upgrade -y
```
2.1 - Instale  LAMP:

- O LAMP Stack consiste em Apache2, PHP e MySQL/MariaDB e pode ser instalado usando as seguintes instruções:
```
sudo apt install -y apache2 php8.1 php8.1-curl php8.1-zip php8.1-gd php8.1-intl php8.1-intl php-pear php8.1-imagick php8.1-imap php-memcache php8.1-pspell php8.1-tidy php8.1-xmlrpc php8.1-xsl php8.1-mbstring php8.1-ldap php8.1-ldap php-cas php-apcu libapache2-mod-php8.1 php8.1-mysql php-bz2 wget php8.1-gd
```
- Assim que a instalação terminar, verifique o status do Apache2
```
sudo systemctl status apache2
```
- Se você ver ativo (em execução), o Apache está instalado e pronto para uso.

2.2 - Instale MariaDB:
```
sudo apt install mariadb-server -y
```
- E verifique o status:
```
sudo systemctl status mariadb
```
- Para ter certeza de que o MariaDB é seguro e tem uma senha de root definida (a configuração padrão é sem senha), execute o comando:
```
sudo mariadb-secure-installation
```
- Quando o script perguntar “Digite a senha atual para root (digite para nenhuma):” pressione enter, pois não há senha definida no momento.
```
- Responda Y para “Mudar para autenticação unix_socket [S/n]”

- Responda Y novamente para “Alterar a senha root? [S/n]”

- Digite a nova senha e pressione Enter quando solicitado “Nova senha:” (Essa senha será utilizada posteriormente para o GLPI acessar o banco de dados)

- Digite novamente a nova senha e pressione Enter quando solicitado “Digite novamente a nova senha:”

- Digite Y e digite quando for perguntado “Remover usuários anônimos? [S/n]”

- Digite N quando for perguntado “Proibir login root remotamente? [S/n]”

- Por fim, digite Y e digite quando for solicitado “Remover banco de dados de teste e acesso a ele? [S/n]”

- O MySQL agora está protegido e sua senha root do mysql está definida.
```

2.3 - Criar banco de dados e usuário GLPI:
```
sudo mysql -u root -p (E digite a senha de acesso definida anteriormente)
CREATE DATABASE glpi;
```
- Crie o usuário e conceda acesso ao banco de dados glpi executando os seguintes comandos alterando `DBNAME`, `GLPIUSER` e `YOUR_PASSWORD` para os nomes e senha que você deseja usar:
```
GRANT ALL PRIVILEGES ON glpi.* TO 'root'@'localhost' IDENTIFIED BY 'SUA_SENHA';

FLUSH PRIVILEGES;

EXIT;
```
- Agora estamos prontos para baixar o GLPI.


3.0 - Baixando o GLPI:

- (Versão atual na data deste tutorial é a 10.0.12:

- Execute os comandos:
```
cd /
wget https://github.com/glpi-project/glpi/releases/download/10.0.12/glpi-10.0.12.tgz
tar xvf glpi-10.0.12.tgz
```
- Mova o diretorio do glpi extraído /var/www/html:
```
sudo mv -v glpi /var/www/html/
```

- Dê ao usuário Apache a propriedade do diretório /var/www/html/glpi e de todo o seu conteúdo:
```
sudo chown -R www-data:www-data /var/www/html/glpi
```
3.1 - Conclua a instalação na GUI:

- O restante da instalação precisa ser feito no navegador da web, acesse via ip estático que configurou no início, no caso deste tutorial:
```
http://192.168.0.90/install.php
```
- E siga as etapas de configurações mencionadas.

3.2 - Selecione o idioma de instalação e clique em OK\
3.3 - Clique em Continuar, e posteriormente em Instalar.\
3.4 - Clique no botão Continuar para ir para a próxima etapa de instalação.\
3.5 - Coloque os dados do seu banco de dados criado quando configuramos o MariaDB e clique em Continuar.\
3.6 - Selecione o banco de dados glpi e clique em Continuar.\
3.7 - A inicialização do banco de dados será executada por um tempo.\
3.8 - Desmarque Enviar “Estatísticas de uso” e clique em Continuar.\
3.9 - Clique em Continuar novamente.\
4.0 - Clique em Usar GLPI para ir para a página de login.\
4.1 - Logue usando o user glpi, e senha glpi.\
4.2 - Siga as instruções para sanar os avisos exibidos na tela\

4.3 - Seu GLPI está instalado e pronto para iniciar as configurações.
















