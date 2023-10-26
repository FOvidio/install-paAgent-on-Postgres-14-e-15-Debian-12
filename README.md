# install-paAgent-on-Postgres-14-e-15-Debian-12
install paAgent on Postgres 14, 15 Debian 12

Para instalação na mesma maquina onde se encontra o banco de dados


## Terminal
1. Instalando o pgagent

```
sudo apt update
sudo apt install pgagent
```


2. Criando o arquivos .pgpass

Devemos criar este aquivo no usuário postgres para que o arquivo fique em $HOME$ do postgres (/var/lib/postgresql)

```
sudo su - postgres
echo localhost:5432:*:pgagent:senha_do_banco >> ~/.pgpass
chmod 600 ~/.pgpass
chown postgres:postgres /var/lib/postgresql/.pgpass
```

3. Criando o diretório de log

Criando o diretório de log
```bash
mkdir /var/log/pgagent
chown -R postgres:postgres /var/log/pgagent
chmod g+w /var/log/pgagent
```


### Testando
4. Test pgAgent connections

Em um terminal separado, `tail -f /var/log/postgresql/postgresql-10-main.log` para ver os logs do pgagent gravados

#### Conexão com o Postgres
Testando a conexão com o banco de dados
```bash
psql -h localhost -d yourdatabase -U pgagent
```

#### pgAgent
```bash
sudo su - postgres
/usr/bin/pgagent -f -l 2 host=localhost port=5432 user=pgagent dbname=postgres
```
Revise os logs e observe se existe alguma mensagem de erro.


### PgAgent carregando como serviço no Debian (testado com debian 12 stable)
5. Configurar o pgAgent para carregar no boot (reboot)

Crie um arquivo de configuração e salve em /etc/pgagent.conf
sudo nano /etc/pgagent.conf
```bash
#/etc/pgagent.conf
DBNAME=postgres
DBUSER=postgres
DBHOST=localhost
DBPORT=5432
# ERROR=0, WARNING=1, DEBUG=2
LOGLEVEL=1
LOGFILE="/var/log/pgagent/pgagent.log"
```

#### Lidando com o Systemd ;)
6. Crie o arquivo pgagente.service

sudo nano /usr/lib/systemd/system/pgagent.service
```
[Unit]
Description=PgAgent PostgreSQL by atma
After=syslog.target
After=network.target

[Service]
Type=forking

User=postgres
Group=postgres

# Local do arquivo de configuração
EnvironmentFile=/etc/pgagent.conf

# Where to send early-startup messages from the server (before the logging
# options of pgagent.conf take effect)
# This is normally controlled by the global default set by systemd
# StandardOutput=syslog

# Disable OOM kill on the postmaster
OOMScoreAdjust=-1000

ExecStart=/usr/bin/pgagent -s ${LOGFILE}  -l ${LOGLEVEL} host=${DBHOST} dbname=${DBNAME} user=${DBUSER} port=${DBPORT}
KillMode=mixed
KillSignal=SIGINT

Restart=on-failure

# Tempo para iniciar o serviço
TimeoutSec=300

[Install]
WantedBy=multi-user.target
```

### Configurando o serviço 

```
sudo -i
systemctl daemon-reload
systemctl disable pgagent
systemctl enable pgagent
systemctl start pgagent
```

### Auto limpesa dos logs (logrotate)
7. Habilita a rotação dos logs (apaga e limpa logs antigos da pasta /var/log/pgagent *pgagent.log*)
Crie o arquivo
sudo nano /etc/logrotate.d/pgagent
 
```
#/etc/logrotate.d/pgagent
/var/log/pgagent/*.log {
       weekly
       rotate 10
       copytruncate
       delaycompress
       compress
       notifempty
       missingok
       su root root
}
```

Teste a rotação dos logs com o comando:
```
logrotate -f /etc/logrotate.d/pgagent
````

### Source ###
- [pgAgent Install Documentation](https://www.pgadmin.org/docs/pgadmin4/dev/pgagent_install.html)
- http://www.ekho.name/2012/03/pgagent-debianubuntu.html / https://gist.github.com/ekho/6636393
