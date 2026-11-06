# Relatório de Incidente — Falha 3: erro 403 por permissão

**Ambiente:** Ubuntu Server LTS (VM `lab-linux`, 1 vCPU / 1 GB RAM) — Lab 1, Cenário A
**Data:** 06/11/2026
**Serviço afetado:** `nginx.service` (site `meusite.com.br`)

## 1. Resumo

A requisição de volta ao site retornou **403 Forbidden** mesmo com o nginx saudável e a porta 80 correta. A causa foi permissão de leitura removida do `index.html`: o modo `600` negou acesso a todos os usuários além do dono, e o nginx — que roda como `www-data` — não pode mais abrir o arquivo.

## 2. Sintoma

```bash
$ curl -I http://localhost
HTTP/1.1 403 Forbidden
Server: nginx/1.28.3 (Ubuntu)
```

Requisição chega ao Virtual Host correto e retorna código de permissão.

## 3. Diagnóstico

**Evidência 1 — log de erro do nginx:**

```bash
$ sudo tail /var/log/nginx/error.log
2026/11/06 18:31:34 [error] 1268#1268: *9 open() "/var/www/meusite/index.html"
failed (13: Permission denied), client: ::1, server: meusite.com.br, request: "HEAD / HTTP/1.1", host: "localhost"
```

`open() ... failed (13: Permission denied)` — o código 13 do sistema (`EACCES`) aponta para permissão no arquivo, não para ausência (`404`) nem defeito do servidor (`500`). O log ainda confirma `server: meusite.com.br`, descartando roteamento errado de Virtual Host.

**Evidência 2 — permissões do arquivo:**

```bash
$ ls -l /var/www/meusite/index.html
-rw------- 1 root root 803 Oct 28 17:24 /var/www/meusite/index.html
```

Modo `600`: dono lê/escreve; grupo e demais — nada. O nginx roda como usuário `www-data`, que se encaixa no "demais": sem bit de leitura, a leitura é negada pelo kernel.

## 4. Causa raiz

Modo de permissão `600` aplicado ao `index.html` removiu o bit de leitura do mundo ("others"), do qual o `www-data` necessita para servir conteúdo estático.

## 5. Solução

```bash
$ sudo chmod 644 /var/www/meusite/index.html   # dono :owner rw, grupo/demais re-read
$ curl http://localhost                        # HTTP 200 + conteúdo HTML entregue
```

Validação: página completa retornada, ciclo fechado.

## 6. Prevenção / aprendizados

- Padrão de permissões para conteúdo web estático: **644 em arquivos, 755 em pastas** (pasta precisa do bit `x` para permitir "atravesar"/listar).
- O usuário do processo (não o root) é quem efetivamente lê os arquivos: sempre conferir `ls -l` pensando em "como o `www-data` vê isso".
- Comparação típica de 403: bits de permissão (aqui) vs. dono errado (remédio: `chown www-data:www-data`) vs. pasta pai fechada (`chmod 755 /var/www/meusite`).
- Hábito de diagnóstico consolidado: `curl -I` para o código → `error.log` para o motivo numerado → `ls -l` para o estado real das permissões.
