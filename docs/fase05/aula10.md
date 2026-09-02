# Aula 10 — Deploy no Render (back + front + banco)

> **Contexto:** app completo (API + React + Turso) publicado no Render como **um único Web Service**.

---

## 1. render.yaml (Infrastructure as Code)

```yaml
services:
  - type: web
    name: gerenciador-tarefas
    runtime: node
    plan: free
    buildCommand: npm install && npm run build
    startCommand: npm start
    envVars:
      - key: NODE_ENV
        value: production
      - key: JWT_SECRET
        generateValue: true
      - key: TURSO_URL
        sync: false
      - key: TURSO_TOKEN
        sync: false
      - key: FRONT_ORIGIN
        value: https://gerenciador-tarefas.onrender.com
```

---

## 2. package.json raiz — scripts de produção

```json
{
  "scripts": {
    "dev": "tsx watch server.ts",
    "start": "node server.js",
    "build": "tsc && cd web && npm install && npm run build"
  }
}
```

> `npm run build` compila o back (`tsc`) E o front (`cd web && npm run build`).

---

## 3. server.ts — ajustes para produção

```ts
// Porta vindas do ambiente (Render injeta PORT)
const PORT = Number(process.env.PORT) || 3000;

// CORS restrito à origem do front
app.use(cors({
    origin: process.env.FRONT_ORIGIN,
    credentials: true,
}));

// Trust proxy (Render usa proxy)
app.set("trust proxy", 1);

// Health check para o Render monitorar
app.get("/api/health", (req, res) => res.json({ status: "ok" }));
```

---

## 4. Passos no painel do Render

1. **New → Web Service** → conecte seu repo GitHub.
2. Render detecta `render.yaml` automaticamente (ou preencha manual):
   - **Build Command**: `npm install && npm run build`
   - **Start Command**: `npm start`
3. **Environment Variables** (Settings → Environment):
   - `JWT_SECRET` → gere valor (botão *Generate*)
   - `TURSO_URL` → `libsql://seu-banco.turso.io`
   - `TURSO_TOKEN` → seu token do Turso
   - `FRONT_ORIGIN` → `https://seu-app.onrender.com`
4. **Create Web Service** → aguarde build e deploy.

---

## 5. Testando depois do deploy

```http
### Health check
GET https://seu-app.onrender.com/api/health

### DB check (Turso conectado)
GET https://seu-app.onrender.com/api/db-check
```

- Abra a URL no navegador → front React carregado.
- Teste login, CRUD de tarefas, prioridades, modal.

---

## 6. Dockerfile (opcional, para controle total)

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

---

> 💡 **Dica:** o plano *free* do Render "dorme" após 15 min sem tráfego. Primeira requisição demora ~30s para acordar.