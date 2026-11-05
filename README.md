# infra-labs

Laboratórios práticos de infraestrutura: Linux, redes, identidade, PowerShell e cloud — com roteiro, provocações de falha, diagnóstico e relatórios de incidente. Material de estudo em formato de cenário real, documentado como faria em produção.

## Status dos laboratórios

| Lab | Escopo | Status |
|---|---|---|
| [01-linux](01-linux/) | Servidor Linux, diagnósticos e hardening | 🚧 em andamento (Cenário A) |
| 02-powershell-git | Inventário de parque e automação | 🗓 planejado (nov/2026) |
| 03-ad-hibrido | Active Directory + Entra ID | 🗓 planejado (dez/2026) |
| 07-docker-nginx | Docker, TLS e proxy reverso | 🗓 planejado (abr/2027) |

## Lab 01 — Linux

### Cenário A: "O sistema interno parou" (diagnóstico)

Servidor web de intranet em Nginx, com seis falhas provocadas e resolvidas uma a uma, cada uma documentada em relatório de incidente de 1 página.

| Falha | Tema | Relatório |
|---|---|---|
| 1 | Nginx sem servidor configurado (`sites-enabled` vazio, `nginx -t` sem sudo) | [falha-1-nginx-sem-listener.md](01-linux/cenario-a-diagnostico/falha-1-nginx-sem-listener.md) |
| 2 | Porta 80 ocupada pelo `httpd` do pacote Snap do Nextcloud | [falha-2-porta-80-ocupada.md](01-linux/cenario-a-diagnostico/falha-2-porta-80-ocupada.md) |
| 3 | Erro 403 por permissão | 🚧 em andamento |
| 4 | Disco cheio por log | 🗓 |
| 5 | Firewall bloqueando (`ufw`) | 🗓 |
| 6 | Nome não resolve (`hosts`, DNS) | 🗓 |

**Ambiente:** Ubuntu Server LTS (VM dedicada, 1 vCPU / 1 GB RAM) · acesso remoto por SSH · snapshot antes de cada risco

### Cenário B: "Servidor novo para a filial" (construção e proteção)

Servidor de filial do zero: usuários por grupo, SSH só por chave, firewall restritivo, LVM, backup com rsync e rotina de saúde em Bash. 🗓 planejado

## Estrutura

```
infra-labs/
├── README.md                        # este índice
└── 01-linux/
    └── cenario-a-diagnostico/       # relatórios de incidente (1 página cada)
```

## Metodologia

- **Provocar → diagnosticar → resolver → documentar**: cada falha é induzida de propósito e resolvida com evidências (logs, status, saída de comandos).
- Ferramentas nativas antes de buscar na internet: `man` e `--help` como pesquisa primária.
- Snapshot da VM antes de cada risco — ambiente sempre recuperável.
- Relatórios no padrão: resumo → sintoma → diagnóstico → causa raiz → solução → prevenção.
