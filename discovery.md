# Discovery — Observability & Dependency Analysis Agent

## 1. Contexto

Hoje a investigação de incidentes exige que um engenheiro correlacione manualmente informações provenientes de diferentes fontes:

Splunk → logs, exceptions, warnings e eventos da aplicação.

Prometheus → CPU, memória, JVM, threads, GC, HTTP, pool de conexões, Kubernetes e métricas de negócio.

Kubernetes → PODs, restarts, OOMKilled, requests/limits e comportamento dos workloads.

Service Dependency Map → dependências entre serviços, tópicos, filas, bancos, APIs internas e vendors externos.

O objetivo deste discovery é avaliar a criação de um **Observability Analysis Agent** capaz de consultar essas fontes, correlacionar os sinais e produzir hipóteses de causa raiz e impacto sistêmico.

O agente inicialmente será apenas **read-only e investigativo**.

Não deverá reiniciar PODs, modificar configurações, alterar deployments, executar queries mutáveis ou realizar qualquer ação corretiva automaticamente.

---

# 2. Problema

Durante uma degradação ou incidente, normalmente analisamos isoladamente:

* aumento de erros;
* stack traces;
* warnings;
* CPU;
* memória;
* heap;
* GC;
* threads;
* latência;
* throughput;
* HTTP 4xx/5xx;
* database connection pool;
* filas;
* dependências downstream;
* vendors.

Esses sinais podem estar relacionados.

Exemplo:

```text
payment-service
      ↓
vendor-api
      ↓
latência sobe
      ↓
threads ficam bloqueadas
      ↓
Jetty thread pool aumenta utilização
      ↓
requests acumulam
      ↓
heap cresce
      ↓
GC aumenta
      ↓
CPU sobe
      ↓
latência aumenta ainda mais
      ↓
timeouts
      ↓
HTTP 5xx
```

Uma análise baseada somente em logs poderia concluir:

```text
"Há muitos SocketTimeoutException."
```

O objetivo do agente é chegar a algo como:

```text
Possível degradação no vendor-api.

Evidências:

14:01 — latência vendor-api começa a subir
14:03 — SocketTimeoutException +230%
14:04 — threads HTTP ocupadas passam de 58% → 91%
14:05 — heap passa de 61% → 82%
14:06 — GC pause aumenta
14:07 — HTTP 5xx passa de 0.3% → 7.4%

Serviço possivelmente causador:
vendor-api

Serviço analisado:
payment-service

Serviços potencialmente impactados:
customer-service
checkout-service
fraud-service

Confiança:
HIGH
```

---

# 3. Objetivo do Discovery

Responder se é tecnicamente viável construir um agente capaz de:

```text
telemetry
   ↓
correlation
   ↓
pattern detection
   ↓
hypothesis
   ↓
dependency analysis
   ↓
blast radius
   ↓
incident report
```

O discovery deve evitar começar diretamente pela implementação do agente.

Primeiro deverão ser respondidas as questões relacionadas a dados, APIs, segurança, cardinalidade, dependências e qualidade das métricas.

---

# 4. Perguntas que o Discovery deve responder

## Splunk

Investigar como podemos consultar Splunk programaticamente.

Responder:

```text
Existe API disponível?
Qual mecanismo de autenticação?
Há REST API habilitada?
Há Search Jobs API disponível?
Podemos executar SPL?
Existem limites de query?
Existem limites de retenção?
Quais indexes o time pode acessar?
Como identificar service/application/pod/environment?
Existe traceId?
Existe correlationId?
Existe requestId?
Existe customerId e ele precisa ser mascarado?
Quanto dado uma consulta típica retorna?
```

Determinar se já existe MCP corporativo para Splunk.

Caso não exista, avaliar criação de:

```text
splunk-observability-mcp
```

---

# 5. MCP Splunk

O MCP NÃO deve oferecer uma ferramenta genérica:

```text
execute_splunk_query(spl)
```

na primeira versão.

Isso aumenta riscos de segurança, geração de queries caras e consultas excessivamente abertas.

Preferir ferramentas orientadas ao domínio.

Exemplo:

```text
search_errors(
    service,
    environment,
    start_time,
    end_time
)

search_warnings(
    service,
    environment,
    start_time,
    end_time
)

get_error_rate(
    service,
    start_time,
    end_time
)

get_top_exceptions(
    service,
    start_time,
    end_time,
    limit
)

get_logs_by_trace_id(
    trace_id
)

compare_error_windows(
    service,
    baseline_window,
    incident_window
)
```

O MCP deve aplicar:

```text
maximum time range
maximum number of events
timeouts
pagination
allowed indexes
allowed fields
field masking
query templates
rate limits
```

---

# 6. Prometheus

Investigar se o Prometheus pode ser consultado diretamente pela API HTTP ou se existe alguma camada intermediária corporativa.

Identificar as métricas disponíveis para:

## Kubernetes

```text
container_cpu_usage_seconds_total

container_memory_working_set_bytes

kube_pod_container_resource_requests

kube_pod_container_resource_limits

kube_pod_container_status_restarts_total

OOMKilled
```

## JVM

Investigar especialmente métricas Micrometer/JVM:

```text
jvm_memory_used_bytes

jvm_memory_max_bytes

jvm_gc_pause_seconds

jvm_gc_memory_allocated_bytes_total

jvm_threads_live_threads

jvm_threads_peak_threads

jvm_classes_loaded_classes

process_cpu_usage

system_cpu_usage
```

Validar quais métricas realmente existem no ambiente.

Não assumir nomes sem verificar.

## HTTP

Verificar existência de:

```text
request rate
error rate
latency
p95
p99
4xx
5xx
timeouts
```

## Jetty

Identificar métricas disponíveis relacionadas a:

```text
thread pool size
threads busy
threads idle
threads max
queue size
connections
requests
```

## HikariCP

Identificar:

```text
active connections
idle connections
pending connections
max connections
connection acquire time
connection timeout
```

---

# 7. MCP Prometheus

Avaliar criação de:

```text
prometheus-observability-mcp
```

Evitar inicialmente uma ferramenta aberta:

```text
execute_promql(query)
```

Preferir ferramentas semânticas.

Exemplo:

```text
get_cpu_usage(service, window)

get_memory_usage(service, window)

get_heap_usage(service, window)

get_gc_activity(service, window)

get_thread_pool_usage(service, window)

get_http_metrics(service, window)

get_pod_restarts(service, window)

get_hikari_metrics(service, window)

compare_metrics(
    service,
    metric,
    baseline_window,
    incident_window
)
```

Uma ferramenta PromQL genérica poderá existir posteriormente para investigação avançada, com controles adicionais.

---

# 8. Dependency Map

O agente precisa conhecer as dependências entre serviços.

Investigar qual fonte deve ser considerada o **Source of Truth**.

Possíveis fontes:

```text
service catalog
Backstage
Compass
Confluence
repository metadata
Kubernetes
OpenTelemetry traces
Terraform
manual YAML
```

Caso ainda não exista um mapa confiável, propor inicialmente um manifesto versionado no Git.

Exemplo:

```yaml
service: payment-service

dependencies:

  services:
    - customer-service
    - fraud-service

  queues:
    - payment-processing
    - fraud-analysis

  databases:
    - payment-db

  vendors:
    - vendor-x

consumers:
  - checkout-service
  - account-service
```

O agente deve conseguir navegar o grafo nos dois sentidos.

```text
payment-service
     |
     +--> customer-service
     |
     +--> vendor-x
     |
     +--> payment-db
```

e:

```text
payment-service
       ↑
       |
checkout-service

account-service
       ↓
payment-service
```

Isso permitirá calcular o **blast radius**.

---

# 9. Vendor Map

Dependências externas devem ser entidades de primeira classe.

Exemplo:

```yaml
vendors:

  vendor-x:
    type: REST_API
    criticality: HIGH

    services:
      - payment-service
      - customer-service

    timeout: 3s
```

O agente deve conseguir responder:

```text
vendor-x está degradado.

Quais serviços dependem dele?
```

Resultado esperado:

```text
DIRECT

payment-service
customer-service


TRANSITIVE

checkout-service
account-service
fraud-service
```

---

# 10. Modelo de arquitetura proposto

Separar claramente:

```text
                Observability Agent
                        |
                Analysis Engine
                 /      |       \
                /       |        \
           Splunk    Prometheus   Dependency Graph
              |          |              |
              |          |              |
         Splunk MCP  Prometheus MCP   Dependency MCP
              |          |              |
           Splunk     Prometheus      Service Catalog
```

O LLM NÃO deve acessar diretamente:

```text
Splunk API
Prometheus API
Kubernetes API
```

Esses acessos deverão ser encapsulados por ferramentas controladas.

---

# 11. Pipeline de investigação

O comportamento esperado do agente deve ser semelhante a:

```text
User
 |
 | "Analyze payment-service last 30m"
 |
 v
Observability Agent
 |
 +--> get_error_rate()
 |
 +--> get_top_exceptions()
 |
 +--> get_cpu_usage()
 |
 +--> get_memory_usage()
 |
 +--> get_gc_activity()
 |
 +--> get_http_metrics()
 |
 +--> get_thread_pool_usage()
 |
 +--> get_dependencies()
 |
 +--> get_dependents()
 |
 v
Correlation Engine
 |
 v
Hypotheses
 |
 v
Blast Radius
 |
 v
Incident Analysis
```

---

# 12. Detecção de padrões

O discovery deve avaliar a capacidade de identificar pelo menos:

## Error burst

```text
baseline

5 errors/min

incident

300 errors/min
```

## Latency degradation

```text
p95

150ms
↓
900ms
↓
2.5s
```

## Memory growth

```text
heap

45%
51%
61%
72%
83%
```

Possível:

```text
memory pressure
memory leak
large workload
cache growth
```

## GC pressure

Correlacionar:

```text
heap ↑
allocation rate ↑
GC frequency ↑
GC pause ↑
CPU ↑
```

## Thread starvation

Correlacionar:

```text
busy threads ↑
available threads ↓
latency ↑
timeouts ↑
```

## Database pool exhaustion

Correlacionar:

```text
Hikari active ≈ max
pending ↑
connection acquire time ↑
HTTP latency ↑
```

## Vendor degradation

Correlacionar:

```text
vendor latency ↑
timeouts ↑
SocketTimeoutException ↑
service latency ↑
5xx ↑
```

## Kubernetes instability

Identificar:

```text
restart
OOMKilled
CPU throttling
memory limit
CrashLoopBackOff
pod churn
```

---

# 13. Baseline

Não utilizar somente limites absolutos.

Exemplo ruim:

```text
CPU > 80% = problema
```

O comportamento normal de um serviço pode ser 75%.

Preferir comparação com baseline:

```text
current

CPU = 82%

baseline same period

CPU = 32%
```

Anomalia:

```text
+156%
```

Avaliar baselines:

```text
previous 30 minutes
same hour yesterday
same hour previous 7 days
rolling average
rolling median
p95 historical
```

Para MVP, começar simples:

```text
incident window

versus

previous equivalent window
```

---

# 14. Correlação temporal

Os sinais devem ser organizados cronologicamente.

Exemplo:

```text
13:58 vendor latency increases

14:01 SocketTimeoutException starts

14:03 Jetty busy threads increases

14:04 request latency increases

14:05 heap increases

14:06 GC pressure detected

14:07 5xx increases
```

A ordem temporal deve influenciar a hipótese de causa raiz.

Correlação não significa causalidade.

O agente deverá diferenciar:

```text
evidence
correlation
hypothesis
confirmed cause
```

---

# 15. Confidence Score

Toda hipótese deve possuir nível de confiança.

Exemplo:

```text
HIGH

multiple independent signals
strong temporal correlation
dependency relationship identified


MEDIUM

signals correlated
insufficient dependency evidence


LOW

single signal
weak temporal relationship
```

Nunca apresentar uma hipótese como causa confirmada sem evidência suficiente.

---

# 16. Resposta esperada do agente

Formato sugerido:

```text
INCIDENT ANALYSIS

Service:
payment-service

Period:
14:00–14:30


SUMMARY

High probability of vendor-api degradation.


EVIDENCE

14:02 vendor latency +340%

14:03 SocketTimeoutException +270%

14:05 Jetty busy threads:
62% → 94%

14:06 HTTP p95:
220ms → 2.8s

14:07 HTTP 5xx:
0.2% → 8.1%


JVM

Heap:
normal

GC:
normal

CPU:
+22%

Threads:
high utilization


DEPENDENCY

payment-service
      ↓
vendor-api


POTENTIAL BLAST RADIUS

checkout-service
customer-service
fraud-service


HYPOTHESIS

External dependency degradation caused request
accumulation and thread pool saturation.


CONFIDENCE

HIGH


RECOMMENDED CHECKS

Check vendor-api availability.

Verify timeout and retry metrics.

Compare with other consumers of vendor-api.
```

---

# 17. Segurança

O discovery deverá verificar:

```text
PII nos logs
customer identifiers
tokens
Authorization headers
credentials
database credentials
API keys
request bodies
vendor payloads
```

Informações sensíveis devem ser filtradas antes de chegar ao LLM.

MCPs devem inicialmente ser:

```text
READ ONLY
```

Nunca expor tokens diretamente ao modelo.

Adicionar auditabilidade:

```text
who executed
which tool
which service
time range
query duration
number of records
```

---

# 18. Controle de custo e contexto

O agente NÃO deve enviar milhares de logs para o LLM.

Fluxo preferencial:

```text
Splunk
  ↓
aggregation
  ↓
grouping
  ↓
top patterns
  ↓
representative samples
  ↓
LLM
```

Exemplo:

```text
12,431 errors
```

não devem virar 12.431 eventos no contexto.

Transformar em:

```text
SocketTimeoutException        8,421
ConnectionResetException     2,311
DatabaseTimeoutException       921
Other                          778
```

Enviar ao agente apenas:

```text
counts
trend
first occurrence
last occurrence
representative examples
```

---

# 19. Estratégia de implementação

## Phase 0 — Discovery

Responder:

```text
Quais APIs estão disponíveis?

Quais métricas existem?

Quais labels estão padronizados?

Como serviço/environment/pod são identificados?

Qual volume médio de logs?

Como consultar Splunk?

Como consultar Prometheus?

Existe service catalog?

Existe dependency map?

Como vendors são identificados?

Existe OpenTelemetry?

Existe traceId/correlationId?

Existem restrições de segurança para MCP?

Kiro pode acessar esses MCPs no ambiente corporativo?
```

Entregável:

```text
discovery.md
```

---

## Phase 1 — MCP Prometheus

Implementar somente leitura.

Objetivo:

```text
analyze payment-service metrics for last 30 minutes
```

Resultado:

```text
CPU
memory
heap
GC
threads
HTTP
Hikari
pod restart
```

Nenhum LLM avançado necessário ainda.

---

## Phase 2 — MCP Splunk

Adicionar:

```text
error rate
warning rate
exception grouping
exception trends
representative logs
```

Resultado:

```text
top error patterns
error anomalies
warning anomalies
```

---

## Phase 3 — Observability Skill

Antes de criar um agente autônomo, criar uma Skill contendo o processo investigativo.

Exemplo conceitual:

```text
observability-analysis
```

A Skill deverá ensinar:

```text
what to query
query order
baseline strategy
correlation strategy
confidence classification
output format
```

Isso permitirá validar o raciocínio antes de aumentar a autonomia.

---

## Phase 4 — Dependency Graph

Adicionar:

```text
get_dependencies(service)

get_dependents(service)

get_vendors(service)

get_databases(service)

get_queues(service)

calculate_blast_radius(service, depth)
```

---

## Phase 5 — Observability Agent

Somente depois das ferramentas estarem estáveis criar:

```text
observability-agent
```

Responsabilidades:

```text
orchestrate MCP tools
choose investigation strategy
correlate signals
construct timeline
generate hypotheses
calculate blast radius
generate report
```

---

# 20. Questão arquitetural importante

O Agent NÃO deve ser responsável por calcular tudo.

Preferir:

```text
                Agent
                  |
          reasoning/orchestration
          /       |        \
         /        |         \
   Splunk MCP  Metrics MCP  Dependency MCP
       |            |            |
 aggregation    calculations    graph traversal
```

Ou seja:

MCP:

```text
data retrieval
aggregation
statistical calculations
normalization
```

Agent:

```text
reasoning
correlation
hypothesis generation
investigation strategy
explanation
```

---

# 21. MVP recomendado

Evitar inicialmente:

```text
automatic alerts
machine learning
automatic remediation
production changes
continuous monitoring
complex anomaly detection
full Kubernetes access
```

MVP:

```text
User:
"Analyze service X for last 30 minutes"

Agent:

1. Splunk errors
2. Splunk warnings
3. CPU
4. memory
5. JVM
6. HTTP
7. thread pool
8. Hikari
9. POD restart
10. dependencies

↓

timeline

↓

patterns

↓

hypothesis

↓

blast radius

↓

report
```

---

# 22. Critérios de sucesso

O discovery estará concluído quando for possível responder:

```text
1. Quais fontes de dados serão utilizadas?

2. Como cada fonte será acessada?

3. Quais MCPs precisam ser construídos?

4. Quais tools cada MCP deverá fornecer?

5. Qual será o source of truth das dependências?

6. Como identificar vendors?

7. Quais métricas JVM/Kubernetes estão disponíveis?

8. Como definir baseline?

9. Como limitar dados enviados ao LLM?

10. Como proteger PII e secrets?

11. Como calcular blast radius?

12. Como distinguir correlação de causa?

13. Como calcular confidence?

14. Qual será o MVP?

15. Quais gaps impedem a implementação?
```

---

# 23. Entregáveis esperados do Kiro

Ao executar este discovery, gerar:

```text
docs/discovery/observability-agent/
│
├── discovery.md
├── data-sources.md
├── splunk-analysis.md
├── prometheus-analysis.md
├── dependency-model.md
├── security-analysis.md
├── mcp-proposal.md
└── architecture.md
```

Não implementar o agente antes que os principais unknowns estejam respondidos.

Ao final, produzir uma recomendação:

```text
GO
GO WITH RESTRICTIONS
NO-GO
```

incluindo justificativa, riscos, gaps e próximos passos.
