# Constitution — Zona Azul Digital

> [!NOTE]
> Este arquivo define regras persistentes para o agente que irá gerar o código. O contrato de API prevalece sobre exemplos ilustrativos e sobre escolhas de implementação.

## 1. Princípios operacionais

| ID | Regra operacional concreta | Impacto no código gerado |
|---|---|---|
| OP-01 | O entregável final é uma API REST para bilhetes de estacionamento rotativo, sem front-end e sem back-office. | Não criar telas, autenticação, painel administrativo ou fluxos fora do contrato. |
| OP-02 | Todos os valores monetários devem ser representados e retornados como centavos inteiros. | Nunca retornar `valor`, decimal, float, string monetária ou separador decimal. |
| OP-03 | Todas as datas/horas externas devem ser ISO-8601 com fuso `-03:00`. | Normalizar respostas de `entrada` e `saida` para strings com offset `-03:00`. |
| OP-04 | O contrato REST é obrigatório e exato: método, rota, status HTTP, nomes de campos e corpos de erro devem bater com a especificação. | Não renomear campos; não trocar códigos de erro; não envolver respostas em objetos extras. |
| OP-05 | A validação de formato vem antes de regra de negócio. | Payload inválido retorna `422` mesmo que também pudesse haver conflito `409`. |
| OP-06 | A aplicação deve ler os parâmetros da variante em `variante/params.json` e usar esses valores em tempo de execução. | Não fixar tarifa, fração, teto, tolerância ou porta no código. |
| OP-07 | O serviço deve escutar em `PORTA_SERVICO` da variante, em `0.0.0.0`, sem exigir variável de ambiente obrigatória. | A suíte acessará `http://localhost:{PORTA_SERVICO}`. |
| OP-08 | A persistência deve sobreviver durante o processo de testes, mas não precisa ser distribuída. | SQLite local ou armazenamento persistente simples é aceitável; memória pura é arriscada se o servidor reiniciar. |
| OP-09 | Logs não devem conter segredos, tokens ou dados sensíveis. Placa pode aparecer somente em logs técnicos essenciais. | Não criar segredos no repositório; não depender de credenciais. |
| OP-10 | Cancelar bilhete não gera cobrança, `saida`, `minutos` nem `valor_centavos`. | O status muda para `cancelado` e a placa fica liberada para novo bilhete. |

## 2. Precedência de fontes

1. `contrato.json` e `ENUNCIADO.md` definem o comportamento obrigatório.
2. `variante/params.json` define os valores concretos da variante.
3. Estes arquivos `.md` orientam o agente de implementação.
4. Exemplos marcados como inconsistentes devem ser ignorados.

> [!WARNING]
> Não copiar o exemplo antigo com `valor: 12.50`. A API deve retornar `valor_centavos` como inteiro e o corpo completo do encerramento.

## 3. Estados permitidos do bilhete

| Status | Significado | Pode encerrar? | Pode cancelar? | Ocupa placa? | Entra no relatório? |
|---|---|---:|---:|---:|---:|
| `aberto` | Bilhete ativo, com entrada e sem saída. | Sim | Sim | Sim | Não |
| `encerrado` | Bilhete finalizado com saída, minutos e valor. | Não | Não | Não | Sim, pela data de `saida` |
| `cancelado` | Bilhete aberto cancelado sem cobrança. | Não | Não | Não | Não |

## 4. Restrições para o agente de código

- Não alterar `scripts/`, `.github/`, `docs/`, `track.json`, `contrato.json`, `rubrica.json` ou `ENUNCIADO.md`.
- Não inserir blocos de implementação nos `.md`.
- Criar código, testes, manifesto de dependências, `Containerfile` e `README.md` fora dos arquivos protegidos.
- Manter nomes de erros exatamente como especificados.
- Usar testes automatizados próprios para validar contrato, bordas de cobrança e relatório.

[^centavos]: Centavos inteiros evitam bugs de ponto flutuante e facilitam a correção automática.
