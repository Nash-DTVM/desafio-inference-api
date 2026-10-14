# Desafio técnico: API de inferência assíncrona

Olá! Obrigado por topar o desafio.

## Contexto

Vários sistemas internos precisam extrair informação de documentos (contratos, comprovantes,
fotos de documentos) com a ajuda de um LLM. Chamar o LLM direto de cada sistema não funciona bem:
a chamada é **lenta** (segundos, às vezes minutos), **falha com frequência** (sobrecarga, limite de
uso, instabilidade) e **custa dinheiro** a cada tentativa.

Sua tarefa é construir o serviço que fica no meio: ele recebe o pedido com os arquivos, responde
na hora com um identificador e processa a inferência em segundo plano, de forma confiável. Quem
pediu consulta o resultado depois.

O LLM já existe neste repositório, simulado: é o serviço `llm-mock` do `docker-compose.yml`, com
contrato em [`MOCK_API.md`](MOCK_API.md). Ele se comporta como um provedor real. **Leia o contrato
com atenção e observe como ele se comporta quando você roda.**

## O que construir

### 1. `POST /tasks`: criar uma task

Requisição `multipart/form-data`:

| Campo | Obrigatório | Descrição |
|---|:---:|---|
| `model` | sim | modelo do LLM (`mock-flash` ou `mock-pro`) |
| `prompt` | sim | instrução para o LLM |
| `response_schema` | não | JSON Schema (enviado como texto) que a resposta deve seguir |
| `files` | sim | de 1 a 5 arquivos |

Limites dos arquivos: tipos **PDF, PNG ou JPEG**; no máximo **10 MB por arquivo** e **20 MB
somados** por task.

Respostas:

```
201 Created
{ "id": "…", "status": "pending" }

400 Bad Request
{ "error": { "code": "…", "message": "…" } }
```

Os arquivos devem ser guardados no **armazenamento S3** do compose (bucket `tasks-files`). O banco
guarda só a referência, nunca os bytes.

### 2. `GET /tasks/:id`: consultar uma task

```
200 OK
{
  "id": "…",
  "status": "pending" | "running" | "succeeded" | "failed",
  "result": …,                                   // só quando succeeded
  "error": { "code": "…", "message": "…" },      // só quando failed
  "attempts": 2,                                 // chamadas feitas ao LLM até agora
  "created_at": "2026-10-14T12:00:00.000Z",
  "updated_at": "2026-10-14T12:00:07.412Z"
}

404 Not Found
```

- Com `response_schema`, o `result` é o **objeto JSON** devolvido pelo LLM. Sem schema, é o
  **texto** devolvido.
- Os nomes dos campos e os quatro valores de `status` são **fixos**. Os valores de `error.code`
  ficam a seu critério.

### 3. O processamento

Um **worker**, em processo separado da API, consome as tasks pendentes de uma **fila persistente**,
busca os arquivos no armazenamento, chama o LLM (os arquivos vão em base64, conforme o
`MOCK_API.md`) e grava o resultado.

O LLM falha. Seu serviço deve:

- tentar de novo quando faz sentido, com **espera crescente entre as tentativas** e um **limite de
  tentativas**;
- **não** tentar de novo quando não faz sentido;
- deixar a task em `failed`, com um `error` que explique o motivo, quando não houver mais o que
  fazer.

Decidir o que é "quando faz sentido" faz parte do desafio.

### 4. Testes

Escreva testes que cubram o comportamento do serviço, **inclusive os casos de falha**, e não só o
caminho feliz. O `MOCK_API.md` descreve gatilhos que tornam as falhas do mock reproduzíveis. A
pasta [`samples/`](samples/) tem arquivos de exemplo para usar à vontade.

### 5. Rodar

**`docker compose up` precisa subir tudo**, inclusive a sua API e o seu worker, sem nenhum passo
manual além do que estiver documentado no seu README. O `docker-compose.yml` tem um bloco
comentado mostrando onde plugar a sua aplicação.

## Stack

- **Obrigatório:** TypeScript, Node.js 24 ou mais recente, e o **Postgres** do compose como banco.
- **Livre:** framework HTTP, ORM ou query builder, biblioteca de testes, e **a tecnologia da
  fila**. Você pode usar o próprio Postgres ou adicionar outra peça ao compose. **Explique a
  escolha no seu README.**
- Para o armazenamento, use qualquer cliente compatível com S3 (por exemplo
  `@aws-sdk/client-s3`, com `forcePathStyle: true`). As credenciais estão no `.env.example`.

## Extras opcionais

Nada daqui é obrigatório. **Os extras só somam pontos: não fazê-los não tira ponto nenhum.** Se
sobrar tempo e vontade, escolha os que achar mais interessantes:

- **Worker que morre:** se o processo do worker morrer no meio de uma task, ela não pode ficar
  presa para sempre.
- **Vários workers:** rodar mais de uma instância do worker ao mesmo tempo sem que duas processem
  a mesma task.
- **Observabilidade:** logs estruturados, métricas (tasks por status, tentativas, latência, custo)
  e/ou traces.
- **Aviso de conclusão:** avisar quem criou a task quando ela terminar (webhook, evento ou outro
  mecanismo), em vez de obrigar a consultar.
- **Validação da saída:** garantir que o resultado de fato respeita o `response_schema` antes de
  marcar a task como `succeeded`.
- **Idempotência no `POST /tasks`:** reenviar o mesmo pedido (por exemplo, depois de um timeout do
  lado de quem chama) não deve criar duas tasks.

## Fora do escopo

Não implemente **autenticação**, frontend nem deploy.

## Regras do jogo

- **Tempo:** esperamos algo em torno de **4 horas** de trabalho. Você tem **5 dias corridos** a
  partir do recebimento para entregar. Não esperamos tudo perfeito: **priorize e documente** o
  que ficou de fora e por quê.
- **IA é permitida.** Use as ferramentas que usaria no dia a dia. Só pedimos que você conte, no
  seu README, **como** usou. Na conversa seguinte, vamos falar sobre o código, então ele precisa
  fazer sentido para você.
- **Dúvidas:** responda o e-mail em que recebeu o desafio. Perguntar é bem-vindo.

## Entrega

1. Clique em **"Use this template"** e crie um repositório **privado** na sua conta do GitHub.
2. Faça commits ao longo do trabalho. **O histórico também conta**, não precisa esmagar tudo num
   commit só.
3. Adicione como colaboradores os usuários [`andrevendramini`](https://github.com/andrevendramini) e
   [`dukilee`](https://github.com/dukilee).
4. Responda o e-mail com o link do repositório até o fim do prazo.

O seu README deve trazer:

- **como rodar** (e como rodar os testes);
- **as decisões principais**, incluindo a escolha da fila, e as alternativas que você descartou;
- **o que ficou de fora** e o que você faria com mais tempo ou para levar isto a produção;
- **como você usou IA**.

Boa sorte, e divirta-se!
