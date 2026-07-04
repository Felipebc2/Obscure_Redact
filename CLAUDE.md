# CLAUDE.md — Obscure-Redacted

Protótipo funcional (não bonito): upload PDF → **tarja preta real** em dados
sensíveis (CPF primeiro) → **preview** → **tarja interativa** (palavra-chave e/ou
seleção visual) → download. Docs completas em `docs/requirements/` e `ROADMAP.md`.

## Regra de segurança nº 1 — Redação REAL, nunca overlay

Retângulo preto desenhado por cima **não remove o texto** — o CPF continua
extraível por baixo. Inaceitável (LGPD). Sempre usar PyMuPDF:

```python
page.add_redact_annot(bbox, fill=(0, 0, 0))
page.apply_redactions()   # apaga o texto subjacente + pinta a caixa
```

Todo redactor DEVE ter teste que reabre o PDF de saída e confirma que o dado
tarjado **não aparece** em `page.get_text()`. Sem esse teste, a feature não está pronta.
Vale para a tarja automática **e** para a interativa (palavra-chave/região).

Além do teste de não-extração, teste de **falso-negativo**: PDF com N CPFs válidos em
posições variadas → **todos** ausentes na saída, contagem = N.

## Stack

- **Backend**: FastAPI + Pydantic + PyMuPDF (`fitz`). Python.
- **Frontend**: Next.js (App Router) + Tailwind + **pdf.js** (`pdfjs-dist`/`react-pdf`) p/ preview.
- **Testes**: Pytest (unit) + Playwright (E2E).
- **Empacotamento**: Docker + docker-compose.
- MVP **stateless**: nada persistido no servidor; o **PDF de trabalho vive no cliente**
  (blob) e é reenviado a cada tarja interativa; buffers descartados por request.

## Arquitetura — Clean Architecture

Dependências apontam para dentro: `web` / `pdf` → `application` → `domain`.

```
backend/app/
├── domain/          # puro, sem fitz nem fastapi
│   ├── entities/    # RedactionMatch, RedactionResult, Region
│   └── patterns/    # base.py (Pattern ABC), cpf.py, registry.py
├── application/
│   ├── ports/       # pdf_redactor_port.py (interface)
│   └── use_cases/   # redact_document.py, redact_by_keyword.py, redact_regions.py
├── infrastructure/
│   ├── pdf/         # pymupdf_redactor.py (implementa o port)
│   └── web/         # routes.py, schemas.py
└── main.py          # app FastAPI + CORS
```

O domínio **não importa** `fitz` nem `fastapi`. PyMuPDF fica isolado atrás de
`PdfRedactorPort` (`redact_patterns` / `redact_keywords` / `redact_regions`, todos
`(pdf: bytes, ...) -> RedactionResult`) — trocar de lib não toca no caso de uso.

## Como estender padrões (RG, CNPJ, e-mail, telefone)

Novo padrão = **1 arquivo + 1 linha + 1 teste**, sem tocar no use case:

1. `domain/patterns/<nome>.py` — classe herdando `Pattern` (ABC), implementa
   `name` e `find(text) -> spans`. Validar dígito verificador quando aplicável (CNPJ).
2. `domain/patterns/registry.py` — registrar a nova classe no `PatternRegistry`.
3. `tests/test_<nome>.py` — válidos passam, inválidos falham.

## API

Todos os endpoints de redação devolvem o **PDF de trabalho atualizado** (stream), pro
cliente seguir com preview e novas tarjas (backend stateless).

- `GET /health` → `{"status": "ok"}`
- `POST /redact` (multipart `file`) → stream `application/pdf` — tarja automática (patterns)
  - `X-Redaction-Count`: total de tarjas
  - `X-Redaction-Counts`: JSON por tipo, ex. `{"cpf": 3}`
- `POST /redact/keyword` (multipart `file` + `keywords` JSON lista + `case_sensitive` + `whole_word` + `normalize_accents`) → PDF atualizado
- `POST /redact/regions` (multipart `file` + `regions` JSON lista em pontos do PDF) → PDF atualizado
- Ações interativas em **batch** (lista por chamada), não uma request por clique
- Erros: `400` (não-PDF/corrompido), `413` (grande demais), `422` (parâmetro ausente)

## CPF (`domain/patterns/cpf.py`)

Regex `\d{3}\.?\d{3}\.?\d{3}-?\d{2}` (formatado e não formatado) + validação
dos **dois dígitos verificadores**. Rejeita `111.111.111-11` e sequências inválidas.

Mapeamento match→bbox via `page.search_for(match)` (retângulos exatos); fallback pro
mapa palavra→bbox quando o CPF cruza spans/linha. **Validar num spike antes do core.**

## Preview + tarja interativa

Depois da tarja automática, o cliente segura o PDF de trabalho (blob) e renderiza no
**preview** (pdf.js — exige `use client` + `next/dynamic ssr:false` + worker; renderizar
só a página atual). Duas formas de tarjar mais (redação real, mesmo critério), sempre em
**batch** (lista por chamada):

- **Palavra-chave** (`/redact/keyword`): `search_for` em cada termo; opções `whole_word`
  (evita "Ana" em "Banana") e `normalize_accents`.
- **Seleção visual** (`/redact/regions`, **stretch**): usuário desenha região; frontend
  converte viewport → pontos do PDF e envia `Region(page, x0, y0, x1, y1)`. Remove texto
  que **intersecta** o retângulo (cuidar rotação/`CropBox`/zoom).

Método da interativa **ainda em decisão** — palavra-chave primeiro (mais simples),
seleção depois (cortável). Desfazer = **1 checkpoint** (resetar ao original/pós-auto),
não histórico. Ver `docs/requirements/frontend.md`.

## Robustez (hardening)

Limite de tamanho de upload (ex. 20 MB → `413`), limite de páginas e timeout contra
PDF-bomb; abrir sempre `fitz.open(stream=..., filetype="pdf")` defensivo → `400`.

## Comandos

```bash
# backend
cd backend && python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload          # http://localhost:8000
pytest                                 # testes unit

# frontend
cd frontend && npm install && npm run dev   # http://localhost:3000

# tudo junto
docker-compose up
```

`NEXT_PUBLIC_API_URL` default `http://localhost:8000`.

## Escopo

- **MVP**: PDF com texto real (selecionável), CPF, redação real, contagem, **preview**,
  **tarja interativa** (palavra-chave; seleção visual como próximo passo), download.
- **Fora do MVP**: OCR/escaneado, login, persistência/audit, desfazer no servidor.
- **Fase 2 (web)**: MinIO/S3, Supabase/Postgres + SQLAlchemy, JWT, deploy Azure/Traefik.

## Workflow deste projeto

- Trabalho quebrado em to-do por sessão/tarefa do `ROADMAP.md`.
- **Pausa entre tarefas**: ao terminar, comentar e perguntar "revisar ou próxima?".
- **Entre sessões do to-do**: lembrar de rodar `/compact` (context save).
- Registrar progresso/decisões no Obsidian second-brain via `/capture`.
- **Antes de editar**: ler o arquivo; antes de mudar função, grep dos callers.
