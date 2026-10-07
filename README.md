# Intake de Demandas: protótipo de front

Protótipo navegável da experiência de intake de demandas de produto. O solicitante descreve o que precisa, uma IA (simulada) faz perguntas de negócio uma de cada vez e monta ao vivo o brief estruturado da demanda, até enviá-lo para planejamento.

Não há backend: a IA é uma simulação com roteiro fixo, isolada numa camada de serviço que pode ser trocada por uma API real com streaming (SSE).

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

## Roteiro de demonstração

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
| `createSseIntakeService` | Adaptador para a API real, com o mesmo contrato. Lê `text/event-stream` via `fetch`. |
| UI (IIFE no fim do arquivo) | Consome eventos e renderiza conversa, atividades, brief e estado da sessão. Não conhece o roteiro. |

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
}
```

Notas do contrato:

- `brief_updated` sempre manda o brief inteiro. A UI compara com o snapshot anterior por `id` para animar o que entrou, o que saiu (pergunta respondida) e o que mudou (destaque de marca-texto).
- `tool_activity` com `id` permite marcar a mesma atividade como `running` e depois `done` com um `result`. Sem `id`, cada atividade nova encerra a anterior.
- Textos aceitam `**negrito**`. O HTML é sempre escapado.
- Eventos de tipo desconhecido são ignorados, o que permite evoluir o backend sem quebrar a UI.

### Trocando pela API real

Defina a URL antes dos scripts do arquivo:

```html
<script>window.INTAKE_API_URL = 'https://intake.exemplo.interno';</script>
```

O adaptador SSE assume esta proposta de API (ajuste quando o backend existir):

- `POST /intake/sessions` responde `text/event-stream`. O primeiro evento é `{ "type": "session", "sessionId": "…" }`, seguido dos eventos de abertura.
- `POST /intake/sessions/{id}/messages` com `{ "text": "…" }` responde `text/event-stream` com os eventos da resposta.

Cada evento SSE traz um `IntakeEvent` em JSON no campo `data:`.

### Ajuste de velocidade (desenvolvimento)

`window.INTAKE_SIM_SPEED` multiplica os tempos da simulação (padrão `1`). Útil para testes automatizados, por exemplo `0.1`.

## Interface

- Desktop: conversa à esquerda, folha do brief à direita. Abaixo de 900 px o brief vira uma gaveta, aberta pelo botão "Brief" com contador de itens. O que muda com a gaveta fechada é destacado quando ela abre.
- Estado da sessão no topo: Entendendo a demanda → Esclarecendo detalhes → Pronto para planejamento → Enviado para planejamento.
- Tema claro e escuro seguindo o sistema. O botão alterna o tema; voltar ao tema do sistema volta a segui-lo.
- Teclado: Enter envia, Shift+Enter quebra linha, Esc fecha a gaveta. O foco volta ao campo após enviar.
- A rolagem acompanha a resposta da IA, mas para se você rolar para cima; nesse caso aparece "Novas mensagens".
- Acessibilidade: região `aria-live` anuncia cada resposta completa da IA (não os pedaços do streaming), mudanças de estado e quantos registros entraram no brief; a gaveta é um diálogo modal com o fundo `inert`; foco visível em todos os controles; `prefers-reduced-motion` desliga os movimentos e mantém só o destaque de cor.
