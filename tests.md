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
