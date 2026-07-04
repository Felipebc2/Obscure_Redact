# ROADMAP — Obscure-Redacted

> Construção do MVP em **3 dias / 24h**. Protótipo funcional (não bonito):
> upload PDF → tarja **real** de dados sensíveis (CPF primeiro) → download.
> Princípio nº 1: **redação real, nunca overlay** (ver `docs/requirements/stack.md`).

Legenda: `[ ]` pendente · `[x]` feito. Estimativas entre parênteses.

---

## Dia 1 (8h) — Backend core

Objetivo: `POST /redact` funcional que tarja CPF de verdade num PDF.

- [ ] **Scaffold + ambiente** (1h)
  - `python -m venv` + `requirements.txt` (fastapi, uvicorn, python-multipart, pymupdf, pytest, reportlab)
  - Dockerfile base do backend
- [ ] **`domain/patterns` — CPF** (2h)
  - `base.py` (`Pattern` ABC: `name`, `find(text) -> spans`)
  - `cpf.py` (regex `\d{3}\.?\d{3}\.?\d{3}-?\d{2}` + validação dos 2 dígitos verificadores)
  - `registry.py` (`PatternRegistry` com CPF ativo)
  - `test_cpf.py` — válidos passam, `111.111.111-11`/sequências falham
- [ ] **`application` — use case + port** (1.5h)
  - `ports/pdf_redactor_port.py` (interface `redact(pdf, patterns) -> result`)
  - `use_cases/redact_document.py` (`RedactDocument` orquestra registry + port)
  - `entities/redaction_match.py`, `entities/redaction_result.py`
  - `test_redact_document.py` — conta matches corretos
- [ ] **`infrastructure/pdf` — PyMuPDFRedactor** (2.5h)
  - `add_redact_annot(bbox, fill=(0,0,0))` + `apply_redactions()`
  - fixtures via `reportlab` (PDF com CPF válido/inválido)
  - `test_pymupdf_redactor.py` — **crítico**: reabrir saída, CPF não extraível em `get_text()`
- [ ] **`web` — endpoints** (1h)
  - `POST /redact` (multipart `file`) → stream `application/pdf` + `X-Redaction-Count` / `X-Redaction-Counts`
  - `GET /health` → `{"status":"ok"}`
  - `main.py` (app FastAPI + CORS)

**Fim Dia 1**: `uvicorn` sobe, `curl -F file=@fixture.pdf localhost:8000/redact` devolve PDF tarjado.

---

## Dia 2 (8h) — Frontend + integração

Objetivo: tela única que sobe PDF, chama API, baixa resultado.

- [ ] **Scaffold Next.js + Tailwind** (1h)
  - App Router, `NEXT_PUBLIC_API_URL` default `http://localhost:8000`
- [ ] **Dropzone** (2h)
  - `components/Dropzone.tsx` — drag-drop + clique, valida extensão `.pdf`
  - Estados `idle` / `selected` (nome + tamanho + botão "Tarjar")
- [ ] **Integração API + download** (2h)
  - `POST /redact` com `FormData`; resposta `blob` → `URL.createObjectURL`
  - `components/ResultPanel.tsx` — lê `X-Redaction-Count(s)`, botão "Baixar PDF tarjado"
  - Estados `processing` / `done`
- [ ] **E2E local (Playwright)** (1h)
  - abrir página → upload fixture → Tarjar → aguardar `done` → validar contagem
- [ ] **Erros / edge cases** (2h)
  - não-PDF bloqueado antes de enviar; `4xx/5xx` → mensagem amigável
  - PDF sem CPF → "0 tarjas" (não é erro); PDF sem texto; arquivo grande

**Fim Dia 2**: fluxo completo no navegador, local, backend + frontend juntos.

---

## Dia 3 (8h) — Extensibilidade + deploy prep

Objetivo: novos padrões via registry + empacotamento.

- [ ] **Novos padrões** (3h)
  - `rg.py`, `cnpj.py`, `email.py`, `telefone.py` — cada um 1 arquivo + 1 linha no registry + teste
  - validação de dígito onde aplicável (CNPJ)
- [ ] **Contagem por tipo na UI** (1.5h)
  - exibir `X-Redaction-Counts` (ex.: `CPF: 3 · e-mail: 1`)
- [ ] **Docker + compose** (2h)
  - Dockerfile frontend; `docker-compose.yml` sobe os dois serviços
  - `docker-compose up` reproduz o fluxo local
- [ ] **Docs + notas de deploy** (1.5h)
  - atualizar `README.md` (como rodar, testar)
  - notas de deploy web fase 2: MinIO/S3, Supabase/Postgres + SQLAlchemy, JWT, Azure/Traefik

**Fim Dia 3**: múltiplos padrões, contagem por tipo, sobe via `docker-compose`, docs atualizadas.

---

## Critérios de "pronto" (MVP)

- 100% dos CPFs válidos de teste tarjados e **não extraíveis** na saída.
- 0 falso-positivo em `111.111.111-11` / sequências inválidas.
- PDF < 20 páginas processado em poucos segundos.
- Novo padrão adicionado **sem alterar** o use case.
- `docker-compose up` roda o fluxo ponta a ponta.
