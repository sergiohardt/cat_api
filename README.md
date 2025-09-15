# API de gatos

# ADR 001 – Arquitetura da Solução "The Cat API Project"

## Status
Aceito – 2025-09-15

## Contexto
O sistema deve coletar dados da **TheCatAPI** (raças, descrições, temperamentos, origens, imagens com filtros específicos) e disponibilizar:
- **APIs REST síncronas** para consulta de raças.
- **Processamento paralelo** para ingestão eficiente das imagens e raças.
- **Persistência em banco** para consultas rápidas e flexíveis.
- **Rotas assíncronas (bônus)** para envio de URLs de imagens via e-mail, com suporte a filas.
- **Logging estruturado** com integração a ferramentas de monitoramento.

O cenário mistura **operações de leitura rápidas** (consultas por filtros) com **operações de ingestão potencialmente demoradas** (coleta de dados externos, fan-out de e-mails).  
Além disso, o projeto deve ser fácil de portar para **AWS** (RDS, Fargate, SQS, SES).

## Decisão
Adotar **Arquitetura Hexagonal (Ports & Adapters)** combinada com **CQRS**:

- **Hexagonal**:  
  - O domínio não depende de frameworks nem de infraestrutura.  
  - Integrações externas (TheCatAPI, DB, filas, e-mail) ficam em *adapters*.  
  - As regras de negócio residem em *use cases* isolados.  

- **CQRS**:  
  - **Ingestão (Write-side)** será **assíncrona**, processada em paralelo, podendo usar fila e workers.  
  - **Consultas (Read-side)** serão **síncronas**, expostas via APIs REST.  
  - Com isso, garantimos **baixo acoplamento** entre ingestão e consulta.  

- **Banco de dados**:  
  - Escolhido **PostgreSQL (RDS)**, devido à necessidade de consultas complexas (por origem, temperamento, filtros textuais) com suporte a índices, arrays e JSONB.  
  - O DynamoDB foi descartado porque exigiria GSIs adicionais e comprometeria flexibilidade em queries ad-hoc.  

- **Concorrência**:  
  - Uso de **Java 21 virtual threads** (ou `CompletableFuture` com `WebClient`) para paralelizar chamadas de imagens por raça.  

- **Logging**:  
  - Uso de **SLF4J + Logback** com formato JSON.  
  - Integração futura com **AWS CloudWatch Logs** (ou ELK/Graylog).  
  - Níveis configurados: `DEBUG`, `INFO`, `WARN`, `ERROR`.  

- **Infraestrutura alvo**:  
  - Local: Docker Compose (Postgres, LocalStack).  
  - Cloud: AWS Fargate + RDS Postgres + SQS + SES.  

## Consequências
### Positivas
- **Alta testabilidade**: casos de uso podem ser testados sem dependências externas.  
- **Baixo acoplamento**: trocar DB, fila ou provedor de e-mail exige apenas novo adapter.  
- **Escalabilidade**: ingestão pode rodar em paralelo em múltiplos workers (EC2/Fargate).  
- **Observabilidade estruturada**: logs centralizados com query em tempo real.  

### Negativas
- **Complexidade inicial maior**: separação em ports/adapters aumenta boilerplate.  
- **Overhead de arquitetura**: para cenários simples (CRUD), pode parecer excessivo.  
- **Custo AWS**: RDS + Fargate + SQS + SES elevam o custo operacional comparado a um monólito simples.  

## Alternativas Consideradas
- **Arquitetura em camadas tradicional**:  
  - Mais simples, mas aumentaria acoplamento entre domínio e infraestrutura.  
  - Menos adequada para troca futura de DB ou fila.  

- **Uso de DynamoDB em vez de RDS**:  
  - Escalável e serverless, mas ineficiente para queries ad-hoc (temperamento + origem).  

- **Somente processamento síncrono**:  
  - Simples de implementar, mas arriscado frente a timeouts e limitações de rate-limit da TheCatAPI.  

## Decisões Futuras
- Avaliar **cache distribuído** (Redis) para respostas mais frequentes (ex.: lista de raças).  
- Estender suporte a **monitoramento distribuído** (tracing com OpenTelemetry).  
- Implementar **retry + backoff exponencial** para chamadas à TheCatAPI.  
- Suportar **multi-tenant** (ex.: diferentes chaves da TheCatAPI por cliente).  
