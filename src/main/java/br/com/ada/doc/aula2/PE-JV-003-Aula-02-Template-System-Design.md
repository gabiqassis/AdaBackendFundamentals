# Template de System Design: Aula 02 - APIs Resilientes

> **Instruções:** Faça uma cópia deste arquivo e preencha as seções abaixo com as conclusões da sua pesquisa e os debates. O foco é a integração da API de Checkout com o Serviço de Pagamento Externo para suportar 10.000 usuários simultâneos.

---

## 0. Premissas e recorte do desenho

Premissa não declarada é furo de avaliação, então declaro tudo antes de decidir qualquer coisa.

**O que é meu e o que é caixa-preta**

| Componente | Responsabilidade |
| :--- | :--- |
| API de Checkout (Spring, Java) | **Minha.** É o que estou desenhando. |
| Banco do Checkout (PostgreSQL) | **Meu.** Mesma transação da aplicação. |
| Parceiro de Pagamentos | **Caixa-preta externa.** Não controlo código, deploy, nem janela de manutenção. |
| App mobile / front-end | **Cliente da minha API.** Posso exigir contrato dele (header obrigatório), mas não controlo o retry dele. |

**Números que o cenário dá**

- 10.000 usuários simultâneos.
- O parceiro de pagamentos é instável.

**Números que o cenário não dá e eu preciso assumir** — são estes que sustentam toda a aritmética de timeout e de bulkhead mais abaixo. Se qualquer um mudar, a configuração muda com ele:

| Premissa | Valor assumido | Por que preciso dela |
| :--- | :--- | :--- |
| Leitura de "10.000 simultâneos" | 10.000 requisições **em voo ao mesmo tempo**, não 10.000 TPS | Define o dimensionamento de threads e de concorrência. Pela Lei de Little, 10k em voo com latência de 1,5s equivale a ~6.600 TPS. Se a leitura certa fosse 10k TPS, a conta de bulkhead dobraria. |
| p99 de latência do parceiro em dia bom | 1,5s | É o piso para calibrar timeout e detecção de lentidão. |
| Teto contratual de TPS do parceiro | 2.000 TPS | Define se RateLimiter entra ou não, e com qual valor. |
| SLA da minha API de Checkout para o app | 5s ponta a ponta | É o orçamento total que preciso respeitar somando tentativas. |
| Réplicas do Checkout em produção | 10 pods | Divide os limites por instância. |
| O parceiro expõe endpoint de consulta de transação | Sim, por `Idempotency-Key` | **Premissa mais crítica do documento.** Sem ela, não existe reconciliação possível após timeout e o desenho da Seção 3 cai. Se o parceiro não oferecer, isso vira requisito contratual bloqueante, não detalhe de implementação. |

**Stack e versões assumidas** (verificadas, não de memória):

- JDK 25 LTS — primeiro LTS a já trazer o JEP 491.
- Spring Boot 4.1.x (a última que verifiquei é a 4.1.1, de 20/08/2026), sobre Spring Framework 7.0.x.
- Resilience4j 2.4.0, módulo `resilience4j-spring-boot4`.
- `spring-boot-starter-aspectj` — no Spring Boot 4 o antigo `spring-boot-starter-aop` foi renomeado. Sem ele os proxies não são criados e as anotações do Resilience4j param de funcionar **em silêncio**, sem erro e sem log.

**Fora de escopo deste documento:** persistência em larga escala, transação distribuída entre serviços, esteira de deploy e observabilidade. Cada um desses é desafio de outro cenário. Aqui o problema é um só: **sobreviver a um parceiro externo instável com 10.000 clientes em voo, sem cobrar ninguém duas vezes e sem perder venda.**

---

## 1. Escolha de Paradigma da API

- **A Escolha:** ( **X** ) Spring MVC com Virtual Threads ou ( ) Spring WebFlux (Reativo)?
- **Justificativa:** Considerando a capacidade de suportar 10k acessos concorrentes, por que você fez essa escolha técnica?

> ### 1.1. Os três caminhos, não dois
>
> O template oferece duas opções, mas o ponto de partida real é um terceiro, que é o que a maioria dos sistemas tem hoje: **Spring MVC com pool de threads de plataforma**. Começo por ele porque é ele que falha, e entender *como* ele falha é o que justifica a escolha entre os outros dois.
>
> **Caminho A — MVC com pool de threads de plataforma (descartado por aritmética)**
>
> O Tomcat trabalha com uma thread por requisição em voo. Enquanto a thread espera a resposta do parceiro, ela está bloqueada e ocupada — não está fazendo nada, mas também não está livre.
>
> Pela Lei de Little, a concorrência sustentável é `vazão × latência`. Invertendo: com 200 threads (o padrão do Tomcat) e p99 de 1,5s no parceiro, a vazão máxima é `200 / 1,5 ≈ 133 req/s`. Os 10.000 usuários simultâneos do cenário precisariam de **~10.000 threads bloqueadas ao mesmo tempo**. Cada thread de plataforma reserva na ordem de 1 MB de stack, o que coloca o custo só de pilha na casa de gigabytes, antes de qualquer heap. Some a isso o custo de context switch do sistema operacional com dezenas de milhares de threads disputando os núcleos.
>
> E o detalhe que mata: quando o parceiro **fica lento** em vez de cair, a latência sobe de 1,5s para 10s, a vazão sustentável cai para `200 / 10 = 20 req/s`, a fila de aceitação do Tomcat enche, e o Checkout para de responder **inclusive nas rotas que não falam com o parceiro**. A lentidão de uma dependência externa virou indisponibilidade total da minha aplicação. É esse acoplamento que preciso quebrar.
>
> **Caminho B — MVC com Virtual Threads (escolhido)**
>
> Cada requisição roda em uma virtual thread. Quando ela bloqueia em I/O, o JDK a desmonta da carrier thread de plataforma, que fica livre para carregar outra virtual thread. O custo de uma virtual thread é de heap, não de stack reservado de SO, e dezenas de milhares delas são normais.
>
> O código continua imperativo e bloqueante: `RestClient`, JDBC, `try/catch`, stack trace legível, debugger que para onde eu pedi, `ThreadLocal` e MDC funcionando. Isso não é conforto, é custo de manutenção e de diagnóstico de incidente às três da manhã.
>
> O risco histórico desse caminho era o **pinning**: até o JDK 23, uma virtual thread que bloqueasse dentro de um bloco `synchronized` ficava presa à carrier thread, e o pool de carriers (paralelismo igual ao número de processadores) esgotava. Com oito núcleos, oito operações bloqueantes dentro de `synchronized` travavam a aplicação inteira. O JEP 491, entregue no JDK 24 e presente no JDK 25 LTS, reimplementou os monitores para que a virtual thread seja identificada por si mesma e não pela carrier — `synchronized` e `Object.wait()` deixaram de prender. Os pins que restam vêm de frames nativos (JNI, FFM) e de inicialização de classe, que são casos estreitos e que eu monitoro (Seção 4). É por isso que a versão de JDK não é detalhe de infraestrutura neste desenho: **no JDK 21 esta escolha seria arriscada; no 25, não é.**
>
> **Caminho C — WebFlux reativo (descartado por custo/benefício, não por capacidade)**
>
> WebFlux é tecnicamente o mais eficiente em uso de recurso por requisição e é o único dos três que oferece **backpressure de ponta a ponta de verdade**, como propriedade do fluxo. Para streaming, SSE, fan-out de muitas chamadas paralelas ou proxy de altíssimo volume, eu escolheria WebFlux.
>
> Descarto aqui por três motivos ligados a *este* cenário:
>
> 1. **O gargalo não é meu, é do parceiro.** A conta é a mesma dos dois lados: nenhum paradigma faz o parceiro responder mais rápido, e nenhum faz o parceiro aceitar mais que os 2.000 TPS contratados. O que eu ganharia de eficiência de CPU e memória não compra vazão nenhuma, porque a vazão está limitada do outro lado do cabo. Virtual Threads já removem o teto de threads, que era o problema real.
> 2. **A cadeia inteira tem que ser não bloqueante, ou não vale nada.** Um `JdbcTemplate` dentro de um `Mono` bloqueia uma thread do event loop, que é um recurso escasso (tipicamente um por núcleo). Um único bloqueio acidental no event loop degrada *toda* a aplicação, de forma muito mais violenta que no modelo de pool. Adotar WebFlux honestamente significa adotar R2DBC, e isso significa reescrever a camada de persistência do Checkout e abrir mão de maturidade de ferramental de transação.
> 3. **Diagnóstico.** Stack trace de pipeline reativo é notoriamente difícil de ler, e `Idempotency-Key` com contexto propagado por `Context`/`ContextView` em vez de MDC adiciona uma camada de erro possível justamente no fluxo que não pode errar. Num serviço que move dinheiro, custo de diagnóstico é um critério de arquitetura, não uma preferência de gosto.
>
> ### 1.2. A decisão
>
> **Spring MVC com Virtual Threads sobre JDK 25 LTS**, habilitado com:
>
> ```properties
> spring.threads.virtual.enabled=true
> ```
>
> Observação importante sobre isso: com Virtual Threads habilitadas, `server.tomcat.threads.max` **deixa de ser o limitador de concorrência**. Quem acreditar que ainda é vai dimensionar o sistema por um número que não tem mais efeito.
>
> ### 1.3. O trade-off assumido — e a consequência que amarra esta seção à próxima
>
> O que eu abro mão: do teto de eficiência e do backpressure nativo do modelo reativo; e assumo o risco residual de pinning por frame nativo, que passa a ser item de monitoração.
>
> Mas o custo que importa de verdade é outro, e é a razão pela qual esta decisão não termina aqui:
>
> **O pool de 200 threads era também o único mecanismo de contrapressão que o sistema tinha, por acidente.** Com 200 threads, nunca existiam mais de 200 chamadas simultâneas ao parceiro — havia um bulkhead implícito, que ninguém configurou e do qual ninguém sabia que dependia. Virtual Threads removem esse teto. Sem um limite explícito, 10.000 requisições em voo se tornam **10.000 chamadas simultâneas a um parceiro que aceita 2.000 TPS**: eu transformo meu ganho de escalabilidade em um ataque de negação de serviço contra a dependência que já estava instável, e o colapso volta pela outra porta.
>
> A própria documentação do Spring Framework 7 é explícita sobre isso ao apresentar o `@ConcurrencyLimit`: limitar concorrência é particularmente útil com Virtual Threads, onde em geral não existe limite de pool em vigor.
>
> Conclusão que levo para a Seção 2: **escolher Virtual Threads torna o limite explícito de concorrência obrigatório, não opcional.** Bulkhead deixa de ser "um dos cinco padrões que eu poderia usar" e passa a ser a peça que repõe a contrapressão que eu removi.

---

## 2. Desenho de Resiliência

Dependemos de uma API terceira instável. Quais recursos do padrão de resiliência você vai utilizar?

- **Proteção contra Falhas e Lentidão:**
  Quais padrões (Circuit Breaker, TimeLimiter, RateLimiter, Bulkhead, Retry) você implementaria na chamada para o Parceiro de Pagamentos? Por que?

> ### 2.1. Resumo da decisão
>
> | Padrão | Entra? | Papel neste cenário | Se eu errar nele, o que acontece |
> | :--- | :--- | :--- | :--- |
> | **Timeout** | **Sim, mas no socket — não via `@TimeLimiter`** | Impedir que uma chamada lenta ocupe recurso indefinidamente | Lentidão do parceiro vira indisponibilidade minha |
> | **Circuit Breaker** | **Sim, com configuração reescrita** | Parar de bater em parceiro degradado e deixá-lo recuperar | Ou nunca abre (default), ou abre por motivo errado e derruba checkout saudável |
> | **Bulkhead** (semáforo) | **Sim — obrigatório** | Repor a contrapressão que Virtual Threads removeram | 10k chamadas simultâneas afogam um parceiro de 2k TPS |
> | **Retry** | **Sim, escopo estreitíssimo** | Absorver falha transitória de rede | Sem jitter e sem idempotência: tempestade de retry e cobrança dupla |
> | **RateLimiter** | **Sim, por contrato — não por estabilidade** | Respeitar o teto de 2.000 TPS acordado | Quebra de contrato comercial / bloqueio pelo parceiro |
>
> ### 2.2. Timeout: a decisão que quase todo desenho erra
>
> O template lista `TimeLimiter` entre as opções, e a resposta instintiva é anotar o método com `@TimeLimiter`. **Nesta arquitetura isso não funciona**, e é importante saber por quê.
>
> O `TimeLimiter` do Resilience4j só opera sobre tipos assíncronos — `CompletableFuture`, `Mono`, `Flux`. O próprio mantenedor do projeto responde isso de forma direta em issue do repositório: só é possível usar TimeLimiter com tipos de retorno CompletableFuture, Reactor ou RxJava2. Aplicado a um método bloqueante que retorna um POJO, ele não impõe prazo nenhum.
>
> Isso deixa duas saídas ruins e uma boa:
>
> - **Ruim 1:** anotar o método bloqueante e seguir a vida. O timeout simplesmente não existe, e ninguém descobre até o incidente.
> - **Ruim 2:** envolver a chamada em `CompletableFuture.supplyAsync(..., executor)` só para habilitar a anotação. Isso reintroduz um pool de threads e **desfaz a escolha da Seção 1**: eu adotei Virtual Threads para não gerenciar pool, e voltaria a gerenciar um — agora escondido dentro do mecanismo de resiliência. O `Bulkhead` do tipo `THREADPOOL` tem o mesmo problema pela mesma razão.
> - **Certa:** **o prazo pertence ao socket, não ao aspecto.** Configuro `connect timeout` e `read timeout` no cliente HTTP. A virtual thread fica bloqueada em I/O com prazo, é desmontada da carrier enquanto espera, e quando o prazo estoura o próprio cliente HTTP devolve exceção. Sem pool extra, sem thread presa, sem camada intermediária.
>
> ```java
> @Bean
> RestClient partnerRestClient(RestClient.Builder builder, PartnerProperties props) {
>     var settings = ClientHttpRequestFactorySettings.defaults()
>         .withConnectTimeout(Duration.ofMillis(500))
>         .withReadTimeout(Duration.ofSeconds(2));
>     return builder
>         .baseUrl(props.baseUrl())
>         .requestFactory(ClientHttpRequestFactoryBuilder.httpComponents().build(settings))
>         .build();
> }
> ```
>
> Um `RestClient` **dedicado ao parceiro**, não o global. O Checkout fala com outras dependências, e cada uma merece seu próprio prazo — um `read timeout` global de 2s estrangularia chamadas internas legítimas mais lentas. Para os defaults da aplicação, `spring.http.client.connect-timeout` e `spring.http.client.read-timeout`; para o parceiro, o bean acima.
>
> **O terceiro timeout, que quase todo desenho esquece: `connectionRequestTimeout`.**
>
> `connect timeout` mede abrir o socket. `read timeout` mede esperar os bytes. Nenhum dos dois cobre o tempo que a thread passa **esperando para pegar uma conexão do pool**. Com Apache HttpClient 5 e pool saturado, a requisição fica parada numa fila antes de sequer tentar conectar — e esse tempo é invisível para os dois timeouts que configurei acima, para o `slowCallDurationThreshold` do disjuntor (que só começa a contar quando a chamada sai) e para qualquer orçamento que eu tenha calculado.
>
> Isso é especialmente grave **por causa da decisão da Seção 1**: com Virtual Threads eu removi o teto de threads, então o pool de conexões HTTP passa a ser o recurso finito de verdade nesse caminho. Dez mil virtual threads disputando um pool de 50 conexões produzem exatamente a latência fantasma que eu disse estar eliminando. O `ClientHttpRequestFactorySettings` não expõe esse parâmetro como propriedade de primeira classe, então ele vai no `RequestConfig` do HttpClient 5 diretamente, junto com o dimensionamento do pool — que eu alinho com o `maxConcurrentCalls` do bulkhead (item 2.4), porque pool menor que o semáforo transforma o semáforo em decoração.
>
> ```java
> // pool e connectionRequestTimeout alinhados ao bulkhead: 300 por instância
> PoolingHttpClientConnectionManager cm = PoolingHttpClientConnectionManagerBuilder.create()
>     .setMaxConnTotal(320)          // folga pequena sobre o semáforo
>     .setMaxConnPerRoute(320)
>     .build();
>
> RequestConfig requestConfig = RequestConfig.custom()
>     .setConnectionRequestTimeout(Timeout.ofMilliseconds(100))  // = maxWaitDuration do bulkhead
>     .build();
> ```
>
> **A aritmética do orçamento de tempo** — é daqui que saem os números, não de chute:
>
> | Item | Valor | Raciocínio |
> | :--- | :--- | :--- |
> | SLA da minha API para o app | 5.000 ms | Premissa declarada |
> | `connectionRequestTimeout` | 100 ms | Alinhado ao `maxWaitDuration` do bulkhead. Espera por conexão é fila, e fila tem que ser curta e visível. |
> | `connect timeout` | 500 ms | Conectar é rápido por natureza. Conexão lenta significa load balancer do parceiro doente — quero descobrir em meio segundo, não em dois. |
> | `read timeout` por tentativa | 2.000 ms | ~1,3× o p99 de 1,5s. Folga para variação normal, apertado o suficiente para que parceiro degradado seja cortado rápido. |
> | Tentativas máximas | 2 (1 retry) | — |
> | Backoff base | 150 ms + jitter | — |
>
> Com dois cenários, porque a diferença entre eles é o que justifica a peça seguinte:
>
> - **Pior caso realista** (pool livre, conexão rápida, leitura estoura): `~70 + 2.000` por tentativa → `2.070 + 150 + 2.070 = 4.290 ms`. Sobram ~700 ms para minha persistência, serialização e rede interna. **Cabe.**
> - **Pior caso patológico** (os três prazos estourando juntos): `100 + 500 + 2.000 = 2.600 ms` por tentativa → `2.600 + 150 + 2.600 = 5.350 ms`. **Não cabe.** Fura o SLA em 350 ms.
>
> Eu não resolvo isso apertando o `read timeout` até o pior caso caber, porque aí eu cortaria chamadas legítimas em dia normal. Resolvo com um **deadline por requisição**: antes de disparar o retry, checo se `tempo decorrido + custo esperado da próxima tentativa` ainda cabe no orçamento de 5s. No caso patológico a conta é `2.750 + 2.600 = 5.350 > 5.000`, então **o retry simplesmente não acontece** — eu devolvo 202 e caio no fluxo de reconciliação em vez de estourar o prazo do cliente.
>
> É por isso que o deadline não é refinamento: **ele é a peça que torna o retry condicional em vez de automático**, e sem ele a minha própria configuração de resiliência fura o meu próprio SLA num cenário que vai acontecer. Um retry que estoura o prazo do cliente não é resiliência, é uma falha mais lenta. O prazo trafega pelo contexto da requisição, e é essa mesma checagem que impede que a soma de camadas bem-intencionadas (retry aqui, retry no cliente HTTP, retry no app mobile) produza uma espera que ninguém orçou.
>
> ### 2.3. Circuit Breaker: os defaults não resolvem o cenário deste template
>
> O Circuit Breaker é o padrão mais necessário aqui — o cenário diz literalmente que o parceiro é instável. E é também onde o erro mais comum acontece: adotar o padrão e manter a configuração padrão.
>
> Os defaults do Resilience4j são: `slidingWindowType` COUNT_BASED, `slidingWindowSize` 100, `minimumNumberOfCalls` 100, `failureRateThreshold` 50%, `slowCallRateThreshold` **100%**, `slowCallDurationThreshold` **60s**, `permittedNumberOfCallsInHalfOpenState` 10, `waitDurationInOpenState` 60s, `automaticTransitionFromOpenToHalfOpenEnabled` false.
>
> Confrontando cada um com este cenário:
>
> **a) `slowCallRateThreshold: 100%` + `slowCallDurationThreshold: 60s` — o default desliga exatamente a proteção que eu preciso.** O modo de falha de um parceiro de pagamentos instável raramente é "cair"; é **ficar lento**. Com o default, uma chamada só conta como lenta acima de 60 segundos, e o disjuntor só abriria se **100%** das chamadas fossem lentas. Na prática: nunca. Eu configuro `slowCallDurationThreshold: 1500ms` (o p99 do parceiro) e `slowCallRateThreshold: 50%`. Assim o disjuntor abre quando metade das chamadas passa do p99 — **antes** da parede dos 2s de timeout, não depois dela. A diferença é detectar degradação enquanto ela é degradação, em vez de esperar que ela vire falha.
>
> **b) `slidingWindowType: TIME_BASED` em vez de COUNT_BASED.** A ~6.600 TPS, uma janela de 100 chamadas é atravessada em 15 milissegundos. Uma janela de contagem nesse volume é uma fotografia de um instante, e o disjuntor passa a oscilar ao ritmo do ruído. Janela temporal raciocina em "os últimos 10 segundos", que é como o incidente de fato se comporta. Vantagem secundária: a janela temporal agrega por segundo, então o custo de memória acompanha o tamanho da janela em segundos, não o volume de chamadas — uma janela de contagem dimensionada para esse volume guardaria dezenas de milhares de medições.
>
> **c) `recordExceptions` / `ignoreExceptions` — a configuração que evita o autogol.** Um HTTP 422 do parceiro dizendo "cartão recusado" ou "saldo insuficiente" **não é falha de infraestrutura, é resposta de negócio**. Se eu deixar 4xx contando como falha, uma manhã com muitas recusas legítimas — um fim de semana de salário atrasado, uma promoção que atraiu cartões sem limite — abre o disjuntor e derruba o checkout de um parceiro que está **perfeitamente saudável**. Só contam: timeout, erro de conexão e 5xx.
>
> **d) `waitDurationInOpenState: 60s` é demais para checkout.** Um minuto de disjuntor aberto em pico de venda é muita receita. Uso 10s, aceitando sondar um parceiro possivelmente ainda doente com mais frequência — risco contido pelos `permittedNumberOfCallsInHalfOpenState`, que limitam o estrago da sondagem.
>
> **e) `automaticTransitionFromOpenToHalfOpenEnabled: true`.** Com o default `false`, a transição de OPEN para HALF_OPEN só acontece quando chega uma chamada. A 6.600 TPS isso é instantâneo, então o comportamento praticamente não muda. O que muda é a **honestidade da métrica**: com `true`, o estado exportado é o estado real, e o painel não mostra OPEN quando o disjuntor já poderia estar sondando. Custa uma thread de scheduler.
>
> ```yaml
> resilience4j:
>   circuitbreaker:
>     instances:
>       parceiroPagamento:
>         slidingWindowType: TIME_BASED
>         slidingWindowSize: 10            # segundos
>         minimumNumberOfCalls: 50
>         failureRateThreshold: 50
>         slowCallDurationThreshold: 1500ms   # = p99 do parceiro
>         slowCallRateThreshold: 50           # default 100 desliga a proteção
>         waitDurationInOpenState: 10s
>         permittedNumberOfCallsInHalfOpenState: 20
>         automaticTransitionFromOpenToHalfOpenEnabled: true
>         recordExceptions:
>           - java.io.IOException
>           - java.net.SocketTimeoutException
>           - br.com.checkout.partner.PartnerServerErrorException
>         ignoreExceptions:
>           - br.com.checkout.partner.PartnerBusinessDeclineException
> ```
>
> **Nota de desenho — o estado do disjuntor é local, e isso é intencional.** Cada um dos 10 pods tem seu próprio Circuit Breaker em memória. Consequências que assumo de forma explícita: (i) os pods aprendem que o parceiro caiu de forma independente, então a proteção entra escalonada em vez de instantânea; (ii) em HALF_OPEN, o parceiro recebe até `10 pods × 20 sondagens = 200` chamadas de sondagem, e é esse número que preciso ter certeza de que ele tolera — não as 20 que eu configurei.
>
> Considerei e **descartei** disjuntor com estado distribuído (em Redis). Motivo: o disjuntor é a peça que tem que funcionar *justamente quando* as dependências estão instáveis. Fazer o estado dele depender de um I/O de rede cria um ponto de falha no mecanismo de proteção contra falhas — e a pergunta "e se o banco cair na hora de atualizar o estado do Circuit Breaker?" deixa de ser hipotética. Estado em memória é uma decisão de disponibilidade, não uma limitação. O preço é a imprecisão escalonada, que o `permittedNumberOfCallsInHalfOpenState` mantém dentro de um envelope conhecido.
>
> ### 2.4. Bulkhead: a peça obrigatória por causa da Seção 1
>
> Tipo **SEMAPHORE**, nunca THREADPOOL — pela mesma razão do item 2.2: um pool de threads desfaria a escolha de Virtual Threads e exigiria retorno assíncrono. Semáforo é um contador, funciona com virtual threads porque não depende da carrier.
>
> Dimensionamento, pela Lei de Little, a partir do teto do parceiro e não do meu desejo:
>
> ```
> concorrência total = TPS contratado × latência = 2.000 × 1,5s = 3.000 chamadas simultâneas
> por instância = 3.000 / 10 pods = 300
> ```
>
> ```yaml
> resilience4j:
>   bulkhead:
>     instances:
>       parceiroPagamento:
>         maxConcurrentCalls: 300
>         maxWaitDuration: 100ms
> ```
>
> `maxWaitDuration: 100ms` e não zero, e não 2s: um valor pequeno absorve rajada curta; um valor grande transforma fila em **latência invisível**, que é o pior dos mundos — o cliente espera sem saber que está na fila, e eu não tenho métrica dizendo que estou saturado, só latência subindo sem causa aparente. Prefiro rejeitar rápido com `BulkheadFullException`, que é um sinal explícito e contável.
>
> **Bulkhead por dependência, não um global.** O Checkout fala com o parceiro de pagamentos, mas também com serviços internos. Um semáforo compartilhado deixa a dependência mais lenta consumir a cota de todas as outras. Um semáforo por dependência é o que garante que parceiro degradado não contamina o resto.
>
> Faço aqui a leitura de 10.000 em voo contra 3.000 de capacidade: **7.000 requisições não vão alcançar o parceiro agora.** Isso não é um defeito do desenho, é a verdade do contrato — o parceiro não aceita mais que 2.000 TPS, e nenhuma arquitetura muda isso. O trabalho da arquitetura é **não perder essas 7.000 intenções de compra**, e é exatamente isso que o terceiro edge case da Seção 3 resolve.
>
> ### 2.5. Retry: o padrão mais perigoso da lista
>
> Retry é o único dos cinco que pode **causar** o incidente que deveria mitigar. Entra com escopo estreito e três travas.
>
> ```yaml
> resilience4j:
>   retry:
>     instances:
>       parceiroPagamento:
>         maxAttempts: 2                  # total de chamadas, inclui a primeira
>         waitDuration: 150ms
>         enableExponentialBackoff: true
>         exponentialBackoffMultiplier: 2
>         exponentialMaxWaitDuration: 1s  # teto, para o backoff não comer o deadline
>         enableRandomizedWait: true      # jitter — não é opcional a 10k
>         randomizedWaitFactor: 0.5
>         retryExceptions:
>           - java.net.SocketTimeoutException
>           - java.io.IOException
>           - br.com.checkout.partner.PartnerServerErrorException
>         ignoreExceptions:
>           - io.github.resilience4j.circuitbreaker.CallNotPermittedException
>           - br.com.checkout.partner.PartnerBusinessDeclineException
> ```
>
> **Trava 1 — jitter é obrigatório, não refinamento.** Com 10.000 requisições em voo e backoff fixo, uma oscilação de 200ms no parceiro produz milhares de retries **sincronizados no mesmo milissegundo**. Isso é uma tempestade de retry: o parceiro que estava se recuperando recebe um pico coordenado e cai de novo, agora por minha causa. `enableRandomizedWait` espalha as tentativas e é a diferença entre ajudar e matar a recuperação.
>
> **Trava 2 — `CallNotPermittedException` tem que estar em `ignoreExceptions`.** Com a ordem padrão de aspectos (próximo item), o Retry envolve o Circuit Breaker. Se o disjuntor estiver aberto, ele devolve `CallNotPermittedException` imediatamente — e um Retry que trate isso como falha transitória vai gastar backoff e orçamento de tempo tentando de novo contra um disjuntor que vai recusar de novo. Resultado: segundos de espera numa chamada que nunca vai sair, durante um incidente. A exclusão é o que faz "disjuntor aberto" significar falha rápida de verdade.
>
> **Trava 3 — retry sobre `POST` de cobrança só é seguro porque existe `Idempotency-Key`.** Repetir uma cobrança cujo destino é desconhecido é como se produz cobrança dupla. Esta trava é o que amarra a Seção 2.5 à Seção 2.6: **sem idempotência, a decisão correta aqui seria `maxAttempts: 1`.** Retry e idempotência não são dois itens do template, são uma decisão só, e a chave tem que ser *a mesma* nas duas tentativas — ponto que detalho a seguir.
>
> ### 2.6. A interação entre os padrões: a ordem importa
>
> A ordem documentada dos aspectos do Resilience4j é esta, de fora para dentro — `Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( Function ) ) ) ) )`, então o Retry é aplicado por último, e pode ser alterada pelas propriedades `retryAspectOrder`, `circuitBreakerAspectOrder` e equivalentes, em que valor maior significa precedência maior.
>
> Com a decisão do item 2.2 — prazo no socket, não via `@TimeLimiter` — a cebola efetiva deste desenho fica:
>
> ```mermaid
> flowchart LR
>     A[Requisição] --> B[Retry]
>     B --> C[CircuitBreaker]
>     C --> D[RateLimiter]
>     D --> E[Bulkhead<br/>semáforo]
>     E --> F[RestClient<br/>connect 500ms<br/>read 2s]
>     F --> G[(Parceiro)]
> ```
>
> **Mantenho o Retry por fora do Circuit Breaker, que é o default, e essa é uma decisão ativa.** As duas opções:
>
> - **Retry por fora (escolhido):** cada tentativa atravessa o disjuntor e é **registrada** por ele. Um parceiro que falha consistentemente acumula falhas mais rápido e o disjuntor abre mais cedo — que é exatamente o que eu quero com um parceiro instável. E no instante em que ele abre, o Retry para de tentar (pela Trava 2).
> - **Retry por dentro:** o disjuntor veria apenas o resultado final da sequência de tentativas, como uma chamada só. Isso **esconde** a falha transitória do disjuntor, que reage tarde; e pior, a duração medida pelo disjuntor passaria a incluir todos os backoffs, poluindo a detecção de lentidão que calibrei com tanto cuidado no item 2.3.
>
> **O preço dessa escolha, que eu assumo e documento:** com o Retry por fora, N tentativas contam como N chamadas nas métricas do disjuntor. `resilience4j.circuitbreaker.calls` passa a medir **tentativas, não requisições de negócio** — a própria documentação do Resilience4j foi ajustada para alertar sobre essa inflação de métrica. Quem ler o painel achando que conta pedidos vai superestimar o volume. A mitigação é um contador de negócio separado, à parte das métricas do Resilience4j.
>
> **Sobre usar o `@Retryable` nativo do Spring Framework 7 em vez do Retry do Resilience4j.** Desde a versão 7.0 o Spring Framework traz resiliência no core, com `@Retryable` e `@ConcurrencyLimit` habilitados por `@EnableResilientMethods`. É uma opção real e recente, e descarto por dois motivos:
>
> 1. **Um modelo mental e uma superfície de métrica.** O core não tem Circuit Breaker, nem RateLimiter, nem `fallbackMethod`, nem integração com Actuator/Micrometer. Como eu preciso de disjuntor de qualquer forma, misturar dois motores de retry no mesmo método me daria duas configurações, duas semânticas e um painel incompleto.
> 2. **A armadilha de contagem, que é um erro de produção esperando acontecer.** `maxAttempts: 3` no Resilience4j significa **3 chamadas no total**. `maxRetries = 3` no Spring Framework significa **4 chamadas** — a documentação oficial é explícita: total de tentativas = 1 tentativa inicial + maxRetries tentativas. Copiar o número de um para o outro pelo nome erra por um, e errar por um em cobrança é uma cobrança a mais.
>
> ### 2.7. RateLimiter: entra por contrato, não por estabilidade
>
> Esta é a decisão mais sutil da lista, e o instinto de "usar todos os cinco" atrapalha. **O RateLimiter não protege a minha estabilidade** — quem faz isso é o Bulkhead, porque quando o parceiro está lento o recurso escasso é *concorrência*, não *taxa*. O RateLimiter existe aqui para **cumprir o teto de 2.000 TPS acordado em contrato** e não ser bloqueado pelo parceiro.
>
> Então: entra, mas pela razão certa. **Se não houvesse cota contratual, eu deixaria fora e diria por quê** — seria uma superfície de configuração e um modo de rejeição a mais, sem ganho de estabilidade.
>
> ```yaml
> resilience4j:
>   ratelimiter:
>     instances:
>       parceiroPagamento:
>         limitForPeriod: 200           # 2.000 TPS ÷ 10 pods
>         limitRefreshPeriod: 1s
>         timeoutDuration: 0            # rejeita na hora; não vira latência escondida
> ```
>
> **Fragilidade que declaro:** dividir a cota pelo número de réplicas quebra no momento em que o autoscaling muda o número de pods. Com 20 pods, o limite efetivo vira 4.000 TPS e eu furo o contrato exatamente no pico, que é quando o autoscaling atua. Mitigação correta de longo prazo: cota distribuída ou aplicação do limite na borda — gateway ou service mesh —, onde existe visão global. Enquanto isso não existe, a divisão por réplica tem que estar amarrada ao `maxReplicas` do HPA, e não ao número de pods de hoje.
>
> ### 2.8. O que não está na lista dos cinco e decide o desenho: a semântica do fallback
>
> Os cinco padrões respondem "como eu falho". Falta responder **"falho dizendo o quê"** — e em pagamento essa é a pergunta mais importante do documento.
>
> Não existe fallback de valor para uma cobrança. Não se degrada graciosamente para uma aprovação fictícia, e não se devolve "recusado" quando o que houve foi um timeout. **Um timeout não significa "não cobrou". Significa "não sei".**
>
> Essa distinção de três estados é o núcleo do desenho:
>
> | Resposta do parceiro | Estado interno | Resposta ao app |
> | :--- | :--- | :--- |
> | 2xx aprovado | `APROVADO` | 201 |
> | 4xx de negócio (recusado) | `RECUSADO` | 200 com motivo |
> | Timeout, 5xx, conexão, disjuntor aberto, bulkhead cheio | **`DESCONHECIDO`** ou `PENDENTE` | **202 Accepted** + `Retry-After` |
>
> Tratar o terceiro grupo como `RECUSADO` é o erro clássico: o cliente vê "falhou", tenta outra forma de pagamento, e horas depois descobre duas cobranças. O estado `DESCONHECIDO` existe para ser resolvido por reconciliação assíncrona, e é isso que o primeiro edge case detalha.

- **Proteção contra Duplicidade:**
  Como a `Idempotency-Key` será utilizada na requisição do Checkout para evitar cobranças duplas? Qual o ciclo de vida dessa chave?

> Sigo o `draft-ietf-httpapi-idempotency-key-header` do grupo de trabalho HTTPAPI da IETF (Standards Track, versão -07 de outubro de 2025 — é rascunho, não RFC publicado, e registro isso porque adotar rascunho é uma decisão, não um detalhe). O ganho de seguir a especificação em vez de inventar: os códigos de erro e a semântica de conflito já estão resolvidos por quem pensou nos casos difíceis, e qualquer cliente que já integra com outro gateway reconhece o comportamento.
>
> Pela especificação, `Idempotency-Key` é um Item Structured Header e seu valor tem que ser uma String.
>
> ### 2.9. São duas chaves, não uma — e confundi-las é o bug clássico
>
> Existem **dois saltos** nessa integração, e cada um tem sua própria chave. Tratar como uma só é a origem de uma classe inteira de cobranças duplas:
>
> ```mermaid
> sequenceDiagram
>     participant App as App mobile
>     participant CK as Checkout (meu)
>     participant PG as Parceiro
>     App->>CK: POST /pagamentos<br/>Idempotency-Key: K1 (por intenção de compra)
>     Note over CK: persiste pagamento + chave K1<br/>na MESMA transação
>     CK->>PG: POST /charges<br/>Idempotency-Key: K2 = f(id do pagamento)
>     Note over CK: retry meu reenvia o MESMO K2
> ```
>
> **Chave K1 — do app para mim.** Gerada pelo **cliente**, uma por *intenção de compra* — no momento em que a tela de pagamento abre ou o pedido é criado, e **não por tentativa de HTTP**. Esse detalhe é tudo: se o app gerar uma chave nova a cada tentativa de rede, o retry dele cria uma segunda cobrança legítima aos meus olhos, e nenhuma proteção minha pega isso. Por isso K1 é requisito de contrato documentado para o cliente, não convenção informal.
>
> **Chave K2 — de mim para o parceiro.** Gerada por mim, **derivada de forma determinística** do identificador do meu pagamento (por exemplo, o UUID do pagamento, ou um hash dele com o número da tentativa de cobrança *do negócio*, nunca da tentativa de rede). Se eu gerasse um UUID novo a cada tentativa, **meu próprio Retry da Seção 2.5 cobraria duas vezes** — eu teria construído a tempestade de cobrança duplicada com as minhas próprias mãos, usando um padrão de resiliência. Derivada e determinística, a mesma chave vai nas duas tentativas, o parceiro reconhece a repetição, e o retry se torna seguro. **É isto que autoriza o `maxAttempts: 2` do item 2.5.**
>
> ### 2.10. Onde a chave é armazenada
>
> **PostgreSQL, no mesmo banco e na mesma transação do pagamento, com `UNIQUE` sobre a chave.**
>
> ```sql
> CREATE TABLE idempotency_record (
>     consumer_id      BIGINT       NOT NULL,
>     idempotency_key  VARCHAR(255) NOT NULL,
>     request_hash     CHAR(64)     NOT NULL,   -- SHA-256 do payload canonicalizado
>     status           VARCHAR(20)  NOT NULL,   -- IN_PROGRESS | COMPLETED | FAILED
>     response_status  INT,
>     response_body    JSONB,
>     payment_id       UUID,
>     created_at       TIMESTAMPTZ  NOT NULL DEFAULT now(),
>     expires_at       TIMESTAMPTZ  NOT NULL,
>     CONSTRAINT pk_idempotency PRIMARY KEY (consumer_id, idempotency_key)
> ) PARTITION BY RANGE (created_at);
> ```
>
> **Por que não Redis como fonte da verdade.** Redis é cache e lock, não garantia de atomicidade. Se ele perder a chave — eviction por memória, failover, restart —, eu cobro duas vezes, e é a pior falha possível neste sistema. Posso usar Redis **na frente**, como caminho rápido para reduzir disputa no banco, mas a autoridade tem que ser o índice único, porque é ele que torna a corrida impossível em vez de improvável. Trade-off aceito: um ida-e-volta ao banco por requisição de pagamento — mitigado pelo fato de que esse mesmo ida-e-volta já é necessário para persistir o pagamento, então não é custo novo, é a mesma transação.
>
> **Chave composta `(consumer_id, idempotency_key)`, não a chave sozinha.** Dois motivos, e o segundo é de segurança, não de correção: (i) dois clientes diferentes não colidem; (ii) sem o escopo, um cliente que adivinhasse ou obtivesse a chave de outro **receberia a resposta armazenada dele** — nome, valor, status do pagamento. Isso é vazamento de informação de terceiro, não apenas um bug de unicidade.
>
> ### 2.11. O ciclo de vida completo
>
> ```mermaid
> stateDiagram-v2
>     [*] --> IN_PROGRESS: 1ª vez (INSERT ganha a corrida)
>     IN_PROGRESS --> COMPLETED: parceiro respondeu (aprovado ou recusado)
>     IN_PROGRESS --> FAILED: estado DESCONHECIDO não resolvido
>     COMPLETED --> [*]: expira (partição descartada)
>     FAILED --> COMPLETED: reconciliação resolveu
> ```
>
> Os cenários de aplicação, mapeados contra a especificação:
>
> | Situação | Decisão | Código HTTP |
> | :--- | :--- | :--- |
> | Chave ausente em operação que a exige | Rejeito, não processo | **400** + `application/problem+json` |
> | Chave nova | Processo. `INSERT` de `IN_PROGRESS` na mesma transação do pagamento | 201 / 200 |
> | Mesma chave, mesmo `request_hash`, `COMPLETED` | Devolvo a **resposta armazenada**, bit a bit — mesmo status, mesmo corpo. Não reexecuto nada | o status original |
> | Mesma chave, mesmo `request_hash`, `IN_PROGRESS` | Há uma requisição em andamento para esta chave | **409 Conflict** |
> | Mesma chave, `request_hash` **diferente** | **Nem devolvo a resposta antiga, nem processo** | **422 Unprocessable Content** |
> | Chave expirada reusada | Rejeito | **422** |
>
> **A linha do 422 é a mais importante da tabela, e é a pergunta explícita do template.** Mesma chave com payload diferente é ambiguidade que não tem resposta certa, só duas respostas erradas: devolver a resposta antiga faz o cliente acreditar que o **novo** valor foi cobrado (cobrei R$ 50 e ele pensa que cobrei R$ 500); processar quebra a promessa da idempotência e cobra duas vezes. A especificação resolve recusando: a chave de idempotência NÃO DEVE ser reusada entre payloads diferentes da mesma operação. A única saída segura é dizer ao cliente que ele está errado.
>
> O `request_hash` é SHA-256 sobre o payload **canonicalizado** — campos relevantes (valor, moeda, pedido, instrumento) em ordem fixa. Canonicalizar importa porque reordenar chaves de um JSON ou mudar espaçamento não é mudar o pedido, e um hash ingênuo sobre o corpo cru produziria 422 em requisições idênticas.
>
> **Concorrência de verdade.** Duas requisições com a mesma chave chegando no mesmo milissegundo em pods diferentes: `SELECT` seguido de `INSERT` é justamente a corrida que eu quero eliminar — ambos os `SELECT` não encontram nada e ambos processam. Quem arbitra é o índice único: um `INSERT` ganha, o outro recebe violação de constraint, e eu traduzo essa violação no caminho do 409 (ou espero e devolvo a resposta armazenada). **A corrida é resolvida pelo banco, não pelo meu código** — e essa é a única forma de resolvê-la com 10 pods.
>
> ### 2.12. Retenção, e por que a tabela é particionada
>
> **TTL de 24h na tabela quente**, e o vínculo `chave → payment_id` guardado **permanentemente no próprio registro do pagamento**, para auditoria.
>
> A janela de 24h precisa cobrir o maior horizonte em que alguém pode repetir: retry do app, meu retry, e o job de reconciliação do edge case 1. Trade-off nos dois sentidos: TTL curto demais e uma repetição tardia cria segunda cobrança; TTL longo demais e a tabela com seu índice único crescem sem fim, num caminho que está no percurso de toda requisição de pagamento. A divisão em "quente 24h" + "auditoria permanente no pagamento" resolve os dois lados.
>
> **Expurgo por partição diária, com `DROP PARTITION` — não com `DELETE` em massa.** Nesse volume, apagar milhões de linhas gera bloat na tabela e no índice, pressão de vacuum e consumo de I/O exatamente no horário em que o sistema deveria estar entregando venda. Descartar partição é operação de metadado.

---

## 3. Os Edge Cases (Engenharia do Caos)

Cole aqui as 3 perguntas de risco que você formulou e defina como a arquitetura responde a cada uma delas.

1. **E se...** o meu timeout de 2s estourar, eu registrar falha, e o parceiro tiver processado a cobrança em 2,3s — debitando o cliente de verdade?

   - **Resposta do Sistema:** Este é o pior caso do sistema inteiro, porque o dinheiro saiu e o meu registro diz que não. A resposta é arquitetural, não um tratamento de exceção.

     **Regra número um: timeout nunca vira `RECUSADO`.** Vira `DESCONHECIDO`. Nenhuma tela, nenhum webhook e nenhum e-mail diz ao cliente que o pagamento falhou com base em um timeout. O app recebe **202 Accepted** com `Retry-After` e a compra aparece como "em processamento".

     **Reconciliação assíncrona resolve o estado.** Um job consulta o parceiro pela `Idempotency-Key` K2 — a mesma que foi enviada — com backoff exponencial e jitter, até obter resposta definitiva. Só então o pagamento transiciona para `APROVADO` ou `RECUSADO`, e só então o cliente é notificado. É aqui que a premissa declarada na Seção 0 cobra seu preço: **sem endpoint de consulta por chave, esse fluxo não existe** e o desenho não fecha. Por isso ela é requisito contratual, não suposição.

     **E a elegância do encaixe:** como K2 é determinística, uma nova tentativa de cobrança com a mesma chave **é** a consulta — o parceiro reconhece a chave e devolve o resultado da operação original em vez de cobrar de novo. O retry deixa de ser um risco e passa a ser o mecanismo de resolução. O que eu nunca faço é disparar cobrança com chave **nova** enquanto o estado está `DESCONHECIDO`: isso é precisamente como se produz a cobrança dupla.

     **Teto de tentativas e alçada humana.** Se a reconciliação não resolver dentro de uma janela definida, o registro vai para uma fila de exceção com alerta — porque divergência financeira não resolvida tem que ter dono, e o último recurso é uma pessoa, não um retry infinito.

2. **E se...** o parceiro responder HTTP 200, rápido, com um corpo dizendo `{"status":"ERRO_INTERNO"}` — ou mudar o contrato de resposta sem avisar?

   - **Resposta do Sistema:** Todo o desenho da Seção 2 observa falha em nível de transporte: status HTTP, timeout, exceção de socket. Um parceiro que devolve 200 em 80ms com erro no corpo é **invisível** para o Circuit Breaker, para o Retry e para qualquer painel baseado em taxa de 5xx. O painel fica verde, a latência fica ótima, e a conversão desaba — e é uma classe de incidente especialmente cruel porque todos os instrumentos dizem que está tudo bem.

     A resposta tem três camadas:

     **Camada 1 — traduzir semântica em exceção antes dos decoradores.** Um `ResponseErrorHandler` no `RestClient` inspeciona o corpo e lança `PartnerServerErrorException` para erro técnico ou `PartnerBusinessDeclineException` para recusa de negócio. Isso acontece **dentro** da função decorada, então o disjuntor e o retry passam a operar sobre a verdade do negócio e não sobre o código de status. Sem isso, as configurações de `recordExceptions` do item 2.3 estão decidindo sobre o sinal errado.

     **Camada 2 — campo desconhecido é falha, não é ignorado.** Se o parceiro muda o contrato e envia um status que eu não conheço, a desserialização falha de forma explícita em vez de deixar o campo nulo seguir adiante. Nulo que escorre por uma máquina de estado de pagamento é como se aprova uma cobrança que nunca aconteceu.

     **Camada 3 — testes de contrato contra a especificação do parceiro, rodando na esteira.** Mudança de contrato é detectada no pipeline, não em produção. Detalho na Seção 4.

3. **E se...** o Circuit Breaker abrir no meio do pico, com 10.000 usuários em voo? O que a plataforma faz com 10.000 pedidos que não podem ser cobrados agora?

   - **Resposta do Sistema:** Esta é a pergunta que o desenho da Seção 2 cria e que ele precisa responder, e é onde a maioria dos desenhos de resiliência silenciosamente perde dinheiro. Disjuntor aberto rejeita em microssegundos. Se a resposta for "503, falha rápida, o padrão funcionou", eu acabei de converter um parceiro lento em **10.000 vendas perdidas** e, pior, em 10.000 apps mobile tentando de novo ao mesmo tempo — uma estampida que atinge o parceiro no exato instante em que ele tentava se levantar.

     Combinado com o item 2.4, a conta já era conhecida: 10.000 em voo contra 3.000 de capacidade de bulkhead significa 7.000 requisições que não alcançam o parceiro **mesmo com tudo saudável**. O número nunca foi "zero rejeição".

     A inversão que resolve: **disjuntor aberto não é caminho de erro, é caminho de fila.**

     - O Checkout **persiste a intenção de pagamento** como `PENDENTE` no meu banco, numa transação que não depende do parceiro para nada, e devolve **202 Accepted** com um identificador e um canal de acompanhamento (polling ou push). A venda não se perde: o que o cliente perde é a confirmação imediata, não o pedido.
     - Um **worker assíncrono** drena essa fila respeitando o bulkhead e o estado do disjuntor, aproveitando a sondagem de HALF_OPEN para retomar vazão de forma gradual em vez de despejar 7.000 chamadas no primeiro sinal de recuperação.
     - **`Retry-After` em toda resposta 202 e 503**, para que a repetição dos clientes seja coordenada por mim em vez de espontânea. Sem esse header, cada app escolhe seu próprio momento e eles coincidem.
     - **A ordem de drenagem é decisão de negócio, não técnica.** FIFO é o padrão óbvio e provavelmente errado: um carrinho de valor alto ou um cliente que já tentou três vezes tem prioridade diferente. Registro isso como ponto que precisa de dono no produto, não como algo que eu decido sozinho na arquitetura.

     O princípio que esse caso revela, e que vale para todo o documento: **resiliência em pagamento não é responder rápido, é não perder a intenção.** Um 503 honesto em 2 milissegundos é um padrão de resiliência aplicado corretamente e um resultado de negócio péssimo.

---

## 4. Testando a Resiliência

- **Estratégia:** Como você vai garantir que o Circuit Breaker abre na hora certa e o Timeout não deixa a Thread presa?

> O critério que uso: **um teste que passa com a configuração errada não é teste.** Asserção sobre tipo de exceção é o exemplo clássico — ela passa igual se o `read timeout` estiver em 2s ou em 30s. Então todo teste abaixo mede **tempo, contagem de chamadas no parceiro, ou estado observável**, nunca só o tipo do erro.
>
> ### Camada 1 — Estado do disjuntor, determinístico e rápido
>
> Teste direto contra o `CircuitBreakerRegistry`, sem rede:
>
> - Alimento falhas até cruzar `minimumNumberOfCalls` e asserto a transição **CLOSED → OPEN**, e que **antes** do mínimo de chamadas o disjuntor **não** abre, mesmo com 100% de falha. Essa segunda asserção é a que pega `minimumNumberOfCalls` mal dimensionado.
> - **Teste específico para lentidão**, porque é o modo de falha do cenário: chamadas que respondem com sucesso mas acima de `slowCallDurationThreshold` têm que abrir o disjuntor. Se este teste passar com `slowCallRateThreshold: 100` (o default), ele está errado — e é o teste que prova que a decisão do item 2.3 foi efetivamente aplicada.
> - OPEN → HALF_OPEN → CLOSED, e HALF_OPEN → OPEN quando a sondagem falha.
> - **Asserto também sobre a métrica `resilience4j.circuitbreaker.state`**, não só sobre o objeto em memória — é a métrica que o painel e o alerta consomem, e testar o objeto sem testar a métrica deixa o painel sem cobertura.
> - Teste de que `PartnerBusinessDeclineException` (cartão recusado) **não** conta como falha: 100 recusas de negócio seguidas, disjuntor fechado no final. É o teste que impede o autogol do item 2.3c.
>
> ### Camada 2 — Parceiro falso com injeção de falha
>
> **WireMock** para o comportamento HTTP: `withFixedDelay` para provar o `read timeout`, cenários com máquina de estado (as 5 primeiras chamadas 500, depois 200) para exercitar abertura e recuperação, e resposta 200 com corpo de erro para cobrir o edge case 2.
>
> **Toxiproxy via Testcontainers** para o que WireMock não alcança, porque ele opera acima do TCP: reset de conexão, conexão meio aberta, estrangulamento de banda e — o mais importante — **black hole** (pacotes descartados sem RST). O black hole é a única forma de exercitar o `connect timeout` de verdade e de provar que o prazo está no socket, e não numa camada de aplicação que só parece funcionar.
>
> **A asserção de timeout é sobre tempo decorrido.** Parceiro atrasado em 10s, `read timeout` em 2s: asserto que a chamada retorna **entre 2,0s e 2,3s**. Com janela nos dois lados, porque retornar em 400ms significa que algo está cortando antes do que eu configurei, e isso também é um defeito.
>
> ### Camada 3 — Provar que o timeout não prende thread
>
> Esta é a parte que a maioria das respostas acena e não demonstra. Prender thread não se prova com teste unitário; se prova sob carga, com instrumento.
>
> Cenário: parceiro estático em 10s de atraso, `read timeout` em 2s, 2.000 requisições concorrentes contra o Checkout. Asserções:
>
> 1. **p99 em torno de 2s**, não em múltiplos de 2s. Se as requisições estivessem enfileirando atrás umas das outras, a latência cresceria em escada — essa é a assinatura de recurso preso.
> 2. **Endpoint de controle, que não toca em nada externo, mantém latência normal durante todo o teste.** Esta é a prova real de isolamento. Se o endpoint de controle degrada enquanto o parceiro está lento, alguma coisa está sendo retida, e é exatamente o acoplamento que eu disse estar quebrando na Seção 1.
> 3. **Evento JFR `jdk.VirtualThreadPinned` com zero ocorrências** durante a janela de carga. Essa é a asserção que prova que as Virtual Threads estão de fato funcionando, e não degenerando silenciosamente. Nota operacional que importa: a flag antiga `-Djdk.tracePinnedThreads` **foi removida no JDK 24**, então runbook que depende dela não funciona mais — JFR é o caminho. No JDK 24+ o evento é emitido em toda ocorrência de pinning e carrega o motivo.
> 4. **`jcmd Thread.dump_to_file -format=json`** para contar virtual threads montadas e desmontadas: milhares desmontadas esperando I/O é o comportamento correto; milhares montadas é pinning.
> 5. **Rejeições de bulkhead aparecem.** `BulkheadFullException` com contagem maior que zero e `resilience4j.bulkhead.available.concurrent.calls` chegando a zero. Ausência de rejeição sob essa carga significa que o semáforo está largo demais e a contrapressão que eu reintroduzi na Seção 1 não está atuando.
> 6. **A rejeição vem do bulkhead, não do pool de conexões.** Teste específico com o pool deliberadamente subdimensionado (20 conexões contra um semáforo de 300): asserto que o erro observado é de espera por conexão e **não** `BulkheadFullException`. É o teste que prova que o alinhamento pool ↔ semáforo do item 2.2 está correto — se o pool estrangula antes do semáforo, meu mecanismo de contrapressão é decoração e a saturação real fica sem métrica. Depois, com o pool em 320, asserto o inverso: a rejeição volta a ser do bulkhead.
>
> ### Camada 4 — Idempotência: o teste tem que ser concorrente
>
> Um teste sequencial de idempotência passa em implementação quebrada, porque não exercita a corrida. Os três que valem:
>
> - **50 requisições em paralelo com a mesma `Idempotency-Key`.** Asserto, com `verify(1, postRequestedFor(...))` no WireMock, que **exatamente uma** cobrança chegou ao parceiro, e que as outras 49 receberam a resposta armazenada ou 409 — nunca uma segunda cobrança. É o único teste que de fato prova "não cobra duas vezes".
> - **Mesma chave, payload alterado → 422.** Com asserção de que o parceiro recebeu **zero** chamadas adicionais.
> - **Retry não duplica:** parceiro falhando na primeira e respondendo na segunda, asserto que as duas tentativas carregaram a **mesma** K2. Se a chave mudar entre tentativas, este teste falha — e é ele que protege a decisão do item 2.9 contra uma refatoração futura bem-intencionada.
>
> ### Camada 5 — Retry e orçamento
>
> - Contagem de tentativas pelo `verify` do WireMock, não pelo log: `maxAttempts: 2` significa 2 chamadas no parceiro, e esse teste é a defesa contra a armadilha de contagem do item 2.6.
> - **Disjuntor aberto não é retentado:** forço OPEN via `transitionToOpenState()`, disparo a chamada e asserto que o parceiro recebeu **zero** chamadas e que a resposta voltou em poucos milissegundos. Sem a exclusão de `CallNotPermittedException` esse teste falha com segundos de espera — é a prova da Trava 2.
> - **Deadline respeitado:** com o orçamento quase esgotado, asserto que o retry **não** é disparado.
>
> ### Camada 6 — Esteira e produção
>
> - **Teste de carga com falha injetada como portão**, k6 ou Gatling a 10k usuários virtuais contra o parceiro falso, com limites de p99 e de taxa de erro. Noturno e não por PR, por custo de execução — e isso é uma escolha consciente: ganho velocidade no PR e aceito descobrir regressão de resiliência em até um dia.
> - **Testes de contrato** (Spring Cloud Contract ou Pact) contra a especificação do parceiro, cobrindo o edge case 2.
> - **Asserto que o alerta dispara, não só que a métrica existe.** Painel verde durante incidente é o modo de falha do edge case 2, e um alerta que nunca foi testado é um alerta que não existe.
> - **Game day em homologação**: abrir o disjuntor com o time presente e verificar se o fluxo de fila do edge case 3 realmente preserva as intenções de compra e se as pessoas sabem o que fazer. Mecanismo de resiliência não exercitado é mecanismo de resiliência presumido.

---

## 5. Decisões que contrariam a resposta padrão — e por quê

Síntese das escolhas em que este desenho divergiu do caminho mais comum, com o trade-off de cada uma explícito.

| # | Resposta comum | Decisão deste desenho | Razão ligada a *este* cenário | Preço que eu pago |
| :--- | :--- | :--- | :--- | :--- |
| 1 | WebFlux, por ser "o reativo escala mais" | MVC com Virtual Threads | O gargalo é o parceiro, não a minha CPU. Reativo não faz um parceiro de 2.000 TPS aceitar mais. | Teto de eficiência menor; pinning por frame nativo vira item de monitoração |
| 2 | Virtual Threads resolvem concorrência | Virtual Threads **exigem** limite explícito | O pool de 200 threads era contrapressão acidental. Removido, 10k em voo afogam o parceiro. | Mais um limite para dimensionar e manter |
| 3 | `@TimeLimiter` no método | Timeout no socket do `RestClient` | `@TimeLimiter` só opera sobre tipos assíncronos; no método bloqueante ele não impõe prazo nenhum | Perco as métricas de timeout do Resilience4j; instrumento no cliente HTTP |
| 3b | Dois timeouts: connect e read | **Três**, com `connectionRequestTimeout` e pool alinhado ao bulkhead | Virtual Threads removem o teto de threads, então o pool de conexões HTTP passa a ser o recurso finito. Espera por conexão é invisível aos outros dois prazos e ao disjuntor | Configuração no `RequestConfig` do HttpClient 5, fora da abstração do Spring |
| 3c | Retry sempre dispara | Retry condicionado a **deadline** | No pior caso patológico a soma das tentativas fura o SLA de 5s em 350 ms | Menos cobertura de falha transitória justamente quando o sistema está pior |
| 4 | Circuit Breaker com config padrão | Config reescrita: janela temporal, `slowCallRateThreshold` 50%, lentidão em 1,5s | O default `slowCallRateThreshold: 100` desliga justamente a detecção de lentidão, que é o modo de falha deste parceiro | Mais parâmetros para calibrar e revisitar quando o p99 do parceiro mudar |
| 5 | 4xx conta como falha | 4xx de negócio **ignorado** pelo disjuntor | Pico de cartões recusados abriria o disjuntor e derrubaria checkout de parceiro saudável | Preciso distinguir recusa de negócio de erro técnico no corpo da resposta |
| 6 | Usar os cinco padrões | RateLimiter entra por **contrato**, não por estabilidade | Com parceiro lento o recurso escasso é concorrência, não taxa. Sem cota contratual, eu deixaria fora | Cota dividida por réplica quebra com autoscaling; mitigação real é na borda |
| 7 | Retry é sempre bom | Retry com jitter, 2 tentativas, deadline, e `CallNotPermitted` excluído | A 10k em voo, retry sem jitter é tempestade sincronizada que mata a recuperação do parceiro | Cobertura menor de falha transitória em troca de não agravar o incidente |
| 8 | Uma `Idempotency-Key` | **Duas** chaves: K1 por intenção do cliente, K2 determinística para o parceiro | Chave nova por tentativa faz o **meu próprio retry** cobrar duas vezes | Contrato explícito com o app sobre quando gerar K1 |
| 9 | Chave de idempotência no Redis | PostgreSQL, índice único, mesma transação | Redis perde chave em eviction ou failover; aqui perder chave é cobrança dupla | Um ida-e-volta ao banco — que já era necessário para persistir o pagamento |
| 10 | Timeout → pagamento recusado | Timeout → **`DESCONHECIDO`** + reconciliação | Timeout não significa "não cobrou", significa "não sei". Marcar recusado é a origem da cobrança dupla vista pelo cliente | Fluxo assíncrono, 202 em vez de resposta imediata, e um job a mais para operar |
| 11 | Disjuntor aberto → 503 rápido | Disjuntor aberto → **fila** + 202 + `Retry-After` | 503 honesto em 2ms é padrão aplicado corretamente e 10.000 vendas perdidas | Complexidade de worker, fila e política de priorização |
| 12 | Teste assertando tipo de exceção | Asserção de **tempo**, **contagem no parceiro** e **ausência de pinning** | Asserção de tipo passa igual com timeout de 2s ou de 30s | Suíte mais lenta e mais caprichosa de manter |

---

## Referências consultadas

- Spring Framework — Resilience Features (`@Retryable`, `@ConcurrencyLimit`, `@EnableResilientMethods`): https://docs.spring.io/spring-framework/reference/core/resilience.html
- Spring Blog — Core Spring Resilience Features: https://spring.io/blog/2025/09/09/core-spring-resilience-features/
- JEP 491: Synchronize Virtual Threads without Pinning: https://openjdk.org/jeps/491
- InfoQ — Virtual Threads After JDK 24: What Changed for Production Java: https://infoq.com/articles/virtual-threads-after-jdk24
- Resilience4j — repositório e módulos: https://github.com/resilience4j/resilience4j
- Resilience4j — limitação do `TimeLimiter` a tipos assíncronos (resposta do mantenedor): https://github.com/resilience4j/resilience4j/issues/430
- Resilience4j — ordem dos aspectos e propriedades `*AspectOrder`: https://godekdls.github.io/Resilience4j/spring-boot-2-getting-started
- Resilience4j — suporte a Spring Boot 4 (`resilience4j-spring-boot4`, 2.4.0): https://github.com/resilience4j/resilience4j/issues/2371
- Defaults do CircuitBreaker (`CircuitBreakerConfig.ofDefaults()`): https://reflectoring.io/circuitbreaker-with-resilience4j/
- Opções e defaults do Resilience4j documentados pelo Apache Camel: https://camel.apache.org/components/eips/resilience4jConfiguration-eip.html
- IETF HTTPAPI — The Idempotency-Key HTTP Header Field (draft-07): https://www.ietf.org/archive/id/draft-ietf-httpapi-idempotency-key-header-07.html
- Spring Boot 4.1.1 release: https://spring.io/blog/2026/08/20/spring-boot-4-1-1-available-now/
- Spring Boot — `ClientHttpRequestFactorySettings` por cliente e discussão de configuração avançada do `RestClient`: https://github.com/spring-projects/spring-boot/issues/48479
- Ausência de propriedade de primeira classe para `connectionRequestTimeout` e configuração via `RequestConfig` do HttpClient 5: https://www.technetexperts.com/spring-boot-httpclient-timeouts/
