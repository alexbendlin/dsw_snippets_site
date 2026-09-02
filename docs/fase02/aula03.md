# Aula 03 — Registro e Login (bcrypt + JWT)

> **Contexto:** seu `server.ts` já tem o CRUD sanitizado. Agora adiciona autenticação.

---

## 1. bcrypt — hash de senha

```ts
import bcrypt from "bcryptjs";

const hash = bcrypt.hashSync(senha, 10);  // custa ~100ms, seguro
// guarda hash no banco, NUNCA a senha pura
```

---

## 2. Rota de REGISTRO

```ts
app.post("/api/auth/register", (req, res) => {
    const { email, senha } = req.body;

    if (typeof email !== "string" || typeof senha !== "string") {
        return res.status(400).json({ error: "E-mail e senha são obrigatórios." });
    }
    if (senha.trim().length < 6) {
        return res.status(400).json({ error: "A senha deve ter ao menos 6 caracteres." });
    }

    const hash = bcrypt.hashSync(senha, 10);

    try {
        const resultado = stmtInserirUsuario.run(email, hash);
        return res.status(201).json({ id: resultado.lastInsertRowid, email });
    } catch (erro) {
        if (erro instanceof Error && erro.message.includes("UNIQUE")) {
            return res.status(409).json({ error: "E-mail já cadastrado." });
        }
        return res.status(500).json({ error: "Erro ao registrar." });
    }
});
```

---

## 3. Rota de LOGIN

```ts
import jwt from "jsonwebtoken";

const JWT_SECRET = process.env.JWT_SECRET || "segredo_didatico_trocar_em_prod";

app.post("/api/auth/login", (req, res) => {
    const { email, senha } = req.body;

    const usuario = stmtBuscarUsuarioPorEmail.get(email) as Usuario | undefined;
    if (!usuario) {
        // fake hash para timing attack
        bcrypt.compareSync(senha, "$2a$10$fakehashfakehashfakehashfakehashfakeha");
        return res.status(401).json({ error: "Credenciais inválidas." });
    }

    if (!bcrypt.compareSync(senha, usuario.senha_hash)) {
        return res.status(401).json({ error: "Credenciais inválidas." });
    }

    const token = jwt.sign(
        { id: usuario.id, email: usuario.email },
        JWT_SECRET,
        { expiresIn: "2h" }
    );

    res.json({ token });
});
```

---

## 4. requests.http — teste

```http
### Registrar
POST http://localhost:3000/api/auth/register
Content-Type: application/json

{
  "email": "voce@teste.com",
  "senha": "minhaSenha1"
}

### Login
POST http://localhost:3000/api/auth/login
Content-Type: application/json

{
  "email": "voce@teste.com",
  "senha": "minhaSenha1"
}
# Copie o token retornado para usar nas próximas rotas
```

---

> 💡 **Segurança:** `JWT_SECRET` vem de `process.env.JWT_SECRET` (nunca hardcoded em produção).