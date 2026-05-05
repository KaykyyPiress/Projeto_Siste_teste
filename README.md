# Projeto de Sistemas Distribuídos — Chat com Python + Java

## Introdução da Parte 4

Na Parte 4, o foco foi resolver os erros de inconsistência que apareciam quando o
broker distribuía requisições para servidores diferentes (Python e Java), como o
clássico `Canal ... nao existe`.

### O que causava o erro

Antes, cada servidor mantinha parte do estado localmente (principalmente canais).
Então podia acontecer:

1. `create_channel` cair no servidor A;
2. `publish_message` cair no servidor B;
3. servidor B não conhecer o canal criado em A.

Resultado: falhas intermitentes mesmo com sistema aparentemente "no ar".

### Como foi feito para evitar esses erros

Foi implementada sincronização entre servidores no tópico `servers.state` com dois
mecanismos complementares:

- **Eventos incrementais** (`channel_created`, depois também `login_created` e
  `publication_created`): rápida propagação logo após cada operação.
- **Snapshot periódico** (`channels_snapshot` / `state_snapshot`): reconciliação
  para convergir estado mesmo em restart, atraso de subscribe ou perda de evento.

Além disso:

- foi adicionada **eleição de coordenador por rank**;
- escritas passaram a poder ser **encaminhadas ao coordenador** (`client_request`),
  seguindo a ideia de permissão centralizada;
- o `reference` ficou responsável por rank, liveness e metadados de peer
  (`host`, `peer_port`).

Com isso, os servidores Python e Java convergem para o mesmo estado com muito
menos erro intermitente.

---

## Resumo do Projeto

Este projeto implementa um chat distribuído com múltiplos serviços em Docker,
usando duas linguagens (Python e Java) para clientes e servidores.

### Componentes principais

- **Broker (REQ/REP)**: encaminha chamadas de clientes para servidores.
- **Proxy Pub/Sub (XSUB/XPUB)**: distribui mensagens publicadas nos canais.
- **Reference service**: mantém servidores ativos, rank e dados para descoberta
  entre peers.
- **Servidores Python e Java**: processam login, canais e publicações.
- **Clientes Python e Java (bots)**: executam fluxo de login/lista/criação/publicação.

### Funcionalidades implementadas ao longo das partes

- relógio lógico de Lamport em mensagens;
- referência com heartbeat e ranking de servidores;
- eleição de coordenador e sincronização entre peers;
- replicação de estado entre servidores por eventos + snapshots;
- interoperabilidade Python ↔ Java no mesmo cluster.

### Execução

Na pasta `request-reply2/src/broker`:

```bash
docker compose up --build
```

Serviços iniciados:

- `broker`
- `pubsub-proxy`
- `reference`
- `servidor` (Python)
- `cliente` (Python)
- `servidor-java`
- `cliente-java`
