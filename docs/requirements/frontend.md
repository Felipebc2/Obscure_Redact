# Frontend — Design

## Stack
Next.js (App Router) + Tailwind. Objetivo: **funcional, não bonito**. Página única.
Preview do PDF via **pdf.js** (`react-pdf` / `pdfjs-dist`) — renderização no cliente.

> ⚠️ **Setup pdf.js no App Router (risco):** exige `use client` + import dinâmico
> (`next/dynamic`, `ssr:false`) e configuração do **worker** (`pdf.worker`). Alocar tempo
> para isso no Dia 2 — não é plug-and-play. Renderizar **só a página atual** (lazy) para
> não travar em PDF grande.

## Tela única (`app/page.tsx`)

```
┌───────────────────────────────────────────────────────┐
│  Obscure-Redacted — Tarja de dados sensíveis          │
│                                                        │
│   ┌───────────────────────────────────┐                │
│   │  Arraste o PDF aqui ou clique      │   <- Dropzone  │
│   │  (somente .pdf)                    │                │
│   └───────────────────────────────────┘                │
│                                                        │
│   arquivo.pdf  •  128 KB              [ Tarjar (auto) ] │
│                                                        │
│   ✅ 3 tarjas automáticas (CPF: 3)                      │
│                                                        │
│   ┌─────────────────── PREVIEW ──────────────────┐     │
│   │  [ página do PDF renderizada, com caixas      │     │
│   │    pretas já aplicadas ]                       │     │
│   │  ◀  pág. 1/4  ▶                                │     │
│   └───────────────────────────────────────────────┘     │
│                                                        │
│   Tarja manual:                                        │
│   [ palavra-chave ______ ] [ Tarjar palavra ]          │
│   [ modo seleção: desenhe no preview ]                 │
│                                                        │
│   [ Baixar PDF tarjado ]                                │
└───────────────────────────────────────────────────────┘
```

## Componentes
- `components/Dropzone.tsx` — seleção/drag-drop, valida extensão `.pdf`.
- `components/PdfPreview.tsx` — renderiza o **PDF de trabalho atual** (blob) via pdf.js;
  navegação de páginas. É o preview do RF05 (mostra o resultado real, já tarjado).
- `components/KeywordRedact.tsx` — input de palavra-chave + botão (RF08a).
- `components/RegionRedact.tsx` — captura seleção de região sobre o preview (RF08b).
- `components/ResultPanel.tsx` — contagem acumulada de tarjas, botão de download.

## Modelo de estado (cliente segura o PDF de trabalho)
O frontend mantém em memória o **blob do PDF de trabalho** (`workingPdf`). Cada ação de
redação envia `workingPdf` ao backend e substitui pelo blob retornado. Isso mantém o
backend stateless (RNF02).

- **Batch:** acumular palavras/regiões no cliente e enviar em **uma** chamada (lista), não
  uma request por clique — reduz tráfego e passadas de `apply_redactions`.
- **Desfazer = 1 checkpoint:** guardar apenas o PDF original (ou o pós-tarja-automática)
  para um botão "Resetar". Não manter histórico de N blobs (estoura memória em PDF grande).

## Estados da UI
| Estado | O que mostra |
|--------|--------------|
| `idle` | Dropzone vazia |
| `selected` | Nome + tamanho do arquivo + botão "Tarjar (auto)" |
| `processing` | Spinner / "processando..." (ações desabilitadas) |
| `preview` | PdfPreview do resultado + contagem + tarja manual + download |
| `error` | Mensagem (ex.: "arquivo não é PDF", "arquivo grande demais", "falha no servidor") |

Após `preview`, cada tarja manual (palavra ou região) volta a `processing` e retorna a
`preview` com o PDF atualizado — laço até o usuário baixar.

## Integração com a API
- `POST {API_URL}/redact` (auto) com `FormData` (`file`).
- `POST {API_URL}/redact/keyword` com `FormData` (`file` = workingPdf, `keywords` JSON, `case_sensitive`, `whole_word`, `normalize_accents`).
- `POST {API_URL}/redact/regions` com `FormData` (`file` = workingPdf, `regions` = JSON).
- Toda resposta é um **blob** (`application/pdf`) → vira o novo `workingPdf` e é
  renderizado no preview; `URL.createObjectURL` para o download.
- Ler headers `X-Redaction-Count` / `X-Redaction-Counts` para exibir contagem.
- `API_URL` via `NEXT_PUBLIC_API_URL` (default `http://localhost:8000`).

## Tarja interativa — método (decisão pendente, RF08)
Duas opções, ambas reusam a redação real do backend:

- **Palavra-chave (RF08a — recomendado primeiro):** usuário digita um ou mais termos;
  backend `search_for` tarja todas as ocorrências (batch). Opções `whole_word` e
  normalização de acento. Simples, sem geometria de viewport.
- **Seleção visual (RF08b — Dia 4, pós-MVP, cortável):** usuário desenha retângulos sobre
  a página no preview; o frontend converte viewport → **pontos do PDF** (via `viewport` do
  pdf.js). Mais UX, mais complexo — **rotação de página, `CropBox`/`MediaBox` e zoom**
  complicam o mapeamento; começa por um spike e é cortável se não fechar. A tarja remove
  texto que **intersecta** o retângulo (avisar o usuário). Ver Dia 4 no `ROADMAP.md`.

## Tratamento de erro
- Extensão inválida → bloqueia antes de enviar.
- `413` (arquivo grande) → mensagem clara de limite de tamanho.
- `4xx/5xx` → exibir mensagem amigável, permitir novo envio.
- Sem CPF encontrado → "0 tarjas" (não é erro); segue para preview com o PDF original.

## E2E (Playwright)
Fluxo: abrir página → upload de fixture → Tarjar (auto) → ver preview → tarjar por
palavra-chave → ver contagem subir → baixar → validar arquivo baixado.
