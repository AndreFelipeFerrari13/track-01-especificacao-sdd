# Plan — Decisões Técnicas e Execução

> [!NOTE]
> Este plano orienta o agente de implementação. Ele deve produzir código validável, mas estes `.md` permanecem como especificação, não implementação.

## 1. Decisões técnicas

| ID | Decisão | Justificativa | Critério de validação |
|---|---|---|---|
| DEC-01 | Usar uma API HTTP simples com framework leve, preferencialmente Python + FastAPI ou stack equivalente disponível. | O contrato é REST, JSON e sem front-end; FastAPI facilita validação e testes. | Todos os endpoints respondem na porta `PORTA_SERVICO`. |
| DEC-02 | Usar SQLite local como persistência padrão. | Evita dependência externa, preserva dados durante a execução da suíte e suporta consultas por status, placa e data. | Reiniciar o processo não deve corromper esquema; testes conseguem criar e consultar bilhetes. |
| DEC-03 | Centralizar regras de cobrança em um serviço de domínio. | Evita divergência entre encerramento, relatório e testes. | Bordas de fração, tolerância e teto passam em testes unitários. |
| DEC-04 | Centralizar validação de placa, data e entrada. | Garante precedência de erros e evita respostas diferentes por endpoint. | Placa inválida sempre retorna `placa_invalida`; data inválida sempre retorna `data_invalida`. |
| DEC-05 | Ler `variante/params.json` na inicialização. | A correção recomputa variante por repositório; valores não podem ser hardcoded. | Alterar `params.json` altera porta e cálculo sem mexer no código. |
| DEC-06 | Armazenar horários normalizados com offset `-03:00`. | O contrato exige ISO-8601 com fuso e relatório por dia local. | Respostas de `entrada` e `saida` incluem `-03:00`. |
| DEC-07 | Criar `Containerfile`, manifesto de dependências, README e testes. | O critério D cobra SDLC do código gerado. | Build e testes documentados no README. |

## 2. Arquitetura recomendada

| Camada | Responsabilidade |
|---|---|
| API/Controllers | Mapear rotas, validar requisição, escolher status HTTP e serializar JSON. |
| Services | Aplicar regras de negócio: abertura, encerramento, cancelamento, histórico, relatório. |
| Domain/Billing | Calcular minutos, frações, tolerância, teto e valor em centavos. |
| Repository | Persistir e consultar bilhetes por id, placa, status e data de saída. |
| Config | Carregar `variante/params.json` e fornecer parâmetros tipados. |
| Tests | Validar contrato REST, regras de borda e SDLC mínimo. |
