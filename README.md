# Systella Connector

Agente do [Systella ERP](https://systella.com.br) para o servidor de arquivos da empresa.

Roda como container Docker no próprio servidor (Linux), lê as pastas liberadas **somente leitura** e fala com o Systella por uma única conexão de **saída** (WSS, porta 443). Não abre porta de entrada, não copia arquivos para a nuvem e não altera nada nas pastas: o Systella guarda só a referência e os metadados (nome, caminho, tamanho, data, hash), e um download pela web passa em streaming, sob demanda, só para quem tem permissão no ERP.

```
ghcr.io/edusalvador/systella-connector:<versão>
```

Arquiteturas: `linux/amd64` e `linux/arm64`.

## Requisitos

1. Linux com Docker Engine 24+ e Docker Compose v2.
2. A pasta compartilhada em **disco local** do servidor (a mesma que o Samba compartilha), não uma montagem de rede.
3. Saída liberada para `connector.systella.com.br:443` (proxy corporativo é suportado).
4. Um usuário/grupo do sistema com leitura na pasta (ex.: o grupo do Samba).
5. 1 GB de RAM e 1 vCPU disponíveis.

## Instalação

1. No Systella, em **TI › Armazenamento › Systella Connector**, clique em **Instalar Connector**. A tela mostra o código de instalação (uso único) e o `docker-compose.yml` pronto.
2. No servidor, numa pasta nova:
   - salve o código em `pairing_code.txt`;
   - salve o `docker-compose.yml` e ajuste o caminho da pasta em `volumes` e o `PUID`/`PGID`.
3. Suba:

   ```sh
   docker compose up -d
   docker compose logs -f
   ```

   O log mostra `paired`, `connected to relay` e `scan complete`.
4. De volta à tela de TI, o Connector aparece como **Conectado**. Ligue cada pasta à pasta que os usuários veem (ex.: `propostas` → `S:\031-Propostas`).

Depois do pareamento o código não é mais usado; a credencial fica no volume `connector-state`.

## Exemplo de compose

```yaml
services:
  systella-connector:
    image: ghcr.io/edusalvador/systella-connector:1.0.1
    restart: unless-stopped
    user: "${PUID:-1000}:${PGID:-1000}"
    read_only: true
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    pids_limit: 256
    cpus: 1.0
    mem_limit: 1g
    tmpfs:
      - /tmp:size=256m,noexec,nosuid,nodev
    environment:
      SYSTELLA_RELAY_URL: wss://connector.systella.com.br
      SYSTELLA_PAIRING_CODE_FILE: /run/secrets/pairing_code
      SYSTELLA_ROOTS: propostas=/data/propostas
    secrets: [pairing_code]
    volumes:
      - /caminho/no/servidor/031-Propostas:/data/propostas:ro
      - connector-state:/var/lib/systella-connector
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "5" }
volumes:
  connector-state:
secrets:
  pairing_code:
    file: ./pairing_code.txt
```

## Variáveis de ambiente

| Variável | Padrão | Uso |
|---|---|---|
| `SYSTELLA_ROOTS` | — (obrigatória) | `chave=/caminho/no/container`, separados por vírgula |
| `SYSTELLA_PAIRING_CODE_FILE` | — | Arquivo com o código de instalação (só no primeiro uso) |
| `SYSTELLA_RELAY_URL` | `wss://connector.systella.com.br` | Endereço do Systella |
| `SYSTELLA_WATCH` | ano atual e anterior | Subpastas com aviso de mudança em tempo real (`chave=subpasta,…`; `none` desliga) |
| `SYSTELLA_SCAN_INTERVAL_MIN` | `1440` | Intervalo da varredura completa |
| `HTTPS_PROXY` | — | Proxy corporativo para a saída |
| `NODE_EXTRA_CA_CERTS` | — | Certificado da CA do proxy, se ele inspecionar TLS |

## Atualizar

Troque a versão da imagem no `docker-compose.yml` e rode:

```sh
docker compose pull && docker compose up -d
```

## Segurança

- Pastas montadas `:ro`; o container roda sem privilégios (`read_only`, `cap_drop: ALL`, `no-new-privileges`, usuário comum).
- Só conexão de saída; nada escuta na rede do servidor.
- Caminhos fora das pastas liberadas e links simbólicos que saem delas são recusados.
- Cada download usa um link de uso único de 60 segundos, emitido só depois de o Systella conferir a permissão do usuário.
- Para desligar: **Desligar** na tela de TI (revoga a credencial) e `docker compose down -v` no servidor.

## Problemas comuns

| Log | O que fazer |
|---|---|
| `not paired` | Falta o código em `pairing_code.txt` |
| `relay refused connection` | Código vencido ou já usado, ou Connector desligado: gere um novo na tela de TI, troque o arquivo e rode `docker compose up -d --force-recreate` |
| `relay connection error` | Verifique a saída para `connector.systella.com.br:443` e o proxy |
| `relay silent, reconnecting` | Normal após atualização do Systella; reconecta sozinho |

Suporte: pela Central de Feedback dentro do Systella.
