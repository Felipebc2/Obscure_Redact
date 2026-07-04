# CLAUDE.md — Obscure-Redacted

Protótipo funcional (não bonito): upload PDF → **tarja preta real** em dados
sensíveis (CPF primeiro) → download. Docs completas em `docs/requirements/`
e `ROADMAP.md`.

## Regra de segurança nº 1 — Redação REAL, nunca overlay

Retângulo preto desenhado por cima **não remove o texto** — o CPF continua
extraível por baixo. Inaceitável (LGPD). Sempre usar PyMuPDF:

```python
page.add_redact_annot(bbox, fill=(0, 0, 0))
page.apply_redactions()   # apaga o texto subjacente + pinta a caixa
```

Todo redactor DEVE ter teste que reabre o PDF de saída e confirma que o dado
tarjado **não aparece** em `page.get_text()`. Sem esse teste, a feature não está pronta.

## Stack

- **Backend**: FastAPI + Pydantic + PyMuPDF (`fitz`). Python.
- **Frontend**: Next.js (App Router) + Tailwind. Página única.
- **Testes**: Pytest (unit) + Playwright (E2E).
- **Empacotamento**: Docker + docker-compose.
- MVP **stateless**: nada persistido em disco; buffers descartados.

## Arquitetura — Clean Architecture

Dependências apontam para dentro: `web` / `pdf` → `application` → `domain`.

```
backend/app/
├── domain/          # puro, sem fitz nem fastapi
│   ├── entities/    # RedactionMatch, RedactionResult
│   └── patterns/    # base.py (Pattern ABC), cpf.py, registry.py
├── application/
│   ├── ports/       # pdf_redactor_port.py (interface)
│   └── use_cases/   # redact_document.py (RedactDocument)
├── infrastructure/
│   ├── pdf/         # pymupdf_redactor.py (implementa o port)
│   └── web/         # routes.py, schemas.py
└── main.py          # app FastAPI + CORS
```

O domínio **não importa** `fitz` nem `fastapi`. PyMuPDF fica isolado atrás de
`PdfRedactorPort` — trocar de lib não toca no caso de uso.

## Como estender padrões (RG, CNPJ, e-mail, telefone)

Novo padrão = **1 arquivo + 1 linha + 1 teste**, sem tocar no use case:

1. `domain/patterns/<nome>.py` — classe herdando `Pattern` (ABC), implementa
   `name` e `find(text) -> spans`. Validar dígito verificador quando aplicável (CNPJ).
2. `domain/patterns/registry.py` — registrar a nova classe no `PatternRegistry`.
3. `tests/test_<nome>.py` — válidos passam, inválidos falham.

## API

- `GET /health` → `{"status": "ok"}`
- `POST /redact` (multipart `file`) → stream `application/pdf`
  - `X-Redaction-Count`: total de tarjas
  - `X-Redaction-Counts`: JSON por tipo, ex. `{"cpf": 3}`
  - Erros: `400` (não-PDF/corrompido), `422` (sem arquivo)

## CPF (`domain/patterns/cpf.py`)

Regex `\d{3}\.?\d{3}\.?\d{3}-?\d{2}` (formatado e não formatado) + validação
dos **dois dígitos verificadores**. Rejeita `111.111.111-11` e sequências inválidas.

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

- **MVP**: PDF com texto real (selecionável), CPF, redação real, contagem, download.
- **Fora do MVP**: OCR/escaneado, login, persistência/audit, edição manual de tarjas.
- **Fase 2 (web)**: MinIO/S3, Supabase/Postgres + SQLAlchemy, JWT, deploy Azure/Traefik.

## Workflow deste projeto

- Trabalho quebrado em to-do por sessão/tarefa do `ROADMAP.md`.
- **Pausa entre tarefas**: ao terminar, comentar e perguntar "revisar ou próxima?".
- **Entre sessões do to-do**: lembrar de rodar `/compact` (context save).
- Registrar progresso/decisões no Obsidian second-brain via `/capture`.
- **Antes de editar**: ler o arquivo; antes de mudar função, grep dos callers.
