# Aula 05 — Intro React + Vite + TypeScript

> **Contexto:** back-end pronto (Fase 01/02). Agora criamos o front em `web/`.

---

## Estrutura do projeto React

```
web/
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── Login.tsx
│   ├── Tarefas.tsx
│   ├── api.ts
│   └── types.ts
└── public/
    └── favicon.svg
```

---

## package.json do front

```json
{
  "name": "web",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "@vitejs/plugin-react": "^4.2.0",
    "typescript": "^5.3.0",
    "vite": "^5.0.0"
  }
}
```

---

## main.tsx — ponto de entrada

```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

---

## App.tsx — decisão login vs tarefas

```tsx
import { useState } from "react";
import Login from "./Login";
import Tarefas from "./Tarefas";
import { getToken } from "./api";

function App() {
    const [logado, setLogado] = useState<boolean>(() => !!getToken());

    return (
        <>
            {logado ? (
                <Tarefas aoSair={() => setLogado(false)} />
            ) : (
                <Login aoLogar={() => setLogado(true)} />
            )}
        </>
    );
}

export default App;
```

---

## api.ts — camada de comunicação com o back

```ts
const API_BASE = import.meta.env.VITE_API_BASE || "http://localhost:3000";

export const login = async (email: string, senha: string) => {
    const res = await fetch(`${API_BASE}/api/auth/login`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ email, senha }),
    });
    if (!res.ok) throw new Error(await res.text());
    const { token } = await res.json();
    localStorage.setItem("token", token);
};

export const getToken = () => localStorage.getItem("token");
export const clearToken = () => localStorage.removeItem("token");

const authHeaders = () => ({
    "Content-Type": "application/json",
    "Authorization": `Bearer ${getToken()}`,
});

export const listarTarefas = async (search = "") => {
    const res = await fetch(`${API_BASE}/api/tasks?search=${search}`, {
        headers: authHeaders(),
    });
    if (!res.ok) throw new Error("Falha ao listar");
    return res.json();
};
// ... criarTarefa, atualizarTarefa, deletarTarefa seguem o mesmo padrão
```

---

> 💡 **Dica:** rode `cd web && npm install && npm run dev` → abre em `http://localhost:5173`.