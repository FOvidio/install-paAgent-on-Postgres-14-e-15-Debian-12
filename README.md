# install-paAgent-on-Postgres-14-e-15-Debian-12
install paAgent on Postgres 14, 15 Debian 12

Para instalação na mesma maquina onde se encontra o banco de dados.
No tutorial abaixo considero que o pgAdmin esta instalado no servidor juntamente com o banco de dados.


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
echo localhost:5432:*:postgres:senha_do_banco >> ~/.pgpass
chmod 600 ~/.pgpass
chown postgres:postgres /var/lib/postgresql/.pgpass
exit
```

3. Criando o diretório de log

No terminal execute como root:
```bash
sudo su
mkdir /var/log/pgagent
chown -R postgres:postgres /var/log/pgagent
chmod g+w /var/log/pgagent
```


### Testando
4. Testando as conexões pgAgent

Em um nova aba do terminal, lembrando de mudar a versão do PG que esta na maquina no nosso caso o PG era o 15 por isso ficou postgresql-15-main
 `sudo tail -f /var/log/postgresql/postgresql-15-main.log` para ver os logs do pgagent gravados
 
#### Conexão com o Postgres
Testando a conexão com o banco de dados
```bash
psql -h localhost -d postgres -U pgagent
```

#### pgAgent Extensão
Para executar o pgagent é preciso habilitar sua extensão, execute no pgadmin conectado ao banco postgres:

```bash
sudo -i -u postgres
psql
CREATE EXTENSION pgagent;
```

#### pgAgent Logs

```bash
sudo su - postgres
/usr/bin/pgagent -f -l 2 host=localhost port=5432 user=postgres dbname=postgres
exit
```
Revise os logs e observe se existe alguma mensagem de erro.


### PgAgent carregando como serviço no Debian (testado com debian 12 stable)
5. Configurar o pgAgent para carregar no boot (reboot)

Crie um arquivo de configuração e salve em /etc/pgagent.conf lembrando de logar no root
sudo su
sudo nano /etc/pgagent.conf

Dentro do arquivo salvar a seguinte informação:
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

#### Lidando com o Systemd
6. Crie o arquivo pgagent.service:

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
systemctl status pgagent
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

## pgAdmin

Você tera que logar no servidor realizar o refresh e verificar se a aba pgAgent Jobs esta ativa, 
se sim você pode realizar a criação de um job para começar a testar.

### Criando um Job 

1. Botão direito no pgAgent Jobs > Create > pgAgent Job...
2. Aba General
configuração da aba General do JOB envolve editar cinco parâmetros nesta tela específica:  [Name, Enabled?, Job Class, Host agent, Comment]
Como exemplo vou passar os seguintes paramentros:
```
Name: JobTeste 
Enabled: True
Job Class: Routien Maintenance
```
![image](https://github.com/FOvidio/install-paAgent-on-Postgres-14-e-15-Debian-12/assets/30088858/a2269fa6-8405-4eb9-8a23-afcfc9ba249a)

3. Aba Steps
Selecionar o Botão + que é apresentado no canto superior direito e vamos realizar a configuração de exemplo:
```
Aba General -------------- Step
Name: Teste
Enabled?: True
Kind: SQL
Connection type: Local
Database: postgres
On error: Fail

Aba Code    -------------- Step
create table if not exists tb_teste (id int, nome varchar(20));
insert into tb_teste values (1, 'Lucas'), (2, 'Fabio');
```
![image](https://github.com/FOvidio/install-paAgent-on-Postgres-14-e-15-Debian-12/assets/30088858/55f55a17-aa0d-4f19-a06e-9e331eaf024d)
![image](https://github.com/FOvidio/install-paAgent-on-Postgres-14-e-15-Debian-12/assets/30088858/7ced5759-8135-4874-a48d-e63f4cf6959c)

4. Schedules
```
Aba General -------------- Schedules
Name: Agenda
Enabled?: True
Start: 2023-10-26 00:00:00 -03:00
End:   2099-12-31 00:00:00 -03:00

Aba Repeat --------------- Schedules
Minutes: Select all  -> obs vai aparecer 00, 01, 02, 03, 04, ...
```
![image](https://github.com/FOvidio/install-paAgent-on-Postgres-14-e-15-Debian-12/assets/30088858/821a85d1-68bd-4732-899d-d9fda4a7a1d7)


5. SQL
Se ficou a configuração correta vai ficar da seguinte forma:
```
DO $$
DECLARE
    jid integer;
    scid integer;
BEGIN
-- Creating a new job
INSERT INTO pgagent.pga_job(
    jobjclid, jobname, jobdesc, jobhostagent, jobenabled
) VALUES (
    1::integer, 'JobTeste'::text, ''::text, ''::text, true
) RETURNING jobid INTO jid;

-- Steps
-- Inserting a step (jobid: NULL)
INSERT INTO pgagent.pga_jobstep (
    jstjobid, jstname, jstenabled, jstkind,
    jstconnstr, jstdbname, jstonerror,
    jstcode, jstdesc
) VALUES (
    jid, 'Teste'::text, true, 's'::character(1),
    ''::text, 'postgres'::name, 'f'::character(1),
    'create table if not exists tb_teste (id int, nome varchar(20));
insert into tb_teste values (1, ''Lucas''), (2, ''Fabio'');'::text, ''::text
) ;

-- Schedules
-- Inserting a schedule
INSERT INTO pgagent.pga_schedule(
    jscjobid, jscname, jscdesc, jscenabled,
    jscstart, jscend,    jscminutes, jschours, jscweekdays, jscmonthdays, jscmonths
) VALUES (
    jid, 'Agenda'::text, ''::text, true,
    '2023-10-26 00:00:00 -03:00'::timestamp with time zone, '2099-12-31 00:00:00 -03:00'::timestamp with time zone,
    -- Minutes
    ARRAY[true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true,true]::boolean[],
    -- Hours
    ARRAY[false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false]::boolean[],
    -- Week days
    ARRAY[false,false,false,false,false,false,false]::boolean[],
    -- Month days
    ARRAY[false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false]::boolean[],
    -- Months
    ARRAY[false,false,false,false,false,false,false,false,false,false,false,false]::boolean[]
) RETURNING jscid INTO scid;
END
$$;
```
So salvar e rodar no banco de dados postgres a seguinte consulta de teste:
```
select * from tb_teste;
```


### Fonte ###
https://www.pgadmin.org/docs/pgadmin4/2.1/pgagent_install.html

