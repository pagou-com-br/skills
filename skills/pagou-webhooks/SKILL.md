---
name: pagou-webhooks
description: Implementa o recebimento de webhooks da Pagou com segurança. Use quando o usuário pedir endpoint de notificação, notification_url, confirmação de pagamento por evento, validação de assinatura X-Pagou-Signature ou X-Pagou-Timestamp, HMAC, deduplicação, reenvios ou os eventos qrcode.completed, qrcode.refunded, charge.created e charge.paid. Cobre captura do corpo bruto, validação HMAC sobre timestamp e corpo bruto, janela de tempo, deduplicação durável, resposta 200, eventos fora de ordem, rotação de chave e testes. Use junto com a skill pagou.
---

# Webhooks da Pagou

Aplique primeiro as regras da skill `pagou`. Esta skill traz só as decisões do recebimento de notificações. Headers, formato da assinatura e payloads estão nas páginas abaixo: **leia-as antes de implementar**.

## Leia antes de implementar

- [Recebendo webhooks](https://docs.pagou.com.br/integracao-com-a-api/recebendo-webhooks.md)
- [Autenticação de Webhooks](https://docs.pagou.com.br/integracao-com-a-api/autenticacao-de-webhooks.md)
- Eventos do meio de pagamento: [Webhooks de Pix](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-pix/webhooks.md) e [Webhooks de boleto](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-boleto/webhooks.md)

## Eventos públicos

| Evento | Significado |
| --- | --- |
| `qrcode.completed` | Pagamento Pix confirmado |
| `qrcode.refunded` | Devolução Pix confirmada, possivelmente parcial |
| `charge.created` | Dados de registro do boleto disponíveis (não é pagamento) |
| `charge.paid` | Pagamento do boleto confirmado |

O envelope traz o nome do evento e os dados; os campos de dados variam por evento, então leia o contrato de cada um. Cartão não tem esses webhooks públicos: acompanhe pela consulta de cobranças e de chargebacks. Não use nomes de webhooks internos do provedor como eventos Pagou.

## Configuração

- Informe `notification_url` na criação do Pix ou do boleto.
- Use uma URL HTTPS pública, com rota capaz de ler os bytes originais do corpo.
- A chave que assina cada evento é a API Key associada à cobrança que o originou. Leia-a de um segredo do servidor.

## Procedimento de recebimento

1. **Capture o corpo bruto antes de qualquer parse.** Configure o framework para entregar os bytes originais nessa rota. JSON remontado, reordenado ou reformatado não serve para validar a assinatura.
2. **Exija os headers** de timestamp e assinatura e uma chave conhecida para a cobrança.
3. **Confira a janela de tempo** documentada entre o timestamp e o relógio do servidor. Mantenha o relógio sincronizado.
4. **Calcule o HMAC-SHA256** sobre o timestamp concatenado ao corpo bruto, conforme a página de autenticação, e **compare em tempo constante**.
5. **Só depois** faça o parse e valide o nome do evento e o subconjunto de dados de que o fluxo precisa.
6. **Confira referência, transação e valor** e relacione-os ao pedido.
7. **Registre o evento de forma durável** com uma identidade de deduplicação que combine nome do evento, ID da cobrança e identificador da transação. Não deduplique só pelo ID da cobrança: registro, pagamento e devolução da mesma cobrança são eventos distintos.
8. **Responda exatamente `200`** depois da aceitação durável. Não use `202` ou `204`. Não responda `200` se o evento se perdeu antes de persistir.
9. **Processe os efeitos uma única vez,** fora da resposta HTTP quando o trabalho for demorado.

Assinatura inválida, corpo alterado ou timestamp fora da janela: rejeite sem aplicar efeitos no pedido. Evento autenticado já registrado: responda `200` de novo sem reaplicar o efeito.

## Regras de decisão

- **Reenvios e ordem:** a entrega pode se repetir e chegar fora de ordem. Um evento antigo não pode fazer o pedido regredir. Quando estado, transação ou valor divergirem, consulte o recurso.
- **Ausência de webhook não prova ausência de pagamento.** Reconcilie pela consulta do recurso.
- **Valores:** em devolução, use o valor efetivamente devolvido. Em boleto, o valor pago pode diferir do original.
- **Rotação de chave:** eventos de cobranças antigas continuam assinados com a chave associada a elas. Não presuma que a chave nova valida tudo e preserve o material de validação durante a transição.
- **Falha de assinatura não autoriza desligar a validação.** Confira corpo bruto, chave da cobrança, headers e relógio.
- **Logs:** não registre a API Key nem o corpo completo com dados pessoais em logs comuns. Guarde referências de investigação.

## Testes obrigatórios

- Notificação autêntica aceita e aplicada uma vez.
- Corpo alterado e assinatura inválida rejeitados.
- Timestamp fora da janela rejeitado.
- Reenvio do mesmo evento respondido com `200` sem duplicar efeito.
- Eventos fora de ordem sem regressão do pedido.

Em Sandbox, gere eventos reais criando a cobrança com `notification_url` e simulando o pagamento (skills `pagou-pix` e `pagou-boleto`). O MCP da Pagou não envia nem lê webhooks.
