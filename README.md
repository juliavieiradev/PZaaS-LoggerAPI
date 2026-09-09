# PZaaS-LoggerAPI
Documentação da Logger API, para o projeto de Arquitetura de Serviços em Nuvem, aula ministrada por Andrews Egas.


## 1\. Visão geral

A **Logger API** é um serviço responsável por receber, armazenar e disponibilizar logs e métricas gerados pelos microsserviços de uma arquitetura distribuída.

O workflow é implementado em **n8n** e utiliza o **Supabase** como camada de persistência.

### Responsabilidades

* Receber logs estruturados dos serviços.
* Validar a chave de acesso das requisições.
* Validar os campos obrigatórios dos logs e métricas.
* Normalizar os dados recebidos.
* Armazenar logs e métricas no Supabase.
* Permitir consulta de logs por filtros e paginação.
* Disponibilizar um endpoint de health check.
* Permitir o rastreamento de operações por `eventId` e/ou `orderId`.

---

## 2\. Arquitetura

```text
Microsserviço
     |
     | HTTP
     v
+------------------+
|    Logger API    |
|      (n8n)       |
+------------------+
     |
     +--------------------+
     |                    |
     v                    v
  Validação           Normalização
     |                    |
     +---------+----------+
               |
               v
          +---------+
          | Supabase|
          +---------+
          | logs    |
          | metrics |
          +---------+
```

---

## 3\. Autenticação

As rotas protegidas utilizam uma API Key enviada no header:

```http
x-api-key: turma2026
```

Também deve ser utilizado:

```http
Content-Type: application/json
```

Para o registro de logs, o identificador do pedido pode ser enviado opcionalmente pelo header:

```http
x-pedido-id: <id do pedido enviado pelo API Gateway>
```

### Chave ausente

Quando o header `x-api-key` não é enviado:

```http
HTTP 401 Unauthorized
```

```json
{
  "error": "x-api-key ausente"
}
```

### Chave inválida

Quando a chave enviada não corresponde à chave configurada:

```http
HTTP 403 Forbidden
```

```json
{
  "error": "x-api-key inválida"
}
```

\---

# 4\. Endpoints

|Método|Endpoint|Autenticação|Descrição|
|-|-|-|-|
|GET|`/log/v1/health`|Não|Verifica a disponibilidade da API e da conexão com o banco|
|POST|`/v1/log`|Sim|Registra um novo log|
|GET|`/v1/logs`|Sim|Consulta logs|
|POST|`/v1/metric`|Sim|Registra uma nova métrica|
|GET|`/v1/metrics`|Sim|Consulta métricas|
|GET|`/v1/chaos-status`|Sim|Consulta o estado do Chaos Monkey|
|POST|`/v1/alter-chaos`|Sim|Habilita ou desabilita o Logger|

---

# 5\. POST /v1/log

Registra um evento de log no Logger API.

## Requisição

```http
POST /v1/log
Content-Type: application/json
x-api-key: turma2026
```

O header `x-pedido-id` é opcional:

```http
x-pedido-id: <id do pedido enviado pelo API Gateway>
```

### Payload

```json
{
  "eventId": "7f3a91c2-1234-4567-8901-abcdef123456",
  "timestamp": "2026-09-02T20:50:31Z",
  "service": "4",
  "action": "PROCESS\_PAYMENT",
  "status": "FAILED",
  "level": "ERROR",
  "message": "Pagamento recusado",
  "metadata": {
    "paymentMethod": "PIX",
    "attempt": 2,
    "reason": "INSUFFICIENT\_FUNDS"
  }
}
```

O campo `orderId` não é enviado no body. Quando o header `x-pedido-id` é informado, seu valor é utilizado como `orderId`. Caso o header não seja informado, `orderId` será `null`.

## Campos

|Campo|Obrigatório no contrato|Tipo|Descrição|
|-|-:|-|-|
|`timestamp`|Não|datetime|Data e hora do evento|
|`service`|Sim|number|Serviço que gerou o log|
|`action`|Sim|string|Operação realizada|
|`status`|Sim|string|Resultado da operação|
|`level`|Sim|string|Severidade do log|
|`message`|Sim|string|Descrição do evento|
|`orderId`|Não|string/null|Identificador do pedido, obtido pelo header `x-pedido-id`|
|`metadata`|Não|object|Informações adicionais específicas do serviço|

### Valores recomendados

#### `level`

* `INFO`
* `WARN`
* `ERROR`
* `DEBUG`

#### `status`

* `SUCCESS`
* `ERROR`
* `FAILED`
* `STARTED`
* `PROCESSING`

#### `service`

Os serviços definidos no modelo são:

|Serviço|ID|
|-|-:|
|API Gateway|1|
|Catálogo/Cardápio|2|
|Disponibilidade/Estoque|3|
|Pagamento|4|
|Orquestrador|5|
|Forno|6|
|Fila de produção|7|
|Cadastro/identidade|8|
|Chaos Monkey|10|

O campo `action` permanece flexível, pois cada microsserviço possui operações diferentes.

\---

## 5.1 `metadata`

`metadata` é um objeto JSON livre utilizado para armazenar informações específicas de cada serviço.

### Exemplo — Cardápio

```json
{
  "metadata": {
    "itemsReturned": 15
  }
}
```

### Exemplo — Estoque

```json
{
  "metadata": {
    "ingredient": "mussarela",
    "available": 8,
    "requested": 2
  }
}
```

### Exemplo — Pagamento

```json
{
  "metadata": {
    "paymentMethod": "PIX",
    "amount": 59.90
  }
}
```

Não devem ser criados campos obrigatórios específicos dentro de `metadata`.

\---

# 6\. Validação do payload

O workflow valida os seguintes campos como obrigatórios para `POST /v1/log`:

```text
service
action
status
level
message
```

Caso algum deles esteja ausente, nulo ou vazio, a API retorna:

```http
HTTP 400 Bad Request
```

Exemplo:

```json
{
  "error": "Payload inválido",
  "campos\_faltando": "service, message"
}
```

\---

# 7\. Normalização do log

Após a validação, o workflow transforma o payload para o formato utilizado no armazenamento.

A estrutura persistida é:

```json
{
  "timestamp": "2026-09-02T20:50:31Z",
  "service": "pagamento",
  "action": "PROCESS\_PAYMENT",
  "status": "FAILED",
  "level": "ERROR",
  "message": "Pagamento recusado",
  "orderId": "PED-10293",
  "metadata": "{\\"paymentMethod\\":\\"PIX\\",\\"attempt\\":2}"
}
```

Caso `timestamp` não seja informado, o workflow utiliza automaticamente a data/hora atual.

Caso o header `x-pedido-id` não seja informado:

```json
"orderId": null
```

Caso `metadata` não seja informado:

```json
"metadata": "{}"
```

\---

# 8\. Persistência dos logs

Os logs são armazenados na tabela:

```text
logs
```

Mapeamento utilizado pelo workflow:

|Payload|Supabase|
|-|-|
|`timestamp`|`timestamp`|
|`service`|`service`|
|`action`|`action`|
|`status`|`status`|
|`level`|`level`|
|`message`|`message`|
|`orderId`|`order\_id`|
|`metadata`|`metadata`|

\---

# 9\. Resposta do POST /v1/log

O workflow atual possui um node de retorno após a criação do registro.

O contrato de sucesso recomendado pelo modelo é:

```http
HTTP 201 Created
```

```json
{
  "id": "9b27f8d1-4c8a-42c3-a1f7-123456789abc",
  "message": "Log registrado com sucesso"
}
```


\---

# 10\. GET /v1/logs

Consulta os logs registrados.

## Requisição

```http
GET /v1/logs
x-api-key: turma2026
```

## Filtros disponíveis

### Por id

```http
GET /v1/logs?id=7f3a91c2-1234-4567-8901-abcdef123456
```

### Por pedido

```http
GET /v1/logs?orderId=PED-10293
```

### Por serviço

```http
GET /v1/logs?service=pagamento
```

### Por nível

```http
GET /v1/logs?level=ERROR
```

### Por status

```http
GET /v1/logs?status=FAILED
```

Os filtros podem ser combinados:

```http
GET /v1/logs?service=pagamento\&level=ERROR\&status=FAILED
```

\---

# 11\. Paginação

A consulta suporta os parâmetros:

* `page`: número da página
* `limit`: quantidade de registros por página

Exemplo:

```http
GET /v1/logs?page=1\&limit=20
```

Valores padrão implementados no workflow:

```text
page = 1
limit = 20
```

O SQL preparado pelo workflow utiliza:

```sql
LIMIT
OFFSET
```

e calcula o total de registros utilizando:

```sql
COUNT(\*) OVER()
```

### Resposta esperada

```json
{
  "data": \[],
  "page": 1,
  "limit": 20,
  "total": 157
}
```
---

# 12\. Ordenação

Os registros da consulta são preparados para serem ordenados por:

```sql
ORDER BY timestamp DESC
```

Ou seja, os logs mais recentes devem aparecer primeiro.

\---

# 13\. GET /log/v1/health

Endpoint utilizado para verificar a disponibilidade do Logger e da conexão com o banco.

```http
GET /log/v1/health
```

O workflow consulta a tabela `logs` no Supabase para verificar a disponibilidade da base.

## Banco conectado

```http
HTTP 200 OK
```

```json
{
  "status": "up",
  "database": "connected",
  "timestamp": "2026-09-08T18:00:00.000Z"
}
```

## Banco indisponível

```http
HTTP 500 Internal Server Error
```

```json
{
  "status": "down",
  "database": "disconnected",
  "timestamp": "2026-09-08T18:00:00.000Z"
}
```

\---

# 14\. POST /v1/metric

Além dos logs, o workflow possui um endpoint específico para registro de métricas.

```http
POST /v1/metric
Content-Type: application/json
x-api-key: turma2026
```

## Campos obrigatórios

O workflow valida:

```text
metricName
value
service
```

## Campos opcionais

```text
unit
orderId
timestamp
metadata
```

## Exemplo

```json
{
  "metricName": "payment\_processing\_time",
  "value": 2.43,
  "unit": "seconds",
  "service": "pagamento",
  "orderId": "PED-10293",
  "timestamp": "2026-09-02T20:50:31Z",
  "metadata": {
    "paymentMethod": "PIX"
  }
}
```

\---

# 15\. Identificação da métrica

Para cada métrica válida, o workflow utiliza um node **Crypto** para gerar um identificador `metric\_id`.

A estrutura normalizada é:

```json
{
  "metricId": "generated-id",
  "metricName": "payment\_processing\_time",
  "value": 2.43,
  "unit": "seconds",
  "service": "pagamento",
  "orderId": "PED-10293",
  "timestamp": "2026-09-02T20:50:31Z",
  "metadata": "{\\"paymentMethod\\":\\"PIX\\"}"
}
```

As métricas são armazenadas na tabela:

```text
metrics
```

\---

# 15.1 GET /v1/metrics

Consulta as métricas armazenadas.

```http
GET /v1/metrics
x-api-key: turma2026
```

## Filtros disponíveis

|Parâmetro|Descrição|
|-|-|
|`metricId`|Identificador da métrica|
|`metricName`|Nome da métrica|
|`service`|Serviço responsável|
|`orderId`|Identificador do pedido|
|`page`|Número da página|
|`limit`|Quantidade de registros por página|

### Exemplo

```http
GET /v1/metrics?service=pagamento&metricName=payment_processing_time&page=1&limit=20
```

---

# 15.2 Fluxo de consulta de métricas

A consulta utiliza a função `buscar_metrics` no Supabase, aplicando os filtros informados e os parâmetros de paginação.

Os valores padrão são:

```text
page = 1
limit = 20
```

---

# 16\. Chaos Monkey

O Logger possui endpoints para simular a indisponibilidade do serviço.

A funcionalidade permite testar o comportamento dos demais microsserviços quando o Logger estiver indisponível.

## GET /v1/chaos-status

Consulta o estado atual do Logger.

```http
GET /v1/chaos-status
x-api-key: turma2026
```

A resposta informa se o serviço está habilitado ou desabilitado.

## POST /v1/alter-chaos

Altera o estado de disponibilidade do Logger.

```http
POST /v1/alter-chaos
Content-Type: application/json
x-api-key: turma2026
```

### Desabilitar o Logger

```json
{
  "enabled": false
}
```

### Habilitar o Logger

```json
{
  "enabled": true
}
```

Quando `enabled` for `false`, os endpoints normais do Logger ficam indisponíveis e retornam `503 Service Unavailable`.

Os endpoints de Chaos continuam disponíveis para permitir a consulta e a reativação do serviço.

O estado do serviço é armazenado na tabela:

```text
service_status
```

com o registro:

```text
service = logger
```

---

# 17\. Tratamento de erros

|HTTP|Situação|
|-:|-|
|`200`|Consulta ou health check realizado com sucesso|
|`201`|Log/métrica criado com sucesso|
|`400`|Payload inválido|
|`401`|`x-api-key` ausente|
|`403`|`x-api-key` inválida|
|`404`|Recurso não encontrado|
|`429`|Rate limit atingido|
|`500`|Erro interno ou indisponibilidade do banco|
|`503`|Logger desabilitado pelo Chaos Monkey|

> Os códigos `404` e `429` fazem parte do contrato de documentação, mas não possuem tratamento explícito identificado no workflow atual.

\---

# 18\. Exemplos de uso

## Registrar um log

```bash
curl -X POST "https://SEU\_HOST/v1/log" \\
  -H "Content-Type: application/json" \\
  -H "x-api-key: turma2026" \\
  -H "x-pedido-id: PED-10293" \\
  -d '{
    "eventId": "7f3a91c2-1234-4567-8901-abcdef123456",
    "service": "pagamento",
    "action": "PROCESS\_PAYMENT",
    "status": "SUCCESS",
    "level": "INFO",
    "message": "Pagamento processado com sucesso",
    "metadata": {
      "paymentMethod": "PIX",
      "amount": 59.90
    }
  }'
```

## Consultar logs

```bash
curl -X GET "https://SEU\_HOST/v1/logs?page=1\&limit=20" \\
  -H "x-api-key: turma2026"
```

## Consultar erros de um serviço

```bash
curl -X GET "https://SEU\_HOST/v1/logs?service=pagamento\&level=ERROR" \\
  -H "x-api-key: turma2026"
```

## Verificar saúde da API

```bash
curl -X GET "https://SEU\_HOST/log/v1/health"
```

## Registrar uma métrica

```bash
curl -X POST "https://SEU\_HOST/v1/metric" \\
  -H "Content-Type: application/json" \\
  -H "x-api-key: turma2026" \\
  -d '{
    "metricName": "payment\_processing\_time",
    "value": 2.43,
    "unit": "seconds",
    "service": "pagamento",
    "orderId": "PED-10293"
  }'
```

## Consultar métricas

```bash
curl -X GET "https://SEU\_HOST/v1/metrics?service=pagamento&page=1\&limit=20" \\
  -H "x-api-key: turma2026"
```

## Consultar status do Chaos Monkey

```bash
curl -X GET "https://SEU\_HOST/v1/chaos-status" \\
  -H "x-api-key: turma2026"
```

## Desabilitar o Logger

```bash
curl -X POST "https://SEU\_HOST/v1/alter-chaos" \\
  -H "Content-Type: application/json" \\
  -H "x-api-key: turma2026" \\
  -d '{
    "enabled": false
  }'
```

## Habilitar o Logger

```bash
curl -X POST "https://SEU\_HOST/v1/alter-chaos" \\
  -H "Content-Type: application/json" \\
  -H "x-api-key: turma2026" \\
  -d '{
    "enabled": true
  }'
```

\---

# 19\. Fluxo de processamento

## Logs

```text
POST /v1/log
       |
       v
Verificar x-api-key
       |
       +---- ausente ----> 401
       |
       +---- inválida ---> 403
       |
       v
Validar payload
       |
       +---- inválido ---> 400
       |
       v
Normalizar log
       |
       v
Supabase - tabela logs
       |
       v
Retorno
```

## Métricas

```text
POST /v1/metric
       |
       v
Verificar x-api-key
       |
       +---- ausente ----> 401
       |
       +---- inválida ---> 403
       |
       v
Validar payload
       |
       +---- inválido ---> 400
       |
       v
Gerar metric_id
       |
       v
Normalizar métrica
       |
       v
Supabase - tabela metrics
       |
       v
Retorno
```

## Chaos Monkey

```text
POST /v1/alter-chaos
       |
       v
Verificar x-api-key
       |
       v
Atualizar service_status
       |
       +---- enabled = true
       |
       +---- enabled = false
```

\---

# 20\. Regras e recomendações

1. Todas as rotas protegidas devem receber `x-api-key`.
2. Requisições com corpo JSON devem utilizar `Content-Type: application/json`.
3. `action` deve permanecer flexível e não deve ser tratado como ENUM fechado.
4. `metadata` deve permanecer como objeto livre.
5. `timestamp` deve utilizar formato datetime, preferencialmente ISO 8601.
6. `orderId` deve ser utilizado quando o evento estiver relacionado a um pedido.
7. `level` deve representar a severidade do evento.
8. `status` deve representar o estado ou resultado da operação.
9. Logs devem ser registrados de maneira estruturada para facilitar consultas e rastreamento.
10. A API Key não deve ficar exposta diretamente no workflow em um ambiente produtivo.

\---

# 21\. Pontos de atenção antes da publicação

O workflow e o modelo de documentação apresentam algumas diferenças que devem ser corrigidas/alinhadas:

### 21.1 Health check

**Workflow:**

```text
/log/v1/health
```

**Modelo:**

```text
/health
```

Escolher um único padrão.

### 21.2 `eventId`

O modelo define `eventId` como obrigatório, porém o node `validar payload` do workflow não o valida como obrigatório.

Além disso, o workflow atual não gera automaticamente `eventId`.

### 21.3 GET ainda em mock

O SQL de consulta já está preparado com filtros, `LIMIT`, `OFFSET` e `COUNT(\*) OVER()`, mas o resultado ainda é direcionado para um node de mock.

### 21.4 Retorno do POST

O workflow possui um node de retorno, mas o código HTTP `201 Created` e o corpo de resposta documentados ainda precisam ser configurados explicitamente.

### 21.5 Métricas

O workflow possui `POST /v1/metric` e `GET /v1/metrics`, com registro e consulta de métricas. A consulta utiliza a função `buscar_metrics` no Supabase.

### 21.6 Chaos Monkey

O workflow possui `GET /v1/chaos-status` e `POST /v1/alter-chaos` para consultar e alterar o estado de disponibilidade do Logger.

\---

# 22\. Resumo do contrato

### Logs

**Endpoint**

```text
POST /v1/log
```

**Obrigatórios no workflow**

```text
service
action
status
level
message
```

**Definidos como obrigatórios no modelo**

```text
eventId
service
action
status
level
message
```

**Opcionais**

```text
timestamp
x-pedido-id
metadata
```

O `x-pedido-id`, quando enviado no header, é utilizado para preencher `orderId`. O `orderId` não é enviado no body.

### Consulta

**Endpoint**

```text
GET /v1/logs
```

**Filtros**

```text
eventId
orderId
service
level
status
```

**Paginação**

```text
page
limit
```

### Métricas

**Registro**

```text
POST /v1/metric
```

**Consulta**

```text
GET /v1/metrics
```

**Obrigatórios**

```text
metricName
value
service
```

**Opcionais**

```text
unit
orderId
timestamp
metadata
```

### Health

**Endpoint**

```text
GET /log/v1/health
```

**Resultado**

```text
UP / DOWN
```

### Chaos Monkey

**Consultar estado**

```text
GET /v1/chaos-status
```

**Alterar estado**

```text
POST /v1/alter-chaos
```

**Body**

```json
{
  "enabled": true
}
```

ou

```json
{
  "enabled": false
}
```

\---

## 23\. Tecnologias

* **n8n** — orquestração do workflow e exposição dos webhooks.
* **Supabase** — persistência dos logs e métricas.
* **PostgreSQL/SQL** — consulta e paginação dos registros.
* **HTTP/JSON** — comunicação entre os microsserviços e o Logger.
