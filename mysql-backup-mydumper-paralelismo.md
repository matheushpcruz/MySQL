# MySQL Backup com MyDumper e Paralelismo

> Post de minha autoria publicado em [blog.4linux.com.br](https://blog.4linux.com.br/mysql-backup-mydumper-paralelismo/)

Em bancos de dados muito grandes, o processo de backup tende a ser lento, pois é executado de forma serial. O `mysqldump`, ferramenta oficial do MySQL, realiza o backup de forma sequencial. Já o `mysqlpump`, também oficial, embora suporte paralelismo no dump, foi descontinuado na versão 8.0.34 do MySQL e não realiza o restore com paralelismo — esse processo continua sendo de forma sequencial.

Para contornar esse problema, existe o **MyDumper**, uma ferramenta open source composta por dois utilitários:

- **mydumper**: exporta backups com suporte a múltiplas threads paralelas.
- **myloader**: restaura os arquivos criados pelo mydumper, também com paralelismo.

O MyDumper é mantido pela comunidade e não é um produto da Percona, MariaDB ou MySQL.

---

## 1. Instalação

### Red Hat / CentOS / Rocky Linux

Importe a chave GPG e configure o repositório:

```bash
wget -O GPG-KEY-MyDumper "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x79EA15C0E82E34BA"
rpm --import GPG-KEY-MyDumper
```

```bash
echo -e "[mydumper]\nname=MyDumper\nbaseurl=https://mydumper.github.io/mydumper/repo/yum/\$releasever\nenabled=1\ngpgcheck=1" | sudo tee /etc/yum.repos.d/mydumper.repo
```

```bash
yum install mydumper
```

Exemplo de output da instalação:

```
[root@rhel-demo ~]# yum install mydumper
MyDumper                                                                                2.4 kB/s | 1.5 kB     00:00
Dependencies resolved.
========================================================================================================================
 Package                     Architecture              Version                        Repository                   Size
========================================================================================================================
Installing:
 mydumper                    x86_64                    0.19.3-3                       mydumper                    3.2 M

Transaction Summary
========================================================================================================================
Install  1 Package

Total download size: 3.2 M
Installed size: 19 M
Is this ok [y/N]: y
```

---

### Debian / Ubuntu

Importe a chave GPG:

```bash
wget -qO- 'https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x1D357EA7D10C9320371BDD0279EA15C0E82E34BA&exact=on' | sudo tee /etc/apt/keyrings/mydumper.asc
```

Configure o repositório — **Ubuntu**:

```bash
echo "deb [signed-by=/etc/apt/keyrings/mydumper.asc] https://mydumper.github.io/mydumper/repo/apt/ubuntu $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/mydumper.list
```

Configure o repositório — **Debian**:

```bash
echo "deb [signed-by=/etc/apt/keyrings/mydumper.asc] https://mydumper.github.io/mydumper/repo/apt/debian $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/mydumper.list
```

```bash
apt-get update && apt-get install mydumper
```

Exemplo de output:

```
root@db1:~# apt-get update && apt-get install mydumper
Hit:1 https://security.debian.org/debian-security bookworm-security InRelease
Hit:2 https://deb.debian.org/debian bookworm InRelease
Hit:3 https://deb.debian.org/debian bookworm-updates InRelease
Hit:4 https://deb.debian.org/debian bookworm-backports InRelease
Get:5 https://mydumper.github.io/mydumper/repo/apt/debian bookworm InRelease [4009 B]
Get:6 https://mydumper.github.io/mydumper/repo/apt/debian bookworm/main amd64 Packages [633 B]
Fetched 4642 B in 12s (396 B/s)
Reading package lists... Done
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  gcc-12-base libatomic1 libgcc-s1 libpcre3 libstdc++6
The following NEW packages will be installed:
  libatomic1 libpcre3 mydumper
The following packages will be upgraded:
  gcc-12-base libgcc-s1 libstdc++6
3 upgraded, 3 newly installed, 0 to remove and 83 not upgraded.
Need to get 9955 kB of archives.
After this operation, 742 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
```

---

## 2. Verificando a Versão

```bash
mydumper --version
myloader --version
```

Output:

```
root@db1:~# mydumper --version
mydumper v0.19.3-3, built against MySQL 8.4.5 with SSL support

root@db1:~# myloader --version
myloader v0.19.3-3, built against MySQL 8.4.5 with SSL support
```

---

## 3. Permissões Necessárias

Crie um usuário dedicado para backup com os privilégios necessários:

```sql
CREATE USER 'mydumper'@'%' IDENTIFIED BY 'senha';
GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER, REPLICATION CLIENT, BACKUP_ADMIN, RELOAD ON *.* TO 'mydumper'@'%';
```

---

## 4. Realizando o Backup

### Comando básico

```bash
mydumper -u mydumper -p senha -h localhost -P 3306 -B employees -o /tmp/employees
```

### Comando completo com opções avançadas

```bash
mydumper -u mydumper -p senha -h localhost -P 3306 -B employees \
-o "/tmp/employees" --sync-thread-lock-mode LOCK_ALL --no-views -G -E -R \
--threads 2 --verbose 3
```

Durante o backup, é possível observar as threads trabalhando em paralelo no `show processlist` do MySQL:

```
mysql> show processlist;
+----+-----------------+-----------+--------------------+---------+-------+------------------------+------------------------------------------------------------------------------------------------------+
| Id | User            | Host      | db                 | Command | Time  | State                  | Info                                                                                                 |
+----+-----------------+-----------+--------------------+---------+-------+------------------------+------------------------------------------------------------------------------------------------------+
|  5 | event_scheduler | localhost | NULL               | Daemon  | 27354 | Waiting on empty queue | NULL                                                                                                 |
| 33 | root            | localhost | NULL               | Query   |     0 | init                   | show processlist                                                                                     |
| 34 | mydumper        | localhost | information_schema | Sleep   |    14 |                        | NULL                                                                                                 |
| 35 | mydumper        | localhost | information_schema | Query   |     1 | executing              | SELECT /*! 40001 SQL_NO_CACHE */ * FROM `employees`.`dept_emp`  WHERE (`emp_no` IS NULL OR(10001 <= ` |
| 36 | mydumper        | localhost | employees          | Query   |     0 | executing              | SELECT /*! 40001 SQL_NO_CACHE */ * FROM `employees`.`dept_emp`  WHERE (377499 <= `emp_no` AND `emp_no |
+----+-----------------+-----------+--------------------+---------+-------+------------------------+------------------------------------------------------------------------------------------------------+
5 rows in set, 1 warning (0.00 sec)

mysql>
```

Output verboso do backup:

```
** Message: 21:37:37.926: Creating workers
** Message: 21:37:37.926: Starting to enqueue non-transactional tables
** Message: 21:37:37.929: Thread 2: connected using MySQL connection ID 32
** Message: 21:37:37.929: Thread 1: connected using MySQL connection ID 31
** Message: 21:37:37.930: Thread 2: Creating Jobs
** Message: 21:37:37.930: Written master status
** Message: 21:37:37.930: Thread 2: dumping db information for `employees`
** Message: 21:37:37.935: Thread 1: Creating Jobs
** Message: 21:37:37.935: Transactions started, unlocking tables
** Message: 21:37:37.935: Waiting database finish
** Message: 21:37:37.965: Thread 1: Processing Schema jobs
** Message: 21:37:37.966: Thread 1: dumping schema create for `employees`
** Message: 21:37:37.966: Thread 1: dumping schema for `employees`.`departments`
** Message: 21:37:37.968: Thread 2: Processing Schema jobs
** Message: 21:37:37.968: Thread 2: dumping schema for `employees`.`dept_emp`
** Message: 21:37:37.968: Shutdown schema jobs
** Message: 21:37:37.968: Waiting threads to complete
** Message: 21:37:37.970: Thread 1: dumping schema for `employees`.`employees`
** Message: 21:37:37.970: Thread 2: dumping schema for `employees`.`dept_manager`
** Message: 21:37:37.971: Thread 2: dumping schema for `employees`.`titles`
** Message: 21:37:37.972: Thread 1: dumping schema for `employees`.`salaries`
** Message: 21:37:37.972: Thread 2: dumping schema for `employees`.`dept_emp`
** Message: 21:37:37.973: Thread 2: Schema jobs are done, Starting exporting data for Non-Transactional tables
** Message: 21:37:37.973: Enqueuing of non-transactional tables completed
** Message: 21:37:37.973: Starting to enqueue transactional tables
** Message: 21:37:37.973: Thread 2: Non-Transactional tables are done, Starting exporting data for Transactional tables
** Message: 21:37:37.973: Thread 1: Schema jobs are done, Starting exporting data for Non-Transactional tables
** Message: 21:37:37.973: Thread 1: Non-Transactional tables are done, Starting exporting data for Transactional tables
** Message: 21:37:37.974: employees.salaries has ~2838426 rows
** Message: 21:37:37.986: Thread 1: `employees`.`salaries` [ 0% ] | Tables: 6/6
** Message: 21:37:38.008: Thread 2: `employees`.`salaries` [ 0% ] | Tables: 6/6
** Message: 21:37:38.075: Thread 1: `employees`.`salaries` [ 3% ] | Tables: 6/6
** Message: 21:37:38.089: Thread 1: `employees`.`salaries` [ 3% ] | Tables: 6/6
```

---

## 5. Backup de Múltiplos Bancos de Dados

```bash
mydumper -u mydumper -p senha -h localhost -P 3306 -B employees,sakila \
-o "/tmp/databases" --sync-thread-lock-mode LOCK_ALL --no-views -G -E -R \
--threads 2 --verbose 3
```

Output verboso:

```
** Message: 21:48:39.860: Thread 1: connected using MySQL connection ID 41
** Message: 21:48:39.861: Thread 2: connected using MySQL connection ID 42
** Message: 21:48:39.862: Thread 1: Creating Jobs
** Message: 21:48:39.863: Written master status
** Message: 21:48:39.863: Thread 1: dumping db information for `employees`
** Message: 21:48:39.867: Thread 2: Creating Jobs
** Message: 21:48:39.867: Thread 2: dumping db information for `sakila`
** Message: 21:48:39.867: Waiting database finish
** Message: 21:48:39.935: Thread 1: Processing Schema jobs
** Message: 21:48:39.935: Thread 1: dumping schema create for `employees`
** Message: 21:48:39.935: Shutdown schema jobs
** Message: 21:48:39.935: Thread 2: Processing Schema jobs
** Message: 21:48:39.936: Thread 2: dumping schema create for `sakila`
** Message: 21:48:39.937: Thread 1: dumping SP and VIEWs for `sakila`
** Message: 21:48:39.937: Thread 2: dumping schema for `employees`.`departments`
** Message: 21:48:39.938: Thread 2: dumping schema for `employees`.`dept_emp`
** Message: 21:48:39.939: Thread 2: dumping schema for `employees`.`dept_manager`
** Message: 21:48:39.940: Thread 2: dumping schema for `employees`.`employees`
** Message: 21:48:39.940: Thread 2: dumping schema for `employees`.`salaries`
** Message: 21:48:39.940: Thread 2: dumping schema for `employees`.`titles`
** Message: 21:48:39.940: Thread 2: dumping schema for `sakila`.`actor`
** Message: 21:48:39.941: Thread 2: dumping schema for `sakila`.`address`
** Message: 21:48:39.942: Thread 2: dumping schema for `sakila`.`category`
```

### Estrutura dos arquivos gerados

```
root@db1:/tmp/databases# ls
employees-schema-create.sql        employees.titles.00000.sql  sakila.film_actor-schema.sql
employees-schema-triggers.sql      employees.titles.00001.sql  sakila.film_actor.00000.sql
employees.departments-schema.sql   metadata                    sakila.film_category-schema.sql
employees.departments.00000.sql    sakila-schema-create.sql    sakila.film_category.00000.sql
employees.dept_emp-schema.sql      sakila-schema-post.sql      sakila.film_text-schema.sql
employees.dept_emp.00000.sql       sakila-schema-triggers.sql  sakila.film_text.00000.sql
employees.dept_emp.00001.sql       sakila.actor-schema.sql     sakila.inventory-schema.sql
employees.dept_manager-schema.sql  sakila.actor.00000.sql      sakila.inventory.00000.sql
employees.dept_manager.00000.sql   sakila.address-schema.sql   sakila.language-schema.sql
employees.employees-schema.sql     sakila.address.00000.sql    sakila.language.00000.sql
employees.employees.00000.sql      sakila.category-schema.sql  sakila.payment-schema.sql
employees.employees.00001.sql      sakila.category.00000.sql   sakila.payment.00000.sql
employees.salaries-schema.sql      sakila.city-schema.sql      sakila.payment.00001.sql
employees.salaries.00000.sql       sakila.city.00000.sql       sakila.rental-schema.sql
employees.salaries.00001.sql       sakila.country-schema.sql   sakila.rental.00000.sql
employees.salaries.00003.sql       sakila.country.00000.sql    sakila.rental.00001.sql
employees.salaries.00007.sql       sakila.customer-schema.sql  sakila.staff-schema.sql
employees.salaries.00011.sql       sakila.customer.00000.sql   sakila.staff.00000.sql
employees.salaries.00019.sql       sakila.film-schema.sql      sakila.store-schema.sql
employees.titles-schema.sql        sakila.film.00000.sql       sakila.store.00000.sql
root@db1:/tmp/databases#
```

O backup gera arquivos separados por tabela. O arquivo `metadata` é crucial, pois contém informações de replicação e contagem de linhas de cada tabela:

```
# Started dump at: 2025-09-05 21:20:19
[config]
quote-character = BACKTICK

[myloader_session_variables]
SQL_MODE='NO_AUTO_VALUE_ON_ZERO,ONLY_FULL_GROUP_BY,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION' /*!40101

[source]
File = binlog.000003
Position = 66379458
Executed_Gtid_Set =

[`employees`.`departments`]
real_table_name=departments
rows = 9

[`employees`.`dept_emp`]
real_table_name=dept_emp
rows = 331603
```

---

## 6. Realizando o Restore com Paralelismo

Diferentemente do `mysqlpump`, o `myloader` permite restauração paralela:

```bash
myloader -u root -p senha -h localhost -P 3306 \
-d "/tmp/databases" --threads 2 \
--verbose 3
```

Output verboso do restore:

```
** Message: 22:04:07.450: Initializing initialize_worker_schema
** Message: 22:04:07.452: Reading metadata: metadata
** Message: 22:04:07.452: Config file loaded
** Message: 22:04:07.452: myloader_session_variables found on metadata
** Message: 22:04:07.452: Change master will be executed for channel: default channel
** Message: 22:04:07.452: metadata pushed
** Message: 22:04:07.452: Intermediate queue: Sending END job
** Message: 22:04:07.455: Intermediate thread ended
** Message: 22:04:07.455: Intermediate thread: SHUTDOWN
** Message: 22:04:07.455: S-Thread 4: Starting import
** Message: 22:04:07.456: Thread 4: restoring create database on `employees` from employees-schema-create.sql. Tables 0 of 22 completed
** Message: 22:04:07.459: S-Thread 5: Starting import
** Message: 22:04:07.459: Thread 5: restoring create database on `sakila` from sakila-schema-create.sql. Tables 0 of 22 completed
** Message: 22:04:07.459: S-Thread 3: Starting import
** Message: 22:04:07.460: S-Thread 6: Starting import
** Message: 22:04:07.460: Executing set session
** Message: 22:04:07.461: Executing set session
** Message: 22:04:07.468: Thread 4: restoring table employees.departments from /tmp/databases/employees.departments-schema.sql
** Message: 22:04:07.468: Thread 6: restoring table employees.dept_manager from /tmp/databases/employees.dept_manager-schema.sql
```

> **Atenção:** Triggers, funções e procedures não são exportados automaticamente pelo MyDumper por questões de consistência durante o restore. Verifique a documentação para exportá-los separadamente quando necessário.

---

## Conclusão

O MyDumper é uma excelente alternativa para ambientes com bancos de dados grandes, onde o tempo de backup e restore é crítico. Com suporte a paralelismo tanto no dump quanto no restore, ele supera as limitações do `mysqldump` e do descontinuado `mysqlpump`, tornando-se uma ferramenta essencial para DBAs que precisam de eficiência em backups MySQL.

Para listar todas as opções disponíveis:

```bash
mydumper --help
```
