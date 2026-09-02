# Aula 09 — Banco em Nuvem (Turso / libSQL)

> **Contexto:** SQLite local apaga na nuvem → migramos para Turso (libSQL remoto).

---

## 1. Por que trocar?

- `better-sqlite3` escreve arquivo local (`tarefas.db`) → **apaga a cada deploy** no Render.
- Turso = SQLite remoto, persistente, acessado via HTTP (libSQL).

---

## 2. Instalação

```bash
npm install @libsql/client
```

---

## 3. Conexão unificada (dev local + nuvem)

```ts
import { createClient } from "@libsql/client";

const TURSO_URL = process.env.TURSO_URL || `file:${path.join(process.cwd(), "tarefas.db")}`;
const TURSO_TOKEN = process.env.TURSO_TOKEN;

const db = createClient(
    TURSO_TOKEN ? { url: TURSO_URL, authToken: TURSO_TOKEN } : { url: TURSO_URL }
);
```

- **Dev local**: `TURSO_URL` não definido → usa `file:./tarefas.db` (SQLite local).
- **Produção**: `TURSO_URL=libsql://seu-banco.turso.io` + `TURSO_TOKEN` → conecta no Turso.

---

## 4. libSQL é **assíncrono**

```ts
// ANTES (better-sqlite3) — síncrono
const tarefas = db.prepare("SELECT * FROM tarefas").all();

// DEPOIS (libSQL) — assíncrono
const result = await db.execute("SELECT * FROM tarefas");
const tarefas = result.rows;
```

---

## 5. Prepared statements (também assíncronos)

```ts
const stmt = await db.prepare("SELECT * FROM tarefas WHERE id = ?");
const result = await stmt.execute([id]);
const tarefa = result.rows[0];
```

---

## 6. DDL e transações

```ts
await db.execute(`
    CREATE TABLE IF NOT EXISTS tarefas (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        titulo TEXT NOT NULL,
        status TEXT DEFAULT 'pending',
        prioridade TEXT DEFAULT 'medium',
        usuario_id INTEGER NOT NULL
    );
`);

// Transação
await db.transaction(async (tx) => {
    await tx.execute("INSERT INTO tarefas ...", [titulo, 'pending', prioridade, usuarioId]);
    await tx.execute("UPDATE usuarios SET ...", [...]);
});
```

---

## 6. Variáveis de ambiente (Render)

```env
TURSO_URL=libsql://seu-banco.turso.io
TURSO_TOKEN=seu-token-aqui
```

---

## 7. requests.http — teste

```http
GET http://localhost:3000/api/db-check
GET http://localhost:3000/api/health
```

---

> 💡 **Dica:** no Turso, **não dá para rodar vários statements numa única chamada** (diferente do `db.exec` do better-sqlite3). Use `await db.execute(...)` um por um, ou transação.