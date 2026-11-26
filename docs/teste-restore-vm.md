# Teste de restore numa VM

Testar os backups do pgBackRest **fora dos servidores**, numa máquina virtual local. É a forma recomendada de fazer o **teste mensal** de restore: prova que os backups não estão corrompidos e que se consegue repor o servidor noutra máquina.

Pressupõe os backups configurados como em [Backups com pgBackRest](backups-pgbackrest.md).

| Fase | O quê |
| :---: | :--- |
| 1 | Criar a VM no VirtualBox (2 discos) |
| 2 | Instalar o Oracle Linux 9 (*Minimal Install*) |
| 3 | 2.º disco: LVM + XFS em `/var/lib/pgsql` |
| 4 | PostgreSQL + pgBackRest, iguais aos do servidor |
| 5 | Copiar o repositório do pgBackRest do servidor para a VM |
| 6 | Restore e verificação |

## 1. Criar a VM (VirtualBox)

| Item | Valor |
| :--- | :--- |
| ISO | Oracle Linux 9, a **mesma versão** do servidor (`cat /etc/oracle-release`) |
| *Skip Unattended Installation* | Marcar (instalar à mão, para controlar os discos) |
| CPU / RAM | 2 CPUs, 4096 MB |
| Disco 1 (sistema) | 20 GB, VDI dinâmico |
| Disco 2 (dados) | Tamanho dos backups + margem, VDI dinâmico, no controlador SATA |
| Rede | NAT |
| Reencaminhamento de portas | TCP, IP anfitrião `127.0.0.1`, porta anfitrião `2222` → porta convidado `22` |

Acesso por SSH a partir do PC: `ssh -p 2222 root@127.0.0.1` (ou num cliente como o MobaXterm).

## 2. Instalar o Oracle Linux

- **Teclado: Português** logo no início (com teclado US as teclas ficam trocadas, ex.: `_` sai `?` na palavra-passe).
- Destino da instalação: **só o disco do sistema**, configuração automática.
- Software: **Minimal Install** (por omissão vem *Server with GUI*).
- Rede ON; KDUMP desligado; palavra-passe do `root` com *Allow root SSH login with password* (só numa VM de testes).

Ajustes, se for preciso:
```bash
localectl set-keymap pt-latin1
echo 'PermitRootLogin yes' > /etc/ssh/sshd_config.d/01-root.conf && systemctl restart sshd
```

Confirmar:
```bash
cat /etc/os-release | grep PRETTY; lsblk; ping -c 3 8.8.8.8
dnf -y update
```

## 3. Disco de dados (LVM + XFS)

No VirtualBox o 2.º disco aparece como `sdb`.
```bash
blkid /dev/sdb; wipefs -n /dev/sdb       # não devem devolver nada

pvcreate /dev/sdb
vgcreate datavg /dev/sdb
lvcreate -n pgdata -l 90%FREE datavg
mkfs.xfs /dev/datavg/pgdata

mkdir -p /var/lib/pgsql
echo '/dev/mapper/datavg-pgdata  /var/lib/pgsql  xfs  defaults,nofail  0 0' >> /etc/fstab
systemctl daemon-reload
mount -a
df -h /var/lib/pgsql
reboot                                    # depois: df -h /var/lib/pgsql
```

> `nofail` deixa a VM arrancar mesmo que o disco de dados falte. O `mount -a` antes do `reboot` apanha erros no fstab, que podem deixar o sistema em modo de emergência.

## 4. PostgreSQL e pgBackRest

A **mesma versão principal** do servidor: um backup de PostgreSQL 18 só restaura em PostgreSQL 18. **Não** correr `postgresql-setup --initdb`: a pasta de dados vem do restore, e o pgBackRest exige que esteja vazia.

```bash
dnf module -y install postgresql:18
chown postgres:postgres /var/lib/pgsql && chmod 700 /var/lib/pgsql
```

**SELinux.** Ao contrário de muitos servidores, a VM tem o SELinux ativo (o padrão no Oracle Linux e no RHEL). As pastas precisam da etiqueta `postgresql_db_t`, senão o PostgreSQL pode ser impedido de arrancar:
```bash
getenforce                        # Enforcing
restorecon -Rv /var/lib/pgsql
ls -Zd /var/lib/pgsql             # … postgresql_db_t …
```

**pgBackRest e configuração:** iguais ao servidor (ver [Backups com pgBackRest](backups-pgbackrest.md#instalação)).

**Idioma.** Se o servidor de origem usa `pt_PT.UTF-8` e a VM foi instalada em inglês, instalar o idioma **antes do restore**:
```bash
dnf install -y glibc-langpack-pt
locale -a | grep -i pt_pt
```

> O servidor de destino de um restore tem de ter os mesmos *locales* do servidor de origem. A língua vem dentro do backup (`lc_messages` e o *locale* de cada base de dados), por isso não se resolve mudando a configuração. Para ver a do servidor: `su - postgres -c "psql -c 'SHOW lc_messages;'"`.

## 5. Copiar o repositório do pgBackRest

Na VM (o `rsync` só lê do servidor):
```bash
dnf install -y rsync
rsync -a --info=progress2 -e "ssh -p PORTA_SSH" root@SERVIDOR:/var/lib/pgsql/pgbackrest/ /var/lib/pgsql/pgbackrest/
chown -R postgres:postgres /var/lib/pgsql/pgbackrest
restorecon -R /var/lib/pgsql/pgbackrest
su - postgres -c "pgbackrest --stanza=main info"   # os mesmos backups do servidor
```

A barra no fim da origem quer dizer "o conteúdo desta pasta". Se for interrompido, ao repetir só copia o que falta.

## 6. Restore e verificação

```bash
systemctl is-active postgresql   # inactive
ls -la /var/lib/pgsql/data       # vazia ou inexistente
# num teste repetido:
# systemctl stop postgresql; rm -rf /var/lib/pgsql/data
# install -d -o postgres -g postgres -m 700 /var/lib/pgsql/data
```

Correr **só** um destes e confirmar `completed successfully`:
```bash
# a) Até ao último momento
su - postgres -c "pgbackrest --stanza=main --type=default --archive-mode=off restore"

# b) Até uma hora (PITR), em UTC
su - postgres -c "pgbackrest --stanza=main --type=time '--target=AAAA-MM-DD HH:MM:SS+00' --target-action=promote --archive-mode=off restore"
```

`--archive-mode=off` é essencial: a VM não pode arquivar WAL no repositório, que é uma cópia dos backups do servidor.

```bash
restorecon -R /var/lib/pgsql/data
systemctl start postgresql                 # sem "enable": só arranca quando se testa
systemctl status postgresql --no-pager     # active (running)
su - postgres -c "psql -c 'SELECT datname FROM pg_database ORDER BY 1;'"
# + uma consulta numa tabela conhecida
```

Para devolver uma só base de dados ao servidor: `pg_dump -Fc` na VM e restore no servidor **com outro nome**.

## Checklist do teste mensal

- [ ] Copiar o repositório (fase 5) e `pgbackrest info`: último completo e incrementais presentes.
- [ ] Restore até ao último momento: `completed successfully`.
- [ ] O PostgreSQL arranca, lista todas as bases de dados e uma consulta conhecida devolve os dados esperados.
- [ ] Registar a data, o servidor, o *backup set* usado e o resultado.

## Erros comuns

| Erro | Solução |
| :--- | :--- |
| *Kernel panic* no 1.º arranque (kernel UEK no VirtualBox) | Escolher a 1.ª entrada do GRUB. Se não chegar: ativar I/O APIC e PAE/NX, paravirtualização KVM; no Windows, desligar o Hyper-V / *Integridade da memória* |
| VM lenta / tartaruga verde no VirtualBox | O Windows tem o Hyper-V ativo: desligá-lo |
| Palavra-passe do `root` não funciona | Instalação com teclado US: experimentar com as teclas trocadas; depois `localectl set-keymap pt-latin1` |
| O cliente SSH não liga a `127.0.0.1:2222` | Rever o reencaminhamento de portas (colunas trocadas) |
| `invalid value for parameter "lc_messages": "pt_PT.UTF-8"` | `dnf install -y glibc-langpack-pt` e arrancar de novo; não é preciso repetir o restore |
| O PostgreSQL não arranca por permissões, com dono e `chmod` certos | SELinux: `restorecon -R /var/lib/pgsql` |
| `ERROR: [040] … because it contains files` | Esvaziar a pasta de destino e repetir |
| `ERROR: [031] option 'target-action' not valid…` | Tirar `--target-action` com `--type=default` |
