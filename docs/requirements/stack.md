# Stack — Recomendada vs Usuário vs Meio-termo

## Decisão central que define a stack: redação REAL

Um retângulo preto **desenhado por cima** não remove o texto — o CPF continua
selecionável/extraível por baixo. Inaceitável para dado sensível (LGPD).

**PyMuPDF (`fitz`)** faz redação real: `page.add_redact_annot(bbox)` +
`page.apply_redactions()` **apaga o conteúdo subjacente** e pinta a caixa.
Esse é o motivo técnico de escolher backend **Python** em vez de redação pura
em JS (`pdf-lib`, que essencialmente sobrepõe).

> Regra do projeto: **sempre redação real, nunca overlay.**

## Comparação

| Camada | Recomendada de mercado | Stack do usuário | Meio-termo escolhido |
|--------|------------------------|------------------|----------------------|
| Backend | Python FastAPI | FastAPI, Flask, NestJS, Node ✅ | **FastAPI + Pydantic** |
| Redação PDF | PyMuPDF (`fitz`) | — (novo, mas Python) | **PyMuPDF** |
| Frontend | Next.js + Tailwind | Next.js, React, Angular, Tailwind ✅ | **Next.js (App Router) + Tailwind** |
| Banco (MVP) | Nenhum (stateless) | Postgres, Supabase ✅ | **Nenhum** (privacidade) |
| Storage (fase 2) | S3/MinIO | MinIO, S3 AWS ✅ | **MinIO/S3** |
| Banco (fase 2) | Postgres + ORM | Supabase, SQLAlchemy ✅ | **Supabase/Postgres + SQLAlchemy** + JWT |
| Testes | pytest + E2E | Pytest, Playwright, Gherkin ✅ | **Pytest** + **Playwright** |
| Deploy | Docker | Docker, Azure, K8s, Traefik ✅ | **Docker** → Azure/Traefik |

## Conclusão

O meio-termo é **100% sobreposição** com a stack do usuário — que já lista
**Clean Architecture, FastAPI/Pydantic, Pytest, Playwright e Gherkin** no currículo.
Única peça nova: **PyMuPDF**, justificada pela exigência de redação real.
**Zero atrito de stack.**

## Dependências principais (MVP)

**Backend** (`backend/requirements.txt`)
- `fastapi`, `uvicorn[standard]` — API
- `python-multipart` — upload
- `pymupdf` — redação real de PDF
- `pytest` — testes
- `reportlab` (dev) — gerar PDFs de fixture para os testes

**Frontend** (`frontend/package.json`)
- `next`, `react`, `react-dom`
- `tailwindcss`, `postcss`, `autoprefixer`
- `pdfjs-dist` / `react-pdf` — **preview** do PDF tarjado no cliente (RF05) e base para a
  tarja interativa por seleção visual (RF08b)
