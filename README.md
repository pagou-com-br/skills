# Skills da Pagou para agentes de IA

Plugin oficial da Pagou para agentes de programação. Ele ensina o agente a integrar a [API Pagou](https://docs.pagou.com.br) com segurança e conecta os servidores MCP da Pagou, que dão acesso à documentação e, com a sua chave, às ferramentas da API.

O pacote segue o padrão [Agent Skills](https://agentskills.io) e traz manifestos para Claude Code, Codex e Cursor. Outros agentes podem instalar apenas as skills.

> **Configuração guiada:** cole no seu agente a frase abaixo e ele conduz a instalação.
>
> `Busque e siga as instruções para me configurar para a Pagou em https://docs.pagou.com.br/agentes/prompt.md`
>
> Guia completo por agente: https://docs.pagou.com.br/agentes

## O que vem no pacote

### Skills

As skills são carregadas automaticamente quando a conversa trata do assunto. Cada uma aponta para as páginas `.md` da documentação oficial e orienta o agente a lê-las antes de implementar; os contratos da API não são copiados para cá.

| Skill | Quando é usada |
| --- | --- |
| `pagou` | Base de qualquer integração: ambientes, autenticação, leitura da documentação, regras de segurança, idempotência, erros e uso das ferramentas MCP |
| `pagou-pix` | Pix imediato ou com vencimento, consulta, cancelamento, devolução e confirmação |
| `pagou-boleto` | Boleto, registro assíncrono, Pix vinculado e confirmação |
| `pagou-cartao` | Cartão de crédito: tokenização no navegador, cobrança pelo backend, estados, captura, estorno e chargebacks |
| `pagou-webhooks` | Recebimento de notificações com validação HMAC e deduplicação |
| `pagou-split` | Divisão de valores entre contas Pagou autorizadas |

### Servidores MCP

| Servidor | Endereço | Variável da chave | Ambiente |
| --- | --- | --- | --- |
| `pagou-sandbox` | `https://mcp.pagou.com.br/sandbox/mcp` | `PAGOU_SANDBOX_API_KEY` | Sandbox (recomendado durante a integração) |
| `pagou` | `https://mcp.pagou.com.br/mcp` | `PAGOU_API_KEY` | Produção |

Os dois servidores usam variáveis diferentes para que um ambiente nunca receba a chave do outro.

| Ferramentas | Disponibilidade |
| --- | --- |
| `search_docs`, `get_doc_page`, `list_doc_pages` | Sempre, inclusive sem chave |
| `get_balance`, `create_pix`, `create_pix_due`, `get_pix`, `list_pix`, `create_boleto`, `get_boleto`, `list_boletos`, `get_card_charge`, `list_card_charges` | Com chave, no ambiente do servidor |
| `simulate_pix_payment`, `simulate_charge_payment` | Com chave, somente em `pagou-sandbox` |

O MCP não oferece estorno, cancelamento, captura, nova tentativa, chargeback, criação de cobrança de cartão, saque nem alteração de pagadores ou cartões. Essas operações ficam no backend da sua aplicação ou no painel.

## Instalação

### Claude Code

No terminal:

```sh
claude plugin marketplace add pagou-com-br/skills
claude plugin install pagou@pagou
```

Dentro de uma sessão aberta, o equivalente é:

```text
/plugin marketplace add pagou-com-br/skills
/plugin install pagou@pagou
/reload-plugins
```

O `/reload-plugins` aplica o plugin na sessão atual sem reiniciar. Depois, confira em `/mcp` se `pagou-sandbox` e `pagou` aparecem como servidores do plugin.

O plugin lê as chaves das variáveis `PAGOU_SANDBOX_API_KEY` e `PAGOU_API_KEY` (veja [Variáveis de ambiente](#variáveis-de-ambiente)). Defina a variável **antes** de abrir o Claude Code; se ela mudar, abra uma nova sessão.

Para atualizar, rode `claude plugin marketplace update pagou` e `claude plugin update pagou@pagou`, e abra uma nova sessão.

### Codex

```sh
codex plugin marketplace add pagou-com-br/skills
codex plugin add pagou@pagou
```

Também é possível instalar pelo navegador de plugins, com `/plugins` dentro do Codex. Abra uma nova sessão depois da instalação.

O plugin instala as skills e os dois servidores MCP em **modo só documentação**: o formato de plugin usado pelo Codex não permite ler cabeçalhos de variáveis de ambiente. Para habilitar as ferramentas da API no Sandbox, acrescente ao `~/.codex/config.toml`:

```toml
[mcp_servers.pagou-sandbox]
url = "https://mcp.pagou.com.br/sandbox/mcp"
env_http_headers = { "X-API-KEY" = "PAGOU_SANDBOX_API_KEY" }
```

A entrada do `config.toml` tem precedência sobre o servidor de mesmo nome do plugin. Se a variável estiver vazia ou ausente, o Codex não envia o cabeçalho e o servidor continua só com a documentação. Configure o servidor `pagou` (produção) da mesma forma, com `PAGOU_API_KEY`, somente quando houver necessidade real.

### Cursor

Em **Customize**, importe o repositório `https://github.com/pagou-com-br/skills` pela opção de repositório do GitHub e instale o plugin `pagou`. Para testar sem importar, copie o repositório para `~/.cursor/plugins/local/pagou` e recarregue o Cursor.

Assim como no Codex, os servidores do plugin funcionam em **modo só documentação**. Para habilitar as ferramentas da API no Sandbox, acrescente ao `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "pagou-sandbox": {
      "url": "https://mcp.pagou.com.br/sandbox/mcp",
      "headers": {
        "X-API-KEY": "${env:PAGOU_SANDBOX_API_KEY}"
      }
    }
  }
}
```

A variável precisa estar visível para o processo do Cursor. Se as ferramentas da API não aparecerem, abra o Cursor a partir de um terminal em que a variável esteja exportada. Se o servidor `pagou-sandbox` do plugin também aparecer na lista de MCP, desative-o para não duplicar as ferramentas de documentação.

### Outros agentes

Instale apenas as skills com a CLI [`skills`](https://skills.sh):

```sh
npx -y skills add pagou-com-br/skills --skill '*'
```

Depois, configure o MCP no seu agente com o endereço do servidor e o cabeçalho `X-API-KEY` lido de uma variável de ambiente. Se o cliente só aceitar token do tipo bearer, o MCP também aceita `Authorization: Bearer <chave>`. Nunca grave a chave por extenso em um arquivo versionado.

## Variáveis de ambiente

| Variável | Uso |
| --- | --- |
| `PAGOU_SANDBOX_API_KEY` | Chave da conta de **Sandbox**. É a única necessária durante a integração. |
| `PAGOU_API_KEY` | Chave da conta de **produção**. Configure somente quando precisar operar em produção pelo agente. |

A chave fica no painel da conta de cada ambiente, em **Configurações → API**. Em produção, o caminho é https://app.pagou.com.br/configuracoes/api.

Prefira carregar a chave do seu gerenciador de segredos em vez de escrevê-la em arquivos. Exemplo no macOS, com a chave guardada no Keychain:

```sh
export PAGOU_SANDBOX_API_KEY="$(security find-generic-password -s pagou-sandbox -w)"
```

## Modo só documentação

Sem a variável, o servidor correspondente conecta normalmente e oferece apenas `search_docs`, `get_doc_page` e `list_doc_pages`. Assim você consulta a documentação no agente sem nenhuma credencial, e a mesma instalação passa a oferecer as ferramentas da API assim que a variável for definida.

## Segurança

- **Nunca cole a chave na conversa.** As skills orientam o agente a nunca pedir a chave no chat; ela deve vir de variável de ambiente ou do gerenciador de segredos.
- **Comece e permaneça no Sandbox.** Com `PAGOU_API_KEY` definida, o servidor `pagou` cria Pix e boletos **reais**. Configure a chave de produção só quando houver necessidade e remova-a ao terminar.
- **A chave de produção é um segredo crítico.** Pela API, ela dá acesso a operações financeiras da conta e também valida a assinatura dos webhooks.
- **Sem repetição automática de criações.** As ferramentas de criação nunca são repetidas pelo MCP. Em resultado incerto, o agente consulta ou lista antes de qualquer nova tentativa.
- **Caminho da chave.** A chave segue no cabeçalho até `mcp.pagou.com.br` e é repassada à API Pagou apenas como `X-API-KEY`. O MCP não registra a chave em log.
- **Repositório público.** Este repositório não contém segredos; os arquivos de configuração referenciam apenas nomes de variáveis.
- **Integração no backend.** O MCP serve para consultar e testar. O código da sua aplicação deve chamar a API pelo backend, com a chave no mecanismo de segredos do projeto.

## Estrutura do repositório

| Caminho | Função |
| --- | --- |
| `skills/<nome>/SKILL.md` | Skills no padrão Agent Skills |
| `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` | Plugin e marketplace do Claude Code |
| `.mcp.json` | Servidores MCP do Claude Code, com a chave lida das variáveis de ambiente |
| `plugin.json`, `mcp.json` | Manifesto portátil [Agent Plugins](https://agent-plugins.org), lido por Codex e Cursor |
| `.codex-plugin/plugin.json`, `.agents/plugins/marketplace.json` | Apresentação e marketplace do Codex |
| `.cursor-plugin/plugin.json`, `.cursor-plugin/marketplace.json` | Plugin e marketplace do Cursor |
| `assets/logo.svg` | Ícone usado nos catálogos |

Para validar o plugin do Claude Code depois de uma alteração:

```sh
claude plugin validate .
```

## Links

- Documentação da API: https://docs.pagou.com.br
- Configuração de agentes: https://docs.pagou.com.br/agentes
- Instruções para o agente: https://docs.pagou.com.br/agentes/prompt.md
- Índice para agentes: https://docs.pagou.com.br/llms.txt
