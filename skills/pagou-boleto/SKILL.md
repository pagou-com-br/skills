---
name: pagou-boleto
description: Implementa e testa boletos na API Pagou. Use quando o usuário pedir boleto, cobrança com vencimento por boleto, linha digitável, código de barras, PDF ou link da fatura, multa, juros ou desconto de boleto, boleto com Pix vinculado, cancelamento de boleto ou os webhooks charge.created e charge.paid. Cobre o registro assíncrono, a diferença entre registro e pagamento, a reconciliação do Pix vinculado, cancelamento versus devolução e a recuperação sem idempotência. Use junto com a skill pagou.
---

# Boleto na API Pagou

Aplique primeiro as regras da skill `pagou` (ambientes, autenticação no backend, segurança e resultados incertos). Esta skill traz só as decisões do boleto. Os campos e payloads estão nas páginas abaixo: **leia-as antes de implementar**.

## Leia antes de implementar

- [Meio de Pagamento: Boleto](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-boleto.md)
- [Criando um Boleto](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-boleto/criando-um-boleto.md)
- [Consultando um Boleto](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-boleto/consultando-um-boleto.md)
- [Webhooks de boleto](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-boleto/webhooks.md)
- Quando o fluxo pedir: [Cancelando um Boleto](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-boleto/cancelando-um-boleto.md)
- Para testar: [Simulando pagamentos em Sandbox](https://docs.pagou.com.br/integracao-com-a-api/simulando-pagamentos-em-sandbox.md)

## Pré-requisitos

Conta habilitada, chave do ambiente, cadastro e endereço do lojista completos, dados válidos do pagador e um endpoint de webhook preparado (skill `pagou-webhooks`). Se algum faltar, informe antes de implementar.

## Regras de decisão

- **Valores:** `amount` em reais, com até duas casas decimais. Multa, juros e desconto têm formatos próprios descritos na página de criação; confira o valor final após desconto.
- **Campos opcionais:** omita quando não usar; não envie `null`.
- **Datas:** vencimento em `YYYY-MM-DD`. Atualize as datas dos exemplos antes de executar. A data limite do desconto não pode ser posterior ao vencimento.
- **Identificador:** guarde o `id` do corpo da resposta `201` e associe-o ao pedido. Use-o nas consultas, mesmo que o header `Location` traga outro formato de caminho. Não o substitua por `transaction_id` ou código do pedido.
- **Registro assíncrono:** a criação não devolve necessariamente linha digitável, código de barras ou PDF. Aguarde o evento `charge.created` ou consulte até os dados de registro aparecerem, e só então apresente linha digitável, código de barras ou links da fatura.
- **Registro não é pagamento:** `charge.created` informa o registro. O pagamento é confirmado por `charge.paid` autenticado ou pela consulta no estado de pago documentado. O valor pago pode diferir do original por encargos ou descontos: use o valor efetivamente pago.
- **Boleto com Pix vinculado:** quando a consulta trouxer um Pix vinculado, trate os identificadores como partes da **mesma** cobrança comercial. A confirmação por um meio exige reconciliar o outro; não crie dois pedidos. Um boleto cancelado não prova que o pedido não foi pago pelo Pix.
- **Cancelamento não devolve dinheiro:** cancelar interrompe a cobrança; não devolve um pagamento já recebido. O sucesso do cancelamento (`204`, sem corpo) não encerra o pedido: consulte o boleto e o meio vinculado até a atualização.
- **Split:** siga a skill `pagou-split`.

## Repetição e recuperação

A criação de boleto **não tem idempotência**. `customer_code` e `metadata` ajudam a correlacionar, mas não impedem duplicidade.

- Nunca repita uma criação ou um cancelamento automaticamente após timeout, conexão interrompida ou `5xx`.
- Se recebeu o `id`, consulte o recurso. Se não recebeu, liste os boletos pela janela de criação e pelo valor e reconcilie antes de emitir outro.
- A listagem de boletos tem formato, filtros de data e paginação próprios, diferentes da listagem de cartão. Confira na página de consulta.

## Ferramentas MCP

| Ferramenta | Uso |
| --- | --- |
| `create_boleto` | Criar boleto |
| `get_boleto`, `list_boletos` | Consultar registro, pagamento e reconciliar |
| `simulate_charge_payment` | Simular o pagamento da cobrança pelo Pix vinculado (só `pagou-sandbox`) |

- Confirme o servidor antes de criar: `pagou-sandbox` para testes; `pagou` emite boleto real e exige pedido explícito do usuário.
- Em "resultado incerto", não chame `create_boleto` de novo: use `list_boletos` e `get_boleto` para reconciliar.
- O MCP não cancela boletos. Implemente o cancelamento no backend conforme a documentação ou oriente o usuário a usar o painel.

## Teste em Sandbox

1. Crie o boleto em Sandbox, sem split, e guarde o `id`.
2. Aguarde `charge.created` ou consulte até o registro e o Pix vinculado estarem disponíveis.
3. Simule o pagamento pelo Pix vinculado (`simulate_charge_payment` ou a rota documentada, usando o `id` da cobrança, não o do QR Code). A simulação não cobre pagamento pela linha digitável, e o aceite não confirma o pagamento.
4. Consulte a cobrança e o Pix vinculado até o estado de pago e, se houver `notification_url`, valide o `charge.paid` recebido.
5. Confirme que o pedido muda uma única vez e que uma falha de comunicação leva à reconciliação, não a um novo boleto.
