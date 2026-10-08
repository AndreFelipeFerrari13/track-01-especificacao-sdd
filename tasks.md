# Tasks — Decomposição de Implementação

> [!NOTE]
> Executar as tarefas em ordem reduz risco de retrabalho. Cada tarefa deve manter rastreabilidade com UC, regra de negócio ou requisito não funcional.

## 1. Backlog priorizado

| ID | Tarefa | UCs/Requisitos | Prioridade | Critério de pronto |
|---|---|---|---|---|
| T01 | Criar estrutura mínima do projeto de API, manifesto de dependências e ponto de entrada HTTP. | RNF-01, RNF-02 | Alta | Serviço inicia e responde healthcheck ou rota real na porta da variante. |
| T02 | Implementar carregamento de `variante/params.json`. | OP-06, OP-07 | Alta | Parâmetros são lidos sem hardcode e usados por porta e cobrança. |
| T03 | Criar modelo lógico e persistência de bilhetes. | UC1–UC6 | Alta | É possível salvar, buscar por id, listar ativos, histórico e relatório. |
| T04 | Implementar validações de placa, entrada e data. | UC1, UC4, UC6 | Alta | Erros 422 retornam corpos exatos. |
| T05 | Implementar UC1 abrir bilhete com conflito de placa aberta. | UC1, UC8 | Alta | Abertura retorna 201; duplicidade aberta retorna 409. |
| T06 | Implementar serviço de cobrança. | UC2, UC7 | Alta | Fração, tolerância e teto passam em testes unitários. |
| T07 | Implementar UC2 encerramento. | UC2 | Alta | Resposta 200 tem campos exatos; 404 e 409 corretos. |
| T08 | Implementar UC3 listar ativos. | UC3 | Média | Lista somente abertos, mais recentes primeiro. |
| T09 | Implementar UC5 cancelamento. | UC5 | Média | Cancela apenas abertos e libera placa. |
| T10 | Implementar UC6 histórico por placa. | UC6 | Média | Retorna qualquer status da placa, mais recentes primeiro. |
| T11 | Implementar UC4 relatório diário. | UC4 | Alta | Filtra por data de saída, soma valores e arredonda média 0,5 para cima. |
| T12 | Criar testes automatizados próprios. | Critério D, tests.md | Alta | Testes cobrem contrato, bordas, erros e SDLC. |
| T13 | Criar `Containerfile`. | Critério D | Alta | Imagem executa API na porta da variante. |
| T14 | Criar `README.md`. | Critério D | Média | Documenta instalação, execução, testes, porta e exemplos. |
| T15 | Revisar higiene do repositório. | Segurança/SDLC | Média | Sem segredos, sem alterações em arquivos protegidos, sem logs sensíveis. |

## 2. Ordem recomendada de execução

| Fase | Tarefas | Resultado |
|---|---|---|
| Base | T01, T02, T03 | Projeto executável com persistência e config. |
| Contrato principal | T04, T05, T06, T07 | Abertura e encerramento corretos. |
| Consultas e estados | T08, T09, T10 | Ativos, cancelamento e histórico. |
| Relatórios | T11 | Agregação diária correta. |
| Qualidade | T12, T13, T14, T15 | SDLC completo para correção escondida. |

## 3. Critérios globais de aceite

| ID | Critério |
|---|---|
| G-01 | Todos os endpoints UC1–UC8 existem exatamente nas rotas especificadas. |
| G-02 | Todos os status HTTP e corpos de erro são exatos. |
| G-03 | `valor_centavos` e `faturamento_centavos` são inteiros. |
| G-04 | A porta do serviço vem de `PORTA_SERVICO`. |
| G-05 | Tolerância, fração e teto usam os valores reais da variante. |
| G-06 | Ativos e histórico são ordenados do mais recente para o mais antigo. |
| G-07 | Relatório diário considera somente bilhetes encerrados pela data de saída. |
| G-08 | Cancelamento não gera cobrança nem saída. |
| G-09 | Existe documentação e teste próprio. |
| G-10 | Arquivos protegidos não são alterados. |

