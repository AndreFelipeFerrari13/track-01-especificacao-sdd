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
