---
name: pagou-cartao
description: Implementa cobranças de cartão de crédito na API Pagou em checkout próprio. Use quando o usuário pedir pagamento com cartão, parcelamento, tokenização no navegador, card_token, cadastro de pagador ou cartão, cobrança por customer_id e card_id, antifraude, captura, cancelamento, estorno, nova tentativa (retry), Idempotency-Key de cartão, chargebacks ou credenciamento de cartão na Pagou. Cobre a arquitetura navegador e backend, a escolha entre dados completos e referências, estados authorized, captured e settled, idempotência e o teste em Sandbox. O MCP da Pagou só lê cobranças de cartão. Use junto com a skill pagou.
---

# Cartão de crédito na API Pagou

Aplique primeiro as regras da skill `pagou` (ambientes, autenticação no backend, segurança e resultados incertos). Esta skill traz só as decisões do cartão. Os campos e payloads estão nas páginas abaixo: **leia-as antes de implementar**.

## Leia antes de implementar

- [Meio de Pagamento: Cartão de Crédito](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito.md)
- [Tokenizando um cartão](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/tokenizando-um-cartao.md)
- [Criando uma cobrança](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/criando-uma-cobranca.md)
- [Consultando e listando cobranças](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/consultando-e-listando-cobrancas.md)
- [Idempotência e repetição](https://docs.pagou.com.br/integracao-com-a-api/idempotencia-e-repeticao.md)
- Fluxo por referências: [Criando um cliente (pagador)](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/criando-um-cliente-pagador.md), [Criando um cartão](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/criando-um-cartao.md), [Gerenciando pagadores](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/gerenciando-pagadores.md) e [Gerenciando cartões](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/gerenciando-cartoes.md)
- Quando o fluxo pedir: [Capturando uma cobrança autorizada](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/capturando-uma-cobranca.md), [Cancelando uma cobrança](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/cancelando-uma-cobranca.md), [Estornando uma cobrança](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/estornando-uma-cobranca.md), [Realizando uma nova tentativa](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/realizando-uma-nova-tentativa.md), [Atualizando uma cobrança](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/atualizando-uma-cobranca.md) e [Chargebacks](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/chargebacks.md)
- Habilitação da conta: [Solicitando credenciamento](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/solicitando-credenciamento.md)

## Pré-requisitos

Conta ativa, **credenciamento de cartão aprovado**, plano com tarifa de cartão configurada e chave válida. O credenciamento é solicitado pelo painel, não pela API. Se a conta não estiver credenciada, informe o usuário antes de implementar.

## Arquitetura obrigatória

1. **Navegador:** o checkout próprio coleta os dados do cartão e gera o token com a biblioteca de tokenização da Pagou, no mesmo ambiente da API usada na cobrança. A chave pública da biblioteca **não** é a `X-API-KEY`.
2. **Navegador para o backend:** envie ao servidor da loja apenas o objeto `card` tokenizado e os dados da compra. Limpe número e CVV dos campos e do estado do formulário depois de tokenizar.
3. **Backend do lojista:** cria a cobrança na API com a `X-API-KEY`, que nunca fica no navegador.

Número completo e CVV não fazem parte do contrato da API. Não registre número, CVV ou token em logs, analytics ou armazenamento do navegador. O checkout Pagou já faz a tokenização: esta arquitetura vale para checkout próprio.

## Regras de decisão

- **Valores em centavos inteiros:** `value: 5000` representa R$ 50,00. Não reaproveite a unidade de Pix ou boleto.
- **Escolha exatamente um fluxo:**
  1. Dados completos: objetos `customer` e `card` (com o token).
  2. Referências: `customer_id` e `card_id` do mesmo pagador, ambos da conta autenticada. Não gere novo token nesse fluxo.

  Não misture os fluxos nem envie apenas um elemento do par. No fluxo completo, guarde os IDs retornados para reutilização futura.
- **Antifraude:** siga os requisitos documentados, incluindo o IP real do comprador. Nunca use o IP ilustrativo dos exemplos.
- **Parcelas:** respeite o limite da bandeira e o cartão não pode estar expirado.
- **Resultado financeiro pelo `status`:** `201` não garante aprovação. Diferencie `authorized` (autorização), `captured` (captura) e `settled` (liquidação registrada). O estado `unknown` da cobrança não é o mesmo que um erro de comunicação com resultado incerto.
- **Captura não é etapa fixa:** a criação pode já retornar `captured`. Só chame a captura explícita para uma cobrança que permanece `authorized` e quando essa for a intenção comercial. Nunca use captura para resolver timeout.
- **Cancelamento ou estorno:** cancelamento vale para cobrança pendente; uma cobrança autorizada, capturada ou liquidada segue as regras de estorno, que é integral no contrato atual. Confira o estado antes de escolher.
- **Nova tentativa (`/retry`):** é outra operação financeira, só após falha confirmada (`failed`), com chave própria e dentro do limite documentado. Nunca use `/retry` para resolver timeout ou resultado incerto.
- **Chargebacks:** use o `id` do chargeback nas operações da disputa; o estado da disputa é diferente do estado `charged_back` da cobrança. Defesa e aceite são decisões do lojista: implemente apenas quando pedido.
- **Split:** siga a skill `pagou-split`; no cartão, a cobrança usa centavos, mas o valor fixo do split usa reais.

## Idempotência

- Gere uma `Idempotency-Key` exclusiva por operação comercial (cadastrar pagador, cadastrar cartão, criar cobrança, capturar, estornar, nova tentativa) e armazene-a com a requisição **antes** do primeiro envio.
- Para repetir a mesma operação, mantenha a mesma chave, método, caminho e os **bytes** do corpo, inclusive `card_token` e o array de split. Não remonte o JSON.
- Nova compra exige nova chave; pode reutilizar `customer_id` e `card_id`.
- Não gere chave nova dentro de um laço de repetição nem para contornar timeout.
- Códigos `idempotency_key_conflict`, `idempotency_in_progress` e `idempotency_outcome_unknown` têm tratamentos distintos: leia a página de idempotência. Resultado incerto exige investigação pela consulta, não nova cobrança.

## Ferramentas MCP

O MCP da Pagou **só lê** cobranças de cartão:

| Ferramenta | Uso |
| --- | --- |
| `get_card_charge` | Consultar uma cobrança pelo ID |
| `list_card_charges` | Listar e reconciliar por status e período |

O MCP não cria, captura, cancela, estorna nem tenta de novo cobranças de cartão, não gerencia pagadores ou cartões e não atua em chargebacks. A criação acontece no backend do lojista, depois da tokenização no navegador. Não tente criar cobrança de cartão por outro caminho a partir do agente.

## Teste em Sandbox

1. Confirme que a conta de Sandbox atende aos pré-requisitos de cartão.
2. Tokenize no ambiente de Sandbox o cartão de teste `4111 1111 1111 1111`, conforme a página de tokenização.
3. Crie a cobrança pelo backend da aplicação contra `https://api.pagou.cloud`, com a chave de Sandbox e uma `Idempotency-Key` própria. As rotas de simulação de Pix não se aplicam a cartão.
4. Confira o `status` retornado e consulte a mesma cobrança (inclusive com `get_card_charge`).
5. Teste a repetição da mesma requisição com a mesma chave e confirme que não surge cobrança duplicada.
