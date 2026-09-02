# Aula 08 — Back serve Front (CORS + static + SPA fallback)

> **Contexto:** o front buildado (`web/dist`) é servido pelo Express na mesma porta.

---

## 1. CORS — libera front e back conversarem

```ts
import cors from "cors";

app.use(cors({
    origin: process.env.FRONT_ORIGIN || "http://localhost:5173",
    credentials: true,
}));
```

---

## 2. Servir arquivos estáticos do front

```ts
import path from "path";
import { fileURLToPath } from "url";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const FRONT_DIST = path.join(__dirname, "..", "web", "dist");

app.use(express.static(FRONT_DIST));
```

---

## 3. SPA fallback — rotas desconhecidas → index.html

```ts
// Express 5: regex em vez de "*"
app.get(/(.*)/, (req, res) => {
    res.sendFile(path.join(FRONT_DIST, "index.html"));
});
```

---

## 4. Ordem dos middlewares (importante!)

```ts
app.use(cors(...));
app.use(express.json());

// Rotas de API ANTES do static/fallback
app.get("/api/health", ...);
app.post("/api/auth/register", ...);
// ... todas as rotas /api/*

// DEPOIS: static + fallback
app.use(express.static(FRONT_DIST));
app.get(/(.*)/, (req, res) => res.sendFile(path.join(FRONT_DIST, "index.html")));
```

---

## 5. Build e execução

```bash
# 1. Build do front
cd web
npm install
npm run build    # gera web/dist
cd ..

# 2. Rode o back (serve front + API)
npm run dev      # ou npm start em produção
```

---

## 6. Testando

- `http://localhost:3000` → abre o React (servido pelo Express)
- `http://localhost:3000/api/health` → API continua funcionando
- No DevTools (Rede): `index.html`, `.js`, `.css` vêm do Express

---

> 💡 **Dica:** em produção no Render, o `buildCommand` faz `npm install && npm run build` (instala e builda front + back juntos).