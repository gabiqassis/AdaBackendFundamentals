# Template de System Design: Aula 03 - Banco de Dados em Larga Escala

> **Contexto do Cenário: Checkout na Black Friday**
>
> Sua equipe está preparando a persistência da API de Checkout para o pico de tráfego da Black Friday:
>
> 1. **Armazenamento:** Transações financeiras de pedidos exigem garantias ACID rigorosas e consistência imediata, enquanto dados de catálogo e sessões de usuários demandam alta vazão de leitura e tolerância a partição.
> 2. **Concorrência:** A atualização de status do pedido tem baixa disputa por registro (apenas o comprador interage), mas a **reserva de estoque** sofre altíssima contenção com centenas de clientes disputando as últimas unidades do mesmo SKU promocional.
> 3. **Incidente em Produção:** Em testes de carga, o tempo de resposta p99 subiu para 4.8s e o pool HikariCP saturou devido a consultas lentas de pedidos (`WHERE customer_id = ?` e `WHERE order_status = 'processing'`).
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Adicione este bloco ao repositório contínuo do seu projeto.

---

## 0. Recorte e premissas assumidas

Antes das decisões, declaro o que é dado pelo cenário e o que estou assumindo. Premissa não declarada é furo de avaliação, e metade das discussões sobre banco de dados acaba sendo discussão sobre premissa escondida.

**O que estou desenhando:** a camada de persistência da **API de Checkout** — as tabelas de pedido, item de pedido, pagamento e reserva de estoque, mais as decisões de concorrência e indexação que as cercam. Catálogo, sessão de usuário, antifraude e o parceiro de pagamento externo entram no desenho como vizinhos, não como responsabilidade minha de implementação.

**O que o cenário dá como dado:**

| Dado | Valor |
| :--- | :--- |
| Exigência de consistência da transação financeira | ACID rigoroso, consistência imediata |
| Exigência de catálogo e sessão | Alta vazão de leitura, tolerância a partição |
| Contenção na mudança de status do pedido | Baixa (um comprador por pedido) |
| Contenção na reserva de estoque | Altíssima, centenas de clientes no **mesmo registro** de SKU |
| Sintoma do incidente | p99 de 4.8s e pool HikariCP saturado |
| Consultas incriminadas | `WHERE customer_id = ?` e `WHERE order_status = 'processing'` |

**O que estou assumindo (premissas):**

| Premissa | Valor assumido | Por que assumo isso |
| :--- | :--- | :--- |
| P1 — Volumetria de pico | ~3.000 a 5.000 requisições de checkout por segundo; leitura de catálogo uma ordem de grandeza acima | O cenário diz "pico de Black Friday" de marketplace sem dar o número; sem uma ordem de grandeza nenhuma decisão de lock ou de pool é justificável |
| P2 — Volume da tabela de pedidos | Dezenas de milhões de linhas, crescendo | É o que explica Seq Scan custar 4.8s; em tabela de 100 mil linhas o problema do cenário não existiria |
| P3 — Motor relacional | PostgreSQL 18 (série estável hoje; a 19 tem GA marcado para 19/11/2026, fora da janela de Black Friday) | O cenário cita HikariCP e consulta SQL; Black Friday não é momento de estrear major version |
| P4 — Stack de aplicação | Java 21, Spring Boot 3.x, JPA/Hibernate, HikariCP, múltiplas réplicas de pod | HikariCP no enunciado implica JVM com pool JDBC |
| P5 — Topologia do banco | Um primário para escrita, réplicas de leitura assíncronas, mesma região | Padrão de RDS/Aurora gerenciado; multi-região muda todas as respostas de consistência |
| P6 — Perfil do SKU promocional | Poucas dezenas de SKUs concentram a disputa; o resto do catálogo tem contenção normal | "SKU promocional" no enunciado; é essa assimetria que justifica caminho especial para poucos SKUs em vez de complexidade no sistema todo |
| P7 — Reserva tem prazo | A reserva de estoque expira se o pagamento não confirmar (assumo 15 minutos) | Sem expiração, carrinho abandonado consome estoque e o problema de concorrência fica insolúvel |

**Formato das respostas:** primeira pessoa, decisão explícita, alternativa descartada nomeada e trade-off assumido em cada item.

---

## 1. Decisão de Armazenamento e Chave Primária (PK)

### 1.1 Motor de Banco de Dados e Paradigma de Persistência

- **A Escolha para o Checkout:** ( ) Relacional Puro (PostgreSQL) **(x) Persistência Poliglota / Híbrida**

- **Justificativa Arquitetural:** Avalie o Teorema CAP e o modelo de consistência (ACID vs BASE). Por que a sua escolha atende aos requisitos do Checkout sem comprometer a integridade financeira?

  > Escolho persistência poliglota com uma regra de ouro: **o dinheiro mora em PostgreSQL e em nenhum outro lugar.** Pedido, item de pedido, pagamento, reserva de estoque e o livro de eventos de auditoria ficam no relacional. Catálogo, sessão, carrinho e busca saem do PostgreSQL para motores próprios.
  >
  > **Por que ACID no caminho do dinheiro.** O requisito do checkout não é "gravar um registro de forma durável" — isso qualquer banco faz. É sustentar invariantes que atravessam mais de uma linha ao mesmo tempo:
  >
  > 1. Não existe pedido confirmado sem reserva de estoque correspondente.
  > 2. `quantidade_reservada` nunca excede `quantidade_disponivel` — saldo nunca negativo.
  > 3. Não existe cobrança sem pedido, nem pedido pago duas vezes pelo mesmo evento.
  >
  > Essas três são invariantes multi-linha. Em um banco BASE eu não tenho atomicidade entre elas: eu tenho que detectar a violação depois e compensar. Compensação de estorno custa tempo de engenharia, abre janela de inconsistência visível ao cliente e gera caso de suporte. Quando as linhas estão no mesmo banco, uma transação ACID resolve isso de graça — é literalmente mais barato ser consistente aqui do que ser eventual. É o oposto do que acontece quando as linhas estão em serviços diferentes, aí sim consistência eventual passa a ser a única opção viável.
  >
  > **CAP aplicado com honestidade.** Vale separar dois casos, porque tratar CAP como se fosse um menu de três opções é o erro mais comum nessa discussão:
  >
  > - **No caminho de escrita do checkout** (um primário PostgreSQL), não há partição entre réplicas de escrita para tolerar. O trade-off real durante uma partição de rede é: o primário fica indisponível e eu recuso a venda, ou eu aceito escrita em outro nó e arrisco vender o que não existe. Escolho **C sobre A**: recusar a compra com HTTP 503 é um problema de conversão; vender uma TV que não existe e debitar via Pix é um problema jurídico. Essa é a escolha explícita do cenário quando ele diz "consistência imediata".
  > - **No caminho de leitura de catálogo e sessão**, escolho **A sobre C**: mostrar preço ou estoque com alguns segundos de atraso é aceitável, cair não é. O número de estoque na vitrine é uma *dica*, não uma promessa. A promessa só é feita na reserva, dentro do PostgreSQL.
  >
  > Na prática, o quadro mais útil aqui não é CAP, é **PACELC**: CAP só descreve o comportamento *durante* uma partição, que é a minoria do tempo. PACELC acrescenta a pergunta que importa nos outros 99,9%: na ausência de partição (`Else`), troco **L**atência por **C**onsistência? Para a escrita do checkout, sim — pago a latência da confirmação síncrona no primário. Para o catálogo, não — leio de réplica ou de cache e aceito o atraso.
  >
  > **Por que não relacional puro, embora seja a opção mais simples.** Rejeito relacional puro por um motivo que está escrito no próprio incidente: **raio de explosão compartilhado**. No cenário, consulta lenta de pedido saturou o pool e derrubou o p99 do checkout. Se catálogo, busca, sessão e carrinho estiverem no mesmo PostgreSQL, qualquer um deles viciando o pool ou o cache de buffers leva o pagamento junto. Separar os motores é bulkhead na camada de dados: tráfego de vitrine, que é o maior volume e o menos crítico, não compete por conexão, por `shared_buffers` nem por tempo de autovacuum com a transação financeira.
  >
  > Em compensação, reconheço que relacional puro seria a escolha certa num cenário menor: volume que cabe em uma instância, sem SKU de disputa extrema, equipe pequena. `tsvector` + GIN resolve busca, `UNLOGGED table` ou um simples cache resolve sessão, e a organização economiza quatro componentes para operar. A complexidade poliglota se paga pela volumetria da P1 e pela assimetria de criticidade, não por elegância.
  >
  > **Trade-off que estou assumindo:** mais componentes para operar, monitorar e versionar; dado duplicado entre o relacional e os motores de leitura; necessidade de um pipeline de sincronização (CDC via Debezium/logical replication, ou publicação de evento pelo próprio serviço) com seu próprio modo de falha — se o pipeline atrasar, a vitrine mostra estoque errado por mais tempo. Aceito isso e mitigo com duas regras: nenhum motor NoSQL é fonte de verdade de dado financeiro, e a vitrine nunca promete estoque, só a reserva promete.

- **Além do Modelo Documental (Ecossistema NoSQL):** A persistência poliglota vai muito além da dicotomia "Relacional vs MongoDB". Considerando as necessidades complementares de uma plataforma de e-commerce (como carrinhos efêmeros, busca textual de produtos, telemetria/logs, auditoria temporal ou análise de conexões/fraude), cite e comente pelo menos outros dois tipos/famílias de bancos NoSQL que poderiam coexistir na arquitetura, indicando para quais casos de uso específicos cada um seria a ferramenta ideal e por quê.

  > Quatro famílias além da documental, com o caso de uso que justifica cada uma e o preço que cada uma cobra. Marco também o que eu adotaria já na primeira fase e o que ficaria para depois, porque adotar quatro motores de uma vez é um jeito eficiente de transformar uma arquitetura boa em uma operação impossível.
  >
  > **1. Chave-valor em memória — Redis / Valkey — *adoto na fase 1***
  >
  > Caso de uso ideal: **carrinho efêmero, sessão e chave de idempotência.** Carrinho é o oposto do pedido — altíssima taxa de escrita, vida curta, e perder um carrinho na queda de um nó é irritante, não é incidente financeiro. O que faz dele a ferramenta certa é o **TTL nativo**: o dado se apaga sozinho. Em PostgreSQL, "carrinho que expira" significa uma tabela que incha com linhas mortas e um job de limpeza disputando autovacuum com a tabela de pedidos — exatamente o tipo de trabalho de fundo que piora o incidente do cenário.
  >
  > Dois usos extras no mesmo motor: **contador de reserva para o SKU de disputa extrema** (volto a isso no item 2.2) e **cota/rate limit** de proteção na borda, porque Redis é single-threaded no comando e serializa incremento sem lock de banco.
  >
  > Preço: durabilidade fraca. Com AOF `everysec` eu perco até um segundo de escrita num crash. Por isso nada que seja fonte de verdade financeira entra aqui.
  >
  > **2. Motor de busca / índice invertido — OpenSearch / Elasticsearch — *adoto na fase 1***
  >
  > Caso de uso ideal: **busca textual do catálogo.** Não é "SELECT com LIKE". É análise linguística de português (stemming, acentuação, stopwords), tolerância a erro de digitação, sinônimos de categoria, facetas com contagem (marca, faixa de preço, avaliação) e ranking por relevância misturado a regra de negócio — produto patrocinado, margem, disponibilidade.
  >
  > Por que não resolver no PostgreSQL: `tsvector` + índice GIN é excelente e resolve busca simples, mas facetas com contagem sobre milhões de documentos, tolerância a *typo* e tuning de relevância são o trabalho para o qual um índice invertido distribuído foi construído. Forçar isso no relacional significa colocar a consulta mais pesada e mais frequente do site no mesmo motor que grava pagamento.
  >
  > Preço: não é fonte de verdade e é *near-real-time* — o refresh do índice tem latência da ordem de segundo, então produto recém-cadastrado leva um instante para aparecer. Preciso de pipeline de indexação e de um plano de reindexação completa.
  >
  > **3. Colunar analítico / série temporal — ClickHouse para funil, Prometheus-Mimir para métrica — *fase 2***
  >
  > Caso de uso ideal: **telemetria e análise de comportamento.** Evento de visualização, clique, adição ao carrinho e abandono: escrita append-only em volume muito maior que o de pedidos, consulta sempre por janela de tempo e por agregação, nunca por linha individual. Armazenamento colunar comprime bem e lê só as colunas da agregação; TTL por partição descarta o passado sem `DELETE`.
  >
  > Por que isso importa no cenário: durante a Black Friday, a pergunta "a conversão caiu?" precisa ser respondida em segundos, e essa consulta analítica **não pode** tocar o banco transacional. Rodar `GROUP BY` sobre a tabela de pedidos no pico é uma das formas clássicas de reproduzir exatamente o incidente descrito.
  >
  > Preço: é um segundo pipeline de dados e um segundo lugar onde a verdade pode divergir. Não sirvo página de cliente a partir dele.
  >
  > **4. Grafo — Neo4j / Amazon Neptune — *fase 2, se o antifraude pedir***
  >
  > Caso de uso ideal: **análise de conexões para fraude.** A pergunta do antifraude não é "esse CPF é bom?", é "quantas contas distintas compartilham cartão, dispositivo, endereço de entrega ou IP com essa conta, em até três saltos?". Em SQL isso é um `WITH RECURSIVE` com join explosivo a cada nível; em banco de grafo é travessia nativa, que é o acesso para o qual a estrutura foi feita.
  >
  > Preço: motor a mais, modelagem a mais, e só se paga quando o antifraude for próprio. Se for parceiro externo, essa necessidade não é minha.
  >
  > **Menção honrosa — auditoria temporal.** O requisito "como estava este pedido às 14h03 do dia 29" eu **não** resolveria com NoSQL novo. Resolvo com tabela append-only de eventos de domínio no próprio PostgreSQL (uma linha por transição, nunca `UPDATE`), replicada para armazenamento de objeto em formato colunar para consulta histórica. Motivo: auditoria financeira precisa estar na mesma transação que o fato auditado. Se o registro de auditoria vive em outro motor, eu criei um *dual write* no caminho mais sensível do sistema — e perder a trilha da operação que deu errado é justamente perder a trilha de que eu mais vou precisar.
  >
  > **Trade-off geral:** cada motor adicionado é mais superfície de operação, mais um lugar para o dado divergir e mais uma coisa que pode cair no pico. É por isso que eu faseio, e é por isso que o relacional continua dono do dinheiro: a regra que mantém isso governável é que **nenhum desses quatro é fonte de verdade de nada que tenha valor monetário.**

---

### 1.2 Estratégia da Chave Primária

- **A Escolha:** ( ) Sequence / Identity ( ) UUIDv4 **(x) UUIDv7 / TSID**

- **Impacto no Armazenamento:** Por que essa estratégia foi escolhida considerando inserções concorrentes e o impacto de fragmentação na árvore B-Tree (page splits)?

  > Escolho **UUIDv7** (RFC 9562) gerado na aplicação, armazenado na coluna nativa `uuid` do PostgreSQL, para `pedido`, `item_pedido` e `pagamento`.
  >
  > **A mecânica do B-Tree, que é o centro da questão.** O índice de chave primária do PostgreSQL é um B-Tree cujas folhas estão ordenadas pelo valor da chave. O que acontece na inserção depende inteiramente de o valor novo ser crescente ou aleatório:
  >
  > - **Chave crescente (Sequence, UUIDv7, TSID):** toda inserção cai na folha mais à direita da árvore. Essa página está quente no `shared_buffers`, e quando enche, o PostgreSQL aplica a otimização de *rightmost page split* — divide deixando a página antiga praticamente cheia em vez de partir ao meio, porque sabe que nada mais será inserido ali. Resultado: índice denso, poucos *splits*, pouco WAL gerado por divisão, e a parte quente do índice é pequena o suficiente para caber em cache.
  > - **Chave aleatória (UUIDv4):** cada inserção cai numa folha diferente, espalhada por todo o índice. Isso produz três custos que se somam na inserção concorrente do pico:
  >   1. **Page splits aleatórios 50/50** no meio da árvore, que deixam páginas pela metade. O índice ocupa mais espaço do que o dado que indexa.
  >   2. **WAL amplificado.** Cada split é registrado, e a primeira modificação de uma página após o *checkpoint* grava a página inteira no WAL (*full page image*). Com inserção espalhada, muitas páginas distintas são tocadas por *checkpoint*, então o volume de WAL explode — o que, na topologia da P5, significa **mais lag nas réplicas de leitura** justamente no pico.
  >   3. **Perda de localidade de cache.** Com chave crescente, as páginas quentes são as últimas. Com chave aleatória, *todo* o índice é quente: se ele não couber inteiro em `shared_buffers`, cada inserção vira leitura aleatória de disco. Esse é o ponto em que o custo deixa de ser teórico e aparece no p99.
  >
  > UUIDv7 elimina os três porque os 48 bits mais significativos são o timestamp Unix em milissegundos — ou seja, ele é ordenável no tempo, e para o B-Tree se comporta como um `Sequence`.
  >
  > **Comparação completa:**
  >
  > | Critério | Sequence / Identity (`bigint`) | UUIDv4 | UUIDv7 / TSID |
  > | :--- | :--- | :--- | :--- |
  > | Tamanho | 8 bytes | 16 bytes | 16 bytes (TSID: 8) |
  > | Localidade no B-Tree | Ótima | Péssima | Ótima |
  > | Page splits | Raros, otimizados à direita | Frequentes, 50/50 no meio | Raros, otimizados à direita |
  > | WAL / lag de réplica | Mínimo | Amplificado | Mínimo |
  > | Gerável no cliente, antes do INSERT | Não (precisa do banco) | Sim | Sim |
  > | Enumerável por terceiros | Sim — vaza volume de vendas | Não | Parcialmente: vaza o instante de criação |
  > | Colisão em merge entre bases/serviços | Alta | Desprezível | Desprezível |
  >
  > **Por que descartei Sequence, apesar de ser a chave mais eficiente da tabela.** Dois motivos concretos, nenhum deles sobre performance de índice:
  >
  > 1. **Eu preciso do identificador antes de gravar.** Idempotência de checkout, publicação de evento em tabela de *outbox* e propagação de `correlation id` para o serviço de pagamento todos querem o id do pedido já formado no momento em que a requisição chega. Com `Sequence` eu depende de ida ao banco para saber quem eu sou.
  > 2. **Detalhe de JPA que morde em produção:** com `GenerationType.IDENTITY`, o Hibernate precisa do valor gerado no `INSERT` e por isso **desabilita o batch de JDBC inserts**. Em carga de pico com itens de pedido em lote, isso vira uma viagem ao banco por linha, cada uma segurando conexão do pool. `SEQUENCE` com `allocationSize` preserva o batch e seria aceitável; `IDENTITY` não.
  >
  > Em contrapartida, se este fosse um sistema de uma instância só, sem integração externa e sem outbox, `bigint` com `SEQUENCE` seria a escolha certa e eu economizaria 8 bytes por chave e por chave estrangeira — o que, em dezenas de milhões de linhas com três ou quatro FKs e seus índices, não é pouco.
  >
  > **Por que descartei UUIDv4:** é a escolha que mais aparece em projeto Java por ser o default de `UUID.randomUUID()`, e é exatamente a pior para este cenário, pelos três custos descritos acima. A conveniência de uma linha de código não paga o preço de WAL e de cache sob inserção concorrente.
  >
  > **Por que UUIDv7 e não TSID:** TSID tem o apelo de ser ordenável no tempo em 8 bytes — metade do custo de armazenamento. Descartei por um motivo operacional: TSID precisa de um *node id* coordenado entre as réplicas para garantir unicidade, e coordenação de identidade de nó em ambiente com pods efêmeros e autoescala é uma fonte de bug silencioso. UUIDv7 não precisa de coordenação nenhuma. Se o tamanho do índice virasse gargalo medido, TSID seria a otimização seguinte — com o custo da coordenação assumido conscientemente.
  >
  > **Implementação concreta na stack assumida:**
  >
  > - Na aplicação: o Hibernate 7 já traz a estratégia de UUIDv7 nativa (`org.hibernate.id.uuid.UuidVersion7Strategy`), selecionável pelo atributo `algorithm` de `@UuidGenerator`. Em Hibernate 6 eu usaria `com.fasterxml.uuid` (java-uuid-generator) atrás de um gerador próprio.
  > - No banco: coluna do tipo **`uuid` (16 bytes binários)**, nunca `varchar(36)`. Guardar UUID como texto mais que dobra o tamanho da chave, da FK e de todo índice que a contenha, e ainda troca comparação de inteiro por comparação de string. É o erro mais caro e mais comum dessa decisão.
  > - Como rede de segurança, o PostgreSQL 18 tem `uuidv7()` nativo, que eu uso como `DEFAULT` da coluna para inserção feita fora da aplicação (script de correção, carga).
  >
  > **Trade-off que estou assumindo:** pago 8 bytes a mais por chave em relação a `bigint`, replicados em cada índice e cada chave estrangeira — custo real de armazenamento e de cache, que aceito em troca de geração no cliente e independência do banco. E assumo o vazamento do instante de criação embutido no id: para pedido isso é irrelevante (o cliente sabe quando comprou), mas se algum dia um desses ids for exposto publicamente num contexto onde o horário seja sensível, a mitigação é o deslocamento de timestamp que o `uuidv7(interval)` do PostgreSQL 18 oferece, ou expor um código público separado do id interno.

---

## 2. A Dualidade da Concorrência: Locks e Mitigações

Analise e defina a abordagem de concorrência para os dois fluxos críticos do negócio:

### 2.1 Mudança de Status do Pagamento (Baixa contenção por registro)

- **Abordagem de Lock:** **(x) Lock Otimista (`@Version`)** ou ( ) Lock Pessimista (`SELECT FOR UPDATE`)?

- **Justificativa:** Por que essa abordagem é a mais adequada para este cenário específico?

  > Lock otimista, com duas salvaguardas que considero inseparáveis dele.
  >
  > **Por que otimista:** o critério de escolha entre otimista e pessimista é a probabilidade de conflito, não a gravidade do dado. Pagamento é dado grave, mas a disputa por registro aqui é quase nula — um pedido, um comprador. Lock otimista não adquire lock nenhum no caminho feliz: nenhuma linha fica travada, nenhuma thread espera, nenhuma conexão do pool fica presa. Custa uma coluna `version` e uma cláusula no `UPDATE`. Lock pessimista nesse fluxo pagaria o custo de serialização para evitar um conflito que praticamente não ocorre.
  >
  > Isso conversa direto com o incidente do cenário: `SELECT FOR UPDATE` segura a linha **até o fim da transação**, e na prática a transação de JPA costuma envolver mais trabalho depois do lock. Multiplicado pela volumetria da P1, é mais uma forma de chegar em pool saturado — o problema que estou tentando não recriar.
  >
  > **A disputa real não é comprador contra comprador — é o gateway contra ele mesmo.** Aqui falo do que vejo no dia a dia trabalhando com gateway de pagamentos: quem concorre pela linha de pagamento é o *webhook* do parceiro, a consulta ativa de status (o *polling* de conciliação) e as retentativas de ambos, todos tentando mover o mesmo pagamento ao mesmo tempo. Lock otimista sozinho não resolve isso. Ele detecta escrita concorrente, mas não distingue "duas escritas do mesmo evento" de "duas escritas de eventos diferentes". Por isso somo:
  >
  > **Salvaguarda 1 — transição condicionada ao estado esperado.** O `UPDATE` carrega o estado de origem no `WHERE`, não só a versão:
  >
  > ```sql
  > UPDATE pagamento
  >    SET status = 'PAGO', version = version + 1, atualizado_em = now()
  >  WHERE id = :id
  >    AND version = :version
  >    AND status = 'AUTORIZADO';
  > ```
  >
  > `rowcount = 0` me diz que alguém mexeu antes — e a releitura me diz **se o resultado dessa outra escrita já é o que eu queria**. Se o pagamento já está `PAGO`, meu trabalho acabou e eu descarto o evento em silêncio; se foi para `CANCELADO`, eu tenho um conflito de verdade para tratar. Sem essa cláusula, versão desatualizada e transição inválida geram o mesmo erro e eu perco a informação que diferencia os dois casos.
  >
  > **Salvaguarda 2 — idempotência pelo id do evento, antes do lock.** Cada notificação do parceiro traz um identificador próprio. Gravo esse identificador numa tabela com `UNIQUE` **na mesma transação** da mudança de status. Evento repetido viola a constraint e é descartado sem nem disputar a linha. Isso tira do lock otimista o trabalho que não é dele.
  >
  > **Salvaguarda 3 — máquina de estados monótona.** As transições permitidas são declaradas no código e no banco; `PAGO` e `CANCELADO` são terminais. Evento que chega fora de ordem (e chega, porque rede reordena) não consegue puxar um pagamento de volta para `PENDENTE`. Lock nenhum protege contra isso — é regra de domínio.
  >
  > **Trade-off que estou assumindo:** `OptimisticLockException` passa a ser evento normal de operação e precisa de tratamento explícito em cada ponto de escrita — retentativa com releitura onde a operação é idempotente, descarte onde o estado final já é o desejado, alarme onde é conflito real. É mais código de tratamento de erro do que o `SELECT FOR UPDATE` exigiria. Aceito isso em troca de nenhuma linha travada e nenhuma conexão retida no fluxo de maior volume do sistema. E reconheço o limite: lock otimista protege uma linha, não protege invariante entre agregados diferentes — essa proteção vem do item 2.2 e das constraints de banco.

---

### 2.2 Controle de Estoque Concorrente (Alta contenção no mesmo registro)

- **Abordagem de Lock:** ( ) Lock Otimista (`@Version`) ou **(x) Lock Pessimista (`SELECT FOR UPDATE`)** — na forma de **`UPDATE` condicional atômico**, que é lock pessimista de linha reduzido à menor duração possível

- **Justificativa:** Centenas de usuários disputando os últimos itens na Black Friday. Como você garante que o saldo nunca fique negativo sem derrubar o banco?

  > A resposta tem duas partes, porque as duas metades da pergunta têm respostas diferentes: **o saldo nunca negativo é garantido pelo banco; não derrubar o banco é garantido por não deixar a disputa chegar nele.**
  >
  > #### Por que lock otimista está descartado aqui
  >
  > Com centenas de requisições no mesmo registro de SKU, `@Version` significa que a primeira escrita vence e **todas as outras falham**. Cada falha vira retentativa, cada retentativa reentra na disputa, e o sistema entra em *livelock*: muito trabalho, pouquíssima reserva concluída, e a vazão cai conforme a concorrência sobe — o inverso do que se espera de um sistema sob carga. Lock otimista é a ferramenta certa no item 2.1 e a errada aqui, pelo mesmo critério: probabilidade de conflito.
  >
  > #### Por que `SELECT FOR UPDATE` seguido de `UPDATE` também não é a forma certa
  >
  > O resultado é correto, mas o lock fica retido do `SELECT` até o `COMMIT`. Nesse intervalo a transação costuma fazer outras coisas — e cada thread esperando na fila da linha está segurando uma conexão do HikariCP. Com o default de `maximumPoolSize = 10` por réplica e `connectionTimeout = 30000ms`, a fila de espera vira o incidente do cenário: pool saturado e p99 em segundos. Também é um `round-trip` extra ao banco por reserva.
  >
  > #### O que eu faço: `UPDATE` condicional atômico, sem `SELECT` antes
  >
  > ```sql
  > UPDATE estoque_sku
  >    SET quantidade_reservada = quantidade_reservada + :qtd,
  >        version = version + 1
  >  WHERE sku = :sku
  >    AND quantidade_disponivel - quantidade_reservada >= :qtd;
  > ```
  >
  > `rowcount = 1` → reservado. `rowcount = 0` → esgotado, e eu respondo `409 Conflict` imediatamente, sem retentativa (não há o que retentar: não tem estoque).
  >
  > Três propriedades fazem disso a forma correta:
  >
  > 1. **O lock de linha existe só durante a execução do comando** — microssegundos — e não do início da transação de negócio até o commit.
  > 2. **A decisão e a escrita são o mesmo ato.** Não há janela entre "li 1 unidade disponível" e "decrementei". É isso que torna impossível a venda duplicada clássica.
  > 3. **Em `READ COMMITTED`, o predicado é reavaliado depois que o lock é obtido.** Quando dois comandos disputam a linha, o segundo espera, relê a versão recém-commitada e **reaplica o `WHERE`** (mecanismo de *EvalPlanQual* do PostgreSQL). Se o estoque acabou nesse meio tempo, a condição falha e nada é gravado. É exatamente essa garantia que um `SELECT` seguido de `UPDATE` não tem, porque ali o `SELECT` leu um retrato que já envelheceu.
  >
  > **Última linha de defesa, no esquema:** `CHECK (quantidade_reservada >= 0 AND quantidade_reservada <= quantidade_disponivel)`. Se um dia um caminho de código novo, um script de correção ou uma migração furar a regra da aplicação, o banco recusa. A integridade do estoque não depende de ninguém lembrar de usar o método certo.
  >
  > **Isolamento:** `READ COMMITTED` (o default). Descarto `SERIALIZABLE` deliberadamente: ele resolveria a correção, mas sob esta contenção produziria `serialization_failure` em massa, e eu teria trocado o problema de lock pelo problema de retentativa — o mesmo buraco do lock otimista, com outro nome.
  >
  > #### Não derrubar o banco: a linha quente continua sendo uma linha quente
  >
  > O `UPDATE` atômico resolve a correção, mas não revoga a física: **todas as requisições do mesmo SKU serializam em um registro.** Se 10.000 requisições chegarem para disputar 50 unidades, mesmo um lock de microssegundos forma fila. É aqui que a resposta precisa ir além do SQL.
  >
  > **Mitigação 1 — transação curtíssima, sem rede dentro.** A reserva é sua própria transação. Nenhuma chamada ao parceiro de pagamento, nenhum `publish` em broker, nenhuma consulta a antifraude acontece com o lock de estoque na mão. O que precisa sair daqui sai por tabela de *outbox*, gravada na mesma transação e despachada depois. Regra que eu levaria para qualquer code review: **nada que espera rede roda dentro de transação que segura lock de linha disputada.**
  >
  > **Mitigação 2 — falhar rápido em vez de enfileirar.** Na conexão da reserva:
  >
  > ```sql
  > SET LOCAL lock_timeout = '150ms';
  > SET LOCAL statement_timeout = '500ms';
  > ```
  >
  > E no pool, `connectionTimeout` na casa de 1–2 segundos, não os 30 segundos do default. Cliente que recebe "tente novamente" em 200ms é cliente frustrado; thread parada por 30 segundos é o sistema inteiro caindo. A escolha entre os dois é fácil.
  >
  > **Mitigação 3 — portão de admissão antes do banco, para os SKUs da P6.** Não deixo 10.000 requisições chegarem ao banco para disputar 50 unidades. Para a lista conhecida de SKUs promocionais, um semáforo distribuído (contador em Redis) admite um múltiplo pequeno do estoque restante e **rejeita o excedente na borda**, sem tocar PostgreSQL. É o que a indústria faz em lançamento disputado, e é a mitigação com melhor relação custo-benefício: ela resolve o problema no único lugar onde é barato resolver, antes do recurso escasso.
  >
  > **Mitigação 4 — fracionamento do contador (*bucket sharding*), se a mitigação 3 não bastar.** O contador do SKU passa a ser N linhas (`sku`, `bucket`), cada uma com uma fatia do estoque. A requisição escolhe um bucket por hash e, se ele estiver vazio, tenta outro — ou usa `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1` para pegar um bucket que não esteja travado naquele instante, em vez de entrar na fila. A contenção cai por um fator de N. Preço: o estoque fica fragmentado, então "faltam 3 unidades" pode significar "faltam 3, mas em buckets diferentes", e eu preciso de rebalanceamento e de um caminho de consolidação para as últimas unidades. Complexidade real — por isso é mitigação 4 e só para os poucos SKUs que exigirem.
  >
  > **Mitigação 5 — reserva com prazo (P7).** A reserva expira em 15 minutos e um job devolve o saldo. Sem isso, carrinho abandonado consome estoque e nenhum lock no mundo resolve. O job de expiração roda em lote pequeno, com `SKIP LOCKED`, para não virar ele mesmo uma fonte de contenção.
  >
  > **O que eu descartei e por quê:**
  >
  > - **Contador autoritativo em Redis, com o PostgreSQL só conciliando depois:** tecnicamente é o que mais escala — Redis serializa incremento em memória sem lock de banco. Descartei como padrão porque cria duas fontes de verdade para o estoque, e divergência entre elas significa vender o que não existe. Mantenho essa opção na gaveta exclusivamente para os SKUs da P6, e **mesmo ali** o Redis é portão de admissão (mitigação 3), não autoridade: a reserva válida continua sendo a linha do PostgreSQL.
  > - **Reserva assíncrona por fila particionada por SKU** (um único consumidor serial por SKU, via Kafka). Elimina a contenção de banco por construção e é um desenho elegante. Descartei porque muda a experiência: o cliente sai do checkout sem saber se conseguiu, e "pedido em processamento" numa compra de Black Friday é conversão perdida e volume de suporte. Se o produto aceitasse esse modelo, seria minha primeira escolha.
  >
  > **Fluxo resultante:**
  >
  > ```mermaid
  > sequenceDiagram
  >     participant C as Cliente
  >     participant API as API de Checkout
  >     participant R as Redis (portão de admissão)
  >     participant PG as PostgreSQL
  >     C->>API: POST /checkout (SKU promocional)
  >     API->>R: DECR admissao:{sku}
  >     alt sem vaga de admissão
  >         R-->>API: < 0
  >         API-->>C: 409 esgotado (não toca o banco)
  >     else vaga liberada
  >         R-->>API: ok
  >         API->>PG: BEGIN; SET LOCAL lock_timeout='150ms'
  >         API->>PG: UPDATE estoque_sku ... WHERE disponivel - reservada >= qtd
  >         alt rowcount = 1
  >             API->>PG: INSERT pedido (PENDENTE) + INSERT outbox
  >             API->>PG: COMMIT
  >             API-->>C: 201 pedido criado
  >         else rowcount = 0 ou lock_timeout
  >             API->>PG: ROLLBACK
  >             API-->>C: 409 esgotado / 503 tente novamente
  >         end
  >     end
  > ```
  >
  > **Trade-off que estou assumindo:** mantenho dois caminhos de reserva — o simples, por `UPDATE` condicional, para o catálogo todo, e o caminho com portão de admissão para a lista curta de SKUs de disputa extrema. Isso é mais código, mais configuração e um segundo modo de falha (a lista de SKUs quentes precisa ser mantida; se ela estiver errada, o portão protege o SKU errado). Aceito porque o oposto — aplicar a complexidade do caminho quente ao catálogo inteiro — custaria muito mais em manutenção para proteger registros que ninguém disputa.

---

### 2.3 Riscos e Mitigações

- **Análise Crítica:** Quais os riscos operacionais das abordagens de lock que você escolheu (ex: excesso de `OptimisticLockException` frustrando o cliente vs `ConnectionTimeoutException` e *deadlocks* enfileirando threads no Lock Pessimista)? Como sua arquitetura mitiga esses riscos?

  > | # | Risco | Como ele aparece no cenário | Sinal que eu monitoro | Mitigação |
  > | :--- | :--- | :--- | :--- | :--- |
  > | R1 | Enxurrada de `OptimisticLockException` no status de pagamento | Webhook do parceiro, polling de conciliação e retentativas movendo o mesmo pagamento ao mesmo tempo | Contador de `OptimisticLockException` por endpoint e por motivo | Deduplicação pelo id do evento antes da escrita; transição condicionada ao estado de origem; releitura que distingue "já está no estado desejado" (descarta) de conflito real (alarma) |
  > | R2 | Retentativa que piora a disputa | Cliente e cliente HTTP retentando automaticamente no mesmo instante | Razão entre tentativas e sucessos | Retentativa só fora da transação, com backoff exponencial **e jitter**, teto de tentativas e DLQ. Falha de estoque (`rowcount = 0`) **não** é retentável — é resposta final |
  > | R3 | **Deadlock em carrinho com múltiplos itens** | Carrinho A reserva SKU 1 e depois SKU 2; carrinho B faz o inverso no mesmo instante — abraço mortal clássico | `deadlock_timeout` disparando, erros `40P01` no log | **Ordenar os itens por SKU antes de reservar.** Ordem de aquisição de lock consistente em todo o código elimina esse deadlock por construção. É a mitigação mais barata do documento e a mais esquecida |
  > | R4 | `ConnectionTimeoutException` e saturação do pool | A fila da linha quente de SKU, ou consulta lenta de pedido (o incidente), retendo conexões | `hikaricp_connections_pending`, `hikaricp_connections_acquire_seconds` (p99), `hikaricp_connections_timeout_total` | `lock_timeout` de 150ms e `statement_timeout` por operação; `connectionTimeout` de 1–2s em vez de 30s; nenhuma chamada de rede dentro de transação com lock |
  > | R5 | **Reflexo de aumentar o pool** | Pool saturou, alguém sobe `maximumPoolSize` de 10 para 100 | Vazão **cai** enquanto `pending` melhora | Não aumentar. Pool maior que o paralelismo útil do banco (regra prática do HikariCP: perto de `núcleos × 2 + discos`) só move a fila da aplicação para dentro do PostgreSQL, onde ela custa mais — cada conexão é um processo com memória própria. O que eu faço é **separar pools por carga**: pool do checkout, pool de relatório/consulta, pool de job de expiração. Bulkhead na camada de dados, para que consulta de "Meus Pedidos" não consuma a conexão da reserva |
  > | R6 | Transação aberta esquecida segurando lock | `idle in transaction` com lock de estoque retido, parando o SKU inteiro | `pg_stat_activity` com `state = 'idle in transaction'` e `xact_start` antigo | `idle_in_transaction_session_timeout` configurado; `@Transactional` de escopo mínimo; alarme sobre transação mais antiga que N segundos |
  > | R7 | Esgotamento silencioso virando erro 500 | `rowcount = 0` tratado como falha genérica | Taxa de 5xx no endpoint de checkout | "Esgotado" é `409` com corpo de domínio, não exceção não tratada. Erro de negócio e erro de infraestrutura precisam de códigos diferentes, senão o painel mente durante o incidente |
  > | R8 | Degradação em cascata sem contenção | O SKU quente puxa o checkout inteiro para baixo | p99 por SKU, não só p99 global | *Shedding* seletivo: quando a taxa de `lock_timeout` de um SKU passa do limite, o portão de admissão fecha para **aquele SKU** e o resto do checkout continua vendendo |
  >
  > **Observação sobre R5, porque é a mitigação que eu mais defenderia numa discussão:** o incidente do cenário é descrito como "o pool saturou", e a leitura natural é que faltou pool. Não faltou. Sobrou tempo de consulta. Pool é fila, e aumentar a fila não aumenta a capacidade de atendimento — só aumenta o tempo que todos esperam antes de descobrir que não vão ser atendidos. A correção é reduzir o tempo que cada conexão fica ocupada (índice certo, transação curta, timeout agressivo) e isolar cargas em pools distintos.

---

## 3. Troubleshooting do Incidente e Gestão de Índices

### 3.1 Investigação de Incidente em Produção

- **Cenário:** Em testes de carga para a Black Friday, o tempo de resposta p99 do Checkout subiu repentinamente para 4.8s e o pool de conexões (HikariCP) saturou. Os logs apontam que a consulta de pedidos mais frequente (`WHERE customer_id = ? AND order_status = 'processing'`) está demorando segundos para responder.

- **Plano de Diagnóstico:** Qual é o seu processo técnico passo a passo para investigar a causa raiz dessa lentidão diretamente no banco de dados? Quais comandos, métricas ou planos você analisaria para comprovar a hipótese antes de propor qualquer mudança?

  > A regra que organiza tudo abaixo: **não mexo em nada antes de ter uma hipótese escrita e uma evidência que a sustente.** "Criei um índice e melhorou" não é diagnóstico — é sorte, e sorte não se repete na próxima Black Friday.
  >
  > #### Passo 0 — Localizar o gargalo: aplicação ou banco?
  >
  > Antes de abrir o banco, comparo duas métricas que normalmente estão em painéis diferentes:
  >
  > - Tempo que a requisição passa **esperando conexão**: `hikaricp_connections_acquire_seconds` (p99) e `hikaricp_connections_pending`.
  > - Tempo que a requisição passa **executando SQL**.
  >
  > A distinção decide o resto da investigação. Se `acquire` está alto e o tempo de execução no banco está baixo, o problema é contenção de pool e eu vou investigar quem está retendo conexão — possivelmente uma chamada de rede dentro da transação, não uma consulta lenta. Se o tempo de execução está alto, o banco é o lugar certo para olhar. No cenário descrito, os logs já apontam consulta de segundos, então sigo para o banco — mas registro essa verificação porque pular o passo 0 é como se chega em "aumentei o pool e não resolveu".
  >
  > #### Passo 1 — Fotografar o estado durante o incidente
  >
  > O teste de carga está rodando, então eu olho o agora, não o histórico:
  >
  > ```sql
  > -- Onde as sessões estão esperando?
  > SELECT state, wait_event_type, wait_event, count(*) AS sessoes,
  >        max(now() - query_start) AS mais_antiga
  >   FROM pg_stat_activity
  >  WHERE backend_type = 'client backend'
  >  GROUP BY 1,2,3
  >  ORDER BY sessoes DESC;
  > ```
  >
  > O que cada resultado me diz:
  >
  > | `wait_event_type` dominante | Interpretação |
  > | :--- | :--- |
  > | `IO` (`DataFileRead`) | Leitura de disco — plano ruim ou dado fora do cache |
  > | `Lock` (`transactionid`, `tuple`) | Contenção de linha — é o problema do item 2.2, não de índice |
  > | `LWLock` | Contenção interna (buffer, WAL) |
  > | `Client` (`ClientRead`) com `idle in transaction` | Transação aberta esquecida (R6) |
  > | Nada esperando, CPU alta | Trabalho de CPU — provável `Seq Scan` com filtro descartando muita linha |
  >
  > Se aparecer `Lock`, eu identifico quem bloqueia quem:
  >
  > ```sql
  > SELECT pid, pg_blocking_pids(pid) AS bloqueado_por, wait_event, left(query, 120)
  >   FROM pg_stat_activity
  >  WHERE cardinality(pg_blocking_pids(pid)) > 0;
  > ```
  >
  > #### Passo 2 — Ranquear por custo total, não por consulta mais lenta
  >
  > Com `pg_stat_statements` habilitado (exige `shared_preload_libraries`, ou seja, precisa já estar ligado antes do incidente — ponto do passo 8):
  >
  > ```sql
  > SELECT queryid, calls, mean_exec_time, total_exec_time, rows,
  >        shared_blks_hit, shared_blks_read
  >   FROM pg_stat_statements
  >  ORDER BY total_exec_time DESC
  >  LIMIT 20;
  > ```
  >
  > Ordeno por `total_exec_time` de propósito: a consulta que está derrubando o sistema raramente é a mais lenta individualmente — é a que é razoavelmente lenta e roda o tempo todo. `shared_blks_read` alto em relação a `shared_blks_hit` confirma leitura de disco. `rows / calls` muito baixo com muitos blocos lidos é a assinatura de varredura grande que devolve pouca linha, que é o retrato da consulta do cenário.
  >
  > #### Passo 3 — Provar o plano da consulta suspeita
  >
  > ```sql
  > EXPLAIN (ANALYZE, BUFFERS)
  > SELECT * FROM pedido
  >  WHERE customer_id = '...' AND order_status = 'processing';
  > ```
  >
  > No PostgreSQL 18 o `BUFFERS` já vem por padrão com `ANALYZE`, então o plano mostra I/O sem eu pedir. O que eu procuro, em ordem:
  >
  > 1. **`Seq Scan` com `Filter` e `Rows Removed by Filter` enorme** — leu a tabela toda para devolver poucas linhas. É a hipótese principal.
  > 2. **Divergência entre `rows` estimado e `rows` real** (uma ordem de grandeza ou mais) — estatística desatualizada ou correlação entre colunas que o *planner* não conhece. Importa porque isso faz o *planner* escolher plano ruim mesmo depois de eu criar o índice.
  > 3. **`shared read` alto** — dado não está em cache; o problema é volume de página lida.
  > 4. **`Planning Time` desproporcional** — sintoma de excesso de índices, que é justamente o que o PR do item 3.2 propõe criar.
  >
  > Cuidado metodológico: rodo isso **com volume de produção**, em réplica ou em cópia restaurada. `EXPLAIN` em base de desenvolvimento com 10 mil linhas mente, porque o *planner* escolhe plano em função do tamanho estimado — e vai preferir `Seq Scan` numa tabela pequena com total razão.
  >
  > #### Passo 4 — Saúde da tabela e dos índices
  >
  > ```sql
  > SELECT relname, seq_scan, seq_tup_read, idx_scan,
  >        n_live_tup, n_dead_tup, last_autovacuum, last_autoanalyze
  >   FROM pg_stat_user_tables
  >  WHERE relname IN ('pedido','pagamento','estoque_sku');
  >
  > SELECT indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
  >   FROM pg_stat_user_indexes
  >  WHERE relname = 'pedido'
  >  ORDER BY idx_scan;
  > ```
  >
  > `seq_scan` alto com `idx_scan` baixo confirma ausência de índice útil. `n_dead_tup` grande com `last_autovacuum` antigo aponta *bloat*: a tabela ocupa muito mais páginas do que o dado vivo exige, e varrer isso custa proporcionalmente mais. Autovacuum que não dá conta é causa raiz comum, e nenhum índice conserta.
  >
  > #### Passo 5 — Estatística e correlação entre as colunas
  >
  > Esse passo específico existe porque as duas colunas do predicado são correlacionadas. O *planner* assume **independência** entre `customer_id` e `order_status` e multiplica as seletividades, o que erra feio: um cliente específico tem distribuição de status muito diferente da distribuição global. Verifico quando foi o último `ANALYZE` e, se a divergência do passo 3 se confirmar:
  >
  > ```sql
  > CREATE STATISTICS stat_pedido_cliente_status (dependencies, ndistinct)
  >     ON customer_id, order_status FROM pedido;
  > ANALYZE pedido;
  > ```
  >
  > Sem isso, pode acontecer o pior cenário de troubleshooting: eu crio o índice correto e o *planner* continua ignorando ele, porque acredita que a consulta devolverá linhas demais para o índice valer a pena.
  >
  > #### Passo 6 — Escrever a hipótese antes de tocar em qualquer coisa
  >
  > Formato que eu usaria no incidente, com os números preenchidos pelos passos anteriores:
  >
  > > *A consulta `pedido WHERE customer_id = ? AND order_status = 'processing'` executa `Seq Scan` sobre N milhões de linhas, lendo B blocos e descartando R linhas no filtro, porque não existe índice que atenda o predicado. Cada execução ocupa uma conexão por ~Xs. Com `maximumPoolSize = 10` por réplica e Y execuções concorrentes, a demanda de conexão excede o pool, `hikaricp_connections_pending` sobe e requisições começam a estourar `connectionTimeout`. O p99 de 4.8s é a soma do tempo de fila no pool com o tempo de execução, não só o tempo da consulta.*
  >
  > A hipótese precisa explicar **as duas** observações do cenário — a consulta lenta e o pool saturado — e precisa de três evidências anexadas: o plano do passo 3, a linha do `pg_stat_statements` do passo 2 e o gráfico de `pending` correlacionado no tempo com a latência. Se a hipótese explica só uma das duas, ela está incompleta.
  >
  > #### Passo 7 — Validar a correção antes de aplicar
  >
  > Em réplica ou ambiente com volume equivalente: criar o índice candidato, repetir o `EXPLAIN (ANALYZE, BUFFERS)`, comparar blocos lidos e tempo, e rodar o mesmo teste de carga medindo p99 **e** o impacto na escrita (latência de `INSERT`/`UPDATE` de pedido, volume de WAL, lag da réplica). Em produção, `CREATE INDEX CONCURRENTLY` — detalhado no item 3.2.
  >
  > #### Passo 8 — Deixar o próximo diagnóstico mais barato
  >
  > Metade do tempo gasto num incidente desses é gasto conseguindo dados que deveriam já estar lá. O que eu deixo ligado **antes** da Black Friday:
  >
  > - `pg_stat_statements` habilitado (não dá para habilitar no meio do incidente — exige restart).
  > - `track_io_timing = on`, para que `EXPLAIN` e `pg_stat_statements` mostrem tempo de I/O real.
  > - `log_min_duration_statement` em um limite sensato e `auto_explain` com `log_analyze`, **amostrado** (`auto_explain.sample_rate`) para não virar o próprio gargalo. Assim o plano da consulta lenta fica no log sem eu precisar reproduzir.
  > - `log_lock_waits = on` e `deadlock_timeout` ajustado — é o que transforma "travou" em "travou esperando esta linha".
  > - `idle_in_transaction_session_timeout` configurado.
  > - Painel único com p99 da aplicação, métricas do Hikari e top consultas do `pg_stat_statements` **no mesmo eixo de tempo**. A correlação visual entre `pending` e p99 é o que fecha o diagnóstico em minutos em vez de horas.

---

### 3.2 Decisão de Engenharia: A Abordagem de Índices

- **Dilema no Pull Request:** Para solucionar a lentidão, um desenvolvedor da equipe abriu um PR criando índices individuais para todas as colunas da tabela de pedidos que aparecem em filtros de consulta.

- **Seu Posicionamento:** Como responsável técnico, você aprovaria esse PR? Qual é a sua avaliação crítica sobre essa estratégia e quais impactos colaterais sistêmicos devem ser ponderados antes de aprovar novos índices?

  > **Não aprovo.** Peço para dividir o PR: entra o índice que tem evidência de plano atrás dele, o resto volta. E deixo o motivo registrado no PR, porque "reprovado" sem explicação não ensina ninguém — e a intenção do colega está certa, a técnica é que não.
  >
  > #### Por que a estratégia está errada tecnicamente
  >
  > **1. Um índice por coluna não atende um predicado composto.** A consulta do incidente filtra `customer_id` **e** `order_status` juntos. Dois índices separados dão ao *planner* duas opções ruins: combinar os dois com `BitmapAnd` (duas varreduras de índice, montagem de bitmap, e ainda a visita ao heap) ou usar só um e filtrar o resto na mão. Um índice composto em `(customer_id, order_status)` resolve o predicado inteiro em uma descida de árvore. A regra é que a estrutura do índice precisa espelhar a forma da consulta; coluna a coluna espelha a forma da tabela, que é outra coisa.
  >
  > **2. O índice em `order_status` isolado é inútil e caro.** São poucos valores distintos, e `'processing'` provavelmente cobre uma fatia grande das linhas. Seletividade baixa significa que o *planner* vai preferir `Seq Scan` de qualquer jeito — ele acerta ao fazer isso. Resultado: um índice que ninguém usa, mantido em toda escrita. Comprovo com `pg_stat_user_indexes.idx_scan` parado em zero depois de alguns dias.
  >
  > **3. Ordem das colunas não é detalhe.** `(customer_id, order_status)` serve consulta por cliente, com ou sem status. `(order_status, customer_id)` serve bem só quem filtra por status primeiro. Índice composto é prefixo utilizável: as colunas de igualdade vêm primeiro, a mais seletiva na frente. Um PR que cria índices "por coluna" não tem onde expressar essa decisão, o que já indica que a abordagem não comporta a pergunta certa.
  >
  > #### Impactos colaterais sistêmicos — a parte que não aparece no PR
  >
  > **4. Amplificação de escrita.** Todo `INSERT` e `UPDATE` mantém **todos** os índices da tabela. Na Black Friday, a tabela de pedidos é a mais escrita do sistema. Cada índice extra é custo fixo em cada escrita, somado ao WAL correspondente.
  >
  > **5. O impacto mais grave, específico do PostgreSQL: morte do HOT update.** No PostgreSQL, `UPDATE` não altera a linha — cria uma nova versão (MVCC). A otimização *HOT* (*Heap-Only Tuple*) permite que essa nova versão fique na mesma página **sem tocar índice nenhum**, mas só vale quando **nenhuma coluna indexada mudou**.
  >
  > `order_status` é precisamente a coluna que muda a cada transição do ciclo de vida do pedido — é o `UPDATE` mais frequente do sistema. Indexar essa coluna **desliga o HOT** para esse `UPDATE`. Consequências em cadeia: cada transição de status passa a inserir em todos os índices, gera mais WAL, infla os índices, aumenta o trabalho de autovacuum e, como mais WAL significa mais tráfego de replicação, aumenta o **lag das réplicas de leitura** justo no pico. Ou seja: o PR que pretende acelerar a leitura degrada a escrita mais frequente do sistema e, de tabela, a leitura das réplicas. Esse é o argumento que, só ele, justifica reprovar.
  >
  > **6. Competição por `shared_buffers`.** Índice mantido em escrita ocupa cache. Páginas de índice morto disputando buffer com o índice que importa é cache desperdiçado no momento em que ele é mais escasso.
  >
  > **7. Custo de planejamento.** Mais índices, mais caminhos para o *planner* avaliar. Com índices redundantes, `Planning Time` sobe em toda execução e aumenta a chance de escolha pior.
  >
  > **8. Autovacuum e manutenção.** `VACUUM` varre cada índice da tabela. Mais índices, ciclo mais longo, mais chance de o autovacuum não dar conta — o que leva ao *bloat* do passo 4 do diagnóstico, que é um dos candidatos a causa do próprio incidente.
  >
  > **9. Risco de aplicação, não de desenho.** `CREATE INDEX` sem `CONCURRENTLY` toma `ACCESS EXCLUSIVE` na tabela e **bloqueia escrita** até terminar. Numa tabela de dezenas de milhões de linhas, isso é minutos de checkout parado. Se isso está numa migração Flyway que roda no boot da aplicação durante a Black Friday, o PR não é só ineficaz: é um incidente agendado. Esse ponto eu bloquearia independentemente do mérito dos índices.
  >
  > #### O que eu aprovaria no lugar
  >
  > Derivado da consulta real, não da lista de colunas:
  >
  > ```sql
  > -- Opção A: composto, serve "pedidos do cliente" com e sem filtro de status
  > CREATE INDEX CONCURRENTLY idx_pedido_cliente_status
  >     ON pedido (customer_id, order_status);
  >
  > -- Opção B: parcial, se 'processing' for um subconjunto pequeno e quente
  > CREATE INDEX CONCURRENTLY idx_pedido_processing
  >     ON pedido (customer_id)
  >  WHERE order_status = 'processing';
  > ```
  >
  > Prefiro a **Opção B** se o diagnóstico confirmar que pedidos em `processing` são uma fração pequena da tabela — o caso típico, já que o status é transitório e a maioria dos pedidos termina em estado final. Um índice parcial é drasticamente menor, cabe em cache com folga, e só é mantido nas linhas que estão naquele estado. Limitação que preciso aceitar: ele só serve consultas cujo predicado o *planner* consiga provar que implica a condição do índice — se o código amanhã passar o status por parâmetro variável em vez de literal, o índice deixa de ser usado silenciosamente. Por isso a Opção B vem acompanhada de monitoramento de `idx_scan`.
  >
  > Se a tela for "Meus Pedidos" com ordenação por data e paginação, o índice certo é outro: `(customer_id, created_at DESC)`, com paginação por *keyset* (`WHERE created_at < :ultimo`) em vez de `OFFSET` — que custa cada vez mais conforme o usuário avança nas páginas.
  >
  > #### Governança que eu passaria a exigir para índice novo
  >
  > Para que essa conversa não se repita a cada PR, um padrão de revisão:
  >
  > 1. A consulta exata que o índice serve, colada no PR.
  > 2. `EXPLAIN (ANALYZE, BUFFERS)` antes e depois, **em volume de produção**.
  > 3. Impacto medido na escrita: latência de `INSERT`/`UPDATE` e volume de WAL.
  > 4. Declaração explícita se alguma coluna indexada é alterada por `UPDATE` frequente (o teste do HOT, item 5).
  > 5. `CONCURRENTLY` obrigatório, fora da migração de boot da aplicação.
  > 6. Checagem de redundância: índice em `(a)` é dispensável se já existe `(a, b)`, porque o prefixo serve.
  > 7. Data de revisão: se `idx_scan` continuar zero em 30 dias, o índice é removido. Índice tem custo recorrente; precisa justificar permanência, não só criação.
  >
  > #### Como eu provaria que a correção funcionou
  >
  > Três evidências, na ordem: o `EXPLAIN` mostrando `Index Scan` com queda de blocos lidos; o teste de carga reproduzindo o cenário original com p99 e `hikaricp_connections_pending` voltando ao normal; e o contra-teste de escrita — rodar o fluxo de transição de status em volume e confirmar que a latência de `UPDATE` e o lag de réplica não pioraram. Sem a terceira, eu não sei se consertei a leitura quebrando a escrita.
  >
  > **E a ressalva mais importante:** índice pode não ser a causa raiz. Se o passo 0 do diagnóstico mostrar que o tempo está em `acquire` e não em execução, ou se o passo 4 mostrar *bloat* com autovacuum atrasado, o índice não resolve nada — e eu terei degradado a escrita por nada. O problema de fundo do PR não é a escolha dos índices: é ter pulado o diagnóstico.

---

## 4. Resumo das decisões e trade-offs

| Item | Decisão | Trade-off assumido | Alternativa descartada |
| :--- | :--- | :--- | :--- |
| 1.1 Motor | Poliglota, com PostgreSQL 18 como única fonte de verdade do dinheiro | Mais componentes para operar; dado duplicado; pipeline de sincronização com modo de falha próprio | Relacional puro — simples, mas raio de explosão compartilhado entre vitrine e pagamento |
| 1.1 NoSQL complementar | Redis/Valkey e motor de busca na fase 1; colunar e grafo na fase 2 | Cada motor é mais superfície de operação; nenhum é fonte de verdade financeira | Adotar os quatro de uma vez |
| 1.2 Chave primária | UUIDv7 gerado na aplicação, coluna `uuid` nativa | +8 bytes por chave vs `bigint`; vazamento do instante de criação | `IDENTITY` (desabilita batch no Hibernate, exige ida ao banco); UUIDv4 (WAL e cache); TSID (exige coordenação de node id) |
| 2.1 Lock de pagamento | Otimista, com transição condicionada ao estado e idempotência por id de evento | `OptimisticLockException` vira caso normal e exige tratamento explícito em cada escrita | Pessimista — serializa e retém conexão para evitar conflito que quase não ocorre |
| 2.2 Lock de estoque | `UPDATE` condicional atômico + `CHECK` no esquema, com `READ COMMITTED` | Dois caminhos de reserva (comum e SKU quente); lista de SKUs quentes para manter | Otimista (livelock); `SELECT FOR UPDATE` longo (pool); `SERIALIZABLE` (falhas em massa); Redis autoritativo (duas verdades); fila por SKU (muda a UX) |
| 2.3 Riscos | Timeouts agressivos, ordem consistente de lock, pools separados por carga | Mais configuração e mais código de tratamento de erro | Aumentar `maximumPoolSize` |
| 3.1 Diagnóstico | Hipótese escrita com três evidências antes de qualquer mudança | Diagnóstico leva mais tempo que mexer direto | Criar índice e ver se melhora |
| 3.2 Índices | Reprovar o PR; um índice composto ou parcial derivado da consulta, com `CONCURRENTLY` e governança | Índice parcial é frágil a mudança de predicado no código | Um índice por coluna filtrada — mata HOT update em `order_status` e degrada a escrita mais frequente do sistema |

---

## Apêndice A — Referências

Documentação consultada ao escrever este documento:

- PostgreSQL — [Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html) · [Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html) · [`CREATE INDEX`](https://www.postgresql.org/docs/current/sql-createindex.html) · [Heap-Only Tuples (HOT)](https://www.postgresql.org/docs/current/storage-hot.html) · [`pg_stat_statements`](https://www.postgresql.org/docs/current/pgstatstatements.html) · [Monitoring Statistics](https://www.postgresql.org/docs/current/monitoring-stats.html) · [`auto_explain`](https://www.postgresql.org/docs/current/auto-explain.html) · [Extended Statistics](https://www.postgresql.org/docs/current/sql-createstatistics.html)
- PostgreSQL 18 — [Release Notes](https://www.postgresql.org/docs/18/release-18.html): função `uuidv7()` nativa e `BUFFERS` ligado por padrão em `EXPLAIN ANALYZE`
- PostgreSQL 19 — [data de GA em 19/11/2026](https://www.postgresql.org/message-id/f161fbb7-e193-42eb-bec7-4cfc1af74a50@postgresql.org) (anúncio na lista `pgsql-hackers`)
- [RFC 9562 — Universally Unique IDentifiers (UUIDs)](https://www.rfc-editor.org/rfc/rfc9562.html): especificação das versões 6, 7 e 8
- Hibernate ORM 7 — [`UuidVersion7Strategy`](https://docs.hibernate.org/orm/7.2/javadocs/org/hibernate/id/uuid/UuidVersion7Strategy.html)
- HikariCP — [About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

---

## Apêndice B — Armadilhas que este documento evita de propósito

Registro aqui os pontos em que a resposta mais comum para cada item divergiria da minha, porque é onde a defesa oral do trabalho vai se concentrar.

| Item | Resposta comum | O que ela deixa passar |
| :--- | :--- | :--- |
| CAP | Tratar CAP como escolha de duas entre três letras, para todo o sistema | CAP só vale durante partição. A escolha relevante no resto do tempo é latência vs consistência (PACELC), e ela é **diferente** para escrita de pedido e leitura de catálogo |
| NoSQL | Listar famílias com um exemplo cada | Não dizer o que cada motor **cobra**, nem que auditoria financeira não pode sair da transação do fato auditado |
| PK | "UUID porque é distribuído" ou "Sequence porque é rápido" | A mecânica do *rightmost page split*, a amplificação de WAL e seu efeito no lag de réplica; e que `IDENTITY` desabilita batch de insert no Hibernate |
| PK | UUID armazenado em `varchar(36)` | Mais que dobra o tamanho de toda chave e índice que a referencie |
| Lock de estoque | "Lock pessimista com `SELECT FOR UPDATE`" | Que o lock fica retido até o commit, segurando conexão — o que recria o incidente do enunciado. O `UPDATE` condicional é lock pessimista com duração mínima |
| Lock de estoque | Lock otimista para ser "mais escalável" | Que em alta contenção ele produz livelock: a vazão cai conforme a concorrência sobe |
| Lock de estoque | Resolver só no banco | Que 10.000 requisições disputando 50 unidades é problema de admissão, resolvido antes do banco e muito mais barato ali |
| Carrinho multi-item | Não mencionar | Deadlock por ordem invertida de aquisição de lock, que se resolve ordenando os SKUs antes de reservar |
| Pool saturado | Aumentar `maximumPoolSize` | Que pool é fila, não capacidade; pool maior move a fila para dentro do banco, onde custa mais |
| Diagnóstico | Ir direto ao `EXPLAIN` | Que o passo anterior é separar tempo de espera por conexão de tempo de execução — se o tempo está na fila, o índice não resolve |
| Diagnóstico | `EXPLAIN` em base de desenvolvimento | Que o *planner* decide em função do volume; plano em 10 mil linhas não prevê plano em 40 milhões |
| Diagnóstico | Ordenar `pg_stat_statements` por `mean_exec_time` | Que quem derruba o sistema é a consulta medianamente lenta executada o tempo todo — `total_exec_time` |
| Índices | Reprovar citando "custo de escrita" genericamente | Que indexar `order_status` desliga o HOT update na coluna que mais muda, inflando índice, WAL e lag de réplica |
| Índices | Aprovar o índice composto e encerrar | Que `CREATE INDEX` sem `CONCURRENTLY` bloqueia escrita, e que índice parcial costuma ser a resposta melhor para status transitório |
