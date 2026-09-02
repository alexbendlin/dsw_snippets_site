# Aula 10b — Tailwind CSS v4 (Tema Escuro slate)

> **Contexto:** estilização utilitária direto no JSX, tema escuro coeso (paleta slate).

---

## 1. Instalação (Vite + Tailwind v4)

```bash
npm install -D tailwindcss @tailwindcss/vite
```

---

## 2. vite.config.ts

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
    plugins: [react(), tailwindcss()],
});
```

---

## 3. index.css — só o essencial

```css
@import "tailwindcss";

:root {
    font-family: system-ui, Arial, sans-serif;
    color-scheme: dark;
}
* { box-sizing: border-box; }
body {
    margin: 0;
    background: #0f172a;
    color: #e2e8f0;
}
```

---

## 4. Cartão de Login (tema slate)

```tsx
<div className="max-w-[360px] mx-auto mt-20 bg-slate-800 p-7 rounded-xl shadow-lg">
    <h1 className="text-xl font-bold text-white">Gerenciador de Tarefas</h1>
    <input
        className="px-3 py-2 border border-slate-600 rounded-md text-base text-white bg-slate-700 placeholder:text-slate-400 focus:outline-none focus:ring-2 focus:ring-blue-500"
    />
</div>
```

---

## 5. Tabela de cores slate (referência)

| Uso | Classe | Cor |
|-----|--------|-----|
| Fundo página | `bg-slate-950` / `bg-[#0f172a]` | `#020617` / `#0f172a` |
| Cartões | `bg-slate-800` | `#1e293b` |
| Inputs | `bg-slate-700` | `#334155` |
| Bordas | `border-slate-600` / `border-slate-700` | `#475569` / `#334155` |
| Texto principal | `text-white` | `#ffffff` |
| Texto secundário | `text-slate-300` / `text-slate-400` | `#cbd5e1` / `#94a3b8` |
| Placeholder | `placeholder:text-slate-400` | `#94a3b8` |
| Erros | `text-red-400` | `#f87171` |
| Botão primário | `bg-blue-600 hover:bg-blue-700` | `#2563eb` → `#1d4ed8` |
| Botão perigo | `bg-red-600 hover:bg-red-700` | `#dc2626` → `#b91c1c` |

---

## 6. Build

```bash
npm run build
# gera web/dist com CSS otimizado (só classes usadas)
```

---

> 💡 **Dica:** o tema escuro foi aplicado em **todas as fases do front (03, 04, 05)** para manter evolução coerente.