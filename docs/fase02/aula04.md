# Aula 04 — Middleware authenticate + isolamento por usuario_id

> **Contexto:** agora protegemos as rotas de tarefas e isolamos por usuário.

---

## 1. Middleware de autenticação

```ts
interface AuthRequest extends Request {
    usuarioId?: number;
}

const authenticate = (req: AuthRequest, res: Response, next: NextFunction) => {
    const auth = req.headers.authorization;
    if (!auth?.startsWith("Bearer ")) {
        return res.status(401).json({ error: "Token ausente." });
    }
    const token = auth.slice(7);

    try {
        const payload = jwt.verify(token, JWT_SECRET) as { id: number; email: string };
        req.usuarioId = payload.id;
        next();
    } catch {
        return res.status(401).json({ error: "Token inválido ou expirado." });
    }
};
```

---

## 2. Aplicar nas rotas de tarefas

```ts
// Todas as rotas de tarefas passam pelo authenticate
app.get("/api/tasks", authenticate, (req: AuthRequest, res) => { ... });
app.post("/api/tasks", authenticate, (req: AuthRequest, res) => { ... });
app.put("/api/tasks/:id", authenticate, (req: AuthRequest, res) => { ... });
app.patch("/api/tasks/:id", authenticate, (req: AuthRequest, res) => { ... });
app.delete("/api/tasks/:id", authenticate, (req: AuthRequest, res) => { ... });
```

---

## 3. Isolamento por usuario_id

```ts
// No POST — insere com usuario_id do token
app.post("/api/tasks", authenticate, (req: AuthRequest, res) => {
    const { titulo, prioridade } = req.body;
    const prioridadeValida = normalizarPrioridade(prioridade);
    
    if (!tituloValido(titulo)) { return res.status(400).json({...}); }

    const resultado = stmtInserirTarefa.run(titulo.trim(), prioridadeValida, req.usuarioId!);
    const novaTarefa = stmtBuscarPorId.get(resultado.lastInsertRowid);
    return res.status(201).json(novaTarefa);
});

// No GET — lista só do usuário
app.get("/api/tasks", authenticate, (req: AuthRequest, res) => {
    const tarefas = stmtListarPorUsuario.all(req.usuarioId!);
    res.json(tarefas);
});

// No PUT/PATCH/DELETE — WHERE id = ? AND usuario_id = ?
app.put("/api/tasks/:id", authenticate, (req: AuthRequest, res) => {
    const id = parsearId(req.params.id);
    if (id === null) return res.status(400).json({ error: "ID inválido." });
    
    const resultado = stmtAtualizarTarefa.run(titulo.trim(), statusValido, prioridadeValida, id, req.usuarioId!);
    if (resultado.changes === 0) return res.status(404).json({ error: "Tarefa não encontrada." });
    ...
});
```

---

## 4. Prepared statements com usuario_id

```ts
const stmtInserirTarefa = db.prepare(
    "INSERT INTO tarefas (titulo, status, prioridade, usuario_id) VALUES (?, 'pending', ?, ?)"
);
const stmtListarPorUsuario = db.prepare("SELECT * FROM tarefas WHERE usuario_id = ?");
const stmtBuscarPorId = db.prepare("SELECT * FROM tarefas WHERE id = ? AND usuario_id = ?");
const stmtAtualizarTarefa = db.prepare(
    "UPDATE tarefas SET titulo = ?, status = ?, prioridade = ? WHERE id = ? AND usuario_id = ?"
);
const stmtDeletarTarefa = db.prepare("DELETE FROM tarefas WHERE id = ? AND usuario_id = ?");
```

---

## 5. requests.http — teste com token

```http
@token = {{login.response.body.token}}

### Listar tarefas (precisa do token)
GET http://localhost:3000/api/tasks
Authorization: Bearer {{token}}

### Criar tarefa
POST http://localhost:3000/api/tasks
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "titulo": "Minha tarefa protegida",
  "prioridade": "high"
}
```

---

> 💡 **Conceito-chave:** autenticação = saber *quem* é; autorização = deixar acessar *apenas o que é dele*.