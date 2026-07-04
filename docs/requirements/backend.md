# Backend — Design

## Stack
FastAPI + Pydantic + PyMuPDF (`fitz`), organizado em Clean Architecture.

## Estrutura (Clean Architecture)

```
backend/app/
├── domain/                     # regras puras, sem I/O nem framework
│   ├── entities/
│   │   ├── redaction_match.py  # RedactionMatch(value, kind, bbox)
│   │   ├── region.py           # Region(page, x0, y0, x1, y1)  -> tarja por seleção
│   │   └── redaction_result.py # RedactionResult(pdf_bytes, counts)
│   └── patterns/
│       ├── base.py             # Pattern (ABC): name, find(text) -> spans
│       ├── cpf.py              # CpfPattern: regex + validação de dígito
│       └── registry.py         # PatternRegistry: lista ativa de patterns
├── application/
│   ├── ports/
│   │   └── pdf_redactor_port.py    # interface (ver contrato abaixo)
│   └── use_cases/
│       ├── redact_document.py      # RedactDocument (tarja automática por patterns)
│       ├── redact_by_keyword.py    # RedactByKeyword (tarja interativa por palavra)
│       └── redact_regions.py       # RedactRegions (tarja interativa por seleção)
├── infrastructure/
│   ├── pdf/
│   │   └── pymupdf_redactor.py     # implementa PdfRedactorPort com fitz
│   └── web/
│       ├── routes.py               # POST /redact, /redact/keyword, /redact/regions, GET /health
│       └── schemas.py              # Pydantic (respostas/erros/entrada)
└── main.py                         # cria app FastAPI, monta rotas, CORS
```

## Direção de dependências
`web` / `pdf` → `application` → `domain`. O domínio não importa `fitz` nem `fastapi`.

## Contrato do Port (`application/ports/pdf_redactor_port.py`)

Assinatura fixada — todos os métodos recebem os **bytes do PDF de trabalho** e devolvem
`RedactionResult(pdf_bytes, counts)`. Isso mantém o backend **stateless**: o cliente
segura o PDF entre as etapas e reenvia os bytes atuais.

```python
class PdfRedactorPort(ABC):
    @abstractmethod
    def redact_patterns(self, pdf: bytes, patterns: list[Pattern]) -> RedactionResult:
        ...

    @abstractmethod
    def redact_keywords(self, pdf: bytes, keywords: list[str],
                        case_sensitive: bool = False,
                        whole_word: bool = False,
                        normalize_accents: bool = True) -> RedactionResult:
        ...

    @abstractmethod
    def redact_regions(self, pdf: bytes, regions: list[Region]) -> RedactionResult:
        ...
```

## Contrato da API

Todos os endpoints de redação devolvem o **PDF de trabalho atualizado** (stream), para o
cliente seguir com preview e novas tarjas. Sem estado no servidor.

### `GET /health`
`200 → {"status": "ok"}`

### `POST /redact` — tarja automática (patterns)
- **Request**: `multipart/form-data`, campo `file` (application/pdf).
- **Response**: `200`, `application/pdf` (stream do PDF tarjado).
  - Header `X-Redaction-Count`: total de tarjas nesta chamada.
  - Header `X-Redaction-Counts`: JSON por tipo, ex. `{"cpf": 3}`.

### `POST /redact/keyword` — tarja interativa por palavra-chave (RF08a)
- **Request**: `multipart/form-data`, campo `file` (PDF de trabalho atual) + campos
  `keywords` (JSON, **lista** de termos — batch), `case_sensitive` (bool, default false),
  `whole_word` (bool, default false), `normalize_accents` (bool, default true).
- **Response**: `200`, `application/pdf` + `X-Redaction-Count` (ocorrências tarjadas).
- **Batch**: aceitar lista evita 1 request por palavra (menos tráfego e menos passadas de `apply_redactions`).

### `POST /redact/regions` — tarja interativa por seleção visual (RF08b)
- **Request**: `multipart/form-data`, campo `file` (PDF de trabalho atual) + campo
  `regions` (JSON, **lista** — batch): `[{"page": 0, "x0": .., "y0": .., "x1": .., "y1": ..}, ...]`
  em **coordenadas de ponto do PDF** (o frontend converte do viewport).
- **Response**: `200`, `application/pdf` + `X-Redaction-Count` (regiões aplicadas).
- **Semântica**: a redação remove todo texto cujo bbox **intersecta** a região (não só o
  "contido"). O frontend deve deixar isso claro ao usuário.

### Erros (todos os endpoints de redação)
- `400` — arquivo não é PDF / corrompido.
- `413` — arquivo acima do limite de tamanho (RNF08).
- `422` — parâmetro obrigatório ausente (arquivo, keyword ou regions).

## Robustez / hardening (RNF08, RNF09)
- **Limite de tamanho**: rejeitar upload acima de um teto configurável (ex.: 20 MB) → `413`.
- **Limite de páginas** e **timeout** de processamento para conter PDF-bomb.
- Abrir sempre via `fitz.open(stream=bytes, filetype="pdf")` com tratamento de exceção → `400`.

## Lógica de redação (PyMuPDFRedactor)

```
abrir doc (fitz.open(stream=bytes, filetype="pdf"))  # defensivo -> 400 se falhar
para cada página:
    # tarja automática:
    words = page.get_text("words")   # [x0,y0,x1,y1, "palavra", ...]
    texto_pagina = reconstruir string + mapa palavra->bbox
    para cada pattern no registry:
        para cada span detectado (com validação):
            rects = page.search_for(texto_do_match)   # bbox(es) exatos do match
            para cada rect: page.add_redact_annot(rect, fill=(0,0,0))
    page.apply_redactions()          # remove texto + pinta preto
retornar doc.tobytes() + contagem por tipo
```

**Mapeamento match → bbox (risco alto — ver Conselho):** o regex roda sobre o texto
concatenado, mas a coordenada precisa vir do PDF. Estratégia MVP: `page.search_for(match)`
para obter os retângulos exatos — mais robusto que reconstruir bbox das "words".
Cuidado com **CPF que cruza spans/linhas**: quando `search_for` não acha o match exato
(quebra de linha, espaço inserido), cair para a reconstrução via mapa palavra→bbox.
**Validar essa abordagem num spike com PDF real antes de escrever o core.**

- **Keywords (batch)**: para cada termo, `page.search_for(keyword)` em todas as páginas →
  `add_redact_annot` → `apply_redactions` uma vez ao fim. Opções: `whole_word` (filtrar
  match colado a letra/dígito, ex. evitar "Ana" em "Banana") e `normalize_accents`
  (comparar sem acento). Cuidado com termo que cruza linha/span (mesmo caso do CPF).
- **Regions (batch)**: para cada `Region`, montar `fitz.Rect` na página indicada →
  `add_redact_annot` → `apply_redactions`. Remove texto que **intersecta** o retângulo.

## Padrão CPF (`domain/patterns/cpf.py`)
- Regex: `\d{3}\.?\d{3}\.?\d{3}-?\d{2}` (aceita formatado e não formatado).
- Validação: cálculo dos **dois dígitos verificadores**; rejeita repetições (`111...`).
- `find(text) -> list[Span]` com `(start, end, value, kind="cpf")`.

## Testes (Pytest)
- `test_cpf.py` — válidos passam, inválidos/repetidos falham.
- `test_pymupdf_redactor.py` — **redação real**: reabrir saída e garantir que o CPF
  não aparece em `page.get_text()`.
- `test_no_false_negative.py` — **crítico**: PDF com N CPFs válidos em posições variadas
  (formatado, não-formatado, dentro de tabela) → **todos** ausentes na saída; contagem = N.
- `test_redact_keyword.py` — palavra-chave é tarjada de verdade (não extraível) em todas ocorrências.
- `test_redact_regions.py` — texto sob a região selecionada some do `get_text()`.
- `test_redact_document.py` — use case conta matches corretamente.
- Fixtures geradas com `reportlab` em `tests/fixtures/` — incluir CPF em posições variadas.
