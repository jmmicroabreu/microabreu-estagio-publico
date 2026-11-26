# Backups do PostgreSQL com pgBackRest

Backups físicos automáticos com **pgBackRest**: completo ao domingo, incremental de hora a hora e arquivo contínuo de WAL, o que permite **PITR** (repor a base de dados para qualquer minuto).

Testado com **Oracle Linux 9** e **PostgreSQL 18** do módulo do sistema (`dnf module`). Para testar os backups numa VM, ver [Teste de restore numa VM](teste-restore-vm.md).

## Conceitos

| Conceito | Significado |
| :--- | :--- |
| **WAL** (*Write-Ahead Log*) | O "diário" do PostgreSQL: cada alteração é escrita aqui antes de ir para as tabelas |
| **Backup completo** | Cópia física do servidor inteiro (todas as bases de dados, utilizadores e configuração) |
| **Backup incremental** | Só os ficheiros que mudaram desde o último backup. Cada incremental depende do anterior; o completo semanal recomeça a cadeia |
| **PITR** | Repor o último completo + incrementais e reaplicar os WAL até ao minuto pedido |
| **Stanza** | O nome que o pgBackRest dá a um servidor PostgreSQL (aqui, `main`) |

```
PostgreSQL ──(cada alteração)──► WAL ──archive-push──► repositório do pgBackRest
                                                            ▲
cron ── domingo 02:00 ── backup COMPLETO ───────────────────┤
     └─ de hora a hora (min. 30) ── INCREMENTAL ────────────┘
```

| | pgBackRest (agendado) | `pg_dump` pontual |
| :--- | :--- | :--- |
| Para quê | Recuperação de desastre | Copiar ou investigar uma base de dados |
| O que copia | O servidor inteiro | Uma só base de dados |
| PITR | ✅ Para qualquer minuto | ❌ Só o momento em que foi feito |
| Restore | Para uma VM ou pasta à parte, nunca direto em produção | Para uma BD com outro nome |

> **Regra de ouro:** nunca restaurar diretamente em produção. O restore repõe o servidor **inteiro**. Restaura-se noutra máquina e tira-se de lá só a base de dados que precisa.

## Disco de dados (LVM + XFS)

Recomenda-se ter os dados num disco próprio. Exemplo com um disco vazio `/dev/vdb` (confirmar o nome com `lsblk`):
```bash
blkid /dev/vdb; wipefs -n /dev/vdb        # não devem devolver nada
pvcreate /dev/vdb
vgcreate datavg /dev/vdb
lvcreate -n pgdata -l 90%FREE datavg       # 10% de margem
mkfs.xfs /dev/datavg/pgdata
```

Se o PostgreSQL já tiver dados, copiá-los com o serviço parado:
```bash
dnf install -y rsync
systemctl stop postgresql
mkdir -p /mnt/novo && mount /dev/datavg/pgdata /mnt/novo
rsync -aAXH /var/lib/pgsql/ /mnt/novo/
umount /mnt/novo
mv /var/lib/pgsql /var/lib/pgsql.antigo && mkdir /var/lib/pgsql
echo '/dev/mapper/datavg-pgdata  /var/lib/pgsql  xfs  defaults  0 0' >> /etc/fstab
systemctl daemon-reload && mount -a
chown postgres:postgres /var/lib/pgsql && chmod 700 /var/lib/pgsql
systemctl start postgresql
reboot                                     # testar: df -h /var/lib/pgsql
```

## Instalação

O pgBackRest vem do repositório oficial do PostgreSQL (PGDG), na parte *common*.
```bash
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf config-manager --set-disabled 'pgdg1*'      # o PostgreSQL vem do módulo do sistema
grep -n 'common/redhat' /etc/yum.repos.d/pgdg-redhat-all.repo | head -1
```

O Oracle Linux apresenta-se como `9.x` e a pasta `rhel-9.x` não existe no PGDG (erro 404). Corrigir na linha que o `grep` devolveu (no nosso caso, 17):
```bash
cp /etc/yum.repos.d/pgdg-redhat-all.repo /root/pgdg-redhat-all.repo.bak
sed -i '17s|rhel-\$releasever_major\.\$releasever_minor|rhel-9|' /etc/yum.repos.d/pgdg-redhat-all.repo
sed -n '17p' /etc/yum.repos.d/pgdg-redhat-all.repo
dnf clean metadata
dnf install pgbackrest
pgbackrest version
```

> ⚠️ Não desativar o módulo `postgresql` do sistema (o guia do PGDG manda fazê-lo, mas o PostgreSQL vem desse módulo). Antes de aceitar o `dnf install`, confirmar que não aparece nenhum pacote `postgresql…` na lista.

## Configuração

`/etc/pgbackrest/pgbackrest.conf` (dono `root:postgres`, permissões `640`):
```ini
[global]
repo1-path=/var/lib/pgsql/pgbackrest
repo1-retention-full=4
compress-type=zst
start-fast=y
log-level-console=info
log-level-file=detail

[main]
pg1-path=/var/lib/pgsql/data
```

| Opção | Significado |
| :--- | :--- |
| `repo1-path` | Onde ficam os backups |
| `repo1-retention-full=4` | Guarda 4 completos (≈1 mês); os mais antigos e os seus WAL são apagados sozinhos |
| `compress-type=zst` | Compressão zstd |
| `start-fast=y` | O backup começa logo, sem esperar pelo próximo *checkpoint* |
| `pg1-path` | Pasta de dados do PostgreSQL |

```bash
mv /etc/pgbackrest.conf /etc/pgbackrest.conf.original
install -d -o postgres -g postgres -m 750 /var/lib/pgsql/pgbackrest
install -d -m 755 /etc/pgbackrest
# criar /etc/pgbackrest/pgbackrest.conf com o conteúdo acima
chown root:postgres /etc/pgbackrest/pgbackrest.conf && chmod 640 /etc/pgbackrest/pgbackrest.conf

su - postgres -c "psql -c \"ALTER SYSTEM SET archive_mode = 'on';\""
su - postgres -c "psql -c \"ALTER SYSTEM SET archive_command = 'pgbackrest --stanza=main archive-push %p';\""
su - postgres -c "psql -c \"ALTER SYSTEM SET archive_timeout = '300';\""   # arquiva WAL pelo menos a cada 5 min
systemctl restart postgresql

su - postgres -c "pgbackrest --stanza=main stanza-create"
su - postgres -c "pgbackrest --stanza=main check"
su - postgres -c "pgbackrest --stanza=main --type=full backup"
su - postgres -c "pgbackrest --stanza=main info"
```

## Agendamento

`crontab -e -u postgres`:
```
# completo ao domingo às 02:00
0 2 * * 0  pgbackrest --stanza=main --type=full backup
# incremental de hora a hora, ao minuto 30 (não coincide com o completo)
30 * * * * pgbackrest --stanza=main --type=incr backup
```

## Verificações

```bash
su - postgres -c "pgbackrest --stanza=main info"            # status: ok, último incremental com menos de 1 h
grep -h -E 'ERROR|WARN' /var/log/pgbackrest/*.log | tail     # sem erros
df -h / /var/lib/pgsql
```

## Restore de teste no próprio servidor (porta 5433)

Para uma pasta à parte, sem tocar no servidor principal:
```bash
install -d -o postgres -g postgres -m 700 /var/lib/pgsql/restore_teste
su - postgres -c "pgbackrest --stanza=main --pg1-path=/var/lib/pgsql/restore_teste \
  --type=time '--target=AAAA-MM-DD HH:MM:SS+00' --target-action=promote --archive-mode=off restore"
```

Só depois de `completed successfully`:
```bash
echo "port = 5433" >> /var/lib/pgsql/restore_teste/postgresql.auto.conf
echo "listen_addresses = 'localhost'" >> /var/lib/pgsql/restore_teste/postgresql.auto.conf
su - postgres -c "pg_ctl -D /var/lib/pgsql/restore_teste -l /var/lib/pgsql/restore_teste.log start"
su - postgres -c "psql -p 5433 -c 'SELECT datname FROM pg_database ORDER BY 1;'"

# no fim
su - postgres -c "pg_ctl -D /var/lib/pgsql/restore_teste stop"
rm -rf /var/lib/pgsql/restore_teste /var/lib/pgsql/restore_teste.log
```

| Opção | Significado |
| :--- | :--- |
| `--pg1-path=…` | Restaurar para esta pasta, e não para a pasta de dados verdadeira |
| `--type=time --target=…` | Parar de aplicar os WAL nesta hora (em UTC, `+00`) |
| `--type=default` | Recuperar até ao último momento possível. **Sem** `--target-action` |
| `--target-action=promote` | Ao chegar à hora, fica um servidor normal |
| `--archive-mode=off` | A cópia não envia WAL para o repositório |

## Backup pontual pelo pgAdmin

Para copiar ou investigar **uma só base de dados**:
- **Backup:** botão direito na BD → *Backup…* → formato *Custom* (igual a `pg_dump -Fc`).
- **Restore:** criar uma BD com **outro nome** → botão direito → *Restore…* → *Custom or tar*.

O pgAdmin usa o `pg_dump` do PC: o ficheiro fica no PC. Se aparecer `server version mismatch`, atualizar o pgAdmin.

## Erros e lições

| Erro | Causa | Solução |
| :--- | :--- | :--- |
| `ERROR: [031]: option 'target-action' not valid…` | `--target-action` com `--type=default` | Tirar o `--target-action` |
| `ERROR: [040]: unable to restore to path … because it contains files` | A pasta de destino não está vazia | Esvaziá-la e repetir |
| `FATAL: recovery ended before configured recovery target was reached` | Não há nenhuma transação **depois** da hora pedida | Usar `--type=default`, ou criar uma alteração e `SELECT pg_switch_wal();` |
| A hora "não bate certo" | O `--target` está em UTC | Converter a hora local para UTC |

> Colar um bloco inteiro de comandos é prático, mas se o primeiro falhar os seguintes correm na mesma. Nos restores, correr o `restore` sozinho e confirmar `completed successfully` antes de avançar.
