---
name: pagou-split
description: Implementa split de pagamento na API Pagou, dividindo o valor de uma venda entre a conta que cobra e outras contas Pagou autorizadas. Use quando o usuário pedir split, divisão de valores, repasse, comissão, marketplace, recebedores, alocação fixa ou percentual em Pix, boleto ou cartão da Pagou. Cobre pré-requisitos de aprovação e autorização de recebedores, unidades de valor por tipo e modalidade, margem para tarifas, estados do split diferentes dos estados do pagamento, parcelamento, estornos e recuperação. Use junto com a skill pagou e a skill do meio de pagamento.
---

# Split de pagamento na Pagou

Aplique primeiro as regras da skill `pagou` e da skill do meio de pagamento (`pagou-pix`, `pagou-boleto` ou `pagou-cartao`). Esta skill traz só as decisões do split. Campos, respostas e exemplos completos estão nas páginas abaixo: **leia-as antes de implementar**.

## Leia antes de implementar

- [Split de pagamento](https://docs.pagou.com.br/integracao-com-a-api/split-de-pagamento.md)
- [Exemplos de split por modalidade](https://docs.pagou.com.br/integracao-com-a-api/split-de-pagamento/exemplos-de-split.md)
- A página de criação da modalidade escolhida: [Pix imediato](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-pix/criando-um-qrcode-imediato.md), [Pix com vencimento](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-pix/criando-um-qrcode-com-vencimento.md), [boleto](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-boleto/criando-um-boleto.md) ou [cartão](https://docs.pagou.com.br/integracao-com-a-api/meio-de-pagamento-cartao-de-credito/criando-uma-cobranca.md)

## Pré-requisitos

Confirme todos antes de implementar. Se algum faltar, informe o usuário em vez de contornar.

- A conta que cria a cobrança tem **aprovação prévia da Pagou** para split, recurso habilitado no plano e tarifa de split configurada. A aprovação é solicitada ao suporte Pagou.
- Cada recebedor tem conta Pagou ativa e foi **autorizado pelo lojista no painel** (Configurações → Split).
- Você tem o **UUID do cliente Pagou recebedor** de cada participante. Ele é diferente do pagador de cartão, do ID de autorização e do número da conta corrente.
- Os requisitos da modalidade continuam valendo (no cartão, credenciamento aprovado).

## Regras de decisão

- **O split vai na criação do pagamento.** A documentação não prevê incluir split depois; decida a divisão antes de criar.
- **Quantidade:** de 1 a 10 instruções, uma por recebedor. A própria conta que cobra não entra como recebedora: o restante fica com ela automaticamente.
- **Tipos e unidades:**
  - Fixo: valor em **reais**, com até duas casas decimais, **inclusive no cartão**, em que a cobrança usa centavos.
  - Percentual: `25` significa 25%, não `0.25`. Incide sobre o valor bruto, antes das tarifas.
- **Reserve margem para tarifas:** a soma das alocações, da tarifa da modalidade e da tarifa de split não pode passar do valor bruto. Dividir 100% entre recebedores tende a ser rejeitado.
- **Controle interno em centavos:** calcule em centavos e serialize cada campo na unidade exigida. Confira o valor resolvido devolvido pela Pagou.
- **Estado do pagamento e estado do split são diferentes.** Um pagamento confirmado pode ter a distribuição pendente. Acompanhe o estado do split e de cada alocação pela consulta do pagamento.
- **Não recrie o pagamento** para corrigir distribuição pendente ou que exija ação. Continue acompanhando o mesmo ID e envolva o suporte quando a documentação indicar.
- **Cartão parcelado:** a alocação vale sobre o total e é distribuída conforme as parcelas são liquidadas.
- **Estornos, devoluções e chargebacks** podem exigir recuperar valores já repassados. Não considere o split revertido só pelo sucesso HTTP; confira pagamento, estado do split e alocações. Cancelar boleto não devolve pagamento recebido.

## Repetição e recuperação

- Cartão: use `Idempotency-Key` e preserve os bytes do corpo, inclusive o array de split, nas repetições (skill `pagou-cartao`).
- Pix e boleto: sem idempotência. Após falha ou resultado incerto, verifique o pagamento antes de criar outro para a mesma venda.
- Erros de split na criação chegam antes da distribuição e podem vir só com mensagem, sem código. Leia a tabela de erros da página de split.

## Ferramentas MCP

O MCP não tem ferramentas próprias de split. As consultas `get_pix`, `get_boleto` e `get_card_charge` mostram o estado do split quando ele existe. Antes de testar split por uma ferramenta de criação do MCP, confira no esquema da ferramenta se ela aceita o campo; a integração definitiva envia o split pelo backend da aplicação.

## Teste em Sandbox

1. Confirme aprovação, habilitação e recebedores autorizados também na conta de Sandbox.
2. Crie o pagamento com split no ambiente de Sandbox e guarde o ID.
3. Confira as alocações resolvidas na resposta e na consulta.
4. Acompanhe pagamento e split separadamente até a conclusão.
5. A simulação de pagamento de cobrança pelo Pix vinculado é documentada para cobranças **sem** split. Não use esse caminho para validar liquidação de split.
