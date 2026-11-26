# Documento de Design — mini-quic (TP CSI510 / Redes de Computadores II)

- **Data:** 2026-11-26
- **Entrega:** até 15/12/2026 (apresentação na sala A409)
- **Valor:** 22 pts + até 10 pts de bônus
- **Linguagem/SO:** Python 3 / Linux
- **Repositório:** `mini-quic` (GitHub: matheusmnn/mini-quic)

## 1. Intenção e o "porquê"

Construir um **protocolo de transferência confiável de arquivos sobre UDP**,
inspirado no QUIC, e **avaliar experimentalmente** como atraso (RTT), perda de
pacotes, tamanho de janela e posição do servidor afetam a distribuição de
conteúdo. É um trabalho acadêmico avaliado: a nota vem tanto do sistema
funcionando quanto de um **relatório (modelo SBC)** com experimentos e análise.

Restrição que molda todo o design: **todos os integrantes precisam entender a
implementação inteira**, pois serão arguidos na apresentação sem acesso a IA.
Portanto, preferimos sempre a solução **clara e legível** à "esperta".

## 2. Objetivos (goals)

1. Cliente-servidor UDP com: lista de arquivos, **download e upload**,
   autenticação por senha, tratamento de erros e de dados corrompidos.
2. Confiabilidade sobre UDP: cabeçalho próprio documentado, **janela deslizante
   Selective Repeat**, timeout/retransmissão, detecção de perda, entrega em ordem.
3. Integridade por **SHA-256** (relatório mostra hash original × recebido).
4. **Experimentos** nos 4 cenários exigidos, medindo RTT, throughput, taxa de
   perda e nº de retransmissões, variando RTT, perda **e tamanho de janela**,
   com arquivo **≥ 80 MB** e resultados em **gráficos**.
5. **Mini-CDN:** duas instâncias de servidor; o cliente mede RTT e escolhe a de
   menor atraso; o relatório responde as 3 perguntas do enunciado.
6. **Capturas Wireshark:** ACKs, retransmissões e seleção de servidor.
7. **Bônus (4):** handshake rápido 1-RTT, migração de conexão, dashboard web,
   deploy em nuvem (comparação local × nuvem).
8. Entregáveis: código completo, **README** (compilação/config/execução),
   relatório SBC.

## 3. Não-objetivos (non-goals)

- Reimplementar QUIC de verdade (TLS 1.3, HTTP/3, 0-RTT com tickets, crypto real).
- **Controle de congestionamento** (AIMD/CUBIC). A janela é de **controle de
  fluxo**, com tamanho **configurável** (variável de experimento). Mencionar como
  "melhoria futura" no relatório.
- **Criptografia ponta a ponta** — descartada por ser difícil de defender com
  segurança real na arguição.
- Multiplexação de fluxos — o cabeçalho a suporta (campo `stream_id`), mas só
  implementamos de fato "se sobrar tempo"; não é compromisso.
- Suporte a múltiplos clientes simultâneos no servidor **não é requisito**;
  atender um cliente por vez já cumpre o enunciado (ver §12, decisão do builder).

## 4. Anti-objetivos (o que seria um fracasso mesmo "funcionando")

- Código que algum integrante não consiga explicar na apresentação → risco de
  zerar (FAQ do enunciado).
- Experimentos **não reprodutíveis** (sem seed, sem parâmetros registrados) que
  não sustentem os gráficos do relatório.
- Over-engineering: abstrações, camadas ou configs que ninguém pediu.

## 5. Restrições

- Arquivo de teste **≥ 80 MB**.
- Relatório no **modelo SBC** (template oficial da SBC, LaTeX ou Word).
- Prazo **15/12/2026**; apresentação presencial.
- Mini-CDN exige **duas máquinas diferentes** (infra ainda a definir pelo grupo —
  ver §11). O design roda inteiro em localhost com atraso/perda artificiais e é
  **portável** para 2ª máquina/nuvem sem mudança de código.
- Só bibliotecas de fácil defesa: padrão (`socket`, `struct`, `hashlib`, `zlib`,
  `threading`, `json`, `argparse`), `matplotlib` para gráficos e `Flask` para o
  dashboard. Nada de framework de rede pronto.

## 6. Abordagens consideradas

**Opção A — Go-Back-N (descartada).** ACK cumulativo, timer único, reenvia tudo a
partir do perdido. Mais simples de entender, porém retransmite demais sob 5% de
perda (cenários 3 e 4) → throughput pior, e **não** oferece caminho barato para
os bônus QUIC.

**Opção B — Selective Repeat + Connection ID (ESCOLHIDA).** ACK seletivo
(bitmap/ranges), reenvia só o que faltou → melhor desempenho sob perda (ganha os
pontos de desempenho). Um campo **Connection ID** no cabeçalho desde o início —
ideia central do QUIC — destrava **quase de graça** dois bônus: **migração de
conexão** (a sessão é identificada pelo connID, não pelo par IP:porta) e
**multiplexação** (via `stream_id`). Mais código que A, mas 100% defensável em
Python.

**Decisão:** Opção B. Como o grupo quer puxar vários bônus, é a abordagem em que
o mesmo cabeçalho entrega desempenho **e** 2 features QUIC.

## 7. Arquitetura e estrutura de arquivos

```
mini-quic/
  mini_quic/
    __init__.py
    protocol.py      # cabeçalho (struct), tipos de msg, (de)serialização, checksum, helpers SHA-256
    netsim.py        # wrapper do socket UDP com perda e atraso artificiais (determinístico por seed)
    reliability.py   # Selective Repeat: sender (janela, timers, retransmissão) e receiver (buffer, ACK bitmap)
    connection.py    # estado da conexão, Connection ID, handshake 1-RTT, migração
    server.py        # loop do servidor: auth, LIST, DOWNLOAD, UPLOAD
    client.py        # CLI do cliente: conecta, lista, baixa, envia, mede RTT (mini-CDN)
    metrics.py       # coleta de métricas e escrita periódica em metrics.json
    dashboard/
      app.py         # Flask: serve página + endpoint /metrics (lê metrics.json)
      templates/index.html
  files/             # arquivos servidos (inclui gerador de arquivo de 80 MB)
  experiments/
    run_scenarios.py # executa os 4 cenários (+ varredura de janela), grava CSV
    plot.py          # gera os gráficos (matplotlib) a partir do CSV
  tests/
    test_protocol.py # round-trip de cabeçalho, checksum, SHA-256
    test_reliability.py # SR entrega correto sob perda simulada (seed fixa)
    test_transfer.py # integração: transfere arquivo em localhost e confere hash
  docs/
    report/          # relatório SBC
    superpowers/specs/2026-11-26-mini-quic-design.md
  README.md
  requirements.txt
```

**Camadas (de baixo para cima):** `netsim` (socket + perda/atraso) → `reliability`
(SR) → `connection` (sessão/connID/handshake) → `server`/`client` (aplicação).
Cada camada só conhece a de baixo.

## 8. Protocolo

### 8.1 Cabeçalho (28 bytes, fixo, big-endian)

Formato `struct`: `!QBBHIIHHI`

| Campo        | Tipo (bytes) | Uso |
|--------------|--------------|-----|
| `conn_id`    | uint64 (8)   | ID da conexão; identifica a sessão (permite migração) |
| `msg_type`   | uint8 (1)    | tipo da mensagem (§8.2) |
| `flags`      | uint8 (1)    | reservado (ex.: FIN, bit de "é retransmissão") |
| `stream_id`  | uint16 (2)   | fluxo; 0 = controle (base para multiplexação) |
| `seq`        | uint32 (4)   | nº de sequência do pacote de dados |
| `ack`        | uint32 (4)   | próximo seq esperado (base cumulativa) |
| `window`     | uint16 (2)   | janela anunciada (em pacotes) |
| `length`     | uint16 (2)   | tamanho do payload em bytes |
| `checksum`   | uint32 (4)   | CRC32 (`zlib.crc32`) de cabeçalho(checksum=0)+payload |

Payload de DATA: até **1024 bytes** (fica confortável abaixo do MTU com o
cabeçalho; valor configurável). Payload de ACK: bitmap de pacotes recebidos acima
de `ack` (SACK), permitindo Selective Repeat.

### 8.2 Tipos de mensagem

| Tipo | Direção | Descrição |
|------|---------|-----------|
| `HELLO`      | C→S | handshake: cliente propõe `conn_id` aleatório + senha |
| `HELLO_ACK`  | S→C | aceita (auth ok) ou recusa; confirma `conn_id` |
| `LIST_REQ`   | C→S | pede lista de arquivos |
| `LIST_RESP`  | S→C | lista (nome, tamanho) |
| `DL_REQ`     | C→S | pede download de um arquivo |
| `FILE_META`  | S→C | tamanho, SHA-256, total de pacotes |
| `UL_REQ`     | C→S | anuncia upload (nome, tamanho, SHA-256) |
| `DATA`       | ↔   | bloco de dados (usa `seq`, `stream_id`) |
| `ACK`        | ↔   | `ack` cumulativo + bitmap SACK no payload |
| `FIN`        | ↔   | fim da transferência |
| `FIN_ACK`    | ↔   | confirma fim |
| `ERROR`      | S→C | código + texto (auth falhou, arquivo inexistente, hash divergente) |

### 8.3 Handshake 1-RTT (bônus: estabelecimento rápido)

Cliente gera `conn_id` aleatório e envia `HELLO{conn_id, senha}`. Servidor valida
a senha; se ok, responde `HELLO_ACK`. **Uma ida e volta** estabelece conexão +
autenticação (contraste com TCP+TLS que leva 2–3 RTTs). Justificativa técnica no
relatório: mapeia o "estabelecimento rápido de conexão" do QUIC.

### 8.4 Selective Repeat

- **Sender:** janela de `W` pacotes não-confirmados; cada um com seu timer
  (ou um timer varrendo a janela). Timeout de um pacote → reenvia **só ele**.
  Ao receber ACK, desliza a base e marca confirmados pelo bitmap.
- **Receiver:** aceita pacotes dentro da janela, **bufferiza fora de ordem**,
  entrega em ordem à aplicação, responde `ACK{ack=próximo_esperado, bitmap}`.
- **RTO:** timeout adaptativo estilo TCP (EWMA de RTT + variância; `SRTT`,
  `RTTVAR`, `RTO = SRTT + 4·RTTVAR`), com piso e teto. Simplificação aceitável:
  RTO fixo configurável — registrar a escolha no relatório.
- **Posse da janela `W`:** `W` do *sender* é definido por **config** (é a variável
  de experimento — 1/4/16/64). O *receiver* anuncia a capacidade do seu buffer no
  campo `window` do cabeçalho; o sender usa `min(W_config, window_anunciada)`. O
  buffer de reordenação do receiver é dimensionado pela janela que ele anuncia
  (em SR, janela do receiver ≥ janela do sender). Assim o campo `window` é
  **funcional** (controle de fluxo real), não decorativo — isso sustenta a
  moldura "janela = controle de fluxo" vs. congestionamento como não-objetivo.

### 8.5 Integridade

Remetente calcula SHA-256 do arquivo e envia em `FILE_META`/`UL_REQ`. Receptor
recalcula ao final e compara. Divergência → `ERROR` + descarte. Relatório mostra
os dois hashes.

### 8.6 Tratamento de erros

- Checksum CRC32 inválido → pacote descartado (conta como perda, dispara
  retransmissão).
- Timeout sem resposta ao `HELLO`/em transferência → nº de tentativas, depois
  mensagem de erro clara e saída limpa.
- Auth incorreta, arquivo inexistente, hash final divergente → `ERROR` tipado.
- Exceções de socket tratadas (sem stack trace cru para o usuário).

### 8.7 Modelo de execução (call do builder — importante)

Para que a janela fique **de fato cheia** (W pacotes em voo) e os experimentos de
desempenho façam sentido, o atraso **não pode** ser um `sleep` bloqueante na hora
do envio, e o envio não pode travar esperando ACK. Decisão:

- **`netsim` entrega de forma assíncrona:** ao "enviar", agenda a entrega do
  datagrama para daqui a `RTT/2` numa **thread de agendamento** (fila ordenada por
  horário de entrega) e retorna na hora. Nada bloqueia. A perda é decidida no
  agendamento (descarta com prob. `p`). Seed fixa → reprodutível.
- **Reliability com 2 threads por transferência:** uma thread de **envio**
  (preenche a janela até W, mantém os timers, retransmite por timeout) e uma
  thread de **recepção** (lê ACKs/bitmap, desliza a base). Estado da janela
  protegido por um `Lock`. Um cliente por vez (§12) mantém isso simples.
- **Alternativa equivalente:** socket não-bloqueante + laço `select` único. Fica
  como opção; o modelo de 2 threads é o escolhido por ser mais fácil de explicar
  na arguição.

### 8.8 Confiabilidade das mensagens de controle (call do builder)

DATA roda sobre o SR. As mensagens de **controle** (`HELLO`, `LIST_REQ/RESP`,
`DL_REQ`, `FILE_META`, `UL_REQ`, `FIN`) usam um auxiliar **stop-and-wait com
retransmissão** em `connection.py`: envia a requisição, espera a resposta
esperada, e em timeout reenvia até N vezes antes de erro. Sem isso, nos cenários
3 e 4 (5% de perda) um `FILE_META` ou `LIST_RESP` perdido travaria a
transferência antes de qualquer dado fluir. Mantém o SR focado só no volume de
dados (fronteira de camadas limpa).

## 9. Simulação de rede (netsim) e cenários

`netsim.py` embrulha `sendto`/`recvfrom`:
- **Perda:** descarta cada pacote com probabilidade `p` (RNG com **seed fixa** →
  reprodutível).
- **Atraso:** aplica `RTT/2` de atraso em cada sentido (via timer/sleep).
- Parâmetros vêm de CLI/env, registrados no CSV de cada corrida.

Os 4 cenários (do enunciado), rodados em localhost:

| Cenário | RTT    | Perda |
|---------|--------|-------|
| 1       | 25 ms  | 0%    |
| 2       | 100 ms | 0%    |
| 3       | 100 ms | 5%    |
| 4       | 400 ms | 5%    |

Para cada cenário medir **RTT, throughput, taxa de perda, nº de retransmissões**;
além disso, uma **varredura de tamanho de janela** (ex.: W = 1, 4, 16, 64) para
mostrar o impacto da janela. Evidência de rede real adicional: comandos
`tc netem` documentados no README para gerar capturas Wireshark fora do netsim.

## 10. Mini-CDN

Dois servidores idênticos em máquinas diferentes. Antes do download, o cliente
envia um `HELLO`/ping leve a cada um, mede o RTT e escolhe o de menor atraso.
Relatório responde: qual foi escolhido, impacto no desempenho, e relação com o
mapeamento de CDNs (redirecionamento por proximidade — DNS/anycast/latência).

## 11. Bônus (detalhamento)

1. **Handshake 1-RTT** — §8.3.
2. **Migração de conexão** — como a sessão é indexada por `conn_id` (não por
   IP:porta), o cliente pode trocar de porta/origem no meio da transferência e o
   servidor continua a sessão. Demo: rebind do socket do cliente a meio download;
   evidência no Wireshark (mesma `conn_id`, novo IP:porta).
3. **Dashboard web (Flask)** — `metrics.py` escreve `metrics.json`
   periodicamente; `dashboard/app.py` serve uma página que faz *polling* e mostra
   RTT, throughput, perdas, retransmissões e **janela atual** ao vivo.
4. **Deploy em nuvem** — sobe o 2º servidor numa VM free-tier (AWS/Oracle). De
   brinde resolve a infra do mini-CDN **e** o bônus de nuvem; relatório compara
   local × nuvem. (Depende da decisão de infra, §12.)

## 12. Decisões deixadas ao builder (calls do implementador)

- **Um cliente por vez no servidor.** O enunciado não exige concorrência; o
  servidor atende uma sessão por vez (mais simples de entender/defender). Se
  surgir necessidade, `threading` por `conn_id` é extensão natural — não agora.
- **RTO:** começar com RTO **fixo configurável**; subir para adaptativo
  (EWMA) só se o tempo permitir. Registrar o que foi usado no relatório.
- **Tamanho do payload:** 1024 bytes (ajustável por constante).
- **Formato do arquivo de 80 MB:** gerado por script (`files/make_testfile.py`),
  não versionado no git (entra no `.gitignore`).
- **Linguagem do relatório/README:** Português.
- **Métricas do relatório:** *throughput* = goodput (bytes do arquivo ÷ tempo de
  parede); *taxa de perda* = pacotes descartados ÷ enviados (medida), reportada ao
  lado do `p` configurado; *RTT* por EWMA dos pares DATA/ACK; *retransmissões* =
  contador do sender.
- **Autenticação:** uma única senha do servidor, lida de config/variável de
  ambiente (não há cadastro de usuários).

## 13. Plano de testes

- `test_protocol.py`: round-trip de cabeçalho (serialize→parse), checksum detecta
  corrupção, SHA-256 confere.
- `test_reliability.py`: com `netsim` de perda determinística (seed fixa), o SR
  entrega o fluxo completo e em ordem; conta retransmissões esperadas.
- `test_transfer.py`: transfere um arquivo pequeno em localhost e confere que o
  hash recebido == hash original (e idem para upload).
- Validação final: transferir o arquivo de 80 MB nos 4 cenários sem corromper.

## 14. Entregáveis

- Código completo no repositório.
- `README.md`: dependências, como configurar (senha, portas, cenário), como rodar
  servidor/cliente/experimentos/dashboard, e comandos `tc netem`.
- Relatório SBC: cabeçalho do protocolo documentado, metodologia, gráficos dos 4
  cenários + varredura de janela, hashes, respostas do mini-CDN, justificativa
  técnica dos bônus, capturas Wireshark, limitações e melhorias futuras.

## 15. Divisão sugerida (grupo de 3)

- **Pessoa A:** `protocol.py` + `netsim.py` + testes de protocolo.
- **Pessoa B:** `reliability.py` + `connection.py` (SR, handshake, migração).
- **Pessoa C:** `server.py` + `client.py` + `experiments/` + dashboard.
- Relatório e capturas Wireshark: os três (cada um documenta sua parte →
  garante que todos entendam tudo).
