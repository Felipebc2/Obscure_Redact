# Frontend — Design

## Stack
Next.js (App Router) + Tailwind. Objetivo: **funcional, não bonito**. Página única.

## Tela única (`app/page.tsx`)

```
┌─────────────────────────────────────────────┐
│  Obscure-Redacted — Tarja de dados sensíveis │
│                                              │
│   ┌───────────────────────────────────┐      │
│   │  Arraste o PDF aqui ou clique      │      │  <- Dropzone
│   │  (somente .pdf)                    │      │
│   └───────────────────────────────────┘      │
│                                              │
│   arquivo.pdf  •  128 KB          [ Tarjar ] │
│                                              │
│   [ status: processando... / pronto ]        │
│                                              │
│   ✅ 3 tarjas aplicadas (CPF: 3)             │
│   [ Baixar PDF tarjado ]                      │
└─────────────────────────────────────────────┘
```

## Componentes
- `components/Dropzone.tsx` — seleção/drag-drop, valida extensão `.pdf`.
- `components/ResultPanel.tsx` — status, contagem de tarjas, botão de download.

## Estados da UI
| Estado | O que mostra |
|--------|--------------|
| `idle` | Dropzone vazia |
| `selected` | Nome + tamanho do arquivo + botão "Tarjar" |
| `processing` | Spinner / "processando..." (botão desabilitado) |
| `done` | Contagem de tarjas + botão "Baixar PDF tarjado" |
| `error` | Mensagem (ex.: "arquivo não é PDF", "falha no servidor") |

## Integração com a API
- `POST {API_URL}/redact` com `FormData` (`file`).
- Resposta é um **blob** (`application/pdf`) → `URL.createObjectURL` para download.
- Ler headers `X-Redaction-Count` / `X-Redaction-Counts` para exibir contagem.
- `API_URL` via `NEXT_PUBLIC_API_URL` (default `http://localhost:8000`).

## Tratamento de erro
- Extensão inválida → bloqueia antes de enviar.
- `4xx/5xx` → exibir mensagem amigável, permitir novo envio.
- Sem CPF encontrado → "0 tarjas" (não é erro; PDF de saída = entrada).

## E2E (Playwright)
Fluxo: abrir página → upload de fixture → clicar Tarjar → aguardar "done" →
baixar → validar contagem exibida.
