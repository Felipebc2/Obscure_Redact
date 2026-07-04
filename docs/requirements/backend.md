# Backend — Design

## Stack
FastAPI + Pydantic + PyMuPDF (`fitz`), organizado em Clean Architecture.

## Estrutura (Clean Architecture)

```
backend/app/
├── domain/                     # regras puras, sem I/O nem framework
│   ├── entities/
│   │   ├── redaction_match.py  # RedactionMatch(value, kind, bbox)
│   │   └── redaction_result.py # RedactionResult(pdf_bytes, counts)
│   └── patterns/
│       ├── base.py             # Pattern (ABC): name, find(text) -> spans
│       ├── cpf.py              # CpfPattern: regex + validação de dígito
│       └── registry.py         # PatternRegistry: lista ativa de patterns
├── application/
│   ├── ports/
│   │   └── pdf_redactor_port.py    # interface: redact(pdf, patterns) -> result
│   └── use_cases/
│       └── redact_document.py      # RedactDocument (orquestra)
├── infrastructure/
│   ├── pdf/
│   │   └── pymupdf_redactor.py     # implementa PdfRedactorPort com fitz
│   └── web/
│       ├── routes.py               # POST /redact, GET /health
│       └── schemas.py              # Pydantic (respostas/erros)
└── main.py                         # cria app FastAPI, monta rotas, CORS
```

## Direção de dependências
`web` / `pdf` → `application` → `domain`. O domínio não importa `fitz` nem `fastapi`.

## Contrato da API

### `GET /health`
`200 → {"status": "ok"}`

### `POST /redact`
- **Request**: `multipart/form-data`, campo `file` (application/pdf).
- **Response**: `200`, `application/pdf` (stream do PDF tarjado).
  - Header `X-Redaction-Count`: total de tarjas.
  - Header `X-Redaction-Counts`: JSON por tipo, ex. `{"cpf": 3}`.
- **Erros**:
  - `400` — arquivo não é PDF / corrompido.
  - `422` — nenhum arquivo enviado.

## Lógica de redação (PyMuPDFRedactor)

```
abrir doc (fitz.open(stream=bytes))
para cada página:
    words = page.get_text("words")   # [x0,y0,x1,y1, "palavra", ...]
    texto_pagina = reconstruir string + mapa palavra->bbox
    para cada pattern no registry:
        para cada span detectado (com validação):
            resolver bbox(s) que cobrem o span
            page.add_redact_annot(bbox, fill=(0,0,0))
    page.apply_redactions()          # remove texto + pinta preto
retornar doc.tobytes() + contagem por tipo
```

Observação: um CPF pode se espalhar por múltiplas "words" (ex. quando espaços/pontuação
separam) — o redactor mapeia o span de caracteres para o(s) bbox(es) correspondente(s).
Estratégia MVP: usar `page.search_for(texto_do_match)` para obter os retângulos exatos,
mais robusto que reconstruir bbox manualmente.

## Padrão CPF (`domain/patterns/cpf.py`)
- Regex: `\d{3}\.?\d{3}\.?\d{3}-?\d{2}` (aceita formatado e não formatado).
- Validação: cálculo dos **dois dígitos verificadores**; rejeita repetições (`111...`).
- `find(text) -> list[Span]` com `(start, end, value, kind="cpf")`.

## Testes (Pytest)
- `test_cpf.py` — válidos passam, inválidos/repetidos falham.
- `test_pymupdf_redactor.py` — **redação real**: reabrir saída e garantir que o CPF
  não aparece em `page.get_text()`.
- `test_redact_document.py` — use case conta matches corretamente.
- Fixtures geradas com `reportlab` em `tests/fixtures/`.
