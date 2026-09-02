# Aula 06 — Tela de Login (React + Tailwind tema escuro)

> **Contexto:** componente `Login.tsx` com formulário controlado, modo login/registro.

---

## Login.tsx completo

```tsx
import { useState } from "react";
import { login, registrar } from "./api";

interface LoginProps {
    aoLogar: () => void;
}

export default function Login({ aoLogar }: LoginProps) {
    const [modo, setModo] = useState<"login" | "registro">("login");
    const [email, setEmail] = useState("");
    const [senha, setSenha] = useState("");
    const [erro, setErro] = useState("");
    const [carregando, setCarregando] = useState(false);

    const enviar = async (e: React.FormEvent) => {
        e.preventDefault();
        setErro("");
        setCarregando(true);
        try {
            if (modo === "login") {
                await login(email, senha);
            } else {
                await registrar(email, senha);
                await login(email, senha);
            }
            aoLogar();
        } catch (err) {
            setErro(err instanceof Error ? err.message : "Falha ao autenticar.");
        } finally {
            setCarregando(false);
        }
    };

    return (
        <div className="max-w-[360px] mx-auto mt-20 bg-slate-800 p-7 rounded-xl shadow-lg">
            <h1 className="text-xl font-bold text-white">Gerenciador de Tarefas</h1>
            <h2 className="text-lg font-semibold text-slate-300 mt-1">
                {modo === "login" ? "Entrar" : "Criar conta"}
            </h2>
            <form onSubmit={enviar} className="flex flex-col gap-3 mt-4">
                <input
                    type="email"
                    placeholder="E-mail"
                    value={email}
                    onChange={(e) => setEmail(e.target.value)}
                    required
                    className="px-3 py-2 border border-slate-600 rounded-md text-base text-white bg-slate-700 placeholder:text-slate-400 focus:outline-none focus:ring-2 focus:ring-blue-500"
                />
                <input
                    type="password"
                    placeholder="Senha (mín. 6 caracteres)"
                    value={senha}
                    onChange={(e) => setSenha(e.target.value)}
                    required
                    className="px-3 py-2 border border-slate-600 rounded-md text-base text-white bg-slate-700 placeholder:text-slate-400 focus:outline-none focus:ring-2 focus:ring-blue-500"
                />
                {erro && <p className="text-red-400 font-semibold">{erro}</p>}
                <button
                    type="submit"
                    disabled={carregando}
                    className="cursor-pointer border-none rounded-md px-4 py-2 bg-blue-600 text-white font-semibold hover:bg-blue-700 disabled:opacity-60"
                >
                    {carregando ? "Aguarde..." : modo === "login" ? "Entrar" : "Registrar"}
                </button>
            </form>
            <button
                className="mt-4 bg-none p-0 text-blue-600 font-semibold cursor-pointer"
                onClick={() => setModo(modo === "login" ? "registro" : "login")}
            >
                {modo === "login" ? "Não tem conta? Registre-se" : "Já tem conta? Entrar"}
            </button>
        </div>
    );
}
```

---

## Pontos-chave do tema escuro (slate)

```tsx
// Cartão
<div className="bg-slate-800 p-7 rounded-xl shadow-lg">

// Inputs com texto branco e fundo escuro
<input className="text-white bg-slate-700 placeholder:text-slate-400 border-slate-600">

// Botão primário
<button className="bg-blue-600 hover:bg-blue-700 text-white">

// Texto secundário
<p className="text-slate-300">
```

---

> 💡 **Dica:** o `index.css` do projeto define `color-scheme: dark` e `body { background: #0f172a; }`.