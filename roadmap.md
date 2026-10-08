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
