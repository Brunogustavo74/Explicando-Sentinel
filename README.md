# SENTINEL

**Sistema Inteligente de Detecção de Comportamentos Anômalos**

Demonstração institucional de uma plataforma de *Insider Threat Detection* integrada a um ERP acadêmico. O projeto simula, em tempo real, como um sistema moderno de segurança deve observar, pontuar e explicar cada ação de usuários privilegiados — sem depender de caixa-preta ou de auditorias tardias.

---

## Visão geral

O ambiente é composto por **dois sistemas conectados**:

1. **Portal Acadêmico** — ERP escolar usado por professores, coordenadores e funcionários. Cada ação (login, edição de notas, download, alteração de usuários, acesso a áreas restritas) gera um evento estruturado.
2. **Console Sentinel** — Recebe os eventos em tempo real, aplica regras determinísticas versionadas sobre os metadados, calcula uma pontuação de risco e abre alertas explicáveis.

O fluxo é totalmente observável: o operador vê o evento chegar, entende **por que** ele foi considerado suspeito e acompanha a resposta automática (bloqueio, solicitação de aprovação, MFA).

---

## Diferenciais em relação a outras soluções

A maioria dos sistemas acadêmicos hoje falha em pontos que o Sentinel resolve de forma explícita:

| Limitação comum no mercado | Como o Sentinel resolve |
| --- | --- |
| Ausência de UEBA (análise comportamental) | Perfil de rotina por usuário (horário, dispositivo, cidade habitual) e desvios pontuados |
| Auditoria apenas *post-mortem* | Monitoramento e alertas **em tempo real** |
| Alertas genéricos ("acesso suspeito") | Cada alerta lista **quais regras dispararam** e o peso de cada uma |
| Falta de detecção de *Impossible Travel* | Correlação GeoIP com janela temporal (login em Recife e 4 min depois em Lisboa → crítico) |
| MFA opcional e sem monitoramento | Falhas de MFA em ações sensíveis viram evento e contribuem para o risco |
| Escalada de privilégios sem rastro | Concessão de papel administrativo fora do fluxo de RH gera alerta dedicado |
| Alterações fora do calendário letivo | Regra específica para edição em disciplinas já encerradas |
| Sessões eternas sem reautenticação | Regra de sessão longa (>8h) integrada à pontuação |
| Bloqueio manual e demorado | Bloqueio **automático** ao atingir 90% de risco, com fluxo de liberação pelo superior |
| Mudança de localização sem controle | Solicitação de mudança de cidade habitual exige **aprovação do administrador** |
| Metadados descartados | Cada evento carrega dispositivo, IP, cidade, contagem e metadados livres, exibidos no console |
| Interfaces inacessíveis | Painel de acessibilidade nativo (contraste, fonte, dislexia, movimento reduzido, cursor grande) |

---

## Arquitetura

- **Frontend:** React 19 + TanStack Start (SSR/SSG), Vite 7, Tailwind CSS v4.
- **Backend:** Supabase (Postgres + Auth + Storage), server functions via `@tanstack/react-start`.
- **Estado do Sentinel:** store reativa (`src/lib/sentinel-store.ts`) que ingere eventos, aplica regras e mantém o cálculo de risco por usuário.
- **Roteamento:** file-based em `src/routes/` (páginas do Portal em `portal.*.tsx`, páginas do console em `sentinel.*.tsx`).

### Estrutura principal

```
src/
├── routes/
│   ├── index.tsx                       Landing institucional
│   ├── portal.*.tsx                    Portal Acadêmico (login, notas, alunos, arquivos, etc.)
│   ├── portal.central-simulacao.tsx    Central de simulação de cenários
│   └── sentinel.*.tsx                  Console Sentinel (dashboard, alertas, eventos, regras, solicitações…)
├── components/
│   ├── portal/PortalShell.tsx
│   └── sentinel/SentinelShell.tsx      + AccessibilityPanel.tsx
├── lib/
│   ├── sentinel-store.ts               Motor de regras e risco
│   └── webhook-service.ts              Integração de eventos
└── integrations/supabase/              Cliente e middleware (auto-gerados)
```

---

## Motor de regras

Cada regra é versionada e possui um peso. A pontuação final do usuário é a soma dos pesos das regras violadas nas últimas janelas relevantes. Ao ultrapassar limiares, o Sentinel classifica o comportamento:

| Faixa de risco | Classificação | Ação automática |
| --- | --- | --- |
| 0–29 | Informativo | Apenas registrado no log |
| 30–59 | Baixo | Alerta amarelo no painel |
| 60–89 | Médio | Destaque no dashboard e no perfil |
| ≥ 70 | **Anômalo destacado** | Realce visível no painel principal |
| ≥ 90 | Crítico | **Bloqueio automático** da conta + fluxo de liberação pelo superior |

### Regras implementadas

| # | Regra | Peso | O que detecta |
| --- | --- | --- | --- |
| 1 | Login em madrugada | 35 | Acesso fora da rotina horária habitual |
| 2 | Novo dispositivo | 30 | Fingerprint desconhecido |
| 3 | Download em massa | 55 | Muitos arquivos em sequência |
| 4 | Alteração em massa de notas | 65 | Edição de dezenas de registros |
| 5 | Edição de usuários | 40 | Alteração de contas fora do fluxo |
| 6 | Burst de ações | 25 | Muitos eventos por minuto |
| 7 | Brute-force | 30 | Falhas consecutivas de login |
| 8 | Acesso a nova área | 20 | Módulo sem histórico de uso |
| 9 | **Impossible Travel** | 90 | Logins em cidades geograficamente incompatíveis |
| 10 | **Falha de MFA** | 60 | Códigos MFA inválidos em ações sensíveis |
| 11 | **Escalada de privilégios** | 70 | Papel administrativo fora do RH |
| 12 | **Alteração fora do calendário** | 45 | Notas em disciplinas encerradas |
| 13 | **Correlação SIEM** | 40 | 3+ tipos distintos de evento suspeito em 5 min |
| 14 | **Sessão excessivamente longa** | 25 | Mais de 8h sem reautenticação |

---

## Fluxos automáticos

- **Bloqueio automático:** ao ultrapassar 90 pontos, a conta é bloqueada e o usuário só pode enviar solicitação de liberação ao superior.
- **Solicitação de mudança de localização:** trocar a cidade habitual não é uma ação livre — vira uma solicitação para o administrador aprovar no console.
- **Solicitação de desbloqueio:** ambas as pendências aparecem em `/sentinel/solicitacoes`, com contexto, motivo e histórico.
- **Alertas explicáveis:** em `/sentinel/alertas/:id`, cada alerta lista as regras acionadas, os pesos e o evento original.

---

## Central de Simulação

Em `/portal/central-simulacao` há 15 cenários prontos, cada um representando um vetor real de risco (login noturno, novo dispositivo, download massivo, MFA falhando, *Impossible Travel*, escalada de privilégios etc.). É possível disparar cenários individualmente ou executar o **Cenário Completo**, que dispara toda a bateria em sequência para observar o motor reagindo.

---

## Console Sentinel — páginas

- `/sentinel` — Dashboard com KPIs, tendência de risco e *Insights de Segurança* (Impossible Travel, MFA, correlações, escaladas).
- `/sentinel/alertas` — Fila de alertas, com detalhe explicável em `/sentinel/alertas/:id`.
- `/sentinel/eventos` — Stream completo com filtros por tipo, usuário, busca e metadados.
- `/sentinel/usuarios` — Perfis, risco atual, histórico e status de bloqueio.
- `/sentinel/regras` — Catálogo versionado das regras e pesos.
- `/sentinel/solicitacoes` — Aprovações pendentes (mudança de localização, desbloqueio).
- `/sentinel/perfis`, `/sentinel/relatorios`, `/sentinel/logs`, `/sentinel/integracoes`, `/sentinel/configuracoes`.

---

## Acessibilidade

O console inclui um **painel de acessibilidade** persistente (`AccessibilityPanel`) com:

- Escala de fonte (1x / 1.15x / 1.3x / 1.5x)
- Alto contraste
- Sublinhado de links
- Fonte para dislexia
- Movimento reduzido
- Cursor grande
- Link "pular para o conteúdo", `aria-current` na navegação, foco visível, alvos de toque ≥ 44px e regiões semânticas rotuladas.

Preferências são salvas em `localStorage`.

---

## Responsividade

Ambos os shells (`PortalShell` e `SentinelShell`) usam sidebar fixa em telas grandes e *drawer* em telas menores, com header adaptativo (busca, notificações e perfil colapsam progressivamente).

---

## Como rodar

```bash
bun install
bun run dev
```

Preview em `http://localhost:8080`.

---

## Princípios do projeto

1. **Apenas metadados.** Nunca conteúdo de mensagens ou de arquivos.
2. **Tempo real.** Eventos aparecem em segundos; dashboards atualizam sozinhos.
3. **Explicável.** Cada alerta mostra as regras e os pesos que levaram à decisão.
4. **Determinístico e versionado.** Regras são auditáveis, não um modelo opaco.
5. **Resposta automática, revisão humana.** Bloqueio é imediato; a liberação é humana.
