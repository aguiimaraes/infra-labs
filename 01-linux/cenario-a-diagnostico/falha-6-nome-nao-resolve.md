# Relatório de Incidente — Falha 6: nome não resolve

**Ambiente:** cliente Windows 10/11 (host Hyper-V) × Ubuntu Server LTS (VM `lab-linux`, `192.168.100.10`)
**Data:** 09/11/2026
**Alvo:** `intranet.test` (TLD `.test`, reservado por RFC 2606)

## 1. Resumo

O site deixou de abrir **pelo nome**, mas continuou acessível **pelo IP**. A causa não era o servidor: a linha correspondente no arquivo `hosts` do Windows estava comentada, e `intranet.test` não existe no DNS público (NXDOMAIN). Restaurada a linha e limpo o cache, a resolução voltou. O incidente separou três camadas: nome → endereço → serviço.

## 2. Sintomas

**Pelo nome (falha):**

```powershell
PS> ping intranet.test
A solicitação ping não pôde encontrar o host intranet.test.

PS> curl.exe -I http://intranet.test
curl: (6) Could not resolve host: intranet.test
```

**Pelo IP (servidor saudável):**

```powershell
PS> curl.exe -I http://192.168.100.10
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Content-Length: 803
```

No navegador: `ERR_NAME_NOT_RESOLVED` — erro ocorre **antes** de qualquer pacote TCP ser enviado ao servidor.

## 3. Diagnóstico

**Estado do `hosts`:**

```powershell
PS> Get-Content C:\Windows\System32\drivers\etc\hosts
127.0.0.1 localhost
::1 localhost

#192.168.100.10 intranet.test   www.intranet.test     ← comentado
```

**Consulta ao DNS — prova de que `nslookup` ignora o `hosts`:**

```powershell
PS> nslookup intranet.test
Servidor:  <DNS da rede corporativa — omitido>
Address:  <omitido>
*** servidor DNS não encontrou intranet.test: Non-existent domain
```

`.test` é TLD reservado (RFC 2606): o DNS público **nunca** responderá por ele — evidência limpa de NXDOMAIN.

**Cache local:**

```powershell
PS> ipconfig /displaydns | Select-String intranet
# (vazio — nada em cache)
```

**Variante "resolve para o lugar errado" (IP inválido no `hosts`):**

```powershell
PS> ping intranet.test
Disparando intranet.test [10.0.0.99] ... Esgotado o tempo limite do pedido (100% de perda)

PS> curl.exe -I http://intranet.test
curl: (28) Failed to connect ... after 21060 ms

PS> curl.exe -v http://intranet.test
* Host intranet.test:80 was resolved.
* IPv4: 10.0.0.99
*   Trying 10.0.0.99:80...
```

**Achado de campo (typo):**

```powershell
PS> curl.exe -I http://instranet.test     ← falta o 't'
curl: (6) Could not resolve host: instranet.test
```

Erro de digitação no nome produz **o mesmo sintoma** de falha de DNS — distinguir os dois exige conferir o nome exato usado.

## 4. Causa raiz

A entrada `192.168.100.10 intranet.test` estava **comentada** no `hosts`. Sem ela, o Windows consulta o DNS da rede, que responde NXDOMAIN para o TLD `.test` (inexistente na internet). Sem resolução, não há conexão: o nginx nunca chega a ser contactado.

## 5. Solução

Restaurada a linha no `hosts` (editor com privilégio de administrador):

```
192.168.100.10   intranet.test   www.intranet.test
```

```powershell
PS> ipconfig /flushdns
PS> ping intranet.test
Disparando intranet.test [192.168.100.10] ... 0% de perda

PS> curl.exe -I http://intranet.test
HTTP/1.1 200 OK
```

## 6. Prevenção / aprendizados

- **Ordem de resolução:** `hosts` é consultado **antes** do DNS. Por isso o furo público de `meusite.com.br` não afetava o lab enquanto a linha estava ativa — e por isso "quebrou só o nome".
- **`nslookup` ignora o `hosts`**: ele fala direto com o servidor DNS. Para validar o que a máquina realmente usa, `ping`, `curl` ou `Resolve-DnsName`; para validar o que o DNS diz, `nslookup`. Confundir os dois é erro clássico de diagnóstico.
- **Dois modos de falha de nome:** *não resolve* (erro imediato, `curl (6)`, NXDOMAIN) × *resolve errado* (timeout ~21s, `curl (28)`, "Trying <IP errado>"). O segundo engana mais — a máquina "acha que sabe".
- **Cache DNS:** `ipconfig /flushdns` após qualquer alteração no `hosts`; `ipconfig /displaydns` para inspecionar.
- **Domínio de laboratório:** usar TLD reservado (`.test`, `.local`, `.invalid`) evita colisão com nomes públicos.
- **Erro de digitação (typo) simula falha de DNS:** conferir o nome exato antes de investigar infraestrutura.
