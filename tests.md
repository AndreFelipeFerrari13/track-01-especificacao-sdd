# Tests — Casos de Borda e Validação

> [!NOTE]
> Estes cenários devem orientar os testes próprios do código gerado. Use os valores reais de `variante/params.json` ao executar.

## 1. Matriz de regras de negócio

| Regra | Caso de borda obrigatório | Resultado esperado |
|---|---|---|
| Fração de cobrança | Duração exatamente `FRACAO_MINUTOS` | Cobra 1 fração. |
| Fração de cobrança | Duração `FRACAO_MINUTOS + 1` | Cobra 2 frações. |
| Fração de cobrança | Duração `2 * FRACAO_MINUTOS` | Cobra 2 frações. |
| Teto diário | Duração suficiente para exceder teto | Valor final igual a `TETO_DIARIO_CENTAVOS`. |
| Tolerância | Duração igual a `TOLERANCIA_MINUTOS` | Valor `0`. |
| Tolerância | Duração `TOLERANCIA_MINUTOS + 1` | Cobra desde o primeiro minuto. |
| Uma vaga por placa | Placa com bilhete aberto | Segundo `POST /bilhetes` retorna `409`. |
| Liberação da placa | Após encerrar | Novo `POST /bilhetes` retorna `201`. |
| Liberação da placa | Após cancelar | Novo `POST /bilhetes` retorna `201`. |
| Relatório | Média com parte decimal `.5` | Arredonda para cima. |
| Relatório | Sem bilhetes encerrados | Zeros em total, faturamento e média. |

## 2. Cálculo de cobrança

Considere:
- `valor_fracao = TARIFA_HORA_CENTAVOS / (60 / FRACAO_MINUTOS)`
- `fracoes = teto(minutos / FRACAO_MINUTOS)`
- se `minutos <= TOLERANCIA_MINUTOS`, valor `0`
- se `minutos > TOLERANCIA_MINUTOS`, valor `min(fracoes * valor_fracao, TETO_DIARIO_CENTAVOS)`

| Cenário | Entrada lógica | Esperado |
|---|---|---|
| CBR-01 | `minutos = 0` | `valor_centavos = 0` |
| CBR-02 | `minutos = TOLERANCIA_MINUTOS` | `valor_centavos = 0` |
| CBR-03 | `minutos = TOLERANCIA_MINUTOS + 1` | Cobrança integral desde 0, com fração para cima. |
| CBR-04 | `minutos = FRACAO_MINUTOS` e fora da tolerância | 1 fração. |
| CBR-05 | `minutos = FRACAO_MINUTOS + 1` e fora da tolerância | 2 frações. |
| CBR-06 | Valor bruto maior que teto | `TETO_DIARIO_CENTAVOS`. |

> [!WARNING]
> A tolerância não é desconto. Ela só zera bilhetes com duração menor ou igual ao limite. Ao ultrapassar o limite, todo o período é cobrado.

## 3. Testes por endpoint

### UC1 — Abrir bilhete

| Caso | Requisição | Status | Corpo esperado |
|---|---|---:|---|
| UC1-T01 | `POST /bilhetes` com `{"placa":"ABC1D23"}` | 201 | `id`, `placa`, `entrada`, `status:"aberto"` |
| UC1-T02 | `POST /bilhetes` com `entrada` válida | 201 | `entrada` igual ao instante informado, normalizada com `-03:00` |
| UC1-T03 | Placa ausente | 422 | `{"erro":"placa_invalida"}` |
| UC1-T04 | Placa `abc1d23` | 422 | `{"erro":"placa_invalida"}` |
| UC1-T05 | Entrada `ontem` | 422 | `{"erro":"entrada_invalida"}` |
| UC1-T06 | Mesma placa já aberta | 409 | `{"erro":"bilhete_em_aberto"}` |

### UC2 — Encerrar bilhete

| Caso | Preparação | Ação | Esperado |
|---|---|---|---|
| UC2-T01 | Abrir bilhete com entrada no passado | Encerrar | `200` com `saida`, `minutos`, `valor_centavos` |
| UC2-T02 | Id inexistente | Encerrar | `404 {"erro":"bilhete_nao_encontrado"}` |
| UC2-T03 | Bilhete já encerrado | Encerrar novamente | `409 {"erro":"bilhete_ja_encerrado"}` |
| UC2-T04 | Duração dentro da tolerância | Encerrar | Valor `0` |
| UC2-T05 | Duração uma unidade acima da tolerância | Encerrar | Valor maior que `0`, salvo teto ou tarifa zero inexistente |
| UC2-T06 | Duração longa | Encerrar | Valor limitado pelo teto |

### UC3 — Listar ativos

| Caso | Preparação | Esperado |
|---|---|---|
| UC3-T01 | Criar dois abertos com entradas diferentes | Lista vem do mais recente para o mais antigo. |
| UC3-T02 | Encerrar um dos bilhetes | Encerrado não aparece em ativos. |
| UC3-T03 | Cancelar um bilhete | Cancelado não aparece em ativos. |
| UC3-T04 | Nenhum aberto | Retorna `[]`. |

### UC4 — Relatório diário

| Caso | Preparação | Esperado |
|---|---|---|
| UC4-T01 | Dois encerrados na mesma data de saída | `total_bilhetes = 2`, soma valores, média dos minutos. |
| UC4-T02 | Um aberto na data | Aberto é ignorado. |
| UC4-T03 | Um cancelado na data | Cancelado é ignorado. |
| UC4-T04 | Data sem encerrados | Zeros. |
| UC4-T05 | Durações 1 e 2 minutos | Média `1.5` arredonda para `2`. |
| UC4-T06 | `data=2026/10/05` | `422 {"erro":"data_invalida"}`. |

### UC5 — Cancelar bilhete

| Caso | Preparação | Esperado |
|---|---|---|
| UC5-T01 | Bilhete aberto | Cancelamento retorna `200` e status `cancelado`. |
| UC5-T02 | Bilhete encerrado | Cancelar retorna `409 {"erro":"bilhete_nao_aberto"}`. |
| UC5-T03 | Bilhete já cancelado | Cancelar retorna `409 {"erro":"bilhete_nao_aberto"}`. |
| UC5-T04 | Id inexistente | Cancelar retorna `404 {"erro":"bilhete_nao_encontrado"}`. |
| UC5-T05 | Após cancelar | Mesma placa pode abrir novo bilhete. |

### UC6 — Histórico por placa

| Caso | Preparação | Esperado |
|---|---|---|
| UC6-T01 | Placa com três bilhetes em status diferentes | Retorna todos, mais recentes primeiro. |
| UC6-T02 | Placa válida nunca usada | Retorna `[]`. |
| UC6-T03 | Placa ausente | `422 {"erro":"placa_invalida"}`. |
| UC6-T04 | Placa inválida | `422 {"erro":"placa_invalida"}`. |

## 4. Testes de contrato e SDLC

| Área | Verificação |
|---|---|
| Porta | Serviço responde em `http://localhost:{PORTA_SERVICO}`. |
| Container | `Containerfile` constrói imagem e inicia API. |
| Dependências | Manifesto existe e instala dependências sem passos ocultos. |
| README | Explica execução local, execução em container, testes e parâmetros da variante. |
| Erros | Todos os corpos de erro usam somente `erro` com string exata. |
| JSON | Respostas de sucesso e erro usam JSON válido. |
| Higiene | Não há segredos, tokens ou credenciais no repositório. |
