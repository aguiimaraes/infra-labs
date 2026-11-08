# Relatório de Incidente — Falha 4: disco cheio por log

**Ambiente:** Ubuntu Server LTS (VM `lab-linux`, 1 vCPU / 1 GB RAM, LVM de 14G + /boot 2G) — Lab 1, Cenário A
**Data:** 07–08/11/2026
**Serviço afetado:** `nginx.service` (site `meusite.com.br`)
**Arquivo crítico:** `/var/log/nginx/access.log`

## 1. Resumo

Uma auditoria de rotina no filesystem raiz encontrou **82% de uso** numa VM de 14G antes de qualquer falha induzida. A investigação por `du` apontou o verdadeiro consumidor: **4,4G de snaps** em `/var/lib/snapd` — não logs. Após a limpeza (79%), o cenário de log gigante foi reproduzido; um `dd` sem `count` encheu o disco a **100%**, e o `rm` subsequente não liberou o espaço — um **inode fantasma** mantido aberto pelo nginx (3,52 GB) só foi liberado com `kill -USR1`. Na reprodução controlada, `truncate` (arquivo esparso) × `fallocate` (blocos físicos) mostraram comportamentos opostos no `df`, e o `logrotate -f` fechou o ciclo.

## 2. Sintomas

```bash
$ df -h /
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv   14G   11G  2.4G  82%   # baseline anômala
...
/dev/mapper/ubuntu--vg-ubuntu--lv   14G   14G     0 100%   # após o dd
```

Após `sudo rm /var/log/nginx/access.log`, o `df` **permaneceu 100%**.

## 3. Diagnóstico

**Fase A — quem ocupa 11G (antes de qualquer falha):**

```bash
$ sudo du -xh --max-depth=2 / | sort -h | tail
1.9G  /usr/lib      2.5G  /usr      4.6G  /var/lib     5.7G  /var
11G   /
$ sudo du -sh /var/lib/*
4.4G  /var/lib/snapd        # 31% do disco em snaps
$ sudo journalctl --disk-usage
Archived and active journals take up 94.8M   # journald inocente
$ sudo sh -c 'du -sh /var/lib/snapd/snaps/* | sort -h | tail'
543M  wekan_4669.snap   172M  microk8s_9072.snap   118M  lxd_40958.snap ...
```

Causa do baseline: **acúmulo de revisões de snap** (revisões `disabled` mantidas para rollback) + cache do apt, não logs.

**Fase B — o `rm` que não libera:**

```bash
$ sudo lsof +L1
nginx  1242 root      3w REG 252,0 3778019328 0 264046 /var/log/nginx/access.log (deleted)
nginx  1246 www-data  3w REG 252,0 3778019328 0 264046 /var/log/nginx/access.log (deleted)
```

`NLINK=0` (sem nome) + `SIZE/OFF = 3.778.019.328` (≈3,52 GB ainda alocados) + fd `3w` aberto nos dois processos do nginx → o `rm` removeu o **nome**, não o **espaço**.

**Fase C — a armadilha do `dd` sem `count`:**

```bash
$ sudo dd if=/dev/zero of=/var/log/nginx/access.log bs=1M status=progress
# sem count → cópia infinita até esgotar o disco (ENOSPC)
```

## 4. Causa raiz

1. **Do incidente "real":** ausência de monitoramento de threshold + acúmulo de revisões de snap (4,4G), levando o filesystem a 82% sem alerta.
2. **Do enchimento:** `dd` sem `count` consome todo o espaço livre disponível.
3. **Do "rm sem efeito":** um arquivo aberto por processo continua ocupando espaço após o `unlink`; só é liberado quando o último descritor é fechado (`kill -USR1` no nginx master reabre os logs).

## 5. Solução

**Limpeza estrutural (devolveu ~3% e preveniu recorrência):**

```bash
$ sudo snap remove wekan --revision=4649        # + core24, hwctl, powershell, snapd (revisões disabled)
$ sudo snap set system refresh.retain=2         # limita revisões futuras
$ sudo apt clean                                # libera /var/cache/apt/archives
$ rm ~/nextcloud-backup.tar.gz                  # resíduo de 13M
```

**Liberação do inode fantasma:**

```bash
$ sudo kill -USR1 $(cat /run/nginx.pid)   # mesmo sinal usado pelo logrotate (postrotate)
$ df -h /
/dev/mapper/ubuntu--vg-ubuntu--lv   14G   11G  2.9G  79%   # 3,5G liberados
```

**Reprodução controlada + prevenção:**

```bash
$ sudo truncate -s 1500M /var/log/nginx/access.log   # esparso: ls=1.5G, blocos reais ~0
$ sudo fallocate -l 1500M /var/log/nginx/access.log  # físico: df → 96%
$ sudo logrotate -f /etc/logrotate.d/nginx

$ ls -lh /var/log/nginx/
-rw-r----- www-data adm     0   access.log       # novo, perms da política (create 0640)
-rw-r--r-- www-data root  1.5G  access.log.1     # gigante rotacionado (espaço ainda ocupado)
-rw-r----- www-data adm   315    access.log.2.gz # ciclos anteriores comprimidos
-rw-r----- www-data adm   483    access.log.3.gz

$ sudo rm /var/log/nginx/access.log.1
$ df -h /
14G  11G  2.2G  84%
$ curl http://localhost   # 200 OK — página HTML5 entregue
```

## 6. Prevenção / aprendizados

- **Threshold:** disco acima de ~80% é alerta. Monitorar com `df -h` (bytes) e `df -i` (inodes) e investigar com `du -x` — sempre filesystem → diretórios → arquivos.
- **`rm` libera nome, não espaço:** checar referências abertas com `sudo lsof +L1`; solução por serviço = reabrir/fechar o fd (`kill -USR1` para nginx, ou restart controlado).
- **`dd` sempre com `count`**; para "criar arquivo grande de teste" usar `fallocate` (blocos físicos) — e saber que `truncate` cria esparso (espaço lógico ≠ espaço físico; `ls` ≠ `du`).
- **Logrotate é a defesa estrutural**: `daily`, `rotate 14`, `compress` + `delaycompress`, `create 0640 www-data adm`, `postrotate kill -USR1` — a rotação move o nome; a compressão vem no ciclo seguinte; remover artefatos de teste é passo manual.
- **Consumo concorrente identificado:** o uso final (84%) ficou acima do baseline pós-limpeza (79%). `snap changes` não mostra refreshes, mas registra sequência de **"Running service command" para wekan e microk8s** (23:30 e 00:04–00:05), coincidindo com a janela; `journald` cresceu 94,8M → 125,6M e `/boot` 96M → 183M. Lição: em VM com serviços ativos, o `df` muda durante o experimento — medir antes/depois em janela curta e considerar processos de segundo plano.
