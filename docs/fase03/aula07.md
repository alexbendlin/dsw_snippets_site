# Aula 07 — CRUD de Tarefas (React + Tailwind)

> **Contexto:** componente `Tarefas.tsx` com lista, criar, concluir, excluir, buscar, modal de confirmação.

---

## Tarefas.tsx — pontos principais

```tsx
interface TarefasProps {
    aoSair: () => void;
    usuario: { id: number; email: string } | null;
}

const PRIORIDADES = ["low", "medium", "high"] as const;
const PRIORIDADES_PT: Record<(typeof PRIORIDADES)[number], string> = {
    low: "Baixa", medium: "Média", high: "Alta",
};

export default function Tarefas({ aoSair, usuario }: TarefasProps) {
    const [tarefas, setTarefas] = useState<Tarefa[]>([]);
    const [titulo, setTitulo] = useState("");
    const [prioridade, setPrioridade] = useState("medium");
    const [search, setSearch] = useState("");
    const [paraExcluir, setParaExcluir] = useState<Tarefa | null>(null);
    // ... estados de erro, carregando

    // Carrega lista (com busca)
    const carregar = async () => {
        setTarefas(await listarTarefas(search));
    };

    // Criar
    const criar = async (e: React.FormEvent) => {
        e.preventDefault();
        await criarTarefa(titulo, prioridade);
        setTitulo(""); setPrioridade("medium");
        await carregar();
    };

    // Alternar status
    const alternar = async (t: Tarefa) => {
        await atualizarTarefa(t.id, { status: t.status === "completed" ? "pending" : "completed" });
        await carregar();
    };

    // Excluir com modal
    const confirmarExcluir = async () => {
        if (!paraExcluir) return;
        await deletarTarefa(paraExcluir.id);
        setParaExcluir(null);
        await carregar();
    };

    // Cabeçalho com e-mail do usuário
    return (
        <div className="max-w-[640px] mx-auto mt-6 px-4">
            <header className="flex justify-between items-center">
                <div>
                    <h1 className="text-3xl font-extrabold text-white tracking-tight">Minhas Tarefas</h1>
                    {usuario && (
                        <p className="text-sm text-slate-400 mt-1">Conectado como {usuario.email}</p>
                    )}
                </div>
                <button onClick={aoSair} className="bg-gray-500 text-white px-4 py-2 rounded hover:bg-gray-600">
                    Sair
                </button>
            </header>

            {/* Form criar + busca */}
            <form onSubmit={criar} className="flex gap-2 my-4">...</form>
            <div className="flex gap-2 my-4">...</div>

            {/* Lista com rolagem */}
            <ul className="list-none p-0 flex flex-col gap-2 max-h-[60vh] overflow-y-auto pr-1">
                {tarefas.map(t => (
                    <li key={t.id} className="bg-slate-800 border border-slate-700 rounded-lg p-3 flex items-center gap-2.5">
                        <span className={`flex-1 font-semibold ${t.status === "completed" ? "line-through text-slate-500" : "text-white"}`}>
                            {t.titulo}
                        </span>
                        <span className="text-slate-400 text-sm">[{PRIORIDADES_PT[t.prioridade]}]</span>
                        <button onClick={() => alternar(t)} className="bg-blue-600 text-white px-3 py-1.5 rounded hover:bg-blue-700">
                            {t.status === "completed" ? "Reabrir" : "Concluir"}
                        </button>
                        <button onClick={() => setParaExcluir(t)} className="bg-red-600 text-white px-3 py-1.5 rounded hover:bg-red-700">
                            Excluir
                        </button>
                    </li>
                ))}
            </ul>

            {/* Modal de confirmação */}
            {paraExcluir && (
                <div className="fixed inset-0 bg-black/60 flex items-center justify-center z-50" onClick={() => setParaExcluir(null)}>
                    <div className="bg-slate-800 border border-slate-700 rounded-xl p-6 max-w-[360px] w-full mx-4" onClick={e => e.stopPropagation()}>
                        <h3 className="text-lg font-bold text-white">Excluir tarefa</h3>
                        <p className="text-slate-300 text-sm mt-2">
                            Tem certeza que deseja excluir a tarefa <span className="font-semibold text-white">"{paraExcluir.titulo}"</span>?
                        </p>
                        <div className="flex justify-end gap-2 mt-5">
                            <button onClick={() => setParaExcluir(null)} className="bg-slate-600 text-white px-4 py-2 rounded hover:bg-slate-500">
                                Cancelar
                            </button>
                            <button onClick={confirmarExcluir} className="bg-red-600 text-white px-4 py-2 rounded hover:bg-red-700">
                                Excluir
                            </button>
                        </div>
                    </div>
                </div>
            )}
        </div>
    );
}
```

---

## Destaques

- **Prioridade PT-BR**: `PRIORIDADES_PT` traduz `low/medium/high` → `Baixa/Média/Alta`
- **Lista com rolagem**: `max-h-[60vh] overflow-y-auto`
- **Modal de exclusão**: estado `paraExcluir` + overlay `fixed inset-0 bg-black/60`
- **Cabeçalho dinâmico**: mostra e-mail do usuário logado