# Orchestrator — Pipeline de Redação

Descreve a orquestração dos componentes do fluxo
`upload → tarja automática → preview → tarja interativa → download`.

## Diagrama do fluxo

```
[Next.js UI]  --upload PDF (multipart)-->  [FastAPI POST /redact]
                                                  |
                                                  v
                                   [UseCase: RedactDocument]
                                                  |
                    1. carrega PDF em memória (fitz.open, defensivo)
                    2. extrai palavras + coordenadas (page.get_text("words"))
                    3. PatternRegistry: aplica cada Pattern (CPF: regex + dígito)
                    4. match -> search_for(bbox) -> add_redact_annot
                    5. apply_redactions()  (REMOVE texto + pinta caixa preta)
                    6. serializa PDF em bytes (doc.tobytes)
                                                  |
                                                  v
[Next.js UI]  <-- PDF tarjado + X-Redaction-Count --  [Response stream]
        |
        |  guarda como PDF de trabalho (blob) e RENDERIZA no preview (pdf.js)
        v
   ┌── laço de tarja interativa (opcional, repetível) ──────────────┐
   │  usuário informa palavra-chave  --> POST /redact/keyword       │
   │  OU desenha região no preview   --> POST /redact/regions       │
   │        (envia o PDF de trabalho atual como file)               │
   │  backend aplica redação real e devolve PDF atualizado          │
   │  UI substitui o blob, re-renderiza o preview, soma a contagem  │
   └───────────────────────────────────────────────────────────────┘
        |
        v
   usuário confere o preview  -->  [ Baixar PDF tarjado ]
```

## Responsabilidades por etapa

| Etapa | Componente | Responsabilidade |
|-------|-----------|------------------|
| Recepção | `infrastructure/web/routes.py` | Validar content-type e tamanho, ler bytes, chamar use case |
| Orquestração (auto) | `application/use_cases/redact_document.py` | Extração → detecção → redação; contar matches |
| Orquestração (palavra) | `application/use_cases/redact_by_keyword.py` | Tarjar ocorrências de uma **lista** de termos (batch); opções whole_word/acento |
| Orquestração (seleção) | `application/use_cases/redact_regions.py` | Tarjar **lista** de regiões informadas pelo cliente (batch) |
| Detecção | `domain/patterns/*` | Regras puras (regex + validação). Sem I/O, sem PDF |
| Redação | `infrastructure/pdf/pymupdf_redactor.py` | Implementa `PdfRedactorPort` (patterns/keyword/regions) |
| Resposta | `infrastructure/web/routes.py` | Devolver stream do PDF + header com contagem |
| Preview | `frontend/components/PdfPreview.tsx` | Renderizar o PDF de trabalho (blob) via pdf.js |

## Regras de orquestração

- **Direção de dependência**: `web`/`pdf` → `application` → `domain`. O domínio não importa nada de fora.
- **Port/Adapter**: os use cases dependem da interface `PdfRedactorPort`, não do PyMuPDF. Trocar de lib = novo adapter.
- **Stateless / privacidade**: nenhum arquivo é escrito em disco persistente; o servidor não guarda estado entre chamadas. O **PDF de trabalho vive no cliente** (blob) e é reenviado a cada tarja interativa.
- **Idempotência**: reprocessar um PDF já tarjado não quebra (matches = 0 se já removidos).
- **Robustez**: limite de tamanho/páginas e timeout contra PDF-bomb; abertura defensiva → `400` em PDF corrompido.
