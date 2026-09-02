# package_inicial.json

> Dependências iniciais do projeto (better-sqlite3, express, tsx, TypeScript).

```json
{
  "name": "gerenciador-de-tarefas",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "tsx watch server.ts",
    "start": "tsx server.ts",
    "build": "tsc"
  },
  "dependencies": {
    "better-sqlite3": "^13.0.3",
    "express": "^5.2.1"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^9.6.0",
    "@types/express": "^5.0.6",
    "@types/node": "^26.2.0",
    "tsx": "^4.23.12",
    "typescript": "^7.0.2"
  },
  "allowScripts": {
    "better-sqlite3@13.0.3": true
  }
}
```

---

## Como usar

```bash
# 1. Copie este arquivo como package.json na raiz do seu projeto
# 2. Instale dependências
npm install

# 3. Rode em modo desenvolvimento
npm run dev
```

---

## O que cada script faz

| Script | Comando | Uso |
|--------|---------|-----|
| `dev` | `tsx watch server.ts` | Desenvolvimento com hot-reload |
| `start` | `tsx server.ts` | Produção local (sem watch) |
| `build` | `tsc` | Compila TypeScript → JavaScript |