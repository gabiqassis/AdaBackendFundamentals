# Projeto Final PE-JV-003 — System Design

Entrega do projeto final do módulo **PE-JV-003 (Backend Fundamentos, Ada)**: cinco documentos de System Design, um por aula (02 a 06), cada um respondendo a um cenário de incidente de um marketplace na Black Friday.

**Autora:** Gabriela Queiroz Assis

---

## Sobre este repositório

Cada aula propôs um desafio de arquitetura com um incidente próprio. Em vez de responder com teoria genérica, cada documento parte do cenário do template, declara o que é dado e o que é premissa, defende as escolhas com trade-offs explícitos e diz o que foi descartado e por quê.

Um ponto importante de leitura: **os cinco documentos são independentes.** Várias aulas falam de "Checkout" e "Black Friday", mas isso é vocabulário compartilhado do curso. Cada documento se sustenta sozinho, e a justificativa de qualquer decisão é reconstruída a partir do cenário daquela aula, sem depender de outro arquivo.

## Os documentos

| Aula | Tema | Incidente do cenário | Documento |
| :--- | :--- | :--- | :--- |
| 02 | APIs Resilientes | Integração do Checkout com pagamento externo instável, 10.000 usuários simultâneos | `src/main/java/br/com/ada/doc/aula2/PE-JV-003-Aula-02-respondido.md` |
| 03 | Banco de Dados em Larga Escala | p99 de 4,8 s e pool HikariCP saturado por consultas lentas de pedidos | [`PE-JV-003-Aula-03-respondido.md`](src/main/java/br/com/ada/doc/aula3/PE-JV-003-Aula-03-respondido.md) |
| 04 | Microsserviços & Padrão Saga | Pix de R$ 3.000 debitado, estoque esgotado no milissegundo seguinte | [`PE-JV-003-Aula-04-respondido.md`](src/main/java/br/com/ada/doc/aula4/PE-JV-003-Aula-04-respondido.md) |
| 05 | Containers, CI/CD & Deploy sem Downtime | `latest` sobrescrito, Flyway renomeou coluna e o rollback ficou impossível | [`PE-JV-003-Aula-05-respondido.md`](src/main/java/br/com/ada/doc/aula5/PE-JV-003-Aula-05-respondido.md) |
| 06 | Observabilidade & Diagnóstico em Produção | Canary com painéis verdes e conversão despencando | [`PE-JV-003-Aula-06-respondido.md`](src/main/java/br/com/ada/doc/aula6/PE-JV-003-Aula-06-respondido.md) |

## Como cada documento está organizado

Todos seguem a mesma espinha, o que facilita a leitura em sequência:

1. **Recorte e premissas** — o que está sendo desenhado, o que o cenário dá como dado e o que foi assumido (com o motivo de cada premissa).
2. **Respostas aos itens do template** — decisão explícita, alternativas descartadas e trade-off assumido.
3. **Resumo de decisões e trade-offs** — tabela final com decisão, custo aceito e alternativa rejeitada.
4. **Apêndices** — referências consultadas, comparação com a resposta mais comum, perguntas difíceis para arguição e sequência de commits sugerida.

## Resumo das decisões por aula

### Aula 02 — APIs Resilientes
Paradigma de API, combinação de padrões de resiliência na chamada ao parceiro (Circuit Breaker, TimeLimiter, RateLimiter, Bulkhead, Retry), ciclo de vida da `Idempotency-Key`, três edge cases autorais e estratégia de teste do Circuit Breaker e do timeout.

### Aula 03 — Banco de Dados em Larga Escala
- **Persistência poliglota**, com uma regra de ouro: o dinheiro mora no PostgreSQL e em nenhum outro lugar. Redis e motor de busca na fase 1; colunar e grafo na fase 2.
- **Chave primária UUIDv7** gerada na aplicação, coluna nativa `uuid`, por causa de *page splits*, WAL e localidade de cache no B-Tree.
- **Lock otimista** no status de pagamento, com transição condicionada ao estado e idempotência por id de evento.
- **`UPDATE` condicional atômico** no estoque, com `CHECK` no esquema, portão de admissão para SKUs quentes e ordenação de SKUs contra deadlock.
- **Diagnóstico** em 8 passos, com hipótese escrita antes de qualquer mudança, e **reprovação do PR de índices por coluna** (indexar `order_status` mata o HOT update).

### Aula 04 — Microsserviços & Padrão Saga
- **Saga com orquestração**, com o orquestrador como componente do Checkout (não serviço novo) e coreografia para as consequências do pedido.
- **Inversão da ordem dos passos**: reserva de estoque antes da cobrança, com o pagamento como pivô.
- **Máquina de estados** de 12 estados, com prazo em todo estado intermediário.
- **Compensação como obrigação em livro**, com chave determinística, classificação de erro, alçada humana e conciliação.
- **Outbox transacional** com CDC (Debezium) e **CQRS** com projeção versionada para "Meus Pedidos".
- Testes centrados em falha, repetição e invariante, com auditor de invariantes que serve de teste e de alarme.

### Aula 05 — Containers, CI/CD & Deploy sem Downtime
- Autópsia do incidente em **seis causas independentes**.
- Imagem multi-stage em **distroless** com UID numérico, deploy por **digest**, tag imutável e `latest` negado na admissão.
- **Build único** e promoção do mesmo digest, com assinatura, provenance e atestação verificadas na admissão.
- **Canary** (Argo Rollouts) com análise comparativa canary-vs-estável. Blue-green foi rejeitado porque, com banco compartilhado, o isolamento que o justifica não existe.
- **Expand/contract em três releases** para `status_pagamento` → `estado_saga`, com trigger bidirecional, backfill em lotes e `lock_timeout`.
- Rollback por abort de rollout, **sem nenhuma operação de banco**, e matriz de coexistência N-1 como gate de PR.

### Aula 06 — Observabilidade & Diagnóstico em Produção
- Autópsia das **oito cegueiras** que mantiveram os painéis verdes, incluindo a prova aritmética de que as requisições travadas não estavam na amostra.
- **Watchdog** que faz a requisição travada registrar a própria pilha, log estruturado com schema compartilhado e `service.version` como campo-chave.
- Propagação **W3C Trace Context** com propagador composto e trace iniciado no app.
- **LGPD**: HMAC com chave no lugar de hash puro, criptoapagamento e três camadas de acesso.
- **RED redefinido** (SLI de razão sobre requisições iniciadas), **USE** com foco em saturação de concorrência e lista de permissão de labels com exemplars.
- **Amostragem combinada** com `spanmetrics` antes do tail sampling.
- Testes que reproduzem o incidente, mais quatro canais de diagnóstico em imagem sem shell.

## Stack assumida

As premissas completas estão no início de cada documento. De forma geral:

| Camada | Tecnologias |
| :--- | :--- |
| Aplicação | Java 21/25, Spring Boot 3.x/4.x, JPA/Hibernate, HikariCP, Resilience4j |
| Dados | PostgreSQL 18, Redis/Valkey, DynamoDB (leitura), motor de busca |
| Mensageria | Kafka, Debezium (CDC) |
| Plataforma | Kubernetes, Argo CD, Argo Rollouts, Kyverno, GitHub Actions |
| Observabilidade | OpenTelemetry (agente e Collector), Prometheus, Grafana, Tempo, Loki |
| Testes | JUnit 5, jqwik, Testcontainers, WireMock, Toxiproxy, k6, Awaitility |

## Convenção de commits

O histórico faz parte da entrega, então os commits são atômicos e descritivos, seguindo commits convencionais:

```
docs(aula-04): resolve dual write com outbox transacional e relay por CDC
docs(aula-05): rejeita blue-green pelo isolamento inexistente com banco compartilhado
docs(aula-06): adiciona watchdog que faz a requisicao travada registrar a propria pilha
```

Cada documento traz, no apêndice final, a sequência de commits sugerida, na ordem em que o raciocínio foi construído.

## Estrutura do repositório

Projeto Maven (`AdaBackendFundamentals`), com um pacote por aula dentro de `doc`. Cada pasta guarda o documento de System Design da respectiva aula:

```
AdaBackendFundamentals
├── README.md
├── pom.xml
└── src/
    ├── main/
    │   ├── java/br/com/ada/doc/
    │   │   ├── aula2/    # APIs Resilientes
    │   │   ├── aula3/    # Banco de Dados em Larga Escala
    │   │   ├── aula4/    # Microsserviços & Padrão Saga
    │   │   ├── aula5/    # Containers, CI/CD & Deploy sem Downtime
    │   │   └── aula6/    # Observabilidade & Diagnóstico em Produção
    │   └── resources/
    └── test/
```

## Como este trabalho foi construído

A regra do projeto foi que **nenhuma resposta entra num documento antes de ser discutida e fechada por mim**. A construção seguiu três etapas por aula, cada aula em uma conversa própria:

1. **Enquadramento** — qual sistema está sendo desenhado, o que o cenário dá como dado e o que precisa ser assumido.
2. **Discussão item a item** — alternativas com trade-offs, recomendação, discordâncias registradas e decisão minha ao final.
3. **Redação e commit** — o documento é escrito a partir das decisões fechadas, com revisão item a item.

O critério de qualidade foi **endereçar todos os problemas que o cenário apresentou**, e não apenas responder o que o template perguntou. Por isso alguns documentos têm seções extras, como o diagnóstico em imagem Distroless na Aula 06.

## Para o professor

Cada documento termina com um apêndice de **perguntas difíceis e respostas** que sintetiza os pontos que eu mais gostaria de defender numa conversa. Se quiser ir direto ao essencial, a tabela "Resumo das decisões e trade-offs" de cada documento mostra, em uma página, o que foi decidido, o que custa e o que foi descartado.

---

*Ada Tech — PE-JV-003 · Backend Fundamentos*
