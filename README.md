
# 1. O Problema / Contexto Atual
O sistema atual é composto por:

- Front-end legado monolítico
- Backend legado acoplado
- Banco de dados único
- Autenticação própria e não padronizada
- Baixa escalabilidade
- Dificuldade de manutenção
- Risco operacional ao evoluir funcionalidades

O negócio exige:

- Modernização sem interrupção
- Melhor performance
- Segurança unificada
- Escalabilidade sob demanda
- Evolução contínua
- Migração sem “big bang”



# 2.Objetivo da Migração
Modernizar o front com Blazor WebAssembly.
Modernizar o backend com arquitetura serverless.
Garantir segurança com OAuth2 + OpenID Connect.
Migrar sem downtime.
Manter o legado funcionando até o fim.
Toda a plataforma — legado e novo — usa **o mesmo Identity Provider** (ex.: Cognito OIDC).

### Benefícios:
- SSO entre front legado e novo front
- Tokens JWT padronizados
- Segurança consistente
- Autorização multi-tenant via claims
- Migração suave sem múltiplos logins

### Fluxo:
- Front legado → IdP
- Novo front Blazor → IdP
- API Gateway → valida JWT
- Lambdas → usam claims (`merchantId`, `roles`)


# 3.Estratégia de Migração — Strangler Fig Pattern
1. **Manter o legado funcionando**
2. Criar o novo sistema ao lado do legado
3. Redirecionar funcionalidades específicas para o novo front/backend
4. Usar **CDC** para sincronizar dados entre legado e novo banco
5. Expandir o novo sistema gradualmente
6. “Estrangular” o legado até substituí-lo por completo



# 4.Arquitetura Atual (Legado)

```mermaid

flowchart TD

    User["🧑‍💼 Comerciante (Front Legado)"]

    subgraph Legacy["🏢 Sistema Legado"]
        LegacyFront["🖥️ Front-End Legado"]
        LegacyAPI["🔧 API Legada"]
        LegacyDB["🗄️ Banco de Dados Legado"]
    end

    User --> LegacyFront
    LegacyFront --> LegacyAPI
    LegacyAPI --> LegacyDB
```



# 5.🏗️ Arquitetura Final (Novo sistema)
## 🧾 Sistema de Fluxo de Caixa, Consolidação Diária e Relatórios  

- Registro de lançamentos (débito/crédito)  
- Consolidação diária assíncrona  
- Relatórios rápidos  
- Arquitetura serverless  
- Alta escalabilidade e baixo acoplamento  
- Migração gradual de ambiente legado  
---

```mermaid

flowchart TD
    User["🧑‍💼 Comerciante<br/>Blazor WebAssembly"]

    subgraph Edge["🌎 CDN + Static Web"]
        CF["🌐 CloudFront"]
        S3["📦 S3 Static Website<br/>Blazor WASM"]
    end

    subgraph AWS_Cloud["☁️ AWS Cloud (Backend)"]
        
        Cognito["🔐 Cognito<br/>OAuth2 + OIDC"]
        APIGW["🛡️ API Gateway<br/>Validação JWT"]
        LambdaL["⚡ Lambda Lançamentos"]
        LambdaC["⚡ Lambda Consolidação"]
        LambdaR["⚡ Lambda Relatórios"]
        SQS["📬 SQS / EventBridge<br/>Eventos Assíncronos"]
        Aurora["🗄️ Aurora Serverless v2<br/>Banco ACID"]
        Redis["🚀 Redis (ElastiCache)<br/>Saldos Consolidados"]
        CloudWatch["📊 CloudWatch<br/>Logs / Métricas / Alarmes"]
    end

    User --> CF
    CF --> S3
    S3 --> User

    User --> Cognito
    User --> APIGW

    APIGW --> LambdaL
    APIGW --> LambdaR

    LambdaL --> Aurora
    LambdaL --> SQS

    SQS --> LambdaC
    LambdaC --> Redis

    LambdaR --> Redis
    LambdaR --> Aurora

    LambdaL --> CloudWatch
    LambdaC --> CloudWatch
    LambdaR --> CloudWatch

```


# 6. Arquitetura de Transição (Migração do Legado) - Strangler

```mermaid
flowchart LR

    User["🧑‍💼 Comerciante"]

    subgraph IdP["🔐 Identity Provider<br/>(Cognito / OIDC)"]
        Auth["Emissão de Tokens<br/>OAuth2 + OpenID Connect"]
    end

    %% FRONT-ENDS
    subgraph Fronts["Interfaces"]
        LegacyFront["🖥️ Front-End Legado<br/>(Integrado ao IdP)"]
        NewFront["🌐 Novo Front Blazor<br/>S3 + CloudFront<br/>(OIDC)"]
    end

    %% LEGADO
    subgraph Legacy["🏢 Sistema Legado"]
        LegacyAPI["🔧 API Legada"]
        LegacyDB["🗄️ Banco Legado"]
    end

    %% MIGRAÇÃO
    subgraph Migration["🔄 Migração (Strangler Fig)"]
        CDC["🔁 CDC / Replicação de Dados"]
    end

    %% NOVO BACKEND
    subgraph NewBackend["☁️ Novo Backend AWS"]
        APIGW["🛡️ API Gateway<br/>Authorizer OIDC"]
        LambdaRel["⚡ Lambda Relatórios"]
        LambdaLanc["⚡ Lambda Lançamentos"]
        Aurora["🗄️ Aurora Serverless"]
        Redis["🚀 Redis (Cache de Saldos)"]
    end

    %% FLUXOS BÁSICOS

    User --> LegacyFront
    User --> NewFront

    %% Ambos os fronts usam o MESMO IdP
    LegacyFront --> Auth
    NewFront --> Auth

    %% Front legado ainda chama APIs legadas
    LegacyFront --> LegacyAPI
    LegacyAPI --> LegacyDB

    %% CDC para alimentar Aurora
    LegacyDB --> CDC --> Aurora

    %% Relatórios: front legado redireciona para novo front
    LegacyFront -->|Relatórios: redirect| NewFront
    NewFront --> APIGW
    APIGW --> LambdaRel
    LambdaRel --> Redis
    LambdaRel --> Aurora

    %% Lançamentos migrados: front legado passa a chamar novo backend
    LegacyFront -->|Lançamentos migrados| APIGW
    APIGW --> LambdaLanc
    LambdaLanc --> Aurora


```

## Fluxo de Migração (Simplificado)


```mermaid

flowchart TD

    A["🏢 1. Sistema Legado em Produção"] --> B["🌱 2. Criar Novo Front Blazor<br/>em S3 + CloudFront"]
    B --> C["🔐 3. Integrar Cognito (OIDC)"]
    C --> D["⚡ 4. Criar Novas APIs Serverless<br/>(API Gateway + Lambda)"]
    D --> E["🔀 5. Redirecionar Funcionalidades<br/>Específicas para o Novo Backend"]
    E --> F["🌳 6. Expandir o Novo Sistema<br/>e Estrangular o Legado"]
    F --> G["🛑 7. Desligar o Legado"]


```



# 7. Domínios Funcionais e Capacidades (Arquitetura de negócio)


**Lançamentos**
---------------

*   Registrar lançamento
    
*   Consultar lançamentos
    
*   Auditar histórico
    
*   Publicar evento “LançamentoCriado”
    

**Consolidação**
----------------

*   Processar eventos
    
*   Calcular saldo diário
    
*   Atualizar Redis
    
*   Reprocessar falhas (DLQ)
    

**Relatórios**
--------------

*   Consultar saldo diário
    
*   Gerar relatórios por período
    
*   Fallback para Aurora
    

**Segurança**
-------------

*   Autenticação (Cognito)
    
*   Autorização por comerciante
    
*   Tokens JWT
    

**Observabilidade**
-------------------

*   Logs estruturados
    
*   Métricas técnicas e de negócio
    
*   Alarmes
    
*   Tracing (X-Ray)
    

#8.  Requisitos Funcionais (RF)


*   **RF01** Registrar lançamento financeiro
    
*   **RF02** Consultar lançamentos
    
*   **RF03** Publicar evento de lançamento criado
    
*   **RF04** Processar eventos de lançamento
    
*   **RF05** Atualizar saldo diário consolidado
    
*   **RF06** Registrar falhas em DLQ
    
*   **RF07** Consultar saldo diário
    
*   **RF08** Gerar relatórios consolidados
    
*   **RF09** Autenticação via Cognito
    
*   **RF10** Autorização por comerciante
    
*   **RF11** Registrar logs estruturados
    
*   **RF12** Monitorar filas, erros e latência
    

# 9. Requisitos Não Funcionais (RNF)


### **Desempenho**

*   Saldo diário: < 50 ms (Redis)
    
*   Registro de lançamento: < 200 ms
    

### **Escalabilidade**

*   Suporte a 50 req/s
    
*   Fila absorve picos
    

### **Disponibilidade**

*   Multi‑AZ
    
*   Tolerância a falhas
    
*   Independência entre serviços
    

### **Segurança**

*   TLS obrigatório
    
*   JWT
    
*   IAM least privilege
    
*   Criptografia KMS
    

### **Manutenibilidade**

*   Baixo acoplamento
    
*   Observabilidade completa
    

### **Custo**

*   Pay‑per‑use
    
*   Cache reduz carga no Aurora
    

# 10. Justificativa da Arquitetura e Tecnologias

Atributos:
### ✅ Escalabilidade

Cada serviço escala de forma independente.

### ✅ Disponibilidade

Falhas isoladas não derrubam o sistema.

### ✅ Performance

Relatórios via Redis, lançamentos via Lambda, consolidação assíncrona.

### ✅ Segurança

Princípio de menor privilégio, JWT por serviço, superfícies menores.

### ✅ Observabilidade

Logs, métricas e alarmes por domínio.

### ✅ Manutenibilidade

Evolução contínua sem impacto no restante.

### ✅ Custo

Pay-per-use, cache reduz carga, Aurora Serverless ajusta capacidade.

### ✅ Suporte ao Strangler Fig Pattern

Permite substituir o legado por partes.

Produtos:

### **Serverless**

*   Escalabilidade automática
    
*   Baixo custo
    
*   Alta disponibilidade
    
*   Zero manutenção
    

### **Aurora Serverless**

*   Transações ACID
    
*   Consistência forte
    
*   SQL completo
    

### **Redis**

*   Leitura ultrarrápida
    
*   Ideal para saldos consolidados
    

### **SQS/EventBridge**

*   Desacoplamento total
    
*   Resiliência
    
*   Reprocessamento via DLQ
    

### **Lambda**

*   Simples
    
*   Escalável
    
*   Barato


# 11. Monitoramento e Observabilidade**

### **Logs**

*   CloudWatch Logs
    
*   Logs estruturados (JSON)
    

### **Métricas**

*   Latência
    
*   Erros
    
*   Tamanho da fila
    
*   Cache hit/miss
    

### **Alarmes**

*   DLQ > 0
    
*   Latência alta
    
*   Erros de Lambda
    

### **Tracing**

*   AWS X-Ray


# 12. Segurança e Integração**
==========================

### **Autenticação e Autorização**

*   Cognito + JWT
    
*   Claims com comercianteId
    

### **Comunicação Segura**

*   TLS 1.2+
    
*   VPC privada
    
*   SGs restritivos
    

### **IAM Least Privilege**

*   Cada Lambda só acessa o que precisa
    

### **Auditoria**

*   CloudTrail
    
*   Logs de acesso
    
*   Logs de falha



# 13. Diagramas de Sequência (High-Level)

## Registrar Lançamento
```mermaid

sequenceDiagram
    participant User as Usuário
    participant APIGW as API Gateway
    participant LambdaL as Lambda Lançamentos
    participant Aurora as Aurora
    participant SQS as SQS/EventBridge

    User->>APIGW: POST /lancamentos
    APIGW->>LambdaL: Invoca função
    LambdaL->>Aurora: Salva lançamento
    LambdaL->>SQS: Publica evento "LançamentoCriado"
    APIGW->>User: Sucesso

```

## Consolidação
```mermaid
sequenceDiagram
    participant SQS as SQS/EventBridge
    participant LambdaC as Lambda Consolidação
    participant Redis as Redis

    SQS->>LambdaC: Evento "LançamentoCriado"
    LambdaC->>Redis: Atualiza saldo diário

```

## Consulta de Saldo
```mermaid
sequenceDiagram
    participant User as Usuário
    participant APIGW as API Gateway
    participant LambdaR as Lambda Relatórios
    participant Redis as Redis
    participant Aurora as Aurora

    User->>APIGW: GET /saldos-diarios
    APIGW->>LambdaR: Invoca função
    LambdaR->>Redis: Consulta saldo
    alt Cache hit
        Redis-->>LambdaR: Retorna saldo
    else Cache miss
        LambdaR->>Aurora: Consulta dados
        Aurora-->>LambdaR: Retorna saldo
    end
    LambdaR->>User: Retorna saldo diário

```
# 14. Finops (High-Level)
## 📊 FinOps – Resumo de Custos AWS

A arquitetura foi projetada seguindo princípios **FinOps** e **Serverless**, priorizando **baixo custo em idle**, **escalabilidade automática** e **pagamento por uso**.

### 📈 Cenário considerado
- ~1.000.000 requisições por mês
- Região AWS: us-east-1
- Perfil de uso: SaaS financeiro (lançamentos, consolidação e relatórios)

### 💰 Custo mensal estimado
**≈ USD 100 / mês**

### 🔍 Principais componentes de custo
- **Aurora Serverless v2 (~75%)**  
  Banco transacional ACID com auto scale e ACU mínimo configurado.
- **ElastiCache Redis (~12%)**  
  Cache de saldos consolidados, reduzindo leituras no banco.
- **Demais serviços (~13%)**  
  CloudFront, S3, API Gateway (HTTP API), Lambda, SQS/EventBridge e CloudWatch.

### ✅ Benefícios FinOps
- Sem servidores dedicados (EC2 ou Kubernetes)
- Zero custo quando não há tráfego
- Escala automática conforme a demanda
- Custos previsíveis por volume de requisições

### ⚠️ Pontos de atenção
- Configurar corretamente o ACU mínimo do Aurora
- Definir política de retenção de logs no CloudWatch
- Aplicar throttling no API Gateway para evitar abuso

> Esta estimativa é aproximada e pode variar conforme o volume real de uso, padrões de acesso e região AWS.



# 15.Como rodar a aplicação localmente

## 🧰 Pré-requisitos – LocalStack em Docker
- .NET 10 SDK installed
- PostgreSQL available and reachable
- Docker
- LocalStack
- (Optional) dotnet-ef tool: `dotnet tool install --global dotnet-ef`

Para executar o LocalStack localmente utilizando Docker, certifique-se de que os seguintes requisitos estejam atendidos.

### 🖥️ Sistema Operacional
- Windows 10/11 (com WSL2)
- macOS
- Linux

1) Subir o localstack/postgree usando o docker
<img width="341" height="477" alt="image" src="https://github.com/user-attachments/assets/0cac707c-48ae-43b4-8bb7-57a2039a96bd" />
```  
docker-compose up -d
```  

2. Verificar connection strings
- Edit the `Default` connection string in `src/Lancamentos.Api/appsettings.json` and `src/Relatorios.Api/appsettings.json`  and `src/Consolidacao.Worker/appsettings.json` or set an environment variable:
3. Build solution
4. Aplicar migrations se necessário
  ```bash
  dotnet ef database update --startup-project src/Infrastructure
  ```

5. Subir os projetos conforme imagem abaixo.

<img width="803" height="541" alt="image" src="https://github.com/user-attachments/assets/7e03168f-463a-4733-9098-53db1716bf6a" />


## Como testar 

## (Lancamento.Api)
 - `http://localhost:5000/swagger`

  - 📘 API de Lançamentos
=====================

API responsável por registrar lançamentos financeiros (crédito e débito) em um fluxo de caixa diário.

## 📌 Endpoint

### ➕ Registrar Lançamento

- `POST /lancamentos`  
- Descrição: Registra um lançamento financeiro para uma data específica.

### 📥 Request

- Headers:
  - `Content-Type: application/json`

- Body (exemplo):
### 🧾 Campos do Request

| Campo     | Tipo                | Obrigatório | Descrição                     |
|-----------|---------------------|-------------|-------------------------------|
| `valor`   | number (double)     | ✅          | Valor do lançamento           |
| `descricao` | string            | ❌          | Descrição opcional            |
| `data`    | string (date)       | ✅          | Data no formato `yyyy-MM-dd`  |
| `tipo`    | integer             | ✅          | Tipo do lançamento (enum)     |

#### 🔢 Enum: `TipoLancamento`

| Código | Descrição |
|--------|-----------|
| 1      | Crédito   |
| 2      | Débito    |

---

### 📤 Response

- Sucesso:
  - Status: `200 OK`
  - Mensagem: "Lançamento registrado com sucesso."

Exemplo de resposta:
### ⚠️ Possíveis Erros

| Status | Descrição                         |
|--------|-----------------------------------|
| 400    | Dados inválidos                   |
| 422    | Violação de regra de negócio      |
| 500    | Erro interno                      |

---

## 🧠 Observações Técnicas

- Arquitetura orientada a CQRS.  
- Validações realizadas na Application Layer.  
- Consolidação diária pode ocorrer de forma assíncrona (event-driven).  
- Compatível com MassTransit / SQS / Kafka.

---

# (Relatorio.Api)
API responsavel por gerar o relatório consolidado do dia. 

**Formato da data do relatório:** `yyyy-MM-dd`

## Exemplo de requisição
`GET /relatorios/2025-01-10`

## Response (200 OK)
Relatório consolidado do dia.
### Campos do Response
| Campo | Tipo            | Descrição                     |
|-------|-----------------|-------------------------------|
| dia   | string (date)   | Data do relatório (yyyy-MM-dd)|
| saldo | number (double) | Saldo consolidado do dia      |

### Possíveis Erros
| Status | Descrição                  |
|--------|---------------------------|
| 400    | Data inválida             |
| 404    | Relatório não encontrado  |
| 500    | Erro interno              |

### Observações Técnicas
- API de consulta (Query)
- Segue padrão CQRS
- Dados consolidados previamente via eventos
- Leitura otimizada (read model)
- Compatível com event-driven architecture

### Exemplo de uso (curl)
    curl -X GET http://localhost:5000/relatorios/2025-01-10# Relatórios — API de Consulta

# (Frontend)

Abaixo conferir a URL do frontend do comerciante.

  - `https://localhost:5280/fluxo-caixa`

Print screen da tela:    
<img width="1359" height="700" alt="image" src="https://github.com/user-attachments/assets/8d4fad41-13df-4f2a-a4cf-10bfb2a000d2" />



# 16. Testes funcionais e unitarios:


<img width="1054" height="418" alt="image" src="https://github.com/user-attachments/assets/f5cbc1d9-02c8-4acb-9236-836cfbae0ff7" />



# 18. Proximos  passos 

Falhas transitórias sejam reprocessadas (retry) dlq

Mensagens duplicadas não gerem efeito colateral 

Implementar log e observability

Implementar autenticação e autorização

Implementar cache em diversas camadas

Implementar testes de contrato de api

Implementar testes de perfomance

Implementar demais testes funcionais e unitarios

Segregar os banco de dados se necessário (Redis) 

Implementar infra com código utilizando cloud formation/terraform

Adicionar novos/acrescentar requisitos funcionais 

Deploy arquitetura na AWS (CI/CD)


