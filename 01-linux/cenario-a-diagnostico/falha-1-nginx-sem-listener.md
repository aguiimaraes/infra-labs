# Relatório de Incidente - Falha 1: nginx sem servidor configurado

**Ambiente:** RockyLinux (VM `lab-linux`, 1 vCPU / 1,5 GB RAM) - Lab 1, Cenário A
**Data:** 28/10/2026
**Serviço afetado:** `nginx.service`

## 1. Resumo

Com a porta 80 finalmente livre, o nginx estava `active (running)` mas **nenhuma requisição era atendida**: ele subiu **sem nenhum listener** porque a configuração do VirtualHost não estava ativada em `sites-enabled/`. Somado a isso, um `nginx -t` executado sem `sudo` gerou um erro enganoso de permissão.

## 2. Sintoma

```bash
$ curl http://localhost
curl: (7) Failed to connect to localhost port 80... Could not connect to server
$ sudo systemctl status nginx          # Active: active (running) ✅
$ sudo ss -tlnp | grep :80             # NENHUM processo nginx na porta ⚠️
```

Contradição inicial: serviço saudável, mas ninguém escutando.

## 3. Diagnóstico

1. `nginx -t` **sem sudo** retornou:

```
[warn] "user" directive makes sense only if the master process runs with super-user privileges
[emerg] open() "/run/nginx.pid" failed (13: Permission denied)
```

→ alarme falso: falta de privilégio, não de sintaxe.

2. `sudo nginx -t` → `syntax is ok / test is successful`.
3. `sudo nginx -T | grep -E "listen|server_name"` + `ls -l /etc/nginx/sites-enabled/` → o nginx carrega tudo via `include /etc/nginx/sites-enabled/*;` e **o symlink do site não estava lá** → zero blocos `server` ativos → nginx sobe e não escuta em porta alguma.

## 4. Causa raiz

Confundiu-se **escrever a configuração** (em `sites-available/`) com **ativá-la** (link simbólico em `sites-enabled/`). Sem o link, o include não traz nada. Além disso, o nginx não relê a config sem reload/restart, e a validação tinha sido feita com privilégios insuficientes.

## 5. Solução

```bash
sudo ln -s /etc/nginx/sites-available/meusite /etc/nginx/sites-enabled/
sudo nginx -t                              # sempre com sudo
sudo systemctl restart nginx
sudo ss -tlnp | grep :80                   # nginx agora é dono da :80
curl -I http://localhost                   # HTTP/1.1 200 OK
```

## 6. Prevenção / aprendizados

- Checklist mental fixado para **qualquer** mudança de virtual host: escrever → `ln -s` → `sudo nginx -t` → `reload` → `curl` de validação.
- `nginx -t`, `nginx -s reload` e similares exigem `sudo`; sem root, os erros de permissão mascaram o diagnóstico.
- "Serviço ativo" ≠ "serviço servindo": **sempre cruzar o status do systemd com a evidência de porta/listener** antes de considerar resolvido.
- Diferença entre `nginx -t` (sintaxe) e `nginx -T` (dump da config completa, mostra o que realmente está sendo carregado).
