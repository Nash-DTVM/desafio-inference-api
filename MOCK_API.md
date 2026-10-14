# LLM simulado: contrato da API

O serviço `llm-mock` do `docker-compose.yml` faz o papel de um provedor de LLM. Ele não tem
modelo nenhum por trás: responde com texto ou JSON gerados a partir do que recebeu. Fora isso,
**se comporta como um provedor real**, ou seja, às vezes é lento e às vezes falha.

- **Da sua máquina:** `http://localhost:8090`
- **De dentro da rede do compose:** `http://llm-mock:8080`
- **Autenticação:** header `Authorization: Bearer mock-dev-key`

## `POST /v1/inference`

### Requisição

```http
POST /v1/inference
Authorization: Bearer mock-dev-key
Content-Type: application/json
```

```json
{
  "model": "mock-flash",
  "prompt": "Extraia o valor de face e o vencimento do contrato.",
  "files": [
    {
      "filename": "contrato.pdf",
      "mime_type": "application/pdf",
      "data_base64": "JVBERi0xLjQK..."
    }
  ],
  "response_schema": {
    "type": "object",
    "required": ["valor", "vencimento"],
    "properties": {
      "valor": { "type": "number" },
      "vencimento": { "type": "string" }
    }
  }
}
```

| Campo | Obrigatório | Regras |
|---|:---:|---|
| `model` | sim | `mock-flash` ou `mock-pro` |
| `prompt` | sim | string não vazia |
| `files` | não | até 10 itens; cada um com `filename`, `mime_type` e `data_base64` |
| `files[].mime_type` | sim | `application/pdf`, `image/png` ou `image/jpeg` |
| `files[].data_base64` | sim | conteúdo do arquivo em base64 padrão (com padding) |
| `response_schema` | não | JSON Schema, no subconjunto descrito abaixo |

Campos desconhecidos são recusados com `400`.

**Tamanho máximo do corpo da requisição: 16 MB (16.777.216 bytes).** O limite vale para o corpo
inteiro, já com o base64.

### Resposta de sucesso: `200`

```json
{
  "id": "gen_7f3a9c1e2b4d6f80a1c3e5d7",
  "model": "mock-flash",
  "output": "{\"valor\":4821.37,\"vencimento\":\"vencimento-3fa2c1\"}",
  "finish_reason": "stop",
  "usage": { "input_tokens": 37, "output_tokens": 14 },
  "cost_usd": 0.0000461
}
```

- **`output` é sempre uma string.** Com `response_schema`, é um JSON **serializado**: o parse é
  com você.
- Sem `response_schema`, o `output` é um texto que lista cada arquivo recebido com tamanho e
  **sha256**. Use para conferir que os bytes chegaram intactos.
- **`finish_reason`** é `stop` (resposta completa) ou `length` (a resposta foi **cortada** antes
  do fim).
- A mesma requisição produz o mesmo `output`. O `id` muda a cada chamada.
- `usage` e `cost_usd` dizem quanto aquela chamada custou. **Toda resposta `200` é cobrada**,
  inclusive as cortadas.

### Subconjunto de JSON Schema aceito em `response_schema`

`type` (`object`, `array`, `string`, `number`, `integer`, `boolean`), `properties`, `required`,
`items`, `enum`, `additionalProperties` (boolean), `description` e `title`. Qualquer outra
palavra-chave é recusada com `400 unsupported_schema`.

### Erros

Todo erro tem o formato:

```json
{ "error": { "type": "service_unavailable", "message": "provedor temporariamente indisponível" } }
```

| Status | `error.type` | Significado |
|:---:|---|---|
| 400 | `invalid_request` | corpo inválido (JSON malformado, campo faltando ou desconhecido, base64 inválido) |
| 400 | `invalid_model` | `model` fora da lista |
| 400 | `unreadable_file` | o conteúdo do arquivo não corresponde ao `mime_type` (arquivo corrompido ou de outro tipo) |
| 400 | `unsupported_schema` | `response_schema` fora do subconjunto aceito |
| 401 | `unauthorized` | header `Authorization` ausente ou errado |
| 413 | `payload_too_large` | corpo acima de 16 MB |
| 429 | `rate_limited` | limite de requisições excedido; veja o header `Retry-After` (em segundos) |
| 500 | `internal_error` | erro interno do provedor |
| 503 | `service_unavailable` | provedor temporariamente indisponível |

Além desses, a chamada pode **demorar sem responder**. O mock não impõe timeout: isso fica do
seu lado.

## `GET /healthz`

Sem autenticação. Responde `200 {"status":"ok"}` quando o serviço está no ar.

## Gatilhos para testes reproduzíveis

Sem gatilhos, o mock se comporta como um provedor real. Para testar um cenário específico de forma
determinística, inclua um gatilho **no texto do `prompt`**:

| Gatilho | Efeito |
|---|---|
| `[[fail:<tipo>]]` | a chamada falha **sempre** com `<tipo>` |
| `[[fail:<tipo>x<n>]]` | as **primeiras `<n>` chamadas idênticas** falham com `<tipo>`; da seguinte em diante, a chamada segue normalmente |
| `[[retry-after:<segundos>]]` | define o valor do `Retry-After` nas falhas `429` |
| `[[delay:<ms>]]` | latência fixa da chamada, em milissegundos |

`<tipo>` pode ser:

| `<tipo>` | Resultado |
|---|---|
| `400` | `400 invalid_request` |
| `429` | `429 rate_limited` com `Retry-After` |
| `500` | `500 internal_error` |
| `503` | `503 service_unavailable` |
| `timeout` | a conexão fica aberta sem resposta |
| `length` | `200` com `finish_reason: "length"` e `output` cortado |
| `schema` | `200` com `output` que **não** respeita o `response_schema` |

- "Chamadas idênticas" são as que têm o mesmo `model`, `prompt`, arquivos e `response_schema`.
- Vários `[[fail:...]]` se encadeiam na ordem em que aparecem:
  `[[fail:503x2]][[fail:429x1]]` produz `503`, `503`, `429` e depois sucesso.
- **Com qualquer gatilho no prompt, o mock não sorteia falhas para aquela chamada.** Só acontece
  o que o gatilho pede.
- Os contadores ficam em memória: reiniciar o container zera tudo.
- Gatilho malformado (por exemplo `[[fail:418]]`) é recusado com `400 invalid_request`.

## Exemplo com curl

```bash
curl -s localhost:8090/v1/inference \
  -H 'Authorization: Bearer mock-dev-key' \
  -H 'Content-Type: application/json' \
  -d "{\"model\":\"mock-flash\",\"prompt\":\"Resuma o documento.\",\"files\":[{\"filename\":\"contrato.pdf\",\"mime_type\":\"application/pdf\",\"data_base64\":\"$(base64 < samples/contrato.pdf | tr -d '\n')\"}]}"
```
