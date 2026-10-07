# Intake de Demandas: protótipo de front

Protótipo navegável da experiência de intake de demandas de produto. O solicitante descreve o que precisa, uma IA faz perguntas de negócio uma de cada vez e monta ao vivo o brief estruturado da demanda, até enviá-lo para planejamento.

A IA tem dois modos, escolhidos no botão **Conexão** do topo:

- **Vertex AI** (padrão): Claude de verdade no Vertex AI, com as tools `update_brief` e `submit_for_planning`. As chamadas saem direto do navegador; serve só para validar a ideia.
- **Simulação**: roteiro fixo, sem chamadas externas. Útil para demonstrar sem token.

Não há backend. Cada modo é uma implementação da mesma camada de serviço, que a UI consome por eventos.

## Como abrir

É um único arquivo, sem build e sem dependências:

```bash
# opção 1: abrir direto no navegador
open index.html          # macOS
xdg-open index.html      # Linux

# opção 2: servir localmente
npx serve .
```

As fontes vêm do Google Fonts. Sem internet, a página usa as fontes do sistema.

## IA real (Vertex AI)

1. Gere um access token com a conta que tem acesso ao projeto:
   ```bash
   gcloud auth print-access-token
   ```
2. Abra o `index.html` (direto ou via `npx serve .`), clique em **Configurar** no topo, informe o ID do projeto, cole o token e salve.
3. Converse normalmente. O token vale cerca de 1 hora; quando expirar, a página avisa e reabre o diálogo com a mensagem preservada no campo.

O ID do projeto do Google Cloud não fica no código: informe-o em **Conexão**, junto com o token. O token fica só no `sessionStorage` da aba (some ao fechá-la). Modo, projeto, região e modelo ficam no `localStorage` do navegador. Região (`global`) e modelo (`claude-sonnet-4-6`) têm padrão e podem ser trocados em **Conexão → Região e modelo**.

> A versão publicada no claude.ai não consegue chamar o Vertex: aquela visualização bloqueia requisições externas. Para usar a IA real, abra o arquivo localmente.

### Como a integração funciona

A cada mensagem do usuário, `createVertexIntakeService` faz o loop de tools no próprio front:

1. `POST …/publishers/anthropic/models/{modelo}:streamRawPredict` com `anthropic_version: "vertex-2023-10-16"`, `stream: true`, `max_tokens: 4096`, as tools e o `system` em três blocos: instruções do intake, perfis dos sistemas (com `cache_control`) e o brief atual em JSON.
2. O texto chega em streaming (`content_block_delta`) e vira `text_delta` na UI. O início de cada `tool_use` aparece como atividade.
3. Se a resposta para em `tool_use`, o front executa as tools e devolve todos os `tool_result` numa única mensagem:
   - `update_brief` aplica as mudanças no brief (adiciona regras, critérios e perguntas sem duplicar, substitui sistemas, move perguntas respondidas para esclarecimentos, casando o texto exato ou muito parecido) e devolve o brief completo em JSON;
   - `submit_for_planning` fecha o brief, gera um protocolo `DEM-xxxx` e devolve `{ status: "submitted", protocol }`.
4. Repete até a resposta terminar (`end_turn`), com no máximo 6 chamadas por mensagem.

O histórico (`messages`) fica em memória. Se uma chamada falhar, ele volta ao estado anterior à mensagem, para que o reenvio não duplique nada.

Além das tools, o front deduz duas coisas do texto da IA:

- **Estado da sessão**: "Esclarecendo detalhes" quando o brief ganha objetivo; "Pronto para planejamento" quando a última fala pergunta se pode enviar para planejamento; "Enviado" quando `submit_for_planning` roda.
- **Respostas rápidas**: quando a IA termina a pergunta com uma lista de 2 a 6 opções, elas viram chips. Quando ela pede confirmação de envio, aparecem "Sim, pode enviar" e "Quero ajustar algo".

Diferença em relação ao exemplo de requisição original: `update_brief` ganhou o campo opcional `title` (título curto da demanda), usado no topo do card do brief.

### Depuração

- Cada requisição e resposta do modelo aparece no console do navegador (`console.debug`, nível "Verbose"), com `stop_reason` e `usage` (inclusive `cache_read_input_tokens`).
- `intakeDebug()` no console devolve o histórico enviado ao modelo e o brief atual no formato da tool.

## Roteiro de demonstração (modo Simulação)

1. Clique na sugestão inicial ("Quero que a tela de estoque mostre a data prevista de entrega do pedido") ou descreva a demanda com suas palavras.
2. A IA consulta o mapa de sistemas (Stock.Web → Stock.Api → Bridge), registra objetivo e sistemas no brief e abre as perguntas em aberto.
3. Responda às perguntas de negócio, pelos chips ou digitando livremente:
   - onde a data aparece (detalhe, listagem ou ambos);
   - o que mostrar quando não há data prevista;
   - se o usuário precisa perceber quando a data muda (essa pergunta surge no meio da conversa, quando a IA descobre que o ERP reprograma datas);
   - quais perfis podem ver a data.
4. A IA resume o entendimento e pergunta se pode enviar. "Quero ajustar algo" abre um ciclo de ajuste; "Sim, pode enviar" fecha o brief, carimba com o protocolo e encerra a sessão.
5. "Reiniciar" (ou "Nova demanda", ao final) recomeça a simulação.

O roteiro avança qualquer que seja o texto digitado. A resposta sempre vira um esclarecimento no brief; quando ela corresponde a uma das sugestões (por palavras-chave), também gera regra de negócio e critério de aceite específicos. Caso contrário, a regra registra a resposta entre aspas.

## Arquitetura

O arquivo tem três blocos:

| Bloco | Responsabilidade |
| --- | --- |
| `createSimulatedIntakeService` | Roteiro da IA simulada. Emite eventos com atrasos realistas e respeita `AbortSignal`. |
| `createVertexIntakeService` | Claude no Vertex AI com tools. Faz o loop de `tool_use`, mantém histórico e brief e traduz o streaming para os eventos da UI. |
| `createSseIntakeService` | Adaptador para uma futura API própria de intake, com o mesmo contrato. Lê `text/event-stream` via `fetch`. |
| UI (IIFE no fim do arquivo) | Consome eventos e renderiza conversa, atividades, brief e estado da sessão. Não conhece o roteiro nem o modelo. |

### Contrato do serviço

```ts
interface IntakeService {
  start(opts?: { signal?: AbortSignal }): AsyncIterable<IntakeEvent>;               // abre a sessão
  sendMessage(text: string, opts?: { signal?: AbortSignal }): AsyncIterable<IntakeEvent>;
}

type IntakeEvent =
  | { type: "thinking" }
  | { type: "tool_activity"; id?: string; label: string; state?: "running" | "done"; result?: string }
  | { type: "text_delta"; text: string }
  | { type: "brief_updated"; brief: Brief }          // snapshot completo
  | { type: "suggestions"; options: string[] }
  | { type: "status"; value: "understanding" | "clarifying" | "ready" | "submitted" }
  | { type: "error"; message: string }
  | { type: "done" };

interface Brief {
  version: number;
  status: "understanding" | "clarifying" | "ready" | "submitted";
  title: string | null;
  origin: string | null;              // pedido nas palavras do solicitante
  objective: string | null;
  rules: { id: string; text: string }[];          // RN01, RN02…
  acceptance: { id: string; text: string }[];     // CA01, CA02… (dado/quando/então)
  systems: { id: string; name: string; role: string; consumes?: string }[];
  openQuestions: { id: string; text: string }[];
  clarifications: { id: string; question: string; answer: string }[];
  protocol: string | null;
  submittedAt: string | null;         // ISO 8601
  updatedAt: string | null;
  readiness?: string | null;          // justificativa do envio (modo Vertex)
}
```

Notas do contrato:

- `brief_updated` sempre manda o brief inteiro. A UI compara com o snapshot anterior por `id` para animar o que entrou, o que saiu (pergunta respondida) e o que mudou (destaque de marca-texto).
- `tool_activity` com `id` permite marcar a mesma atividade como `running` e depois `done` com um `result`. Sem `id`, cada atividade nova encerra a anterior.
- O HTML é sempre escapado.
- Eventos de tipo desconhecido são ignorados, o que permite evoluir o backend sem quebrar a UI.
- `error` pode trazer `code` (`auth`, `config`, `rate`, `network`…). Com `auth` ou `config`, a UI reabre o diálogo de conexão.
- Textos aceitam o markdown simples que a IA costuma usar: parágrafos, listas, títulos, `**negrito**`, `*itálico*` e `` `código` ``.

### Trocando por uma API própria

Quando a chamada ao modelo sair do navegador e for para um backend, defina a URL antes dos scripts do arquivo:

```html
<script>window.INTAKE_API_URL = 'https://intake.exemplo.interno';</script>
```

O adaptador SSE assume esta proposta de API (ajuste quando o backend existir):

- `POST /intake/sessions` responde `text/event-stream`. O primeiro evento é `{ "type": "session", "sessionId": "…" }`, seguido dos eventos de abertura.
- `POST /intake/sessions/{id}/messages` com `{ "text": "…" }` responde `text/event-stream` com os eventos da resposta.

Cada evento SSE traz um `IntakeEvent` em JSON no campo `data:`.

### Ajuste de velocidade (desenvolvimento)

`window.INTAKE_SIM_SPEED` multiplica os tempos da simulação (padrão `1`). Útil para testes automatizados, por exemplo `0.1`. `window.INTAKE_DEFAULT_MODE = 'simulated'` abre a página no modo Simulação quando não há preferência salva.

## Interface

- Desktop: conversa à esquerda, folha do brief à direita. Abaixo de 900 px o brief vira uma gaveta, aberta pelo botão "Brief" com contador de itens. O que muda com a gaveta fechada é destacado quando ela abre.
- Estado da sessão no topo: Entendendo a demanda → Esclarecendo detalhes → Pronto para planejamento → Enviado para planejamento.
- Tema claro e escuro seguindo o sistema. O botão alterna o tema; voltar ao tema do sistema volta a segui-lo.
- Teclado: Enter envia, Shift+Enter quebra linha, Esc fecha a gaveta. O foco volta ao campo após enviar.
- A rolagem acompanha a resposta da IA, mas para se você rolar para cima; nesse caso aparece "Novas mensagens".
- Acessibilidade: região `aria-live` anuncia cada resposta completa da IA (não os pedaços do streaming), mudanças de estado e quantos registros entraram no brief; a gaveta é um diálogo modal com o fundo `inert`; foco visível em todos os controles; `prefers-reduced-motion` desliga os movimentos e mantém só o destaque de cor.
