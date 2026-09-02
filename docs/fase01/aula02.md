# Aula 02 — Sanitização: POST, DELETE, PUT, PATCH

> **Arquivo de trabalho:** `server_inicial.ts` (continue editando o mesmo arquivo)
> **Guia de referência:** `GUIA_SANITIZACAO_PASSO_A_PASSO.md` (Passos 5 a 9)

---

## 🎯 Objetivo da Aula

Aplicar os mesmos pilares nas rotas que **escrevem/modificam** dados:

| Pilar | Onde aparece |
|-------|--------------|
| Helpers reutilizáveis | POST, PUT, PATCH |
| `parsearId()` padronizado | DELETE, PUT, PATCH |
| Transação (`db.transaction`) | PATCH |
| Query dinâmica **segura** | PATCH |
| Erro 400 vs 500 bem distinguido | Todas |

---

## Passo 5: Sanitize POST /api/tasks — Criar Tarefa

**📍 Onde:** Função **inteira** da rota `app.post("/api/tasks", ...)` (aprox. linhas 64–86)

### Substitua TUDO por:

```typescript
app.post("/api/tasks", (req, res) => {
    const { titulo, prioridade } = req.body;
    const prioridadeValida = normalizarPrioridade(prioridade);

    // Validação via helper (type guard)
    if (!tituloValido(titulo)) {
        return res.status(400).json({
            error: "O título da tarefa é obrigatório e deve conter pelo menos 3 caracteres válidos."
        });
    }

    try {
        const resultado = stmtInserirTarefa.run(titulo.trim(), prioridadeValida);
        const novaTarefa = stmtBuscarPorId.get(resultado.lastInsertRowid) as Tarefa;
        return res.status(201).json(novaTarefa);
    } catch {
        return res.status(500).json({ error: "Erro ao processar persistência" });
    }
});
```

### O que cada helper faz:

| Helper | Entrada | Saída | Regra |
|--------|---------|-------|-------|
| `tituloValido()` | `unknown` | `boolean` (type guard) | String + trim ≥ 3 chars |
| `normalizarPrioridade()` | `unknown` | `"low" \| "medium" \| "high"` | Allowlist + default `"medium"` |

> 💡 `normalizarPrioridade("urgent")` → retorna `"medium"` (default seguro). Cliente não recebe 500, recebe comportamento previsível.

---

## Passo 6: Sanitize DELETE /api/tasks/:id — Validar ID

**📍 Onde:** Função **inteira** da rota `app.delete("/api/tasks/:id", ...)` (aprox. linhas 89–111)

### Substitua TUDO por:

```typescript
app.delete("/api/tasks/:id", (req, res) => {
    // Validação de ID padronizada (igual PUT/PATCH)
    const idParaDeletar = parsearId(req.params.id);
    if (idParaDeletar === null) {
        return res.status(400).json({ error: "ID inválido." });
    }

    try {
        const resultado = stmtDeletarTarefa.run(idParaDeletar);

        if (resultado.changes === 0) {
            res.status(404).json({ error: "Tarefa não localizada para exclusão." });
            return;
        }
        res.json({ message: "Tarefa excluída do banco SQLite com sucesso!" });
    } catch {
        res.status(500).json({ error: "Erro interno ao processar a exclusão." });
    }
});
```

### Por que `parsearId()`?

```typescript
// ❌ ANTES — cada rota fazia diferente
const id = parseInt(req.params.id);
if (isNaN(id)) { ... }

// ✅ DEPOIS — uma função, zero duplicação
const idParaDeletar = parsearId(req.params.id);
if (idParaDeletar === null) { ... }
```

---

## Passo 7: Sanitize PUT /api/tasks/:id — Atualização Completa

**📍 Onde:** Função **inteira** da rota `app.put("/api/tasks/:id", ...)` (aprox. linhas 114–152)

### Substitua TUDO por:

```typescript
app.put("/api/tasks/:id", (req, res) => {
    const idParaAtualizar = parsearId(req.params.id);
    if (idParaAtualizar === null) {
        return res.status(400).json({ error: "ID inválido." });
    }

    const { titulo, prioridade, status } = req.body;

    // Validação via helpers
    if (!tituloValido(titulo)) {
        return res.status(400).json({
            error: "O título da tarefa é obrigatório e deve conter pelo menos 3 caracteres válidos."
        });
    }

    const prioridadeValida = normalizarPrioridade(prioridade);
    const statusValido = normalizarStatus(status);

    try {
        // Prepared statement inline (UPDATE completo não tem statement fixo no topo)
        const sql = "UPDATE tarefas SET titulo = ?, status = ?, prioridade = ? WHERE id = ?";
        const resultado = db.prepare(sql).run(titulo.trim(), statusValido, prioridadeValida, idParaAtualizar);

        if (resultado.changes === 0) {
            return res.status(404).json({ message: "Tarefa não encontrada para atualização!" });
        }

        const tarefaAtualizada = stmtBuscarPorId.get(idParaAtualizar) as Tarefa;
        return res.status(200).json(tarefaAtualizada);

    } catch {
        return res.status(500).json({ error: "Erro ao processar a atualização no banco de dados." });
    }
});
```

### Observações importantes:

| Ponto | Explicação |
|-------|------------|
| `db.prepare(sql).run(...)` | UPDATE com todos os campos **não** tem statement fixo no topo. Criamos inline — **mas ainda usamos placeholders `?`** |
| `titulo.trim()` | Sanitização na entrada |
| `prioridadeValida` / `statusValido` | Normalização via allowlist |
| `catch` genérico | Não vaza detalhe do banco |

---

## Passo 8: Sanitize PATCH /api/tasks/:id — Atualização Parcial (Avançado)

**📍 Onde:** Função **inteira** da rota `app.patch("/api/tasks/:id", ...)` (aprox. linhas 155–230) — **é a mais complexa**

### Substitua TUDO por:

```typescript
app.patch("/api/tasks/:id", (req, res) => {
    const idParaAtualizar = parsearId(req.params.id);
    if (idParaAtualizar === null) {
        return res.status(400).json({ error: "ID inválido." });
    }

    if (!req.body || Object.keys(req.body).length === 0) {
        return res.status(400).json({ error: "Nenhum campo fornecido para atualização." });
    }

    const { titulo, prioridade, status } = req.body;

    try {
        const fluxoAtualizacao = db.transaction(() => {
            // Busca com statement singleton
            const tarefaExistente = stmtBuscarPorId.get(idParaAtualizar) as Tarefa | undefined;
            if (!tarefaExistente) return null;

            const camposParaAtualizar: string[] = [];
            const valoresParaAtualizar: unknown[] = [];

            // Título (se enviado)
            if (titulo !== undefined) {
                if (!tituloValido(titulo)) {
                    throw new Error("O título da tarefa deve conter pelo menos 3 caracteres válidos.");
                }
                camposParaAtualizar.push("titulo = ?");
                valoresParaAtualizar.push(titulo.trim());
            }

            // Prioridade (se enviada)
            if (prioridade !== undefined) {
                if (!PRIORIDADES.includes(prioridade as typeof PRIORIDADES[number])) {
                    throw new Error("Prioridade inválida. Use 'low', 'medium' ou 'high'.");
                }
                camposParaAtualizar.push("prioridade = ?");
                valoresParaAtualizar.push(prioridade);
            }

            // Status (se enviado)
            if (status !== undefined) {
                if (!STATUS_VALIDOS.includes(status as typeof STATUS_VALIDOS[number])) {
                    throw new Error("Status inválido. Use 'pending' ou 'completed'.");
                }
                camposParaAtualizar.push("status = ?");
                valoresParaAtualizar.push(status);
            }

            if (camposParaAtualizar.length === 0) return tarefaExistente;

            // Query dinâmica SEGURA: placeholders ? + valores array
            const sql = `UPDATE tarefas SET ${camposParaAtualizar.join(", ")} WHERE id = ?`;
            valoresParaAtualizar.push(idParaAtualizar);

            db.prepare(sql).run(...valoresParaAtualizar);
            return stmtBuscarPorId.get(idParaAtualizar) as Tarefa;
        });

        const resultado = fluxoAtualizacao();

        if (!resultado) {
            return res.status(404).json({ message: "Tarefa não encontrada para atualização parcial!" });
        }

        return res.status(200).json(resultado);

    } catch (erro) {
        // Distingue erro de validação (400) de erro interno (500)
        if (erro instanceof Error && (erro.message.includes("inválid") || erro.message.includes("caracteres"))) {
            return res.status(400).json({ error: erro.message });
        }
        return res.status(500).json({ error: "Erro ao processar a atualização parcial no banco." });
    }
});
```

### Por que transação?

| Sem transação | Com transação |
|---------------|---------------|
| Se falhar no meio, banco fica inconsistente | Tudo ou nada — atomicidade |
| Race condition possível | Isolamento garantido |

### Por que query dinâmica **assim** é segura?

```typescript
// ❌ PERIGOSO — concatenação de valor do usuário
const sql = `UPDATE tarefas SET ${campo} = '${valor}' WHERE id = ${id}`;

// ✅ SEGURO — placeholders + array de valores
const camposParaAtualizar = ["titulo = ?", "status = ?"];
const valoresParaAtualizar = ["Novo título", "completed"];
const sql = `UPDATE tarefas SET ${camposParaAtualizar.join(", ")} WHERE id = ?`;
valoresParaAtualizar.push(id);
db.prepare(sql).run(...valoresParaAtualizar);
```

- **Nomes das colunas** vêm de `camposParaAtualizar` (controlado por você, não pelo usuário)
- **Valores** vão no array `valoresParaAtualizar` → parâmetros `?`
- **Zero** concatenação de valor do usuário na string SQL

---

## Passo 9: Ajuste Final — PORT via Environment

**📍 Onde:** Linha 5, `const PORT = 3000;`

### Substitua por:

```typescript
const PORT = Number(process.env.PORT) || 3000;
```

> 🐳 **Por que?** Em produção (Docker, Kubernetes, Vercel, Railway, Render) a porta vem do ambiente. Local continua 3000.

---

## ✅ Checklist da Fase 01 Completa

| Item | Aula | Guia |
|------|------|------|
| [ ] Interface `Tarefa` adicionada no topo | 01 | Passo 1 |
| [ ] Constantes `PRIORIDADES`, `STATUS_VALIDOS` com `as const` | 01 | Passo 1 |
| [ ] Helpers `tituloValido`, `normalizarPrioridade`, `normalizarStatus`, `parsearId` | 01 | Passo 1 |
| [ ] 7 Prepared Statements singleton criados após `const db` | 01 | Passo 2 |
| [ ] Todos `db.prepare()` removidos de dentro das rotas | 01 | Passo 2 |
| [ ] Inicialização usuários usa statements preparados | 01 | Passo 3 |
| [ ] GET /api/tasks: coerção + statements singleton + catch genérico | 01 | Passo 4 |
| [ ] POST /api/tasks: `tituloValido()` + `normalizarPrioridade()` | 02 | Passo 5 |
| [ ] DELETE /api/tasks/:id: `parsearId()` validando ID | 02 | Passo 6 |
| [ ] PUT /api/tasks/:id: helpers + catch genérico | 02 | Passo 7 |
| [ ] PATCH /api/tasks/:id: transação + validação condicional + query dinâmica segura | 02 | Passo 8 |
| [ ] Todos `catch` retornam mensagem genérica (sem `erro.message`) | 01/02 | Passos 4–8 |
| [ ] Tipagem `as Tarefa` / `as { count: number }` em vez de `as any` | 01 | Passos 1,3 |
| [ ] `PORT` lê de `process.env.PORT` | 02 | Passo 9 |

---

## 🧪 Testes da Fase 01

```bash
# 1. Rode o servidor
npx tsx server_inicial.ts

# 2. POST - título inválido (número)
curl -X POST http://localhost:3000/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"titulo": 123}'
# → 400 "O título da tarefa é obrigatório..."

# 3. POST - prioridade inválida (normaliza para medium)
curl -X POST http://localhost:3000/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"titulo": "Estudar TypeScript", "prioridade": "urgent"}'
# → 201 com prioridade "medium"

# 4. DELETE - ID inválido
curl -X DELETE http://localhost:3000/api/tasks/abc
# → 400 "ID inválido."

# 5. PUT - atualização completa
curl -X PUT http://localhost:3000/api/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"titulo": "Novo título", "prioridade": "high", "status": "completed"}'
# → 200 com tarefa atualizada

# 6. PATCH - atualização parcial (só status)
curl -X PATCH http://localhost:3000/api/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"status": "completed"}'
# → 200 com só status alterado

# 7. PATCH - validação de status inválido
curl -X PATCH http://localhost:3000/api/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"status": "cancelled"}'
# → 400 "Status inválido. Use 'pending' ou 'completed'."

# 8. Erro interno não vaza (force erro no banco)
# → response deve ser "Erro interno..." sem stack trace
```

---

## 🏆 Regra de Ouro para Próximas Rotas

> **Nunca escreva validação inline.**  
> Crie helper no topo → use nas rotas.  
>   
> **Nunca crie `db.prepare()` na rota.**  
> Crie singleton no topo → reutilize.  
>   
> **Nunca retorne `erro.message` pro cliente.**  
> Logue no servidor, retorne genérico.  
>   
> **Nunca concatene valor do usuário no SQL.**  
> Use `?` + array de parâmetros — sempre.

---

Isso transforma sanitização em **arquitetura**, não remendo.  
Na **Fase 02** vamos ver: autenticação (bcrypt + JWT), middleware de autenticação, isolamento por usuário.