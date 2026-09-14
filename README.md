# Bot de Clima no Telegram com n8n

Chatbot no Telegram que recebe o nome de uma cidade do Brasil, consulta a API gratuita da [OpenWeather](https://openweathermap.org/api) e responde com a temperatura atual.

Exemplo de resposta:

> 🌤️ A temperatura em Belo Horizonte é de 25°C.

## Tecnologias

- [n8n](https://n8n.io/) — orquestração do workflow
- [Telegram Bot API](https://core.telegram.org/bots/api) — recebimento e envio de mensagens
- [OpenWeather Current Weather Data](https://openweathermap.org/current) — temperatura atual
- Docker / Docker Compose — execução local do n8n (opcional)

## Pré-requisitos

- Conta no Telegram
- Conta gratuita na [OpenWeather](https://home.openweathermap.org/users/sign_up)
- n8n em execução (local via Docker ou n8n Cloud)
- URL pública para o webhook do Telegram (em ambiente local, use ngrok ou equivalente)
- Acesso às variáveis de ambiente do n8n:
  - `OPENWEATHER_API_KEY`
  - `TELEGRAM_BOT_TOKEN`

## 1. Criar o bot no Telegram com o BotFather

1. Abra o Telegram e busque por [@BotFather](https://t.me/BotFather).
2. Envie `/newbot`.
3. Escolha um nome de exibição, por exemplo: `Bot de Clima`.
4. Escolha um username único que termine com `bot`, por exemplo: `meu_clima_bot`.
5. O BotFather devolve um token no formato `123456789:AAH...`.
6. Guarde esse valor como `TELEGRAM_BOT_TOKEN`. **Não compartilhe e não commite no GitHub.**

Opcional: envie `/setdescription` no BotFather e use um texto como:

> Envie uma cidade no formato Cidade,UF,BR para receber a temperatura atual. Ex.: São Paulo,SP,BR

## 2. Obter a API Key da OpenWeather

1. Crie uma conta em [OpenWeather](https://home.openweathermap.org/users/sign_up).
2. Acesse [API keys](https://home.openweathermap.org/api_keys).
3. Copie a chave padrão ou gere uma nova.
4. Guarde esse valor como `OPENWEATHER_API_KEY`. **Não compartilhe e não commite no GitHub.**

A chave gratuita pode levar alguns minutos para ser ativada. Se a consulta falhar logo após a criação, aguarde e tente de novo.

## 3. Configurar as variáveis de ambiente no n8n

O workflow **não** contém token do Telegram nem chave da OpenWeather. Ele lê:

| Variável | Uso |
| --- | --- |
| `OPENWEATHER_API_KEY` | Query parameter `appid` da OpenWeather |
| `TELEGRAM_BOT_TOKEN` | Credencial do nó Telegram no n8n |

### Docker Compose já existente neste repositório

O `docker-compose.yml` deste projeto já encaminha as duas variáveis para os serviços `n8n-editor` e `n8n-worker`.

1. Copie o arquivo de exemplo:

```bash
cp .env.example .env
```

2. Preencha o `.env` com os seus valores reais:

```env
OPENWEATHER_API_KEY=cole_aqui_a_chave_da_openweather
TELEGRAM_BOT_TOKEN=cole_aqui_o_token_do_botfather
```

3. Recrie os containers para o n8n enxergar as variáveis:

```bash
docker compose up -d
```

O acesso a `$env` nos nós já está liberado com `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`.

### Compose simples (opcional)

Se quiser subir só o n8n, use `docker-compose.bot-clima.yml`:

```bash
cp .env.example .env
docker compose -f docker-compose.bot-clima.yml up -d
```

Para o Telegram Trigger funcionar fora do n8n Cloud, defina também `WEBHOOK_URL` com uma URL pública HTTPS (ngrok, Cloudflare Tunnel, reverse proxy etc.).

### n8n Cloud ou instalação sem Docker

Defina `OPENWEATHER_API_KEY` e `TELEGRAM_BOT_TOKEN` nas variáveis de ambiente do processo/serviço do n8n e reinicie a instância.

## 4. Importar o workflow

1. Abra o n8n (neste projeto, em geral `http://localhost:5678`).
2. No menu, vá em **Workflows**.
3. Clique em **Add workflow** (ou no menu `...`) e escolha **Import from File**.
4. Selecione o arquivo `workflow-chatbot-telegram.json`.
5. Confirme a importação. O fluxo entra inativo (`active: false`) até você ativá-lo.

## 5. Configurar as credenciais do Telegram no n8n

Os nós `Telegram Trigger` e `Enviar resposta Telegram` usam a credencial chamada **Telegram Bot**.

1. Abra qualquer um desses nós.
2. Em **Credential to connect with**, crie uma credencial **Telegram API**.
3. No campo **Access Token**, use a variável de ambiente (recomendado):

```text
{{ $env.TELEGRAM_BOT_TOKEN }}
```

   Alternativa: cole o token do BotFather apenas na interface do n8n, nunca no arquivo JSON.

4. Salve a credencial e associe-a aos dois nós Telegram.
5. Não exporte o workflow depois de preencher um token real. Se precisar versionar de novo, remova o valor secreto antes.

## 6. Executar e testar

1. Confirme que `OPENWEATHER_API_KEY` está disponível no n8n.
2. Confirme que a credencial Telegram está selecionada nos dois nós.
3. Em ambiente local, confirme que `WEBHOOK_URL` aponta para uma URL pública HTTPS.
4. Salve o workflow.
5. Ative o interruptor **Active** no canto superior direito.
6. Abra o bot no Telegram e envie `/start`.
7. Envie uma cidade no formato `Cidade,UF,BR`.

O nó **Definir queue** trata a entrada assim:

- remove espaços extras;
- normaliza a cidade em minúsculas e aplica capitalização (`são paulo` → `São Paulo`);
- coloca UF e país em maiúsculas (`sp,br` → `SP,BR`);
- gera o valor esperado pela OpenWeather, por exemplo `São Paulo,SP,BR`.

Entradas aceitas: `São Paulo,SP,BR`, `são paulo, sp, br`, `Belo Horizonte,MG,BR`.

## Exemplos de mensagens de teste

Envie no Telegram:

```text
São Paulo,SP,BR
Belo Horizonte,MG,BR
Recife,PE,BR
```

Resposta esperada (a temperatura muda conforme o momento da consulta):

```text
🌤️ A temperatura em Belo Horizonte é de 25°C.
```

O nome da cidade vem do campo `name` da OpenWeather. A temperatura vem de `main.temp`, em Celsius (`units=metric`), arredondada para inteiro.

### Cidade inválida

Envie algo que a API não reconheça, por exemplo:

```text
CidadeInexistente,XX,BR
```

Resposta esperada:

```text
❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).
```

`/start` ou mensagem vazia recebem um texto de ajuda com os exemplos acima.

## Segurança

- **Não suba segredos reais no GitHub.** Isso inclui token do Telegram, chave da OpenWeather, `.env`, dumps de `n8n-data` e exports de credenciais.
- Use apenas `.env.example` no repositório, com valores vazios ou placeholders.
- O JSON do workflow usa `={{ $env.OPENWEATHER_API_KEY }}` e credenciais nomeadas, sem tokens embutidos.
- Mantenha `TELEGRAM_BOT_TOKEN` e `OPENWEATHER_API_KEY` só no ambiente local ou no secret manager da sua hospedagem.
- Se um token vazar, revogue no BotFather (`/revoke`) e gere outra API key na OpenWeather.

## Estrutura do workflow

```text
Telegram Trigger
        ↓
  Definir queue          ← salva chatId e queue
        ↓
É consulta de cidade?
   ├─ sim → Consultar OpenWeather
   │              ↓
   │        Resposta válida?
   │         ├─ sim → Montar mensagem de clima
   │         └─ não → Mensagem cidade inválida
   └─ não → Mensagem de ajuda
                    ↓
        Enviar resposta Telegram
```

A consulta usa:

```text
GET https://api.openweathermap.org/data/2.5/weather
  q      = {{ $json.queue }}
  units  = metric
  lang   = pt_br
  appid  = {{ $env.OPENWEATHER_API_KEY }}
```
