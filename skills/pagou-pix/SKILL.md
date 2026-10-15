---
name: pagou-pix
description: Implementa e testa cobranças Pix na API Pagou. Use quando o usuário pedir Pix, QR Code, Pix Copia e Cola, Pix com vencimento, consulta ou listagem de Pix, cancelamento de QR Code, devolução (estorno) de Pix ou confirmação pelo webhook qrcode.completed. Cobre a escolha entre QR Code imediato e com vencimento, apresentação do QR Code, confirmação do pagamento, cancelamento versus devolução, recuperação sem idempotência e simulação de pagamento em Sandbox. Use junto com a skill pagou.
---

# Pix na API Pagou

Aplique primeiro as regras da skill `pagou` (ambientes, autenticação no backend, segurança e resultados incertos). Esta skill traz só as decisões do Pix. Os campos e payloads estão nas páginas abaixo: **leia-as antes de implementar**.

## Leia antes de implementar

- [Meio de Pagamento: Pix](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-pix.md)
- [Criando um QR Code imediato](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-pix/criando-um-qrcode-imediato.md)
- [Criando um QR Code com vencimento](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-pix/criando-um-qrcode-com-vencimento.md)
- [Consultando um QR Code](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-pix/consultando-um-qrcode.md)
- [Webhooks de Pix](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-pix/webhooks.md)
- Quando o fluxo pedir: [Cancelando um QR Code](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-pix/cancelando-um-qrcode.md) e [Estornando um QR Code](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-pix/estornando-um-qrcode.md)
- Para testar: [Simulando pagamentos em Sandbox](https://docs.pagou.com.br/integracao-com-a-api/simulando-pagamentos-em-sandbox.md)

## Escolha a modalidade

| Situação | Modalidade |
| --- | --- |
| Pagamento no momento da compra, com validade curta definida em segundos | QR Code imediato |
| Cobrança com data de vencimento, tolerância e encargos ou desconto opcionais | QR Code com vencimento |

Os encargos do Pix com vencimento têm regras próprias. Não copie o formato de encargos do boleto.

## Regras de decisão

- **Valores:** `amount` em reais, com até duas casas decimais. Não converta para centavos.
- **Campos opcionais:** omita quando não usar; não envie `null`.
- **Datas:** atualize as datas dos exemplos para hoje ou o futuro antes de executar. O desconto não pode ser posterior ao vencimento.
- **Identificador:** guarde o `id` da Pagou retornado na criação e associe-o ao pedido. Use esse `id` nas consultas e operações. Não o substitua por `transaction_id`, código do pedido ou referência do provedor.
- **Apresentação:** mostre o código Copia e Cola e a imagem do QR Code devolvidos pela API. Não gere o QR Code por conta própria. A existência desses dados não confirma pagamento.
- **Confirmação:** confirme o pagamento pelo webhook `qrcode.completed` autenticado ou pela consulta do recurso no estado de pago documentado. Confira conta, referência e valor antes de liberar o pedido, e aplique a confirmação uma única vez.
- **Estados desconhecidos ou de remoção** exigem tratamento explícito; nunca os interprete como pagamento.
- **Cancelamento ou devolução:** cancelamento interrompe uma cobrança ainda não paga; estorno devolve valor de uma cobrança paga. Consulte o estado antes de escolher. Um pagamento pode ser confirmado perto do pedido de cancelamento: se aparecer pago, investigue antes de liberar ou devolver.
- **Devolução parcial:** a devolução pode ser parcial e o contrato registra uma solicitação por cobrança. Confira o valor efetivamente devolvido no evento `qrcode.refunded` ou na consulta; não presuma estorno integral nem vários estornos parciais sucessivos.
- **Pix vinculado a boleto:** não aplique o fluxo de QR Code independente. Siga a skill `pagou-boleto`.
- **Split:** siga a skill `pagou-split`.

## Repetição e recuperação

A criação de Pix **não tem idempotência**. `customer_code` e `metadata` ajudam a correlacionar, mas não impedem duplicidade.

- Nunca repita uma criação automaticamente após timeout, conexão interrompida ou `5xx`.
- Se recebeu o `id`, consulte o recurso. Se não recebeu, liste os Pix pela janela de criação e pelo valor e reconcilie antes de criar outro.
- Cancelamento e estorno também não devem ter repetição automática: consulte antes de reenviar.
- A listagem de Pix tem formato e paginação próprios, diferentes da listagem de cartão. Confira na página de consulta.

## Ferramentas MCP

| Ferramenta | Uso |
| --- | --- |
| `create_pix` | Criar QR Code imediato |
| `create_pix_due` | Criar QR Code com vencimento |
| `get_pix`, `list_pix` | Consultar e reconciliar |
| `simulate_pix_payment` | Simular pagamento do QR Code (só `pagou-sandbox`) |

- Confirme o servidor antes de criar: `pagou-sandbox` para testes; `pagou` cria cobrança real e exige pedido explícito do usuário.
- Em "resultado incerto", não chame `create_pix` ou `create_pix_due` de novo: use `list_pix` e `get_pix` para reconciliar.
- O MCP não cancela nem devolve Pix. Implemente essas operações no backend conforme a documentação ou oriente o usuário a usar o painel.

## Teste em Sandbox

1. Crie o QR Code em Sandbox e guarde o `id`.
2. Consulte o recurso até o código Copia e Cola estar disponível.
3. Simule o pagamento (`simulate_pix_payment` ou a rota documentada, com `Idempotency-Key` própria da intenção). O aceite da simulação não confirma o pagamento.
4. Consulte o mesmo `id` até o estado de pago e, se houver `notification_url`, valide o `qrcode.completed` recebido.
5. Confirme que o pedido muda uma única vez, inclusive com reenvio do evento, e que uma falha de comunicação leva à reconciliação, não a uma nova cobrança.
