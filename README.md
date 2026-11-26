# mini-quic

Protocolo de **transferência confiável de arquivos sobre UDP**, inspirado no QUIC.
Trabalho Prático da disciplina **CSI510 – Redes de Computadores II (UFOP)**.

> **Status:** planejamento concluído (design + plano de implementação). A
> implementação do código ainda não começou.

## A ideia

O UDP não garante entrega, ordem nem integridade. Este projeto constrói, **por
cima do UDP**, um protocolo próprio que oferece essas garantias — do mesmo jeito
que o QUIC faz — e depois **mede experimentalmente** como atraso (RTT), perda de
pacotes, tamanho de janela e posição do servidor afetam a transferência.

O que o sistema faz:

- Cliente e servidor trocam arquivos por UDP, com **autenticação por senha**,
  **lista de arquivos**, **download e upload**.
- Confiabilidade com **cabeçalho próprio**, **janela deslizante Selective
  Repeat**, retransmissão por timeout e **verificação de integridade por
  SHA-256**.
- **Mini-CDN:** dois servidores em máquinas diferentes; o cliente mede o RTT e
  escolhe o mais próximo (como o mapeamento de uma CDN real).
- **Experimentos** nos 4 cenários exigidos, com gráficos de RTT, throughput,
  perda e retransmissões.
- **Bônus:** handshake rápido (1-RTT), migração de conexão, dashboard web e
  deploy em nuvem.

## Antes de clonar / começar, leia (nesta ordem)

1. **`arquivos/TP - RC2.pdf`** — o enunciado do trabalho (a fonte de todos os
   requisitos). O repositório inclui também, em `arquivos/`, **todo o material da
   matéria** (livro, artigos e slides sobre TCP, QUIC, CDNs e desempenho de
   redes), versionado junto para dar contexto a quem for desenvolver.
2. **`docs/superpowers/specs/2026-11-26-mini-quic-design.md`** — o documento de
   **design**: cabeçalho do protocolo, arquitetura em camadas, Selective Repeat,
   handshake, migração e as decisões já tomadas. **É a fonte de verdade.**
3. **`docs/superpowers/plans/2026-11-26-mini-quic.md`** — o **plano de
   implementação**: 10 tarefas, cada uma com os testes que a provam.

Quem for programar deve ler o design inteiro antes de escrever qualquer código —
o cabeçalho, os tipos de mensagem e as camadas já estão definidos lá.

## Estrutura do repositório

```
mini-quic/
├── arquivos/        # enunciado + material da matéria (PDFs)
├── docs/
│   └── superpowers/
│       ├── specs/   # documento de design
│       └── plans/   # plano de implementação
├── mini_quic/       # (a criar) código do protocolo, cliente e servidor
├── experiments/     # (a criar) harness dos cenários e gráficos
└── tests/           # (a criar) testes automatizados
```

## Trabalhando com IA (recomendação: crie um `CLAUDE.md`)

Permitido usar IA generativa como apoio, **mas todos do grupo precisam entender a
implementação** — na apresentação não haverá acesso a IA (ver FAQ do enunciado).

Para manter a IA alinhada ao projeto, recomendamos criar um arquivo **`CLAUDE.md`**
na raiz (não versionado, por ser configuração local de cada um). Ele deve orientar
o assistente a:

- **Ler sempre** o documento de design (`docs/superpowers/specs/...`) no início de
  cada conversa, tratando-o como fonte de verdade.

Exemplo mínimo de `CLAUDE.md`:

```markdown
# CLAUDE.md
- No início de toda conversa, leia docs/superpowers/specs/2026-11-26-mini-quic-design.md.
```

## Execução

Instruções de instalação, configuração e execução (servidor, cliente,
experimentos e dashboard) serão adicionadas conforme o código for implementado
(Tarefa 10 do plano).

> **Nota:** este README é de apresentação/onboarding. **Quando o trabalho estiver
> finalizado**, ele será substituído pelo README exigido pelo professor —
> instruções de compilação, configuração e execução do sistema (enunciado §9).
