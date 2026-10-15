# 8. Plano de implementação: `pwa/` × documentação

Este documento compara o código atual de `pwa/` com os documentos 01 a 07 desta pasta e lista, em ordem de execução, as alterações necessárias para o frontend ficar igual à documentação.

- **Base da análise:** branch atual do repositório (outubro de 2026).
- **Como usar:** siga as fases em ordem. Cada fase lista os arquivos a **criar**, **alterar** e **remover**, o código de referência e os critérios de aceite. Cada fase cabe em um PR (`feat(pwa): …` ou `fix(pwa): …`).
- **Guardrail:** as dependências (Next 16, React 19, React Admin 5, `@vis.gl/react-google-maps` 1.x, `lucide-react` 1.x) mudam rápido. Onde o texto diz **"confirmar"**, abra o código em `pwa/node_modules/` antes de usar a API, como pede o `CLAUDE.md`.

---

## 8.1 Resumo da análise

| Situação | Itens | Destaques |
|---|:-:|---|
| ✅ Implementado e conforme | 9 | Autenticação Keycloak/`better-auth`, `useAccessToken`, logout com `id_token_hint`, papéis via JWT, `ENTRYPOINT`, provedor do React Query, componentes `ui/`, montagem da query de filtros |
| ⚠️ Implementado com erro | 27 | Build quebrado (`EstablishmentDrawer.tsx` vazio), `lang="en"`, mapa em São Paulo e travado, filtros só por cor, `<a href="#">` no cabeçalho, callbacks para `/books`, `fetchApi` sem status HTTP, admin de livros |
| ❌ Não implementado | 11 telas/recursos + 6 bibliotecas | T2–T6, T10, lista em T1, preferências de acessibilidade, VLibras, anunciador global, votos, Mercure, testes E2E do Acessibiliza |

Cobertura atual estimada dos requisitos funcionais: **RF01 e RF03/RF04 parciais, RF06 parcial**. Nenhum dos demais RF está atendido.

---

## 8.2 O que já está implementado e conforme

| # | Item | Arquivo | Referência na doc |
|---|---|---|---|
| C1 | Login OIDC com `better-auth` + `genericOAuth`, PKCE, escopos `openid profile email offline_access roles` | `lib/auth.ts`, `app/api/auth/[...all]/route.ts` | 02 §2.2 |
| C2 | Logout que encerra a sessão local e a do Keycloak com `id_token_hint` | `app/api/auth/keycloak-logout/route.ts` | 04 §4.6 |
| C3 | `useAccessToken()`, `signInWithKeycloak()`, `signOutWithKeycloak()` | `hooks/useAuth.ts` | 02 §2.2 |
| C4 | Leitura de `realm_access.roles` do JWT e bloqueio do admin sem papel `admin` | `lib/jwt.ts`, `components/admin/Admin.tsx` | 02 §2.2, 04 §4.8 |
| C5 | `ENTRYPOINT` com mesma origem no navegador e `NEXT_PUBLIC_ENTRYPOINT` no servidor | `config/entrypoint.ts` | 02 §2.1 |
| C6 | `fetchApi` com `Accept: application/ld+json`, `merge-patch+json` no PATCH, `Bearer`, `hubURL` e `violations → fields` | `utils/dataAccess.ts` | 02 §2.1 (com correções em E9) |
| C7 | `QueryClientProvider` global | `app/providers.tsx` | 06 §6.3 |
| C8 | Componentes Base UI/Shadcn (`button`, `dialog`, `sheet`, `drawer`, `toast`, `input`, `skeleton`…) | `components/ui/` | 01 README, 06 §6.5 |
| C9 | Query `GET /establishments` com `name`, `address` e `criterion_average[<criterio>]=bom|medio|ruim` no formato que a API espera | `app/page.tsx` (`buildEstablishmentsUrl`) | 02 §2.3.1 |

---

## 8.3 O que foi implementado errado

Ordenado por gravidade. **B** = bloqueante (quebra build ou fluxo), **A** = acessibilidade (critério de aceite), **F** = funcional, **L** = limpeza do demo.

| # | Grav. | Arquivo | Problema | O que a doc pede | Fase |
|---|:-:|---|---|---|:-:|
| E1 | B | `components/home/EstablishmentDrawer.tsx` | Arquivo **vazio**; `app/page.tsx` faz `import EstablishmentDrawer from …` (default export inexistente): o build falha | Cartão-resumo do mapa (T1) | 0 |
| E2 | B | `components/admin/Admin.tsx`, `admin/book/*`, `admin/review/*`, `layout/Menu.tsx` | Recursos `admin/books` e `admin/reviews` **não existem** na API do Acessibiliza: o admin abre vazio ou com erro | `admin/establishments`, `admin/evaluations`, `admin/users` (03 T9) | 9 |
| E3 | F | `components/home/MapView.tsx` | Centro em **São Paulo** (`-23.5505, -46.6333`), zoom 13 | Tubarão `{ lat: -28.4667, lng: -49.0069 }`, zoom 14 (02 §2.6) | 3 |
| E4 | F | `components/home/MapView.tsx` | `center={userLocation}` torna a câmera **controlada** sem `onCameraChanged`: o usuário não consegue arrastar o mapa (confirmar em `@vis.gl/react-google-maps`) | Câmera não controlada (`defaultCenter`) + `map.panTo()` no botão "Minha localização" | 3 |
| E5 | F | `components/home/MapView.tsx` | `AdvancedMarker` sem `mapId`: o Google não renderiza marcadores avançados sem Map ID. Com `mapId`, a prop `styles` é ignorada | Definir `mapId` (nova variável `NEXT_PUBLIC_GOOGLE_MAPS_MAP_ID`) e remover `MAP_STYLES` | 3 |
| E6 | A | `components/home/MapView.tsx` | Geolocalização pedida **no carregamento** | Só após clique em "Minha localização" (02 §2.6, 04 §4.2) | 3 |
| E7 | A | `components/home/MapView.tsx` | Todos os pinos iguais (`#0f929a`), `title` só com o nome; sem `role="region"`, sem texto alternativo; `gestureHandling="greedy"`; POIs ocultos; overlay de carregamento sem `role="status"` | Pino com cor + glifo por situação, `title` "Nome, situação geral: Bom", região rotulada, POIs visíveis, `role="status"` (05 §5.4.3, 03 §3.3) | 3 |
| E8 | F | `components/home/MapView.tsx` | `parseGeometry` não aceita espaços (`POINT (x y)`) e está dentro do componente | `parsePoint` em `lib/geo.ts`, regex tolerante (02 §2.3.1) | 1 |
| E9 | F | `utils/dataAccess.ts` | `FetchError.status` recebe `json.description` (texto), não o **status HTTP**; sem `violations`, lança `Error(title)` sem status. A UI não consegue distinguir 400/401/403/422. `JSON.parse` quebra com resposta não JSON (502 HTML, corpo vazio) | Status numérico em todos os erros, para o tratamento da tabela 02 §2.4 | 1 |
| E10 | A | `components/common/Header.tsx` | `<a href="#">` + `onClick` + `role="menuitem"`; textos "Sign out"/"Login"; ícone MUI; `aria-label="Global"`; sem navegação, sem "Avaliar", sem "Aa" | `<button>`, "Entrar"/"Sair", `<nav aria-label="Principal">` com Explorar/Painel/Sobre, botão Avaliar, botão Acessibilidade, menu da conta (03 §3.1, 05 §5.6) | 2 |
| E11 | A | `app/layout.tsx` | `<html lang="en">`, título "Welcome to API Platform!" | `lang="pt-BR"`, título "Acessibiliza" com template "Página – Acessibiliza" | 2 |
| E12 | A | `components/common/Layout.tsx` | Sem skip link, sem `<main id="conteudo">`, sem rodapé, sem região *live*, sem Toaster | 03 §3.1, 05 §5.4.1 | 2 |
| E13 | A | `components/home/FilterSidebar.tsx` | `<label>` sem `htmlFor`; critérios como botões sem `aria-pressed`, estado só por cor; sem opção "Qualquer"; `title` como única explicação; bordas `gray-200` (< 3:1) | `<fieldset>`/`<legend>` + rádios Qualquer/Bom/Médio/Ruim, `htmlFor`, bordas ≥ `gray-500` (05 §5.4.5) | 3 |
| E14 | F | `components/home/FilterSidebar.tsx` | Lista de critérios **duplicada** no componente, com rótulos diferentes da doc ("Acesso p/ Cadeira de Rodas", "Banheiros Acessíveis", "Animais de Serviço Permitidos") | Fonte única em `lib/criteria.ts` (06 §6.2) | 1 |
| E15 | F | `components/home/FilterSidebar.tsx` | "Limpar" só aparece com filtro ativo; botão mobile sem contagem; botão mobile `absolute` sobre o mapa | "Aplicar filtros" + "Limpar filtros" sempre; "Filtros (2 ativos)"; chips de filtros ativos (03 T1) | 3 |
| E16 | F | `app/page.tsx` | Filtros em `useState` (perdidos ao recarregar/voltar); só `page=1` (o mapa mostra no máximo 30 locais); `fetchApi<any>`; tratamento `member`/`hydra:member` inline; altura fixa `calc(100vh-53px)` | Filtros na URL (`?q=&end=&cr=bom&view=lista`), todas as páginas para o mapa, `getMembers()` em `lib/hydra.ts` (03 T1, 06 §6.3) | 3 |
| E17 | F | `app/login/page.tsx` | Callback padrão `/books`; só spinner (`SyncLoader`) | Callback `/`; texto "Redirecionando para a página de login…" com `role="status"` (03 T7) | 0 |
| E18 | F | `hooks/useAuth.ts` | `signOutWithKeycloak()` usa `/books` como destino padrão | `${origin}/` (04 §4.6) | 0 |
| E19 | F | `app/api/auth/keycloak-logout/route.ts` | Fallback de `post_logout_redirect_uri` é `/books` | `/` | 0 |
| E20 | L | `components/admin/AccessDenied.tsx` | Textos em inglês e botão "Browse books" → `/books` | "Acesso negado", "Voltar ao mapa" → `/`, "Sair" | 9 |
| E21 | L | `components/admin/i18nProvider.ts` | Mensagens fr/en | pt-BR (03 T9) | 9 |
| E22 | L | `components/admin/Admin.tsx` | `next/head` não funciona no App Router; título "API Platform Admin" | `metadata` em `app/admin/layout.tsx`: "Administração – Acessibiliza" | 9 |
| E23 | A | `styles/globals.css` | `--color-cyan-700: #0f929a` usado em botão com texto branco (~3,8:1); `a { no-underline }` global; `* { font-family: Poppins }` impede a fonte para dislexia; utilitários do demo (`bg-spider-cover`, `ribbon`) | Tokens da doc (`--color-primary: #155E75`, status, foco), variantes `[data-contrast="alto"]`/`[data-theme="dark"]` (06 §6.5) | 1 |
| E24 | F | `types/Establishment.ts` | `criterion: string`; `Evaluation` sem `votes` e `establishment`; `EvaluationsSummary` com chave `string` | Tipos de 06 §6.2 (`CriterionValue`, `EvaluationVote`, `CreateEvaluationInput`) | 1 |
| E25 | F | `utils/mercure.ts` | Sobrescreve os dados com `setData` (fora do cache do React Query) e tem `console.error` (viola RNF07) | `useMercure(topic, onEvent)` que **invalida** queries (06 §6.3) | 7 |
| E26 | L | `components/common/{Error,Loading,Pagination}.tsx` | Herdados do demo: textos em inglês, atributo JSX inválido (`stroke-linecap`), `Loading` sem `aria-busy`/texto, `Pagination` com MUI e links numerados (a doc pede "Carregar mais") | Estados de 03 §3.2 | 2 |
| E27 | L | `pwa/FRONTEND_ARCHITECTURE.md`, `next.config.js`, `postcss.config.{js,mjs}`, `package.json` | Doc de arquitetura fala de livros, Next 14/15 e Radix; `remotePatterns` para `covers.openlibrary.org`; **dois** arquivos de PostCSS; dependências sem uso (`formik`, `slugify`, `ra-language-french`) | Atualizar/remover (06 §6.1) | 0 |

---

## 8.4 O que não foi implementado

### Telas

| Tela | Rota | RF | Situação |
|---|---|---|---|
| T1 Explorar | `/` | RF01–RF04 | Parcial: mapa e filtros existem (com os erros acima). **Faltam:** visualização em lista, alternador Mapa/Lista, busca no topo com *debounce*, contador `aria-live`, chips, ordenação, legenda, cartão-resumo, estados vazio/erro, *clustering*, "Minha localização" |
| T2 Detalhe | `/locais/[id]` | RF05, RF09, RF13 | Não existe |
| T3 Nova avaliação | `/avaliar`, `/locais/[id]/avaliar` | RF07, RF08, RF10 | Não existe |
| T4 Painel | `/painel` | RF11, RF12 | Não existe |
| T5 Sobre | `/sobre` | RF17 | Não existe |
| T6 Acessibilidade | `/acessibilidade` | RF16, RF17 | Não existe |
| T8 Minhas avaliações | `/minhas-avaliacoes` | RF15 | Bloqueada (L3); **não criar** |
| T10 Erros | `app/not-found.tsx`, `app/error.tsx`, offline | — | Não existe |

### Recursos transversais

| Recurso | Referência | Situação |
|---|---|---|
| Rodapé, skip link, landmarks, `RouteAnnouncer` | 03 §3.1, 05 §5.4.1 | Não existe |
| Anunciador global (`AnnouncerProvider`) | 05 §5.4.9 | Não existe |
| Toaster montado no layout | 03 §3.1 | O componente existe em `ui/toast.tsx`, mas **não está montado** |
| Painel de preferências "Aa" + script antes da hidratação | 05 §5.3.1, 04 §4.9 | Não existe |
| Widget VLibras | 05 §5.3.2 | Não existe |
| Atalhos de teclado eMAG (`Alt+1..4`, `Alt+Shift+A`, `/`) | 05 §5.3.3 | Não existe |
| Navegação mobile (hambúrguer ou barra inferior) | 03 §3.1 | Não existe |
| Votos otimistas | 04 §4.5 | Não existe |
| Tempo real no detalhe (Mercure) | 02 §2.5 | Não existe |

### Bibliotecas internas (06 §6.1)

`lib/criteria.ts`, `lib/rating.ts`, `lib/geo.ts`, `lib/hydra.ts`, `lib/format.ts`, `lib/queryKeys.ts`, `lib/filters.ts`, `lib/csv.ts`, `lib/i18n/pt-BR.ts`; hooks `useEstablishments`, `useAllEstablishments`, `useEstablishment`, `useCreateEvaluation`, `useVote`, `useMyVote`, `usePreferences`, `useAnnounce`, `useMercure`; tipos `Evaluation.ts`, `Vote.ts`, `Image.ts`, `api.d.ts`. **Nenhum existe.**

### Qualidade

| Item | Referência | Situação |
|---|---|---|
| `eslint-plugin-jsx-a11y` com regras recomendadas | 05 §5.8 | O `eslint-config-next` carrega só um subconjunto; falta a configuração recomendada |
| `@axe-core/playwright` nos testes E2E | 05 §5.8 | Não existe |
| Cenários E2E do Acessibiliza | 06 §6.8 | `e2e/tests/` ainda testa livros/bookmarks/GraphQL do demo; o mock server simula a Open Library |
| Testes unitários (Vitest) | 06 §6.8 | Não existe (opcional) |

---

## 8.5 Fases de implementação

```
Fase 0  Limpeza e correções bloqueantes ─┐
Fase 1  Fundação (lib, tipos, fetchApi, tokens) ─┤ Marco 1
Fase 2  Layout acessível ─┤
Fase 3  T1 Explorar ─┤
Fase 4  T2 Detalhe (leitura) ─┤
Fase 5  Preferências, VLibras, T5, T6, T10 ─┘
Fase 6  T3 Avaliação ─┐
Fase 7  Votos e Mercure ─┘ Marco 2
Fase 8  T4 Painel ── Marco 3
Fase 9  Administração ── Marco 1/4
Fase 10 Testes e lint (contínuo; fechamento)
```

---

### Fase 0: limpeza e correções bloqueantes

**Objetivo:** o projeto compila, e nenhum fluxo leva a `/books`.

| Ação | Arquivo | Alteração |
|---|---|---|
| Alterar | `components/home/EstablishmentDrawer.tsx` | Exportar um componente provisório (`export default function EstablishmentDrawer() { return null; }`) até a Fase 3, ou remover o import em `app/page.tsx` |
| Alterar | `app/login/page.tsx` | Ver código abaixo |
| Alterar | `hooks/useAuth.ts` | `redirectUri ?? \`${window.location.origin}/\`` |
| Alterar | `app/api/auth/keycloak-logout/route.ts` | `new URL("/", request.url)` |
| Remover | `postcss.config.js` | Manter só `postcss.config.mjs` |
| Remover | `types/Thumbnails.ts`, `public/api-platform/*` | Restos do demo (06 §6.1) |
| Alterar | `styles/globals.css` | Remover `bg-spider-cover` e `ribbon` (dependem de `public/api-platform/web.svg`) |
| Alterar | `next.config.js` | Trocar `covers.openlibrary.org` pelo host da API (imagens do LiipImagine, para a Fase 4/L1) |
| Alterar | `package.json` | Remover `formik`, `slugify`, `ra-language-french` (confirmar com `grep` que não há uso). `@mui/*` continua, mas **só** para o React Admin |
| Alterar | `pwa/FRONTEND_ARCHITECTURE.md` | Reescrever para o Acessibiliza (Next 16, Base UI) e apontar para `docs/frontend/` |

`app/login/page.tsx`:

```tsx
"use client";

import { useSearchParams } from "next/navigation";
import { Suspense, useEffect } from "react";

import { signInWithKeycloak } from "@/hooks/useAuth";

function LoginRedirect() {
  const searchParams = useSearchParams();

  useEffect(() => {
    signInWithKeycloak(searchParams.get("callbackURL") ?? "/");
  }, [searchParams]);

  return null;
}

const Message = () => (
  <div className="flex min-h-[60vh] items-center justify-center">
    <p role="status" className="text-lg">Redirecionando para a página de login…</p>
  </div>
);

export default function LoginPage() {
  return (
    <Suspense fallback={<Message />}>
      <Message />
      <LoginRedirect />
    </Suspense>
  );
}
```

**Aceite:** `pnpm build` e `pnpm lint` sem erros; `grep -rn "/books" pwa --exclude-dir=node_modules` só encontra arquivos do admin, que saem na Fase 9.

---

### Fase 1: fundação

**Objetivo:** criar as fontes únicas (critérios, situação, geometria, Hydra, formatação, query keys, filtros) e corrigir `fetchApi` e os tipos. Todas as fases seguintes dependem disto.

#### 1.1 Tipos (`types/`)

Criar `types/Evaluation.ts` e `types/Vote.ts` e ajustar `types/Establishment.ts` conforme 06 §6.2. O ideal é gerar `types/api.d.ts` primeiro e derivar os tipos dele:

```bash
cd pwa && npx openapi-typescript ../api/public/docs.jsonopenapi -o types/api.d.ts
```

```ts
// types/Evaluation.ts
import type { Establishment } from "./Establishment";

export type CriterionValue =
  | "wheelchair_accessible" | "accessible_restroom" | "tactile_paving"
  | "braille_signage" | "sign_language" | "service_animal_allowed";

export interface EvaluationRating { criterion: CriterionValue; rating: number } // 0–10

export interface EvaluationVote { "@id": string; value: 1 | -1 }

export interface Evaluation {
  "@id": string;
  comment?: string;
  ratings: EvaluationRating[];
  votes?: EvaluationVote[];
  netVotes: number;
  establishment?: string | Pick<Establishment, "@id" | "name" | "address">;
}

export type EvaluationsSummary = Partial<Record<CriterionValue, { average: number; count: number }>>;

export interface CreateEvaluationInput {
  establishmentGooglePlaceId: string;
  comment?: string;
  ratings: EvaluationRating[];
}
```

```ts
// types/Establishment.ts
import type { Item } from "./item";
import type { Evaluation, EvaluationsSummary } from "./Evaluation";

export interface Establishment extends Item {
  "@id": string;
  googlePlaceId?: string;
  name: string;
  address?: string;
  phoneNumber?: string;
  website?: string;
  location: string; // WKT "POINT(<lng> <lat>)"
  evaluations?: string[] | Evaluation[]; // IRIs na coleção, objetos no detalhe
  evaluationsSummary?: EvaluationsSummary;
}
```

`types/collection.ts`: tornar `@id` e `@type` de `view` opcionais e aceitar `hydra:member`/`hydra:view`/`hydra:totalItems` (o helper da 1.3 normaliza).

#### 1.2 `utils/dataAccess.ts` (corrige E9)

Manter a assinatura de `fetchApi`, mas lançar sempre um erro com status HTTP:

```ts
export class FetchError extends Error {
  constructor(
    message: string,
    public readonly status: number,
    public readonly fields: Record<string, string> = {},
  ) {
    super(message);
    this.name = "FetchError";
  }
}

// dentro de fetchApi, depois do fetch:
const text = await resp.text();
let json: any = null;
try {
  json = text ? JSON.parse(text) : null;
} catch {
  // resposta não JSON (ex.: página de erro do proxy)
}

if (resp.ok) {
  return { hubURL: extractHubURL(resp)?.toString() || null, data: json, text };
}

const fields: Record<string, string> = {};
for (const v of (json?.violations ?? []) as Violation[]) fields[v.propertyPath] = v.message;

throw new FetchError(json?.detail || json?.title || resp.statusText, resp.status, fields);
```

> `detail` traz a mensagem do servidor (ex.: "Invalid Google Place ID…"); `title` é genérico. A UI usa `status` para escolher a ação da tabela 02 §2.4.

#### 1.3 Bibliotecas em `lib/`

```ts
// lib/criteria.ts — fonte única dos critérios (02 §2.3.4). Confirmar os nomes dos ícones em node_modules/lucide-react.
import { Accessibility, Bath, Dog, Footprints, Hand, MessagesSquare, type LucideIcon } from "lucide-react";
import type { CriterionValue } from "@/types/Evaluation";

export interface Criterion {
  value: CriterionValue;
  param: string;        // chave curta na URL (06 §6.3)
  label: string;
  shortLabel: string;   // para chips e gráficos
  description: string;
  Icon: LucideIcon;
}

export const CRITERIA: readonly Criterion[] = [
  { value: "wheelchair_accessible", param: "cr", label: "Acesso para cadeira de rodas", shortLabel: "Cadeira de rodas",
    description: "Rampas, elevadores, portas largas e circulação livre", Icon: Accessibility },
  { value: "accessible_restroom", param: "bn", label: "Banheiro acessível", shortLabel: "Banheiro",
    description: "Banheiro adaptado, com barras de apoio e espaço de giro", Icon: Bath },
  { value: "tactile_paving", param: "pt", label: "Piso tátil", shortLabel: "Piso tátil",
    description: "Piso de alerta e direcional para pessoas com deficiência visual", Icon: Footprints },
  { value: "braille_signage", param: "br", label: "Sinalização em Braille", shortLabel: "Braille",
    description: "Placas em Braille ou alto-relevo", Icon: Hand },
  { value: "sign_language", param: "lb", label: "Atendimento em Libras", shortLabel: "Libras",
    description: "Atendimento na Língua Brasileira de Sinais", Icon: MessagesSquare },
  { value: "service_animal_allowed", param: "an", label: "Animal de serviço permitido", shortLabel: "Animal de serviço",
    description: "Entrada facilitada para cão-guia e outros animais de serviço", Icon: Dog },
] as const;

export const criterionByValue = (v: string) => CRITERIA.find((c) => c.value === v);

export const RATING_ANCHORS: Record<number, string> = {
  0: "Não existe",
  1: "Existe, mas inadequado", 2: "Existe, mas inadequado", 3: "Existe, mas inadequado", 4: "Existe, mas inadequado",
  5: "Parcial: usável com dificuldade", 6: "Parcial: usável com dificuldade",
  7: "Adequado, com pequenos problemas", 8: "Adequado, com pequenos problemas", 9: "Adequado, com pequenos problemas",
  10: "Totalmente adequado",
};
```

```ts
// lib/rating.ts — situação Bom/Médio/Ruim (03 §3.3)
import { CircleAlert, CircleCheck, CircleDashed, CircleX, type LucideIcon } from "lucide-react";
import type { EvaluationsSummary } from "@/types/Evaluation";

export type Status = "bom" | "medio" | "ruim" | "sem-dados";

export const statusFromAverage = (avg?: number | null): Status =>
  avg == null ? "sem-dados" : avg >= 7 ? "bom" : avg >= 5 ? "medio" : "ruim";

export const STATUS_META: Record<Status, { label: string; glyph: string; Icon: LucideIcon; className: string }> = {
  bom:         { label: "Bom",       glyph: "✔", Icon: CircleCheck,  className: "text-status-bom bg-status-bom-bg border-status-bom" },
  medio:       { label: "Médio",     glyph: "!", Icon: CircleAlert,  className: "text-status-medio bg-status-medio-bg border-status-medio" },
  ruim:        { label: "Ruim",      glyph: "✕", Icon: CircleX,      className: "text-status-ruim bg-status-ruim-bg border-status-ruim" },
  "sem-dados": { label: "Sem dados", glyph: "?", Icon: CircleDashed, className: "text-status-nd bg-status-nd-bg border-status-nd" },
};

/** Média simples das médias dos critérios avaliados (03 T2: "Média geral (X de 6 critérios avaliados)"). */
export function overallAverage(summary?: EvaluationsSummary) {
  const values = Object.values(summary ?? {}).map((s) => s!.average);
  return {
    average: values.length ? values.reduce((a, b) => a + b, 0) / values.length : null,
    evaluatedCriteria: values.length,
  };
}

/** Maior `count` entre os critérios: aproximação do nº de avaliações do local (03 T4). */
export const evaluationCount = (summary?: EvaluationsSummary) =>
  Math.max(0, ...Object.values(summary ?? {}).map((s) => s!.count));
```

```ts
// lib/geo.ts
export const TUBARAO_CENTER = { lat: -28.4667, lng: -49.0069 };
export const DEFAULT_ZOOM = 14;

export function parsePoint(wkt?: string | null): { lat: number; lng: number } | null {
  const m = wkt?.match(/POINT\s*\(\s*([-\d.]+)\s+([-\d.]+)\s*\)/i);
  return m ? { lng: parseFloat(m[1]), lat: parseFloat(m[2]) } : null;
}

/** Distância em km (Haversine), para ordenar por proximidade na lista. */
export function distanceKm(a: { lat: number; lng: number }, b: { lat: number; lng: number }) {
  const R = 6371, rad = Math.PI / 180;
  const dLat = (b.lat - a.lat) * rad, dLng = (b.lng - a.lng) * rad;
  const h = Math.sin(dLat / 2) ** 2 + Math.cos(a.lat * rad) * Math.cos(b.lat * rad) * Math.sin(dLng / 2) ** 2;
  return 2 * R * Math.asin(Math.sqrt(h));
}
```

```ts
// lib/hydra.ts — trata os formatos com e sem prefixo "hydra:" (02 §2.1)
export const getMembers = <T>(data: any): T[] => data?.member ?? data?.["hydra:member"] ?? [];
export const getTotal = (data: any): number => data?.totalItems ?? data?.["hydra:totalItems"] ?? 0;
export const getNext = (data: any): string | undefined =>
  (data?.view ?? data?.["hydra:view"])?.next ?? (data?.["hydra:view"])?.["hydra:next"];
```

```ts
// lib/format.ts
const avgFmt = new Intl.NumberFormat("pt-BR", { maximumFractionDigits: 1 });
export const formatAverage = (n: number) => avgFmt.format(n);       // 8,2
export const formatDate = (d: Date | string) => new Intl.DateTimeFormat("pt-BR").format(new Date(d));
export const plural = (n: number, one: string, many: string) => `${n} ${n === 1 ? one : many}`;
```

```ts
// lib/queryKeys.ts
import type { ExploreFilters } from "./filters";

export const queryKeys = {
  establishments: (filters?: ExploreFilters) => ["establishments", filters] as const,
  allEstablishments: (filters?: ExploreFilters) => ["establishments", "all", filters] as const,
  establishment: (id: string) => ["establishment", id] as const,
  myVote: (evaluationIri: string, userIri?: string) => ["my-vote", evaluationIri, userIri] as const,
};
```

```ts
// lib/filters.ts — serialização curta dos filtros na URL (06 §6.3: ?q=&end=&cr=bom&bn=ruim&view=lista)
import { CRITERIA } from "./criteria";
import type { CriterionValue } from "@/types/Evaluation";

export type Band = "bom" | "medio" | "ruim";
export interface ExploreFilters {
  q: string;
  end: string;
  criteria: Partial<Record<CriterionValue, Band>>;
  order: "asc" | "desc";
}

const BANDS: Band[] = ["bom", "medio", "ruim"];

export function filtersFromSearchParams(sp: URLSearchParams): ExploreFilters {
  const criteria: ExploreFilters["criteria"] = {};
  for (const c of CRITERIA) {
    const v = sp.get(c.param) as Band | null;
    if (v && BANDS.includes(v)) criteria[c.value] = v;
  }
  return { q: sp.get("q") ?? "", end: sp.get("end") ?? "", criteria, order: sp.get("ordem") === "desc" ? "desc" : "asc" };
}

export function filtersToSearchParams(f: ExploreFilters, base = new URLSearchParams()): URLSearchParams {
  const sp = new URLSearchParams(base);
  ["q", "end", "ordem", ...CRITERIA.map((c) => c.param)].forEach((k) => sp.delete(k));
  if (f.q) sp.set("q", f.q);
  if (f.end) sp.set("end", f.end);
  if (f.order === "desc") sp.set("ordem", "desc");
  for (const c of CRITERIA) if (f.criteria[c.value]) sp.set(c.param, f.criteria[c.value]!);
  return sp;
}

/** Caminho da API para a 1ª página (substitui buildEstablishmentsUrl de app/page.tsx). */
export function establishmentsPath(f: ExploreFilters): string {
  const sp = new URLSearchParams({ page: "1", "order[name]": f.order });
  if (f.q) sp.set("name", f.q);
  if (f.end) sp.set("address", f.end);
  for (const [criterion, band] of Object.entries(f.criteria)) sp.set(`criterion_average[${criterion}]`, band!);
  return `/establishments?${sp}`;
}

export const activeFilterCount = (f: ExploreFilters) =>
  Object.keys(f.criteria).length + (f.q ? 1 : 0) + (f.end ? 1 : 0);
```

Também criar `lib/i18n/pt-BR.ts` (06 §6.9) e mover para lá os textos de interface à medida que os componentes forem escritos.

#### 1.4 Hooks de dados (`hooks/`)

```ts
// hooks/useEstablishments.ts
"use client";
import { useInfiniteQuery, useQuery } from "@tanstack/react-query";
import { fetchApi } from "@/utils/dataAccess";
import { establishmentsPath, type ExploreFilters } from "@/lib/filters";
import { getMembers, getNext, getTotal } from "@/lib/hydra";
import { queryKeys } from "@/lib/queryKeys";
import type { Establishment } from "@/types/Establishment";

const MAX_PAGES = 10; // limite de segurança enquanto a lacuna L7 existir

export function useEstablishments(filters: ExploreFilters) {
  return useInfiniteQuery({
    queryKey: queryKeys.establishments(filters),
    initialPageParam: establishmentsPath(filters),
    queryFn: async ({ pageParam }) => (await fetchApi<unknown>(pageParam))?.data,
    getNextPageParam: (last) => getNext(last),
    select: (d) => ({
      items: d.pages.flatMap((p) => getMembers<Establishment>(p)),
      total: getTotal(d.pages[0]),
    }),
    staleTime: 60_000,
  });
}

export function useAllEstablishments(filters: ExploreFilters) {
  return useQuery({
    queryKey: queryKeys.allEstablishments(filters),
    queryFn: async () => {
      const items: Establishment[] = [];
      let path: string | undefined = establishmentsPath(filters);
      for (let i = 0; path && i < MAX_PAGES; i++) {
        const data = (await fetchApi<unknown>(path))?.data;
        items.push(...getMembers<Establishment>(data));
        path = getNext(data);
      }
      return items;
    },
    staleTime: 60_000,
  });
}
```

`hooks/useEstablishment.ts`: `useQuery({ queryKey: queryKeys.establishment(id), queryFn, initialData })`, devolvendo também o `hubURL` para a Fase 7.

#### 1.5 Tokens de design (`styles/globals.css`, corrige E23)

```css
@theme {
  --font-sans: Poppins, system-ui, sans-serif;

  --color-primary: #155E75;        /* cyan-800: 7:1 com texto branco */
  --color-primary-hover: #164E63;
  --color-focus: #F59E0B;
  --color-status-bom: #166534;    --color-status-bom-bg: #DCFCE7;
  --color-status-medio: #854D0E;  --color-status-medio-bg: #FEF9C3;
  --color-status-ruim: #991B1B;   --color-status-ruim-bg: #FEE2E2;
  --color-status-nd: #374151;     --color-status-nd-bg: #F3F4F6;
}

@layer base {
  html { font-size: var(--base, 100%); scroll-padding-top: 5rem; }
  body { font-family: var(--font-sans); line-height: 1.5; }
  /* remover o "* { font-family: Poppins }": impede a preferência de fonte para dislexia */

  :focus-visible { outline: 3px solid var(--color-focus); outline-offset: 2px; box-shadow: 0 0 0 5px #1f2937; }
  a { color: var(--color-primary); text-decoration: underline; text-underline-offset: 2px; }

  *, ::after, ::before { border-color: var(--color-gray-500, currentcolor); } /* ≥ 3:1 (1.4.11) */
}

/* Preferências (05 §5.3.1) — atributos aplicados em <html> */
[data-contrast="alto"] {
  --color-primary: #FFD400; --color-primary-hover: #FFE666;
  --color-status-bom: #7CFC9A; --color-status-bom-bg: #000;
  --color-status-medio: #FFD400; --color-status-medio-bg: #000;
  --color-status-ruim: #FF8A8A; --color-status-ruim-bg: #000;
  --color-status-nd: #fff; --color-status-nd-bg: #000;
  background: #000; color: #fff;
}
[data-motion="reduzido"] *, [data-motion="reduzido"] *::before, [data-motion="reduzido"] *::after {
  animation: none !important; transition: none !important; scroll-behavior: auto !important;
}
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation: none !important; transition: none !important; scroll-behavior: auto !important; }
}
[data-font="dislexia"] body { font-family: "Atkinson Hyperlegible", system-ui, sans-serif; }
[data-spacing="ampliado"] body { line-height: 1.8; letter-spacing: .12em; word-spacing: .16em; }
[data-links="sublinhado"] a { text-decoration: underline !important; }
```

Remover as sobrescritas `--color-cyan-*` e `--color-red-500` (mudam a paleta do Tailwind e escondem o problema de contraste) e trocar `bg-cyan-700` por `bg-primary` nos componentes. Instalar a fonte `@fontsource/atkinson-hyperlegible` (confirmar o nome do pacote).

**Aceite da fase:** `pnpm lint` limpo; nenhum componente define lista de critérios própria; `fetchApi` lança `FetchError` com `status` numérico em qualquer resposta não OK.

---

### Fase 2: layout acessível

**Arquivos:**

| Ação | Arquivo |
|---|---|
| Alterar | `app/layout.tsx`, `app/providers.tsx`, `components/common/Layout.tsx`, `components/common/Header.tsx` |
| Criar | `components/common/{SkipLink,Footer,RouteAnnouncer,MobileNav,Toaster}.tsx`, `components/a11y/{AnnouncerProvider,PreferencesProvider,VisuallyHidden}.tsx`, `hooks/useAnnounce.ts` |
| Reescrever | `components/common/{Loading,Error}.tsx` (estados de 03 §3.2, pt-BR) |
| Remover | `components/common/Pagination.tsx` (substituído por "Carregar mais") |

`app/layout.tsx`:

```tsx
import type { Metadata } from "next";
import type { ReactNode } from "react";
import "@fontsource/poppins";
import "@fontsource/poppins/600.css";
import "@fontsource/poppins/700.css";

import { Layout } from "@/components/common/Layout";
import { PREFERENCES_BOOT_SCRIPT } from "@/components/a11y/PreferencesProvider";
import "@/styles/globals.css";
import { Providers } from "./providers";

export const metadata: Metadata = {
  title: { default: "Acessibiliza", template: "%s – Acessibiliza" },
  description: "Mapa colaborativo de acessibilidade em estabelecimentos de Tubarão (SC).",
};

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="pt-BR" suppressHydrationWarning>
      <head>
        {/* Aplica as preferências antes da hidratação, sem "flash" (04 §4.9) */}
        <script dangerouslySetInnerHTML={{ __html: PREFERENCES_BOOT_SCRIPT }} />
      </head>
      <body>
        <Providers>
          <Layout>{children}</Layout>
        </Providers>
      </body>
    </html>
  );
}
```

Não usar `viewport` com `maximumScale`/`userScalable` (05 §5.5).

`app/providers.tsx`: envolver com `PreferencesProvider`, `AnnouncerProvider` e `ToastProvider` (`components/ui/toast.tsx`). Manter `staleTime` global e definir 60 s nos hooks de estabelecimentos.

`components/common/Layout.tsx`:

```tsx
import type { ReactNode } from "react";
import { SkipLink } from "./SkipLink";
import { Header } from "./Header";
import { Footer } from "./Footer";
import { MobileNav } from "./MobileNav";
import { RouteAnnouncer } from "./RouteAnnouncer";
import { Toaster } from "./Toaster";
import { VLibras } from "./VLibras";

export const Layout = ({ children }: { children: ReactNode }) => (
  <>
    <SkipLink />
    <Header />
    <main id="conteudo" tabIndex={-1} className="flex-1 outline-none">{children}</main>
    <Footer />
    <MobileNav />
    <RouteAnnouncer />
    <Toaster />
    <VLibras />
  </>
);
```

`SkipLink`: primeiro elemento focável, `<a href="#conteudo" className="sr-only focus:not-sr-only …">Pular para o conteúdo principal</a>`.

`Header.tsx` (corrige E10):

```tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import { CirclePlus, Type, UserRound } from "lucide-react";

import { useSession, signInWithKeycloak, signOutWithKeycloak } from "@/hooks/useAuth";
import { usePreferences } from "@/components/a11y/PreferencesProvider";

const NAV = [
  { href: "/", label: "Explorar" },
  { href: "/painel", label: "Painel" },
  { href: "/sobre", label: "Sobre" },
];

export const Header = () => {
  const { data: session, isPending } = useSession();
  const pathname = usePathname();
  const { openDialog } = usePreferences();

  return (
    <header className="sticky top-0 z-30 border-b bg-white">
      <div className="mx-auto flex max-w-7xl items-center gap-4 px-4 py-2">
        <Link href="/" className="text-xl font-bold no-underline">Acessibiliza</Link>

        <nav aria-label="Principal" id="menu-principal" className="hidden md:block">
          <ul className="flex gap-1">
            {NAV.map((item) => (
              <li key={item.href}>
                <Link href={item.href} aria-current={pathname === item.href ? "page" : undefined}
                      className="inline-flex min-h-11 items-center px-3">
                  {item.label}
                </Link>
              </li>
            ))}
          </ul>
        </nav>

        <div className="ml-auto flex items-center gap-2">
          <Link href="/avaliar" className="btn-primary hidden md:inline-flex">
            <CirclePlus aria-hidden="true" className="size-5" /> Avaliar um local
          </Link>
          <button type="button" onClick={openDialog} className="btn-ghost min-h-11 min-w-11"
                  aria-haspopup="dialog" aria-keyshortcuts="Alt+Shift+A">
            <Type aria-hidden="true" className="size-5" />
            <span className="sr-only md:not-sr-only">Acessibilidade</span>
          </button>
          {isPending ? null : session ? (
            <AccountMenu name={session.user.name} onSignOut={() => signOutWithKeycloak(`${window.location.origin}/`)} />
          ) : (
            <button type="button" className="btn-ghost min-h-11" onClick={() => signInWithKeycloak(window.location.href)}>
              <UserRound aria-hidden="true" className="size-5" /> Entrar
            </button>
          )}
        </div>
      </div>
    </header>
  );
};
```

`AccountMenu`: disclosure (`<button aria-expanded>` + lista) com "Olá, {nome}" e o botão "Sair". **Não** usar `role="menuitem"` fora de `role="menu"`.

`Footer.tsx`: IFSC Câmpus Tubarão; links para `/sobre`, `/acessibilidade`, código-fonte (com "(abre em nova aba)") e contato para relatar barreiras do site; `id="rodape"` para o atalho `Alt+4`.

`MobileNav.tsx`: em `< md`, barra inferior fixa `<nav aria-label="Principal (celular)">` com Explorar, Avaliar, Painel, Conta (alvos ≥ 44 px). Reservar `padding-bottom` no `<body>` para não cobrir conteúdo, e conferir a posição do botão do VLibras.

`RouteAnnouncer.tsx`: observa `usePathname()`; na troca de rota, foca o `<h1>` da página (ou `#conteudo`) e anuncia `document.title` pela região *live*. Não executar na primeira renderização.

`AnnouncerProvider` (05 §5.4.9):

```tsx
"use client";
import { createContext, useCallback, useContext, useRef, useState, type ReactNode } from "react";

type Politeness = "polite" | "assertive";
const Ctx = createContext<(msg: string, p?: Politeness) => void>(() => {});
export const useAnnounce = () => useContext(Ctx);

export function AnnouncerProvider({ children }: { children: ReactNode }) {
  const [polite, setPolite] = useState("");
  const [assertive, setAssertive] = useState("");
  const timer = useRef<ReturnType<typeof setTimeout>>(undefined);

  const announce = useCallback((msg: string, p: Politeness = "polite") => {
    const set = p === "polite" ? setPolite : setAssertive;
    set("");                              // limpa para repetir a mesma mensagem
    clearTimeout(timer.current);
    timer.current = setTimeout(() => set(msg), 50);
  }, []);

  return (
    <Ctx.Provider value={announce}>
      {children}
      {/* As regiões existem no DOM antes de receber texto */}
      <div aria-live="polite" aria-atomic="true" className="sr-only">{polite}</div>
      <div aria-live="assertive" aria-atomic="true" className="sr-only">{assertive}</div>
    </Ctx.Provider>
  );
}
```

`Toaster.tsx`: monta o viewport de `ui/toast.tsx` com `role="status"` (sucesso) e `role="alert"` (erro), duração ≥ 6 s, pausa com hover/foco e botão de fechar com rótulo "Fechar notificação". Confirmar as props de `@base-ui/react/toast`.

**Aceite:** um `<h1>` por página; landmarks `header`/`nav`/`main`/`footer`; skip link é o primeiro Tab; axe sem violações em `/`.

---

### Fase 3: T1 Explorar (`/`)

**Arquivos:**

| Ação | Arquivo |
|---|---|
| Reescrever | `app/page.tsx` |
| Criar | `components/filters/{FilterPanel,CriterionFilterFieldset,ActiveFilterChips,SearchField}.tsx` |
| Criar | `components/establishment/{EstablishmentList,EstablishmentCard,StatusBadge,CriterionBadge,StatusLegend}.tsx` |
| Criar | `components/map/{MapView,EstablishmentMarker,MarkerCluster,MapSummaryCard,LocateMeButton}.tsx` |
| Remover | `components/home/*` (substituídos pelos acima) |

#### 3.1 Página

```tsx
// app/page.tsx (esqueleto)
"use client";

export default function ExplorePage() {
  const sp = useSearchParams();
  const router = useRouter();
  const filters = useMemo(() => filtersFromSearchParams(sp), [sp]);
  const view = useViewPreference(sp); // "mapa" | "lista": ?view= > localStorage > "Preferir lista" > "mapa"

  const list = useEstablishments(filters);        // lista paginada ("Carregar mais")
  const map = useAllEstablishments(filters);      // todos os pinos (só quando view === "mapa")
  const announce = useAnnounce();

  const apply = (next: ExploreFilters) =>
    router.replace(`?${filtersToSearchParams(next, new URLSearchParams(sp))}`, { scroll: false });

  useEffect(() => {
    if (list.data) announce(`${plural(list.data.total, "local encontrado", "locais encontrados")}`);
  }, [list.data?.total]); // foco não se move (04 §4.2)

  return (
    <div className="md:grid md:grid-cols-[18rem_1fr]">
      <aside aria-label="Filtros" className="hidden md:block"><FilterPanel filters={filters} onApply={apply} /></aside>
      <section aria-labelledby="titulo-explorar">
        <h1 id="titulo-explorar" className="sr-only">Explorar locais</h1>
        <SearchField … />                {/* id="busca" para Alt+3 e "/" */}
        <MobileFiltersButton count={activeFilterCount(filters)} … />  {/* "Filtros (2 ativos)" + Sheet */}
        <ViewToggle value={view} … />    {/* dois botões com aria-pressed */}
        <p aria-hidden="true">{total} locais encontrados</p>   {/* o anúncio vai pela região global */}
        <ActiveFilterChips filters={filters} onRemove={…} />
        <StatusLegend />
        {view === "lista" ? <EstablishmentList query={list} /> : <MapView query={map} />}
      </section>
    </div>
  );
}
```

Envolver o conteúdo que usa `useSearchParams` em `<Suspense>` (exigência do App Router no build).

#### 3.2 Filtros (corrige E13–E15)

```tsx
// components/filters/CriterionFilterFieldset.tsx
import type { Criterion } from "@/lib/criteria";
import type { Band } from "@/lib/filters";

const OPTIONS: { value: Band | ""; label: string }[] = [
  { value: "", label: "Qualquer" },
  { value: "bom", label: "Bom (7 ou mais)" },
  { value: "medio", label: "Médio (5 a 6,9)" },
  { value: "ruim", label: "Ruim (menos de 5)" },
];

export function CriterionFilterFieldset({ criterion, value, onChange }: {
  criterion: Criterion; value: Band | undefined; onChange: (v: Band | undefined) => void;
}) {
  const name = `filtro-${criterion.value}`;
  return (
    <fieldset className="rounded-md border border-gray-500 p-3">
      <legend className="flex items-center gap-2 px-1 font-semibold">
        <criterion.Icon aria-hidden="true" className="size-5" /> {criterion.label}
      </legend>
      <p id={`${name}-desc`} className="sr-only">{criterion.description}</p>
      <div className="grid grid-cols-2 gap-1" aria-describedby={`${name}-desc`}>
        {OPTIONS.map((o) => (
          <label key={o.value} className="flex min-h-11 cursor-pointer items-center gap-2">
            <input type="radio" name={name} value={o.value} checked={(value ?? "") === o.value}
                   onChange={() => onChange(o.value || undefined)} className="size-5" />
            {o.label}
          </label>
        ))}
      </div>
    </fieldset>
  );
}
```

`FilterPanel`: campos "Nome" e "Endereço" com `<label htmlFor>`; um `CriterionFilterFieldset` por item de `CRITERIA`; botões "Aplicar filtros" e "Limpar filtros" sempre visíveis. Aplicar **não** move o foco. No mobile, o mesmo painel dentro de `Sheet` (foco preso, `Esc` fecha e devolve o foco).

`ActiveFilterChips`: `<ul>` de `<button>` "Cadeira de rodas: Bom ✕" com `aria-label="Remover filtro Cadeira de rodas: Bom"`. Ao remover o último chip, mover o foco para o título da lista ou para o campo de busca (o botão removido some).

`SearchField`: `<input type="search" id="busca">` com `<label>` visível "Buscar local"; seletor "Buscar por: Nome / Endereço"; *debounce* de 400 ms; Enter envia na hora.

#### 3.3 Lista

```tsx
// components/establishment/EstablishmentCard.tsx (trecho)
<li>
  <article aria-labelledby={`local-${id}`} className="rounded-lg border p-4">
    <h3 id={`local-${id}`}>
      <Link href={`/locais/${id}`}>{establishment.name}</Link>
    </h3>
    {establishment.address && <p>{establishment.address}</p>}
    <StatusBadge status={statusFromAverage(overall.average)} prefix="Situação geral:" />
    <ul aria-label="Critérios" className="flex flex-wrap gap-1">
      {CRITERIA.map((c) => <CriterionBadge key={c.value} criterion={c} data={summary?.[c.value]} />)}
    </ul>
    <p>{plural(evaluationCount(summary), "avaliação", "avaliações")}{distance && ` · ${formatAverage(distance)} km`}</p>
  </article>
</li>
```

- `id` = último segmento do `@id` (`establishment["@id"].split("/").pop()`).
- `StatusBadge` sempre com **ícone + texto + cor** (03 §3.3); ícones com `aria-hidden`.
- "Carregar mais" chama `fetchNextPage()`, anuncia "Mais 30 locais carregados" e **não** move o foco.
- Estados: skeleton com `aria-busy="true"` + "Carregando…" `sr-only`; vazio ("Nenhum local encontrado com esses filtros." + "Limpar filtros" + "Não achou o local? Avalie-o agora" → `/avaliar`); erro ("Não foi possível carregar os locais." + "Tentar novamente" → `refetch()`).

#### 3.4 Mapa (corrige E3–E7)

```tsx
// components/map/MapView.tsx (trecho) — confirmar as APIs em node_modules/@vis.gl/react-google-maps
<APIProvider apiKey={process.env.NEXT_PUBLIC_GOOGLE_MAPS_API_KEY ?? ""} language="pt-BR" region="BR">
  <div role="region" aria-label="Mapa de locais avaliados" className="relative h-[60vh] md:h-[calc(100dvh-10rem)]">
    <Map
      mapId={process.env.NEXT_PUBLIC_GOOGLE_MAPS_MAP_ID}   /* obrigatório para AdvancedMarker */
      defaultCenter={TUBARAO_CENTER}                       /* câmera NÃO controlada */
      defaultZoom={DEFAULT_ZOOM}
      gestureHandling="cooperative"                        /* "greedy" só em tela cheia */
      zoomControl
      streetViewControl={false}
      mapTypeControl={false}
    >
      <MarkerCluster establishments={establishments} onSelect={setSelected} />
      {selected && <MapSummaryCard establishment={selected} onClose={closeAndRestoreFocus} />}
    </Map>
    <LocateMeButton />  {/* pede geolocalização só no clique; panTo(); erro: "Não foi possível obter sua localização. O mapa continua centrado em Tubarão." */}
    {isLoading && <div role="status" className="absolute inset-0 grid place-items-center bg-white/60">Carregando locais…</div>}
  </div>
  <p className="sr-only">
    O mapa mostra {establishments.length} locais. A mesma informação está na <a href="?view=lista">visualização em lista</a>.
  </p>
</APIProvider>
```

`EstablishmentMarker`:

```tsx
const status = statusFromAverage(overallAverage(est.evaluationsSummary).average);
const { label, glyph } = STATUS_META[status];

<AdvancedMarker position={coords} title={`${est.name}, situação geral: ${label}`} onClick={() => onSelect(est)}>
  {/* Se o marcador não ficar focável no navegador, usar um <button aria-label=…> aqui dentro (05 §5.4.3) */}
  <Pin background={PIN_COLORS[status].bg} borderColor={PIN_COLORS[status].border} glyph={glyph} glyphColor="#fff" />
</AdvancedMarker>
```

- Cores do pino a partir dos tokens de status (texto 800 como fundo, para contraste com o glifo branco).
- *Clustering* com `@googlemaps/markerclusterer` (nova dependência) acima de ~50 pinos; nome acessível do cluster: "12 locais nesta área. Ampliar".
- `MapSummaryCard`: `InfoWindow` no desktop e `Drawer` (bottom sheet) no mobile; nome, endereço, situação geral, 3 critérios, "Ver detalhes de {nome}" e "Avaliar {nome}". `Esc` fecha e devolve o foco ao pino.
- Remover `MAP_STYLES` (POIs visíveis; com `mapId`, o estilo vem do Cloud Console).
- Adicionar `NEXT_PUBLIC_GOOGLE_MAPS_MAP_ID` em `compose*.yaml`, `helm/` e na tabela 06 §6.6.

**Aceite:** cenários E2E 1 e 2 de 06 §6.8; filtros sobrevivem ao recarregar e ao "voltar"; mapa arrastável; ≥ 31 locais aparecem no mapa.

---

### Fase 4: T2 Detalhe do local (`/locais/[id]`)

**Arquivos:** criar `app/locais/[id]/page.tsx` (Server Component), `app/locais/[id]/not-found.tsx`, `components/establishment/{EstablishmentHeader,SummaryTable,ShareButton}.tsx`, `components/evaluation/{EvaluationList,EvaluationItem}.tsx`, `hooks/useEstablishment.ts`.

```tsx
// app/locais/[id]/page.tsx
import type { Metadata } from "next";
import { notFound } from "next/navigation";
import { fetchApi, FetchError } from "@/utils/dataAccess";
import type { Establishment } from "@/types/Establishment";
import { EstablishmentDetail } from "@/components/establishment/EstablishmentDetail";

type Props = { params: Promise<{ id: string }> };   // params é Promise no Next 15+ (confirmar no Next 16)

async function load(id: string) {
  try {
    return await fetchApi<Establishment>(`/establishments/${encodeURIComponent(id)}`, { cache: "no-store" });
  } catch (e) {
    if (e instanceof FetchError && e.status === 404) notFound();
    throw e;
  }
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { id } = await params;
  const res = await load(id);
  return { title: `${res?.data.name} – Acessibilidade` }; // template → "… | Acessibiliza" conforme 03 T2
}

export default async function Page({ params }: Props) {
  const { id } = await params;
  const res = await load(id);
  return <EstablishmentDetail id={id} initialData={res!.data} hubURL={res!.hubURL} />;
}
```

> A doc pede o título "Farol Shopping – Acessibilidade | Acessibiliza", mas o template global usa "–". Escolher um formato e alinhar 03 T2 com 05 §5.4.1.

`EstablishmentDetail` (Client):

- "‹ Voltar para resultados" com `router.back()` quando o histórico vier de `/`, senão link para `/`; trilha `nav aria-label="Trilha"`.
- `<h1>` com o nome (recebe o foco na chegada, junto com o `RouteAnnouncer`); `<address>`; `tel:`; site com `target="_blank" rel="noopener"` + "(abre em nova aba)"; "Como chegar": `https://www.google.com/maps/search/?api=1&query=<nome>&query_place_id=<googlePlaceId>`.
- `ShareButton`: `navigator.share` ou `navigator.clipboard.writeText` + anúncio "Link copiado".
- "Avaliar este local" → `/locais/[id]/avaliar`.

`SummaryTable`:

```tsx
<table>
  <caption>Resumo de acessibilidade, {plural(n, "avaliação", "avaliações")}</caption>
  <thead><tr><th scope="col">Critério</th><th scope="col">Média</th><th scope="col">Situação</th><th scope="col">Avaliações</th></tr></thead>
  <tbody>
    {CRITERIA.map((c) => {
      const s = summary?.[c.value];
      return (
        <tr key={c.value}>
          <th scope="row"><c.Icon aria-hidden="true" /> {c.label}</th>
          <td>{s ? `${formatAverage(s.average)} de 10` : "—"}</td>
          <td><StatusBadge status={statusFromAverage(s?.average)} /></td>
          <td>{s?.count ?? 0}</td>
        </tr>
      );
    })}
  </tbody>
</table>
<p>Média geral ({overall.evaluatedCriteria} de 6 critérios avaliados): {…}</p>
```

Critérios sem dados aparecem como "Sem dados" (não ocultar). Barra visual por critério com `aria-hidden`.

`EvaluationList` / `EvaluationItem`:

- Ordenação no cliente: "Mais úteis" (`netVotes` desc). "Mais recentes" fica oculta até a lacuna L2.
- Cada item: `<article id="avaliacao-<id>">`, `<h3 className="sr-only">Avaliação 1 de 12</h3>`, notas em `<dl>` (`<dt>Cadeira de rodas</dt><dd>9 de 10</dd>`), comentário, `VoteButtons` (Fase 7).
- `netVotes ≤ -3`: dentro de `<details>` com `<summary>` "1 avaliação oculta por ser marcada como pouco útil" e a frase "Esta avaliação foi marcada como pouco útil pela comunidade e não entra no resumo."
- Sem avaliações: "Ainda não há avaliações. Seja a primeira pessoa a avaliar!" + botão.
- Seção "Localização": mapa pequeno (`gestureHandling="cooperative"`) + endereço em texto.

**Aceite:** tabela semântica validada pelo axe; 404 mostra "Local não encontrado" + link para o mapa; foco no `<h1>` ao chegar.

---

### Fase 5: preferências, VLibras, atalhos, T5, T6 e T10

#### 5.1 Preferências (RF16, 05 §5.3.1, 04 §4.9)

`components/a11y/PreferencesProvider.tsx`:

```tsx
"use client";

export interface Preferences {
  fontScale: 100 | 125 | 150 | 200;
  contrast: "normal" | "alto";
  theme: "sistema" | "claro" | "escuro";
  motion: "normal" | "reduzido";
  dyslexiaFont: boolean;
  wideSpacing: boolean;
  underlineLinks: boolean;
  preferList: boolean;
  slashShortcut: boolean;
}

export const STORAGE_KEY = "acessibiliza:preferencias";
export const DEFAULT_PREFERENCES: Preferences = { fontScale: 100, contrast: "normal", theme: "sistema",
  motion: "normal", dyslexiaFont: false, wideSpacing: false, underlineLinks: false, preferList: false, slashShortcut: true };

export function applyPreferences(p: Preferences, el = document.documentElement) {
  el.style.setProperty("--base", `${p.fontScale}%`);
  el.dataset.contrast = p.contrast;
  el.dataset.motion = p.motion;
  el.dataset.font = p.dyslexiaFont ? "dislexia" : "padrao";
  el.dataset.spacing = p.wideSpacing ? "ampliado" : "normal";
  el.dataset.links = p.underlineLinks ? "sublinhado" : "padrao";
  if (p.theme === "sistema") delete el.dataset.theme; else el.dataset.theme = p.theme === "escuro" ? "dark" : "light";
}

/** Script inline do <head>: mesma lógica, sem React, envolta em try/catch. */
export const PREFERENCES_BOOT_SCRIPT = `try{var p=JSON.parse(localStorage.getItem("${STORAGE_KEY}")||"{}");var e=document.documentElement;
if(p.fontScale)e.style.setProperty("--base",p.fontScale+"%");if(p.contrast)e.dataset.contrast=p.contrast;
if(p.motion)e.dataset.motion=p.motion;if(p.dyslexiaFont)e.dataset.font="dislexia";if(p.wideSpacing)e.dataset.spacing="ampliado";
if(p.underlineLinks)e.dataset.links="sublinhado";if(p.theme&&p.theme!=="sistema")e.dataset.theme=p.theme==="escuro"?"dark":"light";}catch(_){}`;
```

O provider expõe `{ preferences, update, reset, openDialog }`, salva em `localStorage` (com `try/catch`), aplica na hora e anuncia "Preferência aplicada".

`components/a11y/PreferencesDialog.tsx`: `Dialog` de `components/ui/dialog.tsx` (foco preso, `Esc`, foco volta ao botão "Aa"); cada opção como `fieldset` de rádios ou `checkbox` nativo; botão "Restaurar padrão". O mesmo formulário é reutilizado em `/acessibilidade`.

#### 5.2 Atalhos eMAG (05 §5.3.3)

`hooks/useKeyboardShortcuts.ts`, montado no `Layout`:

| Tecla | Ação |
|---|---|
| `Alt+1` | foca `#conteudo` |
| `Alt+2` | foca o primeiro link de `#menu-principal` |
| `Alt+3` | foca `#busca` (se não existir na página, vai para `/` e foca) |
| `Alt+4` | foca `#rodape` |
| `Alt+Shift+A` | abre as preferências |
| `/` | foca `#busca`, só fora de `input`/`textarea`/`[contenteditable]` e se `slashShortcut` estiver ligado |

Usar `event.code` (`Digit1`…) para funcionar em teclados ABNT2.

#### 5.3 VLibras (05 §5.3.2)

```tsx
// components/common/VLibras.tsx
"use client";
import Script from "next/script";

export function VLibras() {
  return (
    <>
      {/* "vw" não tem hífen, então o TypeScript só o aceita via spread */}
      <div {...{ vw: "" }} className="enabled">
        <div vw-access-button="" className="active" />
        <div vw-plugin-wrapper=""><div className="vw-plugin-top-wrapper" /></div>
      </div>
      <Script src="https://vlibras.gov.br/app/vlibras-plugin.js" strategy="lazyOnload"
              onLoad={() => new (window as any).VLibras.Widget("https://vlibras.gov.br/app")} />
    </>
  );
}
```

Conferir que o botão não cobre "Minha localização" nem a barra inferior. Liberar `vlibras.gov.br` em uma eventual CSP.

#### 5.4 Páginas estáticas

| Arquivo | Conteúdo |
|---|---|
| `app/sobre/page.tsx` | 7 seções de 03 T5; os 6 critérios gerados a partir de `CRITERIA` e `RATING_ANCHORS` (exemplos de nota 0, 5 e 10); regras de votos e moderação; vídeo em Libras com legenda quando houver. `metadata.title = "Sobre"` |
| `app/acessibilidade/page.tsx` | Declaração (normas, situação, limitações conhecidas do mapa, data da última avaliação, canal de contato), tabela de atalhos e o formulário de preferências |
| `app/not-found.tsx` | `<h1>Página não encontrada</h1>` + links para Explorar e Sobre |
| `app/error.tsx` | Client Component: "Algo deu errado." + "Tentar novamente" (`reset()`) |
| `components/common/OfflineBanner.tsx` | Ouve `online`/`offline`; `role="status"`: "Você está sem conexão. Os dados podem estar desatualizados." |
| Sessão expirada | Interceptar `FetchError.status === 401` nas mutações: toast com "Entrar novamente" (sem perder a tela) |

**Aceite:** cenário 6 de 06 §6.8 (axe com e sem alto contraste); preferências persistem após recarregar sem "flash".

---

### Fase 6: T3 Nova avaliação (`/avaliar`)

**Arquivos:** `app/avaliar/page.tsx`, `app/locais/[id]/avaliar/page.tsx`, `components/evaluation/EvaluationWizard/{index,StepPlace,StepCriteria,StepComment,StepReview,StepIndicator,ErrorSummary}.tsx`, `components/evaluation/{RatingRadioGroup,PlacesCombobox,PhotoUploader}.tsx`, `hooks/{useCreateEvaluation,useEvaluationDraft}.ts`, `lib/evaluation.ts`.

#### 6.1 Estado, rascunho e schema

```ts
// lib/evaluation.ts
import * as yup from "yup";
import type { CriterionValue, CreateEvaluationInput } from "@/types/Evaluation";

export type RatingChoice = number | "nao-sei";
export interface EvaluationDraft {
  step: 1 | 2 | 3 | 4;
  placeId?: string;
  placeLabel?: string;
  ratings: Record<CriterionValue, RatingChoice>;
  comment: string;
}

export const DRAFT_KEY = "acessibiliza:rascunho-avaliacao";

export const schema = yup.object({
  placeId: yup.string().required("Escolha o local que você quer avaliar."),
  ratings: yup.object().test("ao-menos-um", "Avalie pelo menos um critério.",
    (r) => Object.values(r ?? {}).some((v) => v !== "nao-sei")),
  comment: yup.string().max(1000, "O comentário pode ter no máximo 1000 caracteres."),
});

export function buildBody(draft: EvaluationDraft): CreateEvaluationInput {
  return {
    establishmentGooglePlaceId: draft.placeId!,
    comment: draft.comment?.trim() || undefined,
    ratings: Object.entries(draft.ratings)
      .filter(([, v]) => v !== "nao-sei")
      .map(([criterion, v]) => ({ criterion: criterion as CriterionValue, rating: Number(v) })),
  };
}

/** Mapeia propertyPath do servidor para campos do formulário (02 §2.4). */
export function mapServerErrors(fields: Record<string, string>, sent: CreateEvaluationInput) {
  const out: { step: 1 | 2 | 3; field: string; message: string }[] = [];
  for (const [path, message] of Object.entries(fields)) {
    const m = path.match(/^ratings\[(\d+)\]/);
    if (m) out.push({ step: 2, field: `ratings.${sent.ratings[Number(m[1])]?.criterion}`, message });
    else if (path === "comment") out.push({ step: 3, field: "comment", message });
    else out.push({ step: 1, field: "placeId", message });
  }
  return out;
}
```

`useEvaluationDraft`: `react-hook-form` com `yupResolver` (adicionar `@hookform/resolvers`), salvando o rascunho em `sessionStorage` a cada troca de passo e limpando após o sucesso. O passo atual também vai na URL (`?etapa=4`) para o retorno do login.

#### 6.2 Passos

| Passo | Implementação |
|---|---|
| Indicador | `<ol>` com "Passo 2 de 4: Critérios" e `aria-current="step"`; ao trocar de passo, foco no `<h2>` do passo e `document.title` atualizado |
| 1. Local | `PlacesCombobox` (abaixo). Ao selecionar, consultar `GET /establishments?name=<nome>` e mostrar "já tem N avaliações — ver". Erro 400 volta para cá: "Não conseguimos identificar este local. Tente buscá-lo novamente." |
| 2. Critérios | 6 × `RatingRadioGroup`: `<fieldset>` com `<legend>` (ícone + rótulo + botão (?) em disclosure), rádio "Não sei / não se aplica" **marcado por padrão**, rádios 0–10 com alvo ≥ 44×44 px, âncora textual do valor selecionado ligada por `aria-describedby`. Sem *slider* |
| 3. Comentário | `<textarea>` com `maxLength` 1000 e contador "restam N caracteres", anunciado (`aria-live="polite"`) só abaixo de 100. `PhotoUploader` atrás de *feature flag* desligada (lacuna L1) |
| 4. Revisão | Resumo com links "Editar" que levam ao passo e focam o campo; checkbox "Declaro que esta avaliação reflete minha experiência"; botão "Enviar avaliação" com estado "Enviando…" via `aria-disabled` |
| Resumo de erros | `ErrorSummary` no topo com `role="alert"`, links para cada campo e foco movido para ele; cada campo com `aria-invalid` + `aria-describedby` |

`PlacesCombobox` (05 §5.4.4), padrão APG combobox + listbox:

```tsx
// trecho — confirmar a API Places (New) em @vis.gl/react-google-maps e @types/google.maps
const places = useMapsLibrary("places");
const [token] = useState(() => places && new places.AutocompleteSessionToken());

const { suggestions } = await places.AutocompleteSuggestion.fetchAutocompleteSuggestions({
  input,
  sessionToken: token,
  includedRegionCodes: ["br"],
  locationBias: { center: TUBARAO_CENTER, radius: 15000 },
  language: "pt-BR",
});
// suggestion.placePrediction.placeId → draft.placeId
```

Markup: `<input role="combobox" aria-expanded aria-controls="sugestoes" aria-activedescendant aria-autocomplete="list">` + `<ul id="sugestoes" role="listbox">` com `<li role="option" aria-selected>`; setas, Enter, Esc; anúncio "5 sugestões disponíveis. Use as setas para navegar."; texto "Powered by Google" visível. Botão "Usar locais perto de mim" pede geolocalização só no clique.

#### 6.3 Envio e erros

```ts
// hooks/useCreateEvaluation.ts
export function useCreateEvaluation() {
  const qc = useQueryClient();
  const { accessToken } = useAccessToken();
  return useMutation({
    mutationFn: async (body: CreateEvaluationInput) =>
      (await fetchApi<Evaluation>("/evaluations", { method: "POST", body: JSON.stringify(body) }, accessToken))!.data,
    onSuccess: (evaluation) => {
      qc.invalidateQueries({ queryKey: ["establishments"] });
      const est = typeof evaluation.establishment === "string" ? evaluation.establishment : evaluation.establishment?.["@id"];
      if (est) qc.invalidateQueries({ queryKey: queryKeys.establishment(est.split("/").pop()!) });
    },
  });
}
```

| Resposta | Ação |
|---|---|
| 201 | Limpar rascunho; `router.push("/locais/<id>")`; toast "Obrigado! Sua avaliação foi publicada."; foco no `<h1>` |
| 400 | Voltar ao passo 1 com a mensagem |
| 401 | Salvar rascunho; `/login?callbackURL=/avaliar?etapa=4`; ao voltar, restaurar o passo 4 |
| 422 `comment` | Voltar ao passo 3, mensagem da moderação no campo e no resumo |
| 422 `ratings[i]` | Voltar ao passo 2, erro no critério `sent.ratings[i].criterion` |
| 5xx / rede | Manter os dados; mensagem com "Tentar novamente" |

- Anônimo vê "Para avaliar, entre com sua conta" + "Entrar", mas pode preencher (não bloquear).
- `/locais/[id]/avaliar`: carrega o local, preenche `placeId` com `googlePlaceId` e começa no passo 2. Sem `googlePlaceId`: aviso de que não é possível avaliar pela API atual (L13).
- Sair com dados preenchidos: `beforeunload` + diálogo "Descartar avaliação?" nos links internos.

**Aceite:** cenários E2E 3 e 4 de 06 §6.8.

---

### Fase 7: votos e tempo real

#### 7.1 Votos (RF09, 04 §4.5)

`components/evaluation/VoteButtons.tsx`:

- Dois `<button aria-pressed>`: "👍 Sim (5)" com `aria-label="Marcar avaliação como útil"` e "👎 Não (1)" com `aria-label="Marcar avaliação como não útil"`; saldo visível.
- Anônimo: `signInWithKeycloak(\`${location.origin}/locais/${id}#avaliacao-${evId}\`)`.
- Clicar no voto já ativo: não envia; anuncia "Você já marcou esta avaliação como útil" (não há DELETE, lacuna L5). Texto de apoio: "Você pode trocar seu voto, mas não removê-lo."

```ts
// hooks/useVote.ts — otimista com rollback (06 §6.3)
export function useVote(establishmentId: string) {
  const qc = useQueryClient();
  const { accessToken } = useAccessToken();
  const announce = useAnnounce();
  const key = queryKeys.establishment(establishmentId);

  return useMutation({
    mutationFn: ({ evaluation, value }: { evaluation: string; value: 1 | -1 }) =>
      fetchApi("/evaluation_votes", { method: "POST", body: JSON.stringify({ evaluation, value }) }, accessToken),
    onMutate: async ({ evaluation, value }) => {
      await qc.cancelQueries({ queryKey: key });
      const previous = qc.getQueryData<Establishment>(key);
      qc.setQueryData<Establishment>(key, (old) => old && applyOptimisticVote(old, evaluation, value, /* voto anterior */));
      announce("Voto registrado");
      return { previous };
    },
    onError: (_e, _v, ctx) => {
      qc.setQueryData(key, ctx?.previous);
      announce("Não foi possível registrar o voto", "assertive");
    },
    onSettled: () => qc.invalidateQueries({ queryKey: key }),
  });
}
```

`useMyVote(evaluationIri)`: `GET /evaluation_votes?evaluation=<iri>&user=<iri>` só com sessão; o IRI do usuário vem do `sub` do token (`/users/<id>`; confirmar no backend como o `User` é identificado). Para muitas avaliações, buscar por `evaluation` em lote.

#### 7.2 Mercure (RF13, corrige E25)

Substituir `utils/mercure.ts` por um hook que só invalida:

```ts
// hooks/useMercure.ts
"use client";
import { useEffect, useRef } from "react";

export function useMercure(hubURL: string | null | undefined, topic: string | undefined, onEvent: () => void) {
  const handler = useRef(onEvent);
  handler.current = onEvent;

  useEffect(() => {
    if (!hubURL || !topic) return;
    const url = new URL(hubURL, window.origin);
    url.searchParams.append("topic", new URL(topic, window.origin).toString());
    const es = new EventSource(url.toString());
    es.addEventListener("message", () => handler.current());
    return () => es.close();
  }, [hubURL, topic]);
}
```

No detalhe: ao receber evento, **não** recarregar sozinho; mostrar "Há novas avaliações. [Atualizar]" e anunciar via região *live*; o botão chama `invalidateQueries(["establishment", id])`. Nenhum arquivo importa `utils/mercure.ts` hoje; ele pode ser removido junto com o `console.error`.

**Aceite:** cenário E2E 5 de 06 §6.8.

---

### Fase 8: T4 Painel (`/painel`)

**Arquivos:** `app/painel/page.tsx`, `components/dashboard/{KpiCard,CriterionAverageChart,StatusDistributionChart,RankingTable,CsvExportButton,ChartFigure}.tsx`, `lib/stats.ts`, `lib/csv.ts`.

- Dados: `useAllEstablishments({ ..., end })` (agregação no cliente até a lacuna L4).
- `lib/stats.ts`: total de locais avaliados; total de avaliações (`totalItems` de `GET /evaluations` ou soma de `evaluationCount`); média por critério (média ponderada por `count`); distribuição Bom/Médio/Ruim por critério; critério com pior média; ranking por média geral com o pior critério.
- `ChartFigure`: `<figure>` com `<h2>`, frase-resumo ("Piso tátil tem a pior média: 3,1"), SVG `role="img" aria-labelledby="<titulo> <resumo>"`, rótulos diretos, hachuras (`<pattern>`) além de cor, e `<details><summary>Ver como tabela</summary><table>…</table></details>`. Seguir o skill/guia de dataviz para a paleta.
- `RankingTable`: `<th aria-sort>` com `<button>` dentro; contêiner com `role="region" aria-label tabIndex={0}` para rolagem horizontal.
- CSV:

```ts
// lib/csv.ts
export function toCsv(rows: Establishment[]): string {
  const header = ["nome", "endereço", "latitude", "longitude",
    ...CRITERIA.flatMap((c) => [`${c.value}_media`, `${c.value}_n`])];
  const esc = (v: unknown) => `"${String(v ?? "").replace(/"/g, '""')}"`;
  const lines = rows.map((e) => {
    const p = parsePoint(e.location);
    return [e.name, e.address, p?.lat, p?.lng,
      ...CRITERIA.flatMap((c) => {
        const s = e.evaluationsSummary?.[c.value];
        return [s ? formatAverage(s.average) : "", s?.count ?? 0];
      })].map(esc).join(";");
  });
  return "﻿" + [header.map(esc).join(";"), ...lines].join("\r\n"); // BOM + ";" para Excel pt-BR
}
```

- "Baixar dados (CSV)" gera o `Blob`, dispara o download e anuncia "Download iniciado". Mostrar "Dados atualizados em …" (`formatDate`) e a nota metodológica (faixas, dados colaborativos, regra de saldo ≤ -3).

---

### Fase 9: Administração (`/admin`)

| Ação | Arquivo | Alteração |
|---|---|---|
| Remover | `components/admin/book/*`, `components/admin/review/*`, `components/admin/Logo.tsx` (se for do API Platform), `layout/HydraLogo.tsx`, `layout/OpenApiLogo.tsx` | Recursos do demo |
| Criar | `components/admin/establishment/{EstablishmentsList,EstablishmentsEdit}.tsx` | Lista pelo endpoint público (`/establishments`, lacuna L8); edição de `name`, `address`, `phoneNumber`, `website` com PATCH em `/admin/establishments/{id}`; remover com diálogo de confirmação |
| Criar | `components/admin/evaluation/{EvaluationsList,EvaluationsEdit}.tsx` | Lista por `/evaluations` ordenável por `netVotes`; DELETE em `/admin/evaluations/{id}` com confirmação. "Ocultar" (`active=false`) só após L2 |
| Criar | `components/admin/user/UsersList.tsx` | `admin/users` com busca `?name=` |
| Alterar | `components/admin/Admin.tsx` | `<ResourceGuesser name="admin/establishments" …/>`, `admin/evaluations`, `admin/users`; remover `next/head` |
| Alterar | `components/admin/layout/Menu.tsx` | "Estabelecimentos", "Avaliações", "Usuários" |
| Alterar | `components/admin/i18nProvider.ts` | pt-BR (pacote `ra-language-portuguese` se for compatível com React Admin 5 — confirmar — ou mensagens próprias); remover fr/en |
| Alterar | `components/admin/AccessDenied.tsx` | "Acesso negado", "Você não tem permissão para acessar a administração.", "Voltar ao mapa" (`/`), "Sair" |
| Criar | `app/admin/layout.tsx` | `metadata.title = "Administração"`; ocultar o `Layout` público, se necessário |

> O React Admin precisa ler/escrever em URLs diferentes (lista pública, ação em `/admin/...`). Isso exige um `dataProvider` customizado (envolver `hydraDataProvider` e reescrever `getList`/`getOne` para o caminho público). Documentar o desvio em comentário, como pede o `CLAUDE.md`.

---

### Fase 10: testes e lint

| Ação | Arquivo | Alteração |
|---|---|---|
| Alterar | `pwa/package.json` | `devDependencies`: `eslint-plugin-jsx-a11y`; opcional `vitest` + `@testing-library/react` + script `"test": "vitest"` |
| Alterar | `pwa/eslint.config.mjs` | Adicionar a configuração recomendada do `jsx-a11y` (confirmar o export `flatConfigs.recommended` na versão instalada) |
| Remover | `e2e/tests/{BookView,BookmarksList,BooksList,GraphQL,Homepage,User}.spec.ts`, `e2e/tests/admin/*`, mocks da Open Library | Testes do demo |
| Alterar | `e2e/mock-server/server.js` | Mockar Google Places e a moderação (422) em vez da Open Library |
| Alterar | `e2e/package.json` | `@axe-core/playwright` |
| Criar | `e2e/tests/a11y.spec.ts` | Exemplo de 05 §5.8 em `/`, `/painel`, `/sobre`, `/acessibilidade`, `/locais/<id>`, com e sem `data-contrast="alto"` |
| Criar | `e2e/tests/{explorar,detalhe,avaliar,votar,auth}.spec.ts` | Cenários 1–6 de 06 §6.8 |
| Criar | `pwa/lib/__tests__/*.test.ts` | `statusFromAverage`, `parsePoint`, `buildBody`, `mapServerErrors`, `toCsv`, `filtersFromSearchParams`/`filtersToSearchParams` |

---

## 8.6 Variáveis de ambiente e configuração

| Variável | Situação | Ação |
|---|---|---|
| `NEXT_PUBLIC_GOOGLE_MAPS_API_KEY` | usada | Habilitar Places API (New) e restringir por domínio |
| `NEXT_PUBLIC_GOOGLE_MAPS_MAP_ID` | **nova** (não está em 06 §6.6) | Necessária para `AdvancedMarker`; adicionar à doc, ao `compose*.yaml` e ao `helm/` |
| `NEXT_PUBLIC_MAP_DEFAULT_CENTER` | prevista, não usada | Opcional; ler em `lib/geo.ts` com fallback para Tubarão |
| Keycloak / `better-auth` | usadas | Sem mudança; tema do Keycloak acessível e em pt-BR (03 T7) fica no escopo do frontend |

## 8.7 Ajustes na própria documentação

Pontos encontrados durante a análise que valem uma correção nos documentos 02–06:

1. 06 §6.6: incluir `NEXT_PUBLIC_GOOGLE_MAPS_MAP_ID`.
2. 03 T2 × 05 §5.4.1: o `<title>` usa "|" em um lugar e "–" no outro; padronizar.
3. 03 §3.1: o status "Parcial" de T1 omite que `EstablishmentDrawer.tsx` quebra o build hoje.
4. `pwa/FRONTEND_ARCHITECTURE.md` (referenciado em 06) descreve o demo de livros; reescrever na Fase 0.

## 8.8 Checklist de entrega

- [ ] Fase 0: build verde, sem `/books`
- [ ] Fase 1: `lib/`, tipos, `fetchApi`, tokens
- [ ] Fase 2: layout, cabeçalho, rodapé, anunciador, toaster
- [ ] Fase 3: T1 com lista, mapa em Tubarão, filtros acessíveis na URL
- [ ] Fase 4: T2 detalhe
- [ ] Fase 5: preferências, VLibras, atalhos, T5, T6, T10
- [ ] Fase 6: T3 assistente de avaliação
- [ ] Fase 7: votos otimistas e Mercure
- [ ] Fase 8: T4 painel e CSV
- [ ] Fase 9: admin do Acessibiliza em pt-BR
- [ ] Fase 10: `jsx-a11y`, axe e cenários E2E
- [ ] Checklist de aceite de 05 §5.7 em cada tela
