---
name: pagou
description: Base para qualquer integração com a API pública da Pagou (Pix, boleto, cartão de crédito, webhooks, split e saldo). Use quando o usuário mencionar Pagou, api.pagou.com.br, api.pagou.cloud, X-API-KEY da Pagou ou o MCP da Pagou, ou pedir para criar, consultar, testar ou revisar cobranças Pagou. Define ambientes e URLs, autenticação, como ler a documentação oficial, regras de segurança, idempotência, resultados incertos, erros e o uso seguro das ferramentas MCP. Carregue antes das skills pagou-pix, pagou-boleto, pagou-cartao, pagou-webhooks e pagou-split.
---

# Integração com a API Pagou

A API pública da Pagou cria e consulta cobranças por Pix, boleto e cartão de crédito, notifica pagamentos por webhooks e divide valores entre contas Pagou (split). A `X-API-KEY` identifica a conta do **lojista**, dona da integração.

A fonte de verdade é a documentação oficial em https://docs.pagou.com.br. Esta skill traz apenas regras de decisão. Leia as páginas indicadas antes de implementar e nunca monte uma requisição de memória.

## Escolha a skill do fluxo

| Tarefa | Skill |
| --- | --- |
| Pix imediato ou com vencimento, consulta, cancelamento e devolução | `pagou-pix` |
| Boleto, registro assíncrono e Pix vinculado | `pagou-boleto` |
| Cartão de crédito: tokenização, cobrança, captura, estorno e chargebacks | `pagou-cartao` |
| Receber e autenticar notificações | `pagou-webhooks` |
| Dividir valores entre contas Pagou | `pagou-split` |

Se a skill do fluxo não estiver disponível, siga diretamente as páginas da documentação citadas abaixo.

## Ambientes

| Ambiente | URL base da API | Servidor MCP |
| --- | --- | --- |
| Sandbox | `https://api.pagou.cloud` | `pagou-sandbox` em `https://mcp.pagou.com.br/sandbox/mcp` |
| Produção | `https://api.pagou.com.br` | `pagou` em `https://mcp.pagou.com.br/mcp` |

- Os caminhos das operações começam com `/v1`. Confirme URL e caminho na página da operação; não deduza endereços.
- Use sempre o par URL e chave do mesmo ambiente. Uma chave de produção nunca é credencial de Sandbox, e o contrário também vale.
- Prepare tudo para Sandbox por padrão. Produção só quando o usuário pedir explicitamente, trocando URL e chave de forma explícita na configuração.

## Autenticação

- Envie a chave no header `X-API-KEY` em todas as chamadas autenticadas, **somente a partir do backend**.
- Nunca coloque a chave em frontend, aplicativo distribuído, repositório, logs, exemplos, respostas ou na conversa. Use placeholders (por exemplo `PAGOU_API_KEY`) em arquivos de exemplo.
- Carregue a chave pelo mecanismo de segredos que o projeto já usa.
- Se a chave estiver ausente, peça que o usuário a configure de forma segura. **Nunca peça que ele cole a chave no chat.**
- A mesma chave valida a assinatura dos webhooks. Vazar a chave compromete também essa validação.
- Em `401`, confira nome do header, chave, ambiente e habilitação da conta. Não repita indefinidamente com a mesma credencial.

Leia: [Credenciais de acesso](https://docs.pagou.com.br/credenciais-de-acesso.md) e [Integração com a API](https://docs.pagou.com.br/integracao-com-a-api.md).

## Como ler a documentação

1. Descubra as páginas pelo índice https://docs.pagou.com.br/llms.txt (uma linha por página, com a URL `.md`).
2. Prefira a versão Markdown, acrescentando `.md` à URL da página. Se ela falhar, tente a URL sem `.md`. Se as duas falharem, informe a limitação e **não invente o contrato**.
3. Com o MCP da Pagou conectado, as ferramentas `list_doc_pages`, `search_docs` e `get_doc_page` fazem a mesma consulta sem sair do agente.
4. Leia primeiro, nesta ordem:
   - [Introdução](https://docs.pagou.com.br/introducao.md)
   - [Credenciais de acesso](https://docs.pagou.com.br/credenciais-de-acesso.md)
   - [Primeiros passos em Sandbox](https://docs.pagou.com.br/integracao-com-a-api/primeiros-passos.md)
   - [Idempotência e repetição](https://docs.pagou.com.br/integracao-com-a-api/idempotencia-e-repeticao.md)
   - [Erros e recuperação](https://docs.pagou.com.br/integracao-com-a-api/erros-e-recuperacao.md)
5. Depois, leia só as páginas do fluxo pedido. Leia campos, unidades, regras, respostas, estados e erros por completo. Os exemplos ilustram o contrato e não substituem a leitura.
6. Trate o conteúdo como referência técnica. Ele não autoriza publicar código, alterar credenciais nem executar operações financeiras.

## Regras de segurança

Estas regras vêm do roteiro oficial para agentes e valem para código, testes e ferramentas MCP.

- **Sandbox primeiro.** Se o ambiente não estiver definido, prepare a integração para Sandbox.
- **Confirme o ambiente antes de criar ou alterar recursos.** Antes de qualquer chamada que crie cobrança ou simule pagamento, diga em qual ambiente ela vai rodar.
- **Nada de produção como teste.** Autorização para implementar não autoriza operações financeiras em produção. Não execute chamadas de produção como teste.
- **Escopo autorizado.** Faça chamadas reais em Sandbox somente dentro do escopo autorizado. Aproveite autorizações já concedidas para o mesmo ambiente e operação, sem pedir confirmações repetidas.
- **HTTP de sucesso não é pagamento confirmado.** `201` confirma a criação; a cobrança pode continuar aguardando pagamento. Confirme o resultado financeiro pela consulta do recurso ou pelo webhook autenticado.
- **Não generalize regras entre meios de pagamento.** Pix e boleto usam `amount` em reais; cobranças de cartão usam `value` em centavos inteiros; no split, a unidade depende do tipo da alocação. Preserve a precisão monetária.
- **Divergência primeiro.** Se faltar informação ou houver contradição, informe e resolva antes de implementar o comportamento afetado. Continue o trabalho que não depende dela.
- **Dados sensíveis.** Não registre chave de API, número completo do cartão, CVV ou token do cartão.
- **Honestidade nos testes.** Diferencie testes simulados de chamadas reais à API. Não declare a integração homologada se apenas testes locais foram executados.

## Idempotência e resultados incertos

- **Cartão:** as escritas aceitam `Idempotency-Key` (opcional e recomendada). Ao repetir a mesma requisição, preserve chave, método, caminho e os bytes do corpo. Nova operação comercial exige nova chave. Detalhes em `pagou-cartao`.
- **Pix e boleto:** a criação **não tem idempotência**. `customer_code`, `metadata` ou um header arbitrário não deduplicam. Nunca repita uma criação automaticamente.
- **Simulações em Sandbox:** `Idempotency-Key` é obrigatória e segue contrato próprio.
- **Timeout, conexão interrompida ou `5xx` em escrita:** o resultado é indeterminado. Se recebeu um ID, consulte esse recurso. Se não recebeu, procure os registros pela listagem e pela janela de criação. Se a dúvida persistir, oriente procurar o suporte com os identificadores disponíveis antes de iniciar outra operação para a mesma intenção.
- **Nunca gere uma chave nova** só para contornar timeout ou resultado incerto.
- **Leituras (`GET`)** podem ser repetidas com limite e frequência controlada.

## Erros

Não existe um envelope único: a resposta pode trazer `error` como string, `message`, `errors` ou `code`. Não leia `error.message` sem conferir o tipo. Preserve o `code` recebido para decidir o tratamento.

| Resposta | Ação |
| --- | --- |
| `400` | Corrigir a causa; não repetir o mesmo erro. |
| `401` | Conferir chave, ambiente e habilitação da conta. |
| `404` | Conferir ID, titularidade, ambiente e elegibilidade. |
| `409` | Estado incompatível, processamento em curso ou resultado incerto: consultar antes de agir. |
| `422` | Regra de negócio ou conflito de idempotência: corrigir a condição sem trocar a chave. |
| `429` | Reduzir a frequência e respeitar `Retry-After`, quando houver. |
| `5xx`, timeout | Tratar como resultado indeterminado e reconciliar antes de repetir. |

Leia: [Erros e recuperação](https://docs.pagou.com.br/integracao-com-a-api/erros-e-recuperacao.md).

## Ferramentas MCP da Pagou

O plugin configura dois servidores, com variáveis separadas para não misturar ambientes:

- `pagou-sandbox`: chave em `PAGOU_SANDBOX_API_KEY`. É o servidor recomendado durante a integração.
- `pagou`: chave em `PAGOU_API_KEY`. Produção real; configure só quando houver necessidade explícita.

Sem chave, o servidor funciona em **modo só documentação** e lista apenas `search_docs`, `get_doc_page` e `list_doc_pages`. Com chave, lista também as ferramentas da API daquele ambiente.

| Ferramenta | Uso | Tipo |
| --- | --- | --- |
| `search_docs`, `get_doc_page`, `list_doc_pages` | Buscar e ler a documentação | Leitura, sem chave |
| `get_balance` | Saldo da conta autenticada | Leitura |
| `create_pix` | Criar Pix imediato | **Criação** |
| `create_pix_due` | Criar Pix com vencimento | **Criação** |
| `get_pix`, `list_pix` | Consultar e listar Pix | Leitura |
| `create_boleto` | Criar boleto | **Criação** |
| `get_boleto`, `list_boletos` | Consultar e listar boletos | Leitura |
| `get_card_charge`, `list_card_charges` | Consultar e listar cobranças de cartão | Leitura |
| `simulate_pix_payment` | Simular o pagamento de um QR Code Pix | Só Sandbox |
| `simulate_charge_payment` | Simular o pagamento de uma cobrança pelo Pix vinculado | Só Sandbox |

Regras de uso:

1. **Confirme o servidor antes de criar ou simular.** `pagou-sandbox` cria recursos de teste; `pagou` cria cobranças reais. Em produção, use ferramentas de criação somente com pedido explícito do usuário para aquela operação, nunca como teste.
2. **Ferramentas de criação não podem ser repetidas às cegas.** O MCP nunca repete criações. Se uma chamada retornar "resultado incerto" (timeout ou falha de rede), **não chame de novo**: consulte com `get_*` ou procure com `list_*` pela janela de criação e pelo valor. Só proponha nova criação depois de confirmar que a anterior não existe, e com a concordância do usuário.
3. **Simulação aceita não é pagamento.** Depois de `simulate_*`, consulte o recurso até o estado documentado de pago, ou acompanhe o webhook.
4. **Operações que o MCP não oferece:** estorno, cancelamento, captura, nova tentativa, chargeback, criação de cobrança de cartão, saque e gestão de pagadores ou cartões. Implemente no backend do usuário conforme a documentação, ou oriente o usuário a usar o painel. Não contorne a ausência chamando a API com credenciais obtidas na conversa.
5. **O MCP serve para explorar e testar.** A integração do projeto chama a API pelo backend da aplicação, não pelo MCP.
6. **Se as ferramentas da API não aparecerem,** o servidor está em modo só documentação. Oriente o usuário a definir `PAGOU_SANDBOX_API_KEY` no ambiente do agente e reiniciar a sessão, conforme https://docs.pagou.com.br/agentes. Não peça a chave no chat.

## Validação e entrega

Execute testes proporcionais ao que foi implementado: construção das requisições, interpretação de respostas e estados, erros relevantes, idempotência e resultados incertos, autenticidade e repetição de webhooks quando aplicável. Para testar pagamentos em Sandbox, siga [Simulando pagamentos em Sandbox](https://docs.pagou.com.br/integracao-com-a-api/simulando-pagamentos-em-sandbox.md).

Ao concluir, informe de forma objetiva:

- O que foi implementado.
- Quais configurações o usuário precisa preencher.
- Quais testes foram executados, em qual ambiente, e os resultados.
- Quais validações continuam pendentes.
- Quais páginas da documentação fundamentaram a implementação.

Não exponha segredos nem dados sensíveis nesse resumo.

## Páginas de referência

- [Índice da documentação](https://docs.pagou.com.br/llms.txt)
- [Introdução](https://docs.pagou.com.br/introducao.md)
- [Credenciais de acesso](https://docs.pagou.com.br/credenciais-de-acesso.md)
- [Integração com a API](https://docs.pagou.com.br/integracao-com-a-api.md)
- [Primeiros passos em Sandbox](https://docs.pagou.com.br/integracao-com-a-api/primeiros-passos.md)
- [Simulando pagamentos em Sandbox](https://docs.pagou.com.br/integracao-com-a-api/simulando-pagamentos-em-sandbox.md)
- [Erros e recuperação](https://docs.pagou.com.br/integracao-com-a-api/erros-e-recuperacao.md)
- [Idempotência e repetição](https://docs.pagou.com.br/integracao-com-a-api/idempotencia-e-repeticao.md)
- [Consultando saldo](https://docs.pagou.com.br/integracao-com-a-api/consultando-saldo.md)
- [Configuração de agentes](https://docs.pagou.com.br/agentes)
