# requests_inicial.http

> Testes da API (REST Client) — arquivo que cresce junto com o curso.

```
# 📡 requests.http — Testes da API (REST Client)
#
# Como usar (VS Code + extensão REST Client):
#   1. Copie este arquivo para a pasta do SEU app como `requests.http`.
#   2. Deixe o servidor rodando (`npm run dev` em outro terminal).
#   3. Clique em "Send Request" acima de cada bloco para testar.
#
# Este arquivo cresce com o curso. As seções marcadas "Aula 00 / Fase 02 / Fase 05"
# são habilitadas conforme você implementa cada parte no seu server.ts.
# Comente (ou apague) os blocos de rotas que ainda não existem no seu servidor.

@baseUrl = http://localhost:3000

### ============================================================
### AULA 00 — Back-end inicial (server_inicial.ts)
### ============================================================

### Health check — confirma que o servidor está no ar
GET {{baseUrl}}/api/health

### Versão da API
GET {{baseUrl}}/api/version

### Listar todas as tarefas
GET {{baseUrl}}/api/tasks

### Listar tarefas filtrando por título (busca)
GET {{baseUrl}}/api/tasks?search=comprar

### Criar uma nova tarefa
POST {{baseUrl}}/api/tasks
Content-Type: application/json

{
  "title": "Estudar SQL Injection",
  "prioridade": "high"
}

### Atualizar uma tarefa inteira (PUT) — troque o id na URL
PUT {{baseUrl}}/api/tasks/1
Content-Type: application/json

{
  "title": "Estudar SQL Injection (revisado)",
  "prioridade": "medium",
  "status": "completed"
}

### Atualização parcial (PATCH) — troque o id na URL
PATCH {{baseUrl}}/api/tasks/1
Content-Type: application/json

{
  "status": "pending"
}

### Excluir uma tarefa — troque o id na URL
DELETE {{baseUrl}}/api/tasks/1


### ============================================================
### FASE 02 — Autenticação (adicione depois da Aula 04)
### Descomente estes blocos quando criar as rotas /api/auth/*
### ============================================================

### Registrar um novo usuário
# POST {{baseUrl}}/api/auth/register
# Content-Type: application/json
#
# {
#   "email": "voce@teste.com",
#   "senha": "minhaSenha1"
# }

### Login — copie o token retornado para usar abaixo
# POST {{baseUrl}}/api/auth/login
# Content-Type: application/json
#
# {
#   "email": "voce@teste.com",
#   "senha": "minhaSenha1"
# }

### Tarefas protegidas — cole o token onde está <SEU_TOKEN>
### (substitua <SEU_TOKEN> pelo valor recebido no login)
# GET {{baseUrl}}/api/tasks
# Authorization: Bearer <SEU_TOKEN>


### ============================================================
### FASE 05 — Banco em nuvem (adicione depois da Aula 09)
### Descomente quando criar a rota /api/db-check
### ============================================================

### Verifica a conexão com o banco
# GET {{baseUrl}}/api/db-check
```

---

## Como usar no VS Code

1. Instale a extensão **REST Client** (Huachao Mao)
2. Abra este arquivo (`requests.http`)
3. Clique em **"Send Request"** acima de qualquer bloco
4. Veja a resposta no painel lateral

---

## Estrutura do arquivo

| Seção | Quando desbloquear |
|-------|-------------------|
| **Aula 00** | Disponível desde o início |
| **Fase 02** | Após implementar `/api/auth/register` e `/api/auth/login` (Aula 04) |
| **Fase 05** | Após implementar `/api/db-check` (Aula 09) |

> 💡 Mantenha este arquivo no seu projeto e vá descomentando conforme avança nas aulas.