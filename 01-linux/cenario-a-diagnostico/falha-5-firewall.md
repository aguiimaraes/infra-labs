# Relatório de Incidente — Falha 5: firewall bloqueando

**Ambiente:** Ubuntu Server LTS (VM `lab-linux`, 1 vCPU / 1 GB RAM) — Lab 1, Cenário A
**Data:** 09/11/2026
**Serviço envolvido:** `nginx.service` (site `meusite.com.br`) × `ufw`

## 1. Resumo

O site parou de responder **somente para acesso externo**: `curl` local continuava 200, mas o navegador/host não conectava. A causa não era o nginx (que seguia escutando na porta 80), e sim o **ufw**, que com política padrão `deny incoming` não tinha regra liberando HTTP. A pista decisiva veio do `ufw.log` (`[UFW BLOCK] ... DPT=80`) e a correção foi uma única regra.

## 2. Sintoma

**Do host Windows (192.168.100.1):**

```powershell
PS> curl.exe -I http://192.168.100.10
curl: (28) Failed to connect to 192.168.100.10 port 80 after 21057 ms: Could not connect to server
```

**Do próprio servidor (loopback):**

```bash
$ curl -I http://localhost
HTTP/1.1 200 OK
```

Contradição característica: funciona em casa, não pela rua.

## 3. Diagnóstico

**1) O serviço está de pé?**

```bash
$ ss -tlnp | grep :80
LISTEN 0 511 0.0.0.0:80   0.0.0.0:*    users:(("nginx",...))
LISTEN 0 511 [::]:80      [::]:*       users:(("nginx",...))
```

Sim: nginx escutando em IPv4 e IPv6 na 80 — camada de aplicação saudável.

**2) E o firewall?**

```bash
$ sudo ufw status verbose
Status: active
Default: deny (incoming), allow (outgoing), deny (routed)

To                         Action      From
22/tcp (OpenSSH)           ALLOW IN    Anywhere
Anywhere on vxlan.calico   ALLOW IN    Anywhere      # regra injetada pelo microk8s (Calico)
Anywhere on cali+          ALLOW IN    Anywhere
# (nenhuma regra para 80/tcp ou 443/tcp)
```

Ativo, com `deny (incoming)` — e **sem liberação para HTTP**. Só o SSH (22) e as interfaces do Calico estavam liberados.

**3) A prova cabal — log do firewall:**

```bash
$ sudo tail /var/log/ufw.log
[UFW BLOCK] IN=eth0 OUT= MAC=... SRC=192.168.100.1 DST=192.168.100.10 ... PROTO=TCP SPT=61481 DPT=80 ... SYN
```

Repetição de `SYN` com `DPT=80` (tentativa de abrir conexão na 80) vinda do host `192.168.100.1`, bloqueada pelo kernel antes de chegar ao nginx. Os intervalos crescentes entre pacotes (1s, 2s, 4s, 8s...) são as retransmissões TCP clássicas que geram o timeout de ~21s no `curl`.

## 4. Causa raiz

`ufw` habilitado com política padrão de negar entrada e **sem regra de liberação para a porta 80** — a requisição era descartada na camada de rede (netfilter), não no servidor web.

## 5. Solução

```bash
$ sudo ufw allow 'Nginx Full'
Rule added
Rule added (v6)
```

Validação **do host** (a prova que importa):

```powershell
PS> curl.exe -I http://192.168.100.10
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Content-Length: 803
```

## 6. Prevenção / aprendizados

- **Dois planos distintos de verificação:** *serviço* (`ss -tlnp` — quem escuta) × *rede* (`ufw status` / `ufw.log` — quem pode entrar). Sintoma "localhost sim, externo não" aponta decisivamente para o segundo.
- **Ordem de ativação do ufw:** liberar o acesso administrativo **antes** de ativar (`sudo ufw allow OpenSSH`) — caso contrário, o administrador se tranca fora do servidor.
- **Privilégio mínimo:** só liberar as portas realmente usadas (`Nginx Full` = 80/443). As portas 8080 (node) e 8088 (ferretdb/local) seguem fechadas — corretamente.
- **`ufw show added` × `ufw status`:** o primeiro lista as regras gravadas; o segundo mostra o estado efetivo — conferir ambos ao auditar.
- **Log como evidência:** `[UFW BLOCK] ... DPT=80` com retransmissões crescentes conta a história do timeout sem precisar de packet capture.
