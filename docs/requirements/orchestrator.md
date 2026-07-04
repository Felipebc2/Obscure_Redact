# Orchestrator — Pipeline de Redação

Descreve a orquestração dos componentes do fluxo upload → tarja → download.

## Diagrama do fluxo

```
[Next.js UI]  --upload PDF (multipart)-->  [FastAPI POST /redact]
                                                  |
                                                  v
                                   [UseCase: RedactDocument]
                                                  |
                    1. carrega PDF em memória (fitz.open)
                                                  |
                    2. extrai palavras + coordenadas (page.get_text("words"))
                                                  |
                    3. PatternRegistry: aplica cada Pattern
                       - CPF: regex + validação de dígito verificador
                                                  |
                    4. para cada match -> bbox -> page.add_redact_annot(bbox)
                                                  |
                    5. page.apply_redactions()  (REMOVE texto + pinta caixa preta)
                                                  |
                    6. serializa PDF em bytes (doc.tobytes)
                                                  |
                                                  v
[Next.js UI]  <-- PDF tarjado + X-Redaction-Count --  [Response stream]
                                                  |
                         UI: link de download + contagem de tarjas
```

## Responsabilidades por etapa

| Etapa | Componente | Responsabilidade |
|-------|-----------|------------------|
| Recepção | `infrastructure/web/routes.py` | Validar content-type PDF, ler bytes, chamar use case |
| Orquestração | `application/use_cases/redact_document.py` | Coordenar extração → detecção → redação; contar matches |
| Detecção | `domain/patterns/*` | Regras puras (regex + validação). Sem I/O, sem PDF |
| Redação | `infrastructure/pdf/pymupdf_redactor.py` | Implementa `PdfRedactorPort` com PyMuPDF |
| Resposta | `infrastructure/web/routes.py` | Devolver stream do PDF + header com contagem |

## Regras de orquestração

- **Direção de dependência**: `web`/`pdf` → `application` → `domain`. O domínio não importa nada de fora.
- **Port/Adapter**: o use case depende da interface `PdfRedactorPort`, não do PyMuPDF. Trocar de lib = novo adapter.
- **Privacidade**: nenhum arquivo é escrito em disco persistente; buffers descartados ao fim da request.
- **Idempotência**: reprocessar um PDF já tarjado não quebra (matches = 0 se já removidos).
