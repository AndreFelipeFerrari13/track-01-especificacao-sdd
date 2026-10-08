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

## 3. Estrutura de arquivos sugerida

| Caminho | Propósito |
|---|---|
| `app/` | Código da API e domínio. |
| `app/main.*` | Ponto de entrada HTTP. |
| `app/config.*` | Leitura de `variante/params.json`. |
| `app/models.*` | Modelo lógico do bilhete. |
| `app/repository.*` | Persistência SQLite ou equivalente. |
| `app/services.*` | Casos de uso UC1–UC8. |
| `app/billing.*` | Cálculo de cobrança. |
| `tests/` | Testes próprios automatizados. |
| `Containerfile` | Build e execução do serviço. |
| `README.md` | Como instalar, executar, testar e verificar endpoints. |
| `requirements.txt` ou `pyproject.toml` | Manifesto de dependências. |

## 4. Contrato de execução

| Item | Regra |
|---|---|
| Host | `0.0.0.0` |
| Porta | `PORTA_SERVICO` carregada da variante |
| Base URL | `http://localhost:{PORTA_SERVICO}` |
| Formato | JSON |
| Timezone | `-03:00` |
| Variáveis obrigatórias | Nenhuma |
| Arquivos protegidos | Não alterar `scripts/`, `.github/`, `docs/`, `track.json`, `contrato.json`, `rubrica.json`, `ENUNCIADO.md` |

> [!WARNING]
> Não depender de porta padrão fixa como `8000` ou `8080` para a suíte. O servidor precisa escutar na porta definida pela variante.

## 5. Estratégia de persistência

| Requisito | Decisão |
|---|---|
| Identificador | Gerar `id` inteiro positivo único. |
| Consulta ativos | Índice ou filtro por `status = aberto`, ordenado por `entrada` desc e `id` desc. |
| Consulta histórico | Filtro por `placa`, qualquer status, ordenado por `entrada` desc e `id` desc. |
| Relatório diário | Filtrar encerrados por data local da `saida`. |
| Concorrência mínima | Garantir que duas aberturas simultâneas da mesma placa não criem dois abertos. |

## 6. Estratégia de testes próprios

| Tipo | Cobertura mínima |
|---|---|
| Unitários | Cálculo de fração, tolerância, teto, arredondamento da média. |
| Integração REST | UC1 a UC8 com status, body e ordenação. |
| Erros | 422, 404 e 409 conforme tabela do contrato. |
| SDLC | Verificar que serviço inicia, porta correta, dependências e README existem. |

## 7. Riscos e mitigação

| Risco | Impacto | Mitigação |
|---|---|---|
| Usar float em dinheiro | Falha em valores escondidos | Usar centavos inteiros e operações determinísticas. |
| Ignorar `entrada` opcional | Suíte não consegue testar cobrança sem esperar | Implementar `entrada` em UC1. |
| Ordenação incorreta | Falha em UC3/UC6 | Ordenar por `entrada` desc e `id` desc. |
| Relatório usar data de entrada | Falha em UC4 | Filtrar pela data local de `saida`. |
| Tolerância descontada indevidamente | Falha em UC7 | Se passou da tolerância, cobrar desde o minuto zero. |
| Porta hardcoded | Serviço inacessível pela suíte | Ler `PORTA_SERVICO` de `params.json`. |

[^sdlc]: A suíte escondida pode avaliar higiene de projeto, dependências, containerização, README e testes próprios, além do contrato REST.
