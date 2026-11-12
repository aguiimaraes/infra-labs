# Dia Caos (Ensaiado) — Cenário A: 6/6 em sequência aleatória

**Data:** 12/11/2026
**Ambiente:** Ubuntu Server LTS (`lab-linux`, `192.168.100.10`) + cliente Windows (host Hyper-V)
**Formato:** runbook aberto · ordem sorteada · meta ≤ 10 min/falha
**Ordem sorteada:** 1 → 6 → 2 → 4 → 3 → 5

## 1. Resumo

As 6 falhas do Cenário A foram executadas em sequência, em ordem aleatória, com o runbook de diagnóstico aberto. Resultado: **6/6 resolvidas**, total **27 min 30 s** (média ~4,6 min/falha). O ensaiado não só validou o runbook como revelou **5 defeitos** nele, todos corrigidos.

## 2. Desempenho

| Pos. | Falha | Tempo | Comando decisivo | Causa raiz |
|---|---|---|---|---|
| 1 | 1 — config inválida | 4 min | `nginx -t` → `meusite:8` | `;` ausente no `root` |
| 2 | 6 — nome não resolve | 7 min | `curl (6)` + `Get-Content hosts` | linha do `hosts` comentada |
| 3 | 2 — porta 80 ocupada | 7 min 40 s | `ss 'sport = :80'` → apache2 | apache2 na `:80` |
| 4 | 4 — disco cheio | 3 min 50 s | `lsof +L1` → inode órfão | arquivo deletado ainda aberto |
| 5 | 3 — 403 permissão | 3 min | `tail error.log` + `ls -l` | `chmod 600` sem leitura p/ outros |
| 6 | 5 — firewall | 2 min | `ufw status` + `[UFW BLOCK]` | regra `Nginx Full` removida |

## 3. Incidentes dentro do ensaiado (o que quase deu errado)

### 3.1 Falha 1 "fechada" com serviço ainda caído
O arquivo foi corrigido, mas o nginx **não foi reiniciado**. Na provação seguinte, o `curl` do Windows deu `(7) Failed to connect` — nome ok, porta fechada. Foi preciso `systemctl restart nginx`.
**Lição:** `nginx -t` ok só valida o arquivo. Fechamento exige `systemctl is-active nginx` = `active` **e** `curl -I localhost` = `200`.

### 3.2 Falha 2 — `apt install` deu 404
O índice local apontava para uma versão (`2.4.66-2ubuntu2.6`) já removida do repositório. `sudo apt update` trouxe a `.7` e o install funcionou.
**Lição:** 404 em `apt install` = índice velho → `apt update` antes.

### 3.3 Falha 4 — `logrotate` não liberou o espaço
Depois do `rm`, o `df` continuou em 96%, mas o `du` voltou a 11 G. `lsof +L1` mostrou o inode de 2 GB ainda aberto pelo nginx. O `logrotate -f` não resolveu (o arquivo já não existia e o `postrotate` via `invoke-rc.d` não disparou o sinal). `kill -USR1` no master liberou na hora.
**Lição:** `rm` libera o nome, não o espaço. Quem libera é o processo fechando o descritor.

## 4. Defeitos corrigidos no runbook

1. Fechamento obrigatório: `is-active` + `curl 200` (não basta `nginx -t`)
2. Cartão da Falha 2 ganhou `systemctl stop nginx` antes de subir o apache
3. Nota: `apt install` 404 → `apt update`
4. `grep :80` casava com `8080`/`8088` → usar `ss -tlnp 'sport = :80'`
5. Falha 4: `rm` → `df` → `lsof +L1` → `kill -USR1` (logrotate pode não disparar)

## 5. Aprendizados de diagnóstico

- **`(6)` × `(7)`**: `(6) Could not resolve` = nome; `(7) Failed to connect` = porta/conexão. O mesmo par de erros separou "nginx caído" de "DNS quebrado".
- **`df` × `du`**: `df` alto + `du` normal + `lsof +L1` com `(deleted)` = inode órfão.
- **`server_name` mascarado**: com um único server block, ele vira *default* e atende qualquer Host — mascarou o `server_name` antigo.
- **`curl intranet.test` na VM sempre dá `(6)`**: o nome só existe no `hosts` do Windows; testar por IP, `localhost` ou `--resolve`.
- **Higiene**: o `nslookup` expôs o DNS da rede corporativa — mascarar antes de publicar.

## 6. Conclusão

O ensaiado cumpriu o objetivo: 6/6, rápido, e o runbook saiu melhor do que entrou. Próximo passo: **caos oficial** — checkpoint `cenario-a-ok-limpo`, runbook fechado, nova ordem, ≤ 12 min/falha.
