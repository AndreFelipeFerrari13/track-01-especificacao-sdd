# Roadmap — Geração da API Zona Azul Digital

> [!NOTE]
> Este arquivo complementa `constitution.md`, `spec.md`, `plan.md`, `tests.md` e `tasks.md`. Ele define a ordem recomendada de implementação e quais módulos/dependências são realmente necessários para o agente gerar uma API REST simples, testável e containerizável.

## 1. Objetivo do roadmap

Guiar o agente de desenvolvimento na construção incremental da API REST de bilhetes de estacionamento rotativo, priorizando:

| Prioridade | Foco | Resultado esperado |
|---|---|---|
| P0 | Contrato REST exato | Endpoints, status HTTP e corpos JSON compatíveis com `contrato.json`. |
| P0 | Regras de cobrança | Fração, tolerância, teto, centavos inteiros e relatório diário corretos. |
| P1 | Casos de borda | Testes próprios cobrindo conflitos, validações e arredondamentos. |
| P1 | SDLC | `Containerfile`, manifesto de dependências, README, testes e organização de pastas. |
| P2 | Higiene | Logs sem segredos, separação de responsabilidades, código simples e manutenível. |

> [!WARNING]
> Este arquivo é especificação. Não colar implementação longa. Qualquer bloco de código acima de 20 linhas em `.md` zera a prova.

## 2. Pesquisa de módulos e decisão de dependências

A API pode ser feita com qualquer stack que respeite o contrato, mas a opção recomendada é Python com FastAPI por simplicidade, validação de JSON e facilidade de testes.

| Módulo/dependência | Necessário? | Uso no projeto | Justificativa |
|---|---:|---|---|
| `fastapi` | Sim, se a stack for Python | Definir rotas REST, serializar JSON e validar payloads. | Framework direto para APIs HTTP; depende de Pydantic e Starlette, evitando instalar esses módulos manualmente em muitos cenários. |
| `uvicorn` | Sim, se usar FastAPI/ASGI | Servir a aplicação na porta `PORTA_SERVICO`. | Servidor ASGI adequado para executar a API em container. |
| `pytest` | Sim | Rodar testes próprios do critério D e validar contrato localmente. | A documentação do FastAPI recomenda uso direto com pytest para testes. |
| `httpx` | Sim para testes com `TestClient` | Permitir testes HTTP internos com cliente de teste. | O `TestClient` do FastAPI depende de HTTPX para testes. |
| `sqlite3` | Sim, mas sem instalar via pip | Persistência local simples. | Faz parte da biblioteca padrão do Python; atende sem serviço externo. |
| `datetime` / `zoneinfo` | Sim, mas sem instalar via pip | Cálculo de entrada, saída, duração e fuso `-03:00`. | Biblioteca padrão; evita dependência externa para datas. |
| `json` / `pathlib` / `os` | Sim, mas sem instalar via pip | Ler `variante/params.json` e configurar porta. | Biblioteca padrão. |
| `pydantic` | Indireto | Modelos de entrada e saída se usado via FastAPI. | Normalmente vem como dependência do FastAPI; instalar separado só se o agente escolher uso explícito avançado. |
| `sqlalchemy` | Não obrigatório | ORM opcional. | Para esta prova, SQLite simples com camada repository é suficiente e reduz complexidade. |
| `alembic` | Não obrigatório | Migrações. | Desnecessário para escopo pequeno e esquema inicial único. |
| `python-dotenv` | Não obrigatório | Variáveis de ambiente. | O contrato informa que não deve haver env obrigatória; os parâmetros vêm de `variante/params.json`. |
| `python-multipart` | Não | Upload de arquivos/form-data. | Não há upload nem multipart no contrato. |
| `requests` | Não | Cliente HTTP externo. | A API não consome serviços externos; para testes, usar `httpx`/`TestClient`. |
| Banco externo, Redis, filas | Não | Infraestrutura distribuída. | Escopo é API local avaliada em container, sem back-office ou integração externa. |

## 3. Estrutura interna recomendada

| Módulo interno | Responsabilidade | Critério de pronto |
|---|---|---|
| `app/main` | Criar aplicação HTTP, registrar rotas e iniciar servidor. | Serviço responde em `http://localhost:{PORTA_SERVICO}`. |
| `app/config` | Ler `variante/params.json` e expor tarifa, fração, teto, tolerância e porta. | Nenhum parâmetro de variante fica hardcoded. |
| `app/models` | Representar bilhete e status lógico. | Campos compatíveis com `spec.md`. |
| `app/repositories` | Salvar e consultar bilhetes. | Busca por id, placa, status aberto e saída por data. |
| `app/services/billing` | Calcular minutos e `valor_centavos`. | Passa casos de fração exata, `+1 min`, tolerância e teto. |
| `app/services/tickets` | Orquestrar abrir, encerrar, cancelar, listar e histórico. | Regras de conflito e transição de status ficam centralizadas. |
| `app/api/routes` | Mapear endpoints e erros HTTP. | Rotas retornam status/body exatos do contrato. |
| `tests` | Testes automatizados próprios. | Cobre UCs 1–8, erros e bordas de cobrança. |

## 4. Roadmap por fases

### Fase 0 — Preparação e leitura obrigatória

| ID | Ação | Saída esperada | Bloqueia |
|---|---|---|---|
| R00 | Ler `ENUNCIADO.md`, `contrato.json`, `variante/params.json`, `docs/REGRAS.md` e `FONTES.md`. | Entendimento completo do contrato e restrições. | Todas as fases. |
| R01 | Confirmar valores reais da variante. | Tabela local com `TARIFA_HORA_CENTAVOS`, `FRACAO_MINUTOS`, `TETO_DIARIO_CENTAVOS`, `TOLERANCIA_MINUTOS`, `PORTA_SERVICO`. | Cobrança e porta. |
| R02 | Garantir que arquivos proibidos não serão alterados. | Nenhuma mudança em `scripts/`, `.github/`, `docs/`, `track.json`, `contrato.json`, `rubrica.json`, `ENUNCIADO.md`. | Push final. |

### Fase 1 — Base técnica mínima

| ID | Ação | Critério de aceite |
|---|---|---|
| R10 | Criar manifesto de dependências com apenas módulos necessários. | Contém `fastapi`, `uvicorn`, `pytest`, `httpx`; não adiciona ORM ou infra sem necessidade. |
| R11 | Criar ponto de entrada da API. | API sobe em `0.0.0.0:{PORTA_SERVICO}` lido da variante. |
| R12 | Criar persistência SQLite local. | Tabela/esquema inicial permite criar, atualizar e consultar bilhetes. |
| R13 | Criar `Containerfile`. | Container inicia o serviço sem pedir variável obrigatória. |

### Fase 2 — Núcleo de domínio

| ID | Ação | Critério de aceite |
|---|---|---|
| R20 | Implementar validação de placa. | Só aceita 7 caracteres alfanuméricos maiúsculos. |
| R21 | Implementar parsing de `entrada` opcional. | ISO-8601 com fuso válido; erro exato `entrada_invalida` quando inválido. |
| R22 | Implementar cálculo de duração em minutos. | Resultado inteiro não negativo entre `entrada` e `saida`. |
| R23 | Implementar cálculo de cobrança. | Centavos inteiros; tolerância não desconta; fração arredonda para cima; teto aplicado. |
| R24 | Implementar arredondamento do tempo médio. | Média diária arredonda `0,5` para cima. |

> [!WARNING]
> A regra mais propensa a erro é a tolerância: duração `<= TOLERANCIA_MINUTOS` custa `0`; duração `> TOLERANCIA_MINUTOS` cobra desde o primeiro minuto.

### Fase 3 — Endpoints UC1–UC8

| Ordem | UC | Endpoint | Critério de aceite |
|---:|---|---|---|
| 1 | UC1 | `POST /bilhetes` | Retorna `201` com `id`, `placa`, `entrada`, `status:"aberto"`. |
| 2 | UC8 | `POST /bilhetes` duplicado | Placa com bilhete aberto retorna `409 {"erro":"bilhete_em_aberto"}`. |
| 3 | UC2 | `POST /bilhetes/{id}/encerramento` | Encerra aberto e retorna `saida`, `minutos`, `valor_centavos`. |
| 4 | UC3 | `GET /bilhetes/ativos` | Retorna apenas abertos, mais recentes primeiro. |
| 5 | UC5 | `POST /bilhetes/{id}/cancelamento` | Cancela apenas aberto, sem cobrança, sem `saida`. |
| 6 | UC6 | `GET /bilhetes?placa=...` | Retorna histórico completo da placa, qualquer status. |
| 7 | UC4 | `GET /relatorios/diario?data=AAAA-MM-DD` | Considera somente bilhetes encerrados no dia local da `saida`. |
| 8 | Erros | Todos | Retorna status e `{"erro":"..."}` exatos. |

### Fase 4 — Testes próprios

| Grupo | Casos mínimos |
|---|---|
| Contrato REST | Status e JSON exatos para UC1–UC8. |
| Validação | Placa ausente, minúscula, curta, longa, com símbolo; entrada inválida; data inválida. |
| Conflitos | Encerrar inexistente; encerrar encerrado; cancelar encerrado; cancelar cancelado; abrir placa ocupada. |
| Cobrança | `0 min`, tolerância exata, tolerância + 1, fração exata, fração + 1, teto. |
| Ordenação | Ativos e histórico mais recentes primeiro. |
| Relatório | Dia sem encerrados; soma; média com `.5`; cancelados ignorados. |
| Variante | Teste que altera parâmetros controlados ou usa fixtures para confirmar que não há hardcode. |

### Fase 5 — SDLC e documentação

| Entregável | Conteúdo mínimo |
|---|---|
| `README.md` | Como instalar, executar, testar, subir container, ler variante e chamar endpoints. |
| `Containerfile` | Build reprodutível, porta correta e comando de execução. |
| Manifesto de dependências | Dependências mínimas e explícitas. |
| Testes | Como executar e o que cobrem. |
| Logs | Erros controlados sem segredos. |
| `FONTES.md` | Registros de consultas, incluindo IA e documentação usada. |

## 5. Sequência de validação antes do push

| Passo | Verificação |
|---|---|
| V01 | Rodar testes próprios localmente. |
| V02 | Subir API e testar manualmente UC1–UC4 pelo menos uma vez. |
| V03 | Testar casos UC5–UC8: cancelamento, histórico, tolerância e placa ocupada. |
| V04 | Verificar que nenhum `.md` tem bloco de código com mais de 20 linhas. |
| V05 | Conferir `FONTES.md` com links e onde cada fonte aparece. |
| V06 | Rodar `/auto-correcao` na issue “🎯 Prova”. |
| V07 | Corrigir apontamentos do Summary/comentário e só então fazer push final. |

## 6. Riscos e mitigação

| Risco | Impacto | Mitigação |
|---|---|---|
| Hardcode de variante | Falha em repositório com parâmetros diferentes. | Ler sempre `variante/params.json`. |
| Retornar dinheiro em float | Quebra contrato e testes escondidos. | Usar centavos inteiros em todos os cálculos. |
| Aplicar tolerância como desconto | Cobrança errada em UC7. | Testar `tolerância + 1` cobrando desde o minuto zero. |
| Relatório filtrar por `entrada` | Totais errados. | Filtrar por data local de `saida` de bilhetes encerrados. |
| In-memory puro | Perda de dados se processo reiniciar. | Preferir SQLite local simples. |
| Dependências excessivas | Maior chance de falha no container. | Manter apenas módulos necessários. |
| Alterar arquivos proibidos | Nota zero. | Nunca editar paths bloqueados pelo enunciado. |

## 7. Fontes consultadas para decisão de módulos

| Fonte | Uso |
|---|---|
| Documentação FastAPI — página principal | Confirma relação com Pydantic/Starlette e uso com Uvicorn. |
| Documentação FastAPI — Testing | Confirma uso de `pytest` e necessidade de `httpx` para `TestClient`. |
| Documentação Uvicorn — Installation/Deployment | Confirma Uvicorn como servidor ASGI instalável e executável por porta. |
| Contrato da prova (`contrato.json`) | Define endpoints, erros, variante, fuso, porta e regra fatal de `.md`. |
