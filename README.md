# observability-agent

Este repositório contém o **discovery** de um **Observability & Dependency Analysis Agent**: um estudo de viabilidade técnica, ainda anterior à implementação, documentado em [`discovery.md`](./discovery.md).

## O que é o discovery

Hoje, investigar um incidente exige que um engenheiro correlacione manualmente sinais de fontes diferentes:

- **Splunk** — logs, exceptions, warnings e eventos da aplicação.
- **Prometheus** — CPU, memória, JVM (heap, GC, threads), HTTP, Jetty, HikariCP, Kubernetes e métricas de negócio.
- **Kubernetes** — PODs, restarts, OOMKilled, requests/limits e comportamento dos workloads.
- **Service Dependency Map** — dependências entre serviços, filas, tópicos, bancos, APIs internas e vendors externos.

O discovery avalia a criação de um agente capaz de consultar essas fontes, **correlacionar os sinais** e produzir **hipóteses de causa raiz** e **impacto sistêmico (blast radius)** — indo além de uma leitura isolada de logs.

O agente é, nesta fase, **read-only e apenas investigativo**: não reinicia PODs, não altera configurações ou deployments e não executa nenhuma ação corretiva.

## O problema em um exemplo

Em vez de concluir apenas *"há muitos SocketTimeoutException"*, o objetivo é chegar a uma análise correlacionada no tempo, como:

```text
Possível degradação no vendor-api.

14:01 — latência vendor-api começa a subir
14:04 — threads HTTP ocupadas 58% → 91%
14:05 — heap 61% → 82%
14:07 — HTTP 5xx 0.3% → 7.4%

Serviço causador: vendor-api
Serviços impactados: customer-service, checkout-service, fraud-service
Confiança: HIGH
```

## Arquitetura proposta (resumo)

O LLM nunca acessa as APIs diretamente. Os acessos são encapsulados por MCPs controlados e read-only:

```text
        Observability Agent
                |
          Analysis Engine
         /      |        \
   Splunk MCP  Prometheus MCP  Dependency MCP
        |          |               |
     Splunk    Prometheus     Service Catalog
```

- **MCPs**: recuperação de dados, agregação, cálculos estatísticos, normalização.
- **Agent**: raciocínio, correlação, geração de hipóteses, estratégia de investigação e explicação.

As ferramentas são orientadas ao domínio (ex.: `get_error_rate`, `get_cpu_usage`, `calculate_blast_radius`) em vez de expor queries genéricas abertas (`execute_splunk_query`, `execute_promql`), reduzindo riscos de segurança e custo.

## Temas cobertos no discovery

- Perguntas de acesso a Splunk e Prometheus (APIs, autenticação, limites, retenção, labels).
- Proposta de MCPs (`splunk-observability-mcp`, `prometheus-observability-mcp`, dependency MCP).
- Modelo de dependências e vendor map como *source of truth* versionado.
- Detecção de padrões (error burst, latency degradation, memory/GC pressure, thread starvation, pool exhaustion, vendor degradation, instabilidade no Kubernetes).
- Baseline, correlação temporal e **confidence score**.
- Segurança: filtragem de PII/secrets antes do LLM, auditabilidade e MCPs read-only.
- Controle de custo/contexto: agregação e amostras representativas em vez de milhares de logs.

## Estratégia de implementação

O discovery propõe uma evolução em fases, começando pelos *unknowns* de dados e segurança antes de qualquer agente autônomo:

- **Phase 0** — Discovery (este documento)
- **Phase 1** — MCP Prometheus (somente leitura)
- **Phase 2** — MCP Splunk
- **Phase 3** — Observability Skill (processo investigativo)
- **Phase 4** — Dependency Graph
- **Phase 5** — Observability Agent

## Entregável

O discovery se conclui respondendo às questões de dados, acesso, MCPs, dependências, vendors, métricas, baseline, segurança e blast radius, e termina com uma recomendação: **GO / GO WITH RESTRICTIONS / NO-GO**, com justificativa, riscos, gaps e próximos passos.

Para os detalhes completos, veja [`discovery.md`](./discovery.md).
