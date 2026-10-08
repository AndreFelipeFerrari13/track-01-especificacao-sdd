# Spec — API REST Zona Azul Digital

> [!NOTE]
> Objetivo: especificar uma API REST para abrir, encerrar, cancelar, listar e relatar bilhetes de Zona Azul Digital. O agente que gerar código deve seguir este contrato sem adicionar envelopes, nomes alternativos ou campos monetários em ponto flutuante.

## 1. Configuração da variante

| Parâmetro | Origem | Uso obrigatório |
|---|---|---|
| `TARIFA_HORA_CENTAVOS` | `variante/params.json` | Base do cálculo do valor por hora cheia. |
| `FRACAO_MINUTOS` | `variante/params.json` | Granularidade mínima de cobrança, sempre arredondada para cima. |
| `TETO_DIARIO_CENTAVOS` | `variante/params.json` | Valor máximo por bilhete encerrado. |
| `TOLERANCIA_MINUTOS` | `variante/params.json` | Duração gratuita inicial; se ultrapassar, cobra desde o primeiro minuto. |
| `PORTA_SERVICO` | `variante/params.json` | Porta HTTP usada pelo serviço em `localhost`. |

A base URL esperada pela suíte é `http://localhost:{PORTA_SERVICO}`.

## 2. Modelo de dados lógico

| Campo | Tipo | Status aplicável | Regra |
|---|---|---|---|
| `id` | inteiro positivo | todos | Sequencial ou único, estável após criação. |
| `placa` | string | todos | Exatamente 7 caracteres alfanuméricos maiúsculos. |
| `entrada` | string ISO-8601 | todos | Sempre com fuso `-03:00`. |
| `saida` | string ISO-8601 | encerrado | Gerada no encerramento, com fuso `-03:00`. |
| `minutos` | inteiro não negativo | encerrado | Duração entre entrada e saída em minutos inteiros. |
| `valor_centavos` | inteiro não negativo | encerrado | Valor calculado com tolerância, fração e teto. |
| `status` | string | todos | `aberto`, `encerrado` ou `cancelado`. |

## 3. Regras de validação comuns

| Regra | Resultado esperado |
|---|---|
| Placa ausente, nula, vazia, com minúsculas, símbolos, espaços ou tamanho diferente de 7 | `422 {"erro":"placa_invalida"}` |
| `entrada` presente fora de ISO-8601 com fuso | `422 {"erro":"entrada_invalida"}` |
| `data` ausente ou fora de `AAAA-MM-DD` | `422 {"erro":"data_invalida"}` |
| `id` inexistente | `404 {"erro":"bilhete_nao_encontrado"}` |
| Payload malformado | Retornar erro de validação sem quebrar o servidor. |

> [!WARNING]
> Precedência obrigatória: validar formato antes de conflito de estado. Exemplo: placa inválida em `POST /bilhetes` retorna `422 placa_invalida`, não `409`.

## 4. UC1 — Abrir bilhete

| Item | Especificação |
|---|---|
| Método e rota | `POST /bilhetes` |
| Body obrigatório | `placa` |
| Body opcional | `entrada` ISO-8601 com fuso |
| Sucesso | `201 {"id":1,"placa":"ABC1D23","entrada":"<ISO-8601 -03:00>","status":"aberto"}` |
| Erro de placa | `422 {"erro":"placa_invalida"}` |
| Erro de entrada | `422 {"erro":"entrada_invalida"}` |
| Erro de placa ocupada | `409 {"erro":"bilhete_em_aberto"}` |

### Critérios de aceite UC1

| ID | Dado | Quando | Então |
|---|---|---|---|
| AC-UC1-01 | Placa válida sem `entrada` | Abrir bilhete | Retorna `201`, status `aberto`, entrada atual com fuso `-03:00`. |
| AC-UC1-02 | Placa válida com `entrada` ISO-8601 | Abrir bilhete | Retorna `201` usando a entrada enviada. |
| AC-UC1-03 | Placa inválida | Abrir bilhete | Retorna `422 {"erro":"placa_invalida"}`. |
| AC-UC1-04 | Entrada inválida | Abrir bilhete | Retorna `422 {"erro":"entrada_invalida"}`. |
| AC-UC1-05 | Placa com bilhete `aberto` | Abrir outro bilhete | Retorna `409 {"erro":"bilhete_em_aberto"}`. |
| AC-UC1-06 | Placa com bilhete `encerrado` ou `cancelado` | Abrir novo bilhete | Retorna `201` e cria novo id. |

## 5. UC2 — Encerrar bilhete

| Item | Especificação |
|---|---|
| Método e rota | `POST /bilhetes/{id}/encerramento` |
| Body | Nenhum body obrigatório |
| Sucesso | `200 {"id":1,"placa":"ABC1D23","entrada":"...","saida":"...","minutos":95,"valor_centavos":1250}` |
| Inexistente | `404 {"erro":"bilhete_nao_encontrado"}` |
| Já encerrado | `409 {"erro":"bilhete_ja_encerrado"}` |
| Cancelado | `409 {"erro":"bilhete_nao_aberto"}` |

### Regras de cálculo do valor

| Passo | Regra |
|---|---|
| 1 | Calcular `minutos` como duração inteira não negativa entre `entrada` e `saida`. Se houver segundos, arredondar duração para cima ao minuto seguinte. |
| 2 | Se `minutos <= TOLERANCIA_MINUTOS`, `valor_centavos = 0`. |
| 3 | Se `minutos > TOLERANCIA_MINUTOS`, cobrar desde o primeiro minuto; a tolerância não é abatida. |
| 4 | Calcular quantidade de frações como teto matemático de `minutos / FRACAO_MINUTOS`. |
| 5 | Calcular valor da fração como `TARIFA_HORA_CENTAVOS / (60 / FRACAO_MINUTOS)`. |
| 6 | Calcular valor bruto como `frações * valor_da_fração`, em centavos inteiros. |
| 7 | Aplicar `min(valor_bruto, TETO_DIARIO_CENTAVOS)`. |

> [!WARNING]
> Nunca usar ponto flutuante em resposta. Se a divisão de tarifa por fração produzir valor não inteiro em alguma variante futura, arredondar de forma determinística para centavos inteiros antes de retornar.

### Critérios de aceite UC2

| ID | Dado | Quando | Então |
|---|---|---|---|
| AC-UC2-01 | Bilhete aberto existente | Encerrar | Retorna `200` com `saida`, `minutos` inteiro e `valor_centavos` inteiro. |
| AC-UC2-02 | Duração exatamente igual a uma fração | Encerrar | Cobra exatamente uma fração. |
| AC-UC2-03 | Duração uma unidade acima da fração | Encerrar | Cobra a próxima fração. |
| AC-UC2-04 | Duração maior que o teto | Encerrar | `valor_centavos` não ultrapassa `TETO_DIARIO_CENTAVOS`. |
| AC-UC2-05 | Bilhete inexistente | Encerrar | Retorna `404 {"erro":"bilhete_nao_encontrado"}`. |
| AC-UC2-06 | Bilhete já encerrado | Encerrar novamente | Retorna `409 {"erro":"bilhete_ja_encerrado"}`. |
| AC-UC2-07 | Duração dentro da tolerância | Encerrar | Retorna `valor_centavos = 0`. |
| AC-UC2-08 | Duração excede tolerância em 1 minuto | Encerrar | Cobra desde o primeiro minuto. |

## 6. UC3 — Listar ativos

| Item | Especificação |
|---|---|
| Método e rota | `GET /bilhetes/ativos` |
| Sucesso | `200` com array de bilhetes `aberto` |
| Ordenação | Mais recentes primeiro por `entrada`; em empate, maior `id` primeiro |
| Campos | `id`, `placa`, `entrada`, `status` |

### Critérios de aceite UC3

| ID | Dado | Quando | Então |
|---|---|---|---|
| AC-UC3-01 | Existem bilhetes abertos | Listar ativos | Retorna somente status `aberto`. |
| AC-UC3-02 | Existem encerrados e cancelados | Listar ativos | Eles não aparecem. |
| AC-UC3-03 | Nenhum aberto | Listar ativos | Retorna `200 []`. |
| AC-UC3-04 | Vários abertos | Listar ativos | Retorna mais recentes primeiro. |

## 7. UC4 — Relatório diário

| Item | Especificação |
|---|---|
| Método e rota | `GET /relatorios/diario?data=AAAA-MM-DD` |
| Sucesso | `200 {"data":"2026-10-05","total_bilhetes":12,"faturamento_centavos":8400,"tempo_medio_minutos":47}` |
| Data inválida | `422 {"erro":"data_invalida"}` |
| Escopo | Somente bilhetes encerrados cuja `saida` cai no dia informado no fuso `-03:00` |
| Arredondamento | `tempo_medio_minutos` arredonda 0,5 para cima |

### Critérios de aceite UC4

| ID | Dado | Quando | Então |
|---|---|---|---|
| AC-UC4-01 | Há encerrados no dia | Gerar relatório | Soma total, faturamento e média dos encerrados do dia. |
| AC-UC4-02 | Há abertos ou cancelados no dia | Gerar relatório | Eles são ignorados. |
| AC-UC4-03 | Não há encerrados no dia | Gerar relatório | Retorna total `0`, faturamento `0`, tempo médio `0`. |
| AC-UC4-04 | Média termina em `.5` | Gerar relatório | Arredonda para cima. |
| AC-UC4-05 | `data` inválida | Gerar relatório | Retorna `422 {"erro":"data_invalida"}`. |

## 8. UC5 — Cancelar bilhete

| Item | Especificação |
|---|---|
| Método e rota | `POST /bilhetes/{id}/cancelamento` |
| Sucesso | `200` com `id`, `placa`, `entrada`, `status:"cancelado"` |
| Inexistente | `404 {"erro":"bilhete_nao_encontrado"}` |
| Não aberto | `409 {"erro":"bilhete_nao_aberto"}` |

### Critérios de aceite UC5

| ID | Dado | Quando | Então |
|---|---|---|---|
| AC-UC5-01 | Bilhete aberto existente | Cancelar | Retorna `200` com status `cancelado`. |
| AC-UC5-02 | Bilhete cancelado | Consultar histórico | Não possui `saida`, `minutos` nem `valor_centavos`. |
| AC-UC5-03 | Bilhete encerrado ou cancelado | Cancelar | Retorna `409 {"erro":"bilhete_nao_aberto"}`. |
| AC-UC5-04 | Bilhete inexistente | Cancelar | Retorna `404 {"erro":"bilhete_nao_encontrado"}`. |
| AC-UC5-05 | Placa cancelada | Abrir novo bilhete | Retorna `201`; a placa foi liberada. |

## 9. UC6 — Histórico por placa

| Item | Especificação |
|---|---|
| Método e rota | `GET /bilhetes?placa=ABC1D23` |
| Sucesso | `200` com array de todos os bilhetes da placa |
| Ordenação | Mais recentes primeiro por `entrada`; em empate, maior `id` primeiro |
| Placa nunca usada | `200 []` |
| Placa inválida ou ausente | `422 {"erro":"placa_invalida"}` |

### Critérios de aceite UC6

| ID | Dado | Quando | Então |
|---|---|---|---|
| AC-UC6-01 | Placa com bilhetes abertos, encerrados e cancelados | Buscar histórico | Retorna todos os status da placa. |
| AC-UC6-02 | Placa válida sem histórico | Buscar histórico | Retorna `200 []`. |
| AC-UC6-03 | Placa inválida ou ausente | Buscar histórico | Retorna `422 {"erro":"placa_invalida"}`. |
| AC-UC6-04 | Vários bilhetes da placa | Buscar histórico | Retorna mais recentes primeiro. |

## 10. UC7 — Tolerância gratuita

| Item | Especificação |
|---|---|
| Regra | Duração `<= TOLERANCIA_MINUTOS` é grátis |
| Ultrapassagem | Se passar 1 minuto da tolerância, cobra integral desde o minuto zero |
| Tolerância zero | Todo minuto positivo é cobrado conforme fração |

### Critérios de aceite UC7

| ID | Dado | Quando | Então |
|---|---|---|---|
| AC-UC7-01 | `minutos == TOLERANCIA_MINUTOS` | Encerrar | `valor_centavos = 0`. |
| AC-UC7-02 | `minutos == TOLERANCIA_MINUTOS + 1` | Encerrar | Cobra frações desde o início. |
| AC-UC7-03 | `TOLERANCIA_MINUTOS = 0` | Encerrar duração positiva | Cobra conforme fração. |

## 11. UC8 — Uma vaga por placa

| Item | Especificação |
|---|---|
| Regra | Uma placa pode ter no máximo um bilhete `aberto` ao mesmo tempo |
| Conflito | `POST /bilhetes` com placa já aberta retorna `409 {"erro":"bilhete_em_aberto"}` |
| Liberação | Encerramento ou cancelamento libera a placa |

### Critérios de aceite UC8

| ID | Dado | Quando | Então |
|---|---|---|---|
| AC-UC8-01 | Placa com bilhete aberto | Abrir novo | Retorna `409 {"erro":"bilhete_em_aberto"}`. |
| AC-UC8-02 | Placa com último bilhete encerrado | Abrir novo | Retorna `201`. |
| AC-UC8-03 | Placa com último bilhete cancelado | Abrir novo | Retorna `201`. |

## 12. Requisitos não funcionais

| ID | Requisito | Critério mensurável |
|---|---|---|
| RNF-01 | API deve iniciar sem dependências externas pagas ou credenciais. | `Containerfile` sobe o serviço e expõe a porta da variante. |
| RNF-02 | Dependências devem estar declaradas. | Existir manifesto como `requirements.txt`, `pyproject.toml` ou equivalente. |
| RNF-03 | Deve haver testes próprios. | Testes cobrem endpoints, bordas de cobrança, erros e relatório. |
| RNF-04 | Deve haver documentação operacional. | `README.md` explica executar, testar, porta e parâmetros. |
| RNF-05 | Respostas devem ser JSON. | Erros e sucessos retornam `application/json`. |
