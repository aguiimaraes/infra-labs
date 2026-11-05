# Relatório de Incidente — Falha 2: porta 80 ocupada

**Ambiente:** Ubuntu Server LTS (VM `lab-linux`, 1 vCPU / 1 GB RAM) — Lab 1, Cenário A
**Data:** 27–28/10/2026
**Serviço afetado:** `nginx.service`
**Duração:** ~40 min (sintoma notado às 00:36 UTC de 28/10)

## 1. Resumo

Ao tentar iniciar o nginx logo após a instalação, o serviço falhou porque a porta 80 já estava ocupada por um terceiro servidor HTTP. A investigação superficial sugeriu configurações do nginx e até um XAMPP — a causa real era o **pacote Snap do Nextcloud, que embute um Apache próprio (`httpd`)** já ativo na porta 80.

## 2. Sintoma

```
Oct 28 00:36:27 lab-linux nginx[24761]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
nginx.service: Failed with result 'exit-code'
```

O acesso via browser/curl à porta 80 entregava página do Nextcloud, não a intranet.

## 3. Diagnóstico

1. `sudo journalctl -xeu nginx` → erro 98 (Address already in use) confirmou disputa de porta.
2. `sudo ss -tlnp | grep :80` → dono da porta: `httpd` (pids 881–101557) — **não era nginx nem apache2**.
3. Trilha falsa 1: `sudo grep -rni nextcloud /etc/nginx/` não retornou nada (a config não estava no nginx).
4. Trilha falsa 2: suspeita de XAMPP (`/opt/lampp`) — descartada.
5. `ps -p 101557 -o pid,cmd` revelou a causa:

```
httpd -d /snap/nextcloud/54589 -k start -DFOREGROUND
```

## 4. Causa raiz

O **Nextcloud instalado via Snap** é um serviço autocontido: empacota seu próprio Apache como `httpd` e assume as portas 80/443. Não deixa rastro em `/etc/nginx/` nem aparece como `apache2.service` no systemd — impedindo qualquer web server que pretenda usar a porta 80.

## 5. Solução

```bash
sudo snap stop nextcloud        # para os serviços do snap
sudo snap disable nextcloud     # impede boot futuro
sudo ss -tlnp | grep :80        # confirmação: porta livre
sudo systemctl daemon-reload
sudo systemctl enable --now nginx
```

Validação: `curl http://localhost` retornou o `index.html` do site.

## 6. Prevenção / aprendizados

- Antes de instalar um web server, **auditar quem já escuta nas portas alvo**: `sudo ss -tlnp`.
- Serviços em Snap são caixas-pretas: não aparecem nas configs padrão — `ps -p <PID> -o cmd` é a ferramenta certa para achar a origem de um processo desconhecido.
- Nesta VM, serviços concorrentes documentados: `node` (:8080) e `ferretdb` (127.0.0.1:8088).
- Lição processual: quando `grep` não encontra o culpado no lugar esperado, **seguir pelo processo vivo** (donos da porta), não por arquivos de config.
