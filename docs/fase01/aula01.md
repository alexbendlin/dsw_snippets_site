# Aula 01 — Sanitização: Tipagem, Validação, Prepared Statements e GET

> **Arquivo de trabalho:** `server_inicial.ts` (abra este arquivo e edite junto)
> **Guia de referência:** `GUIA_SANITIZACAO_PASSO_A_PASSO.md` (Passos 1 a 4)

---

## 🎯 Objetivo da Aula

Transformar o servidor "cru" em código seguro aplicando 4 pilares:

| Pilar | O que resolve |
|-------|---------------|
| **Tipagem explícita** | Elimina `as any` — TypeScript protege você |
| **Validação na entrada** | Rejeita dados ruins **antes** de irem pro banco |
| **Prepared Statements** | Previne SQL Injection + performance |
| **Erros controlados** | Não vaza stack trace para o cliente |

---

## Passo 1: Topo do Arquivo — Tipos, Constantes e Helpers

**📍 Onde:** Linhas 1–6, **logo após** `const PORT = 3000;`

```typescript
// [1] Modelo da entidade — substitui "as any" espalhado
interface Tarefa {
    id: number;
    titulo: string;
    status: string;
    prioridade: string;
}

// [2] Regras de negócio centralizadas (Single Source of Truth)
const PRIORIDADES = ["low", "medium", "high"] as const;
const STATUS_VALIDOS = ["pending", "completed"] as const;

// [3] Helpers de validação — escreva UMA vez, use EM TODAS as rotas
const tituloValido = (t: unknown): t is string =>
    typeof t === "string" && t.trim().length >= 3;

const normalizarPrioridade = (p: unknown) =>
    (PRIORIDADES as readonly string[]).includes(p as string) ? p : "medium";

const normalizarStatus = (s: unknown) =>
    (STATUS_VALIDOS as readonly string[]).includes(s as string) ? s : "pending";

// [4] Helper de ID padronizado para TODAS as rotas (DELETE, PUT, PATCH)
const parsearId = (idParam: string): number | null => {
    const id = parseInt(idParam);
    return isNaN(id) ? null : id;
};
```

### Por que colocar **aqui**?

| Antes (cru) | Depois (sanitizado) |
|-------------|---------------------|
| `as any` em 5+ lugares | Interface `Tarefa` em 1 lugar |
| `"high"` hardcoded nas rotas | `PRIORIDADES` centralizada |
| `parseInt` + `isNaN` copiado 4x | `parsearId()` reutilizável |
| Validação inline repetida | Helpers testáveis isoladamente |

> 💡 **Regra:** *Nunca escreva validação inline. Crie helper no topo → use nas rotas.*

---

## Passo 2: Prepared Statements Singleton — Fora das Rotas

**📍 Onde:** Linha 10–11, **logo após** `const db = new Database("tarefas.db");`

```typescript
// Prepared Statements criados UMA vez (performance + segurança)
const stmtContarUsuarios = db.prepare("SELECT COUNT(*) as count FROM usuarios");
const stmtInserirUsuario = db.prepare("INSERT INTO usuarios (email, senha) VALUES (?, ?)");
const stmtListarTodas = db.prepare("SELECT * FROM tarefas");
const stmtBuscarPorTitulo = db.prepare("SELECT * FROM tarefas WHERE titulo LIKE ?");
const stmtBuscarPorId = db.prepare("SELECT * FROM tarefas WHERE id = ?");
const stmtInserirTarefa = db.prepare("INSERT INTO tarefas (titulo, status, prioridade) VALUES (?, 'pending', ?)");
const stmtDeletarTarefa = db.prepare("DELETE FROM tarefas WHERE id = ?");
```

### ⚠️ Ação obrigatória depois disto:

**Apague TODOS** os `db.prepare("...")` que estiverem **DENTRO** das rotas:
- GET `/api/tasks`
- POST `/api/tasks`
- DELETE `/api/tasks/:id`
- PUT `/api/tasks/:id`
- PATCH `/api/tasks/:id`

### Por que fora das rotas?

```typescript
// ❌ ANTES (dentro da rota) — compila SQL a cada request
app.get("/api/tasks", (req, res) => {
    const tarefas = db.prepare("SELECT * FROM tarefas").all();
});

// ✅ DEPOIS (fora da rota) — statement já compilado, reutilizado
app.get("/api/tasks", (req, res) => {
    const tarefas = stmtListarTodas.all();
});
```

> ⚡ **Performance:** SQLite compila uma vez. Em 10.000 requests, economiza 10.000 compilações.

---

## Passo 3: Inicialização do Banco — Use os Statements Preparados

**📍 Onde:** Bloco `usuariosExistentes` (após criação das tabelas, aprox. linhas 27–33)

### Substitua ESTE bloco:

```typescript
// ANTES (linhas ~28-33)
const usuariosExistentes = db.prepare("SELECT COUNT(*) AS count FROM usuarios").get() as any;
if (usuariosExistentes.count === 0) {
    db.exec(`
        INSERT INTO usuarios (email, senha) VALUES ('admin@senail.com', 'senha_super_segura_123')
    `);
}
```

### Por ESTE bloco:

```typescript
// DEPOIS — tipado + prepared statements
const usuariosExistentes = stmtContarUsuarios.get() as { count: number };
if (usuariosExistentes.count === 0) {
    stmtInserirUsuario.run("admin@senai.com", "senha_super_secreta_123");
}
```

### O que mudou:

| Item | Antes | Depois |
|------|-------|--------|
| Tipagem | `as any` | `as { count: number }` |
| Insert | `db.exec()` + template string | `stmtInserirUsuario.run()` + placeholders |
| Segurança | Vulnerável a injection | 100% parametrizado |

---

## Passo 4: Sanitize GET /api/tasks

**📍 Onde:** Função **inteira** da rota `app.get("/api/tasks", ...)` (aprox. linhas 46–62)

### Substitua TUDO por:

```typescript
app.get("/api/tasks", (req, res) => {
    // Coerção segura: query pode ser string, array ou undefined
    const search = typeof req.query.search === "string" ? req.query.search : "";

    try {
        if (search) {
            // Placeholder ? = proteção contra SQL Injection
            const tarefas = stmtBuscarPorTitulo.all(`%${search}%`);
            res.json(tarefas);
        } else {
            const tarefas = stmtListarTodas.all();
            res.json(tarefas);
        }
    } catch {
        // NUNCA vaze erro interno para o cliente
        res.status(500).json({ error: "Erro interno ao processar a listagem." });
    }
});
```

### 3 conceitos aplicados aqui:

| Conceito | Código | Por que |
|----------|--------|---------|
| **Coerção segura** | `typeof req.query.search === "string"` | Express pode passar `string[]` ou `undefined` — evita *array injection* |
| **Placeholder `?`** | `stmtBuscarPorTitulo.all(`%${search}%`)` | O `%` vai **dentro** do parâmetro, não na query SQL |
| **Catch genérico** | `catch { ... }` sem `erro.message` | Cliente recebe "Erro interno", servidor loga o detalhe real |

---

## ✅ Checklist da Aula 01

- [ ] Interface `Tarefa` criada no topo (após `PORT`)
- [ ] Constantes `PRIORIDADES` e `STATUS_VALIDOS` com `as const`
- [ ] Helpers `tituloValido`, `normalizarPrioridade`, `normalizarStatus`, `parsearId`
- [ ] 7 Prepared Statements singleton criados após `const db`
- [ ] **Removidos** todos `db.prepare()` de dentro das rotas
- [ ] Inicialização de usuários usa statements preparados + tipagem
- [ ] GET `/api/tasks` usa coerção segura + statements singleton + catch genérico

---

## 🧪 Testes Rápidos

```bash
# 1. Rode o servidor
npx tsx server_inicial.ts

# 2. Teste busca normal
curl "http://localhost:3000/api/tasks"

# 3. Teste busca com termo
curl "http://localhost:3000/api/tasks?search=estudar"

# 4. Teste injeção SQL (deve ignorar/retornar vazio, não quebrar)
curl "http://localhost:3000/api/tasks?search='; DROP TABLE tarefas; --"

# 5. Teste array injection (?search=a&search=b)
curl "http://localhost:3000/api/tasks?search[]=a&search[]=b"
# Deve tratar como string vazia, não quebrar
```

---

## 📚 Próxima Aula

**Aula 02** — Rotas de escrita: **POST, DELETE, PUT, PATCH**  
Mesmos pilares: helpers reutilizáveis, statements preparados, validação na entrada, erros controlados.