# ROADMAP — Obscure-Redacted

> Protótipo funcional (não bonito): upload PDF → tarja **real** de dados sensíveis
> (CPF primeiro) → **preview** → **tarja interativa** (palavra-chave/seleção) → download.
> Princípio nº 1: **redação real, nunca overlay** (ver `docs/requirements/stack.md`).
>
> - **MVP core (Dias 1–3, 24h)**: backend, frontend, preview, tarja por palavra-chave, extensibilidade, Docker local.
> - **Dia 4** — seleção visual por viewport (RF08b, cortável do MVP) + revisão geral.
> - **Dia 5** — deploy web (fase 2): storage, banco, auth, Azure/Traefik.

Legenda: `[ ]` pendente · `[x]` feito. Estimativas entre parênteses.

---

## Dia 1 (8h) — Backend core

Objetivo: `POST /redact` funcional que tarja CPF de verdade num PDF.

- [ ] **Spike de coordenadas** (0.5h) — **risco alto, fazer primeiro**
  - validar `page.search_for(match)` num PDF real; CPF formatado, não-formatado, cruzando linha
  - decidir fallback (mapa palavra→bbox) antes de escrever o core
- [ ] **Scaffold + ambiente** (1h)
  - `python -m venv` + `requirements.txt` (fastapi, uvicorn, python-multipart, pymupdf, pytest, reportlab)
  - Dockerfile base do backend
- [ ] **`domain/patterns` — CPF** (2h)
  - `base.py` (`Pattern` ABC: `name`, `find(text) -> spans`)
  - `cpf.py` (regex `\d{3}\.?\d{3}\.?\d{3}-?\d{2}` + validação dos 2 dígitos verificadores)
  - `registry.py` (`PatternRegistry` com CPF ativo)
  - `test_cpf.py` — válidos passam, `111.111.111-11`/sequências falham
- [ ] **`application` — use case + port** (1.5h)
  - `ports/pdf_redactor_port.py` (`redact_patterns` / `redact_keywords` / `redact_regions`)
  - `use_cases/redact_document.py` (`RedactDocument` orquestra registry + port)
  - `entities/redaction_match.py`, `entities/redaction_result.py`, `entities/region.py`
  - `test_redact_document.py` — conta matches corretos
- [ ] **`infrastructure/pdf` — PyMuPDFRedactor** (2h)
  - `add_redact_annot(bbox, fill=(0,0,0))` + `apply_redactions()`; via `search_for` + fallback
  - fixtures via `reportlab` (CPF válido/inválido em **posições variadas**: formatado, não-formatado, tabela)
  - `test_pymupdf_redactor.py` — **crítico**: reabrir saída, CPF não extraível em `get_text()`
  - `test_no_false_negative.py` — **crítico**: N CPFs válidos → todos ausentes, contagem = N
- [ ] **`web` — endpoints + hardening** (1h)
  - `POST /redact` (multipart `file`) → stream `application/pdf` + `X-Redaction-Count` / `X-Redaction-Counts`
  - `GET /health` → `{"status":"ok"}`
  - **hardening**: limite de tamanho (→ `413`), abertura defensiva (→ `400`), limite de páginas/timeout
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
- [ ] **Integração API + estado do PDF de trabalho** (1.5h)
  - `POST /redact` com `FormData`; resposta `blob` vira `workingPdf` (cliente segura)
  - `components/ResultPanel.tsx` — lê `X-Redaction-Count(s)`, botão "Baixar PDF tarjado"
- [ ] **Preview (pdf.js)** (1.5h)
  - **setup do worker** (`use client` + `next/dynamic` `ssr:false` + `pdf.worker`) — **risco, não é plug-and-play**
  - `components/PdfPreview.tsx` — renderiza `workingPdf` (`pdfjs-dist`/`react-pdf`), **só a página atual** (lazy)
  - estado `preview`; botão "Resetar" = 1 checkpoint (não histórico); usuário confere antes de baixar (RF05)
- [ ] **E2E local (Playwright)** (1h)
  - abrir página → upload fixture → Tarjar → ver preview → validar contagem
- [ ] **Erros / edge cases** (2h)
  - não-PDF bloqueado antes de enviar; `4xx/5xx` → mensagem amigável
  - PDF sem CPF → "0 tarjas" (não é erro); PDF sem texto; arquivo grande

**Fim Dia 2**: fluxo completo no navegador (upload → tarja auto → **preview** → download), local, backend + frontend juntos.

---

## Dia 3 (8h) — Tarja interativa + extensibilidade + deploy prep

Objetivo: usuário complementa as tarjas + novos padrões via registry + empacotamento.

- [ ] **Tarja interativa — palavra-chave** (2h) — RF08a (recomendado primeiro)
  - backend: `use_cases/redact_by_keyword.py` + `POST /redact/keyword` — aceita **lista (batch)**
  - opções `whole_word` + `normalize_accents` (evitar "Ana" em "Banana", ignorar acento)
  - frontend: `components/KeywordRedact.tsx`; reenvia `workingPdf`, atualiza preview + contagem
  - `test_redact_keyword.py` — palavra tarjada de verdade (não extraível); teste de `whole_word`
- [ ] **Novos padrões** (2h)
  - `rg.py`, `cnpj.py`, `email.py`, `telefone.py` — cada um 1 arquivo + 1 linha no registry + teste
  - validação de dígito onde aplicável (CNPJ)
- [ ] **Contagem por tipo na UI** (1h)
  - exibir `X-Redaction-Counts` (ex.: `CPF: 3 · e-mail: 1`)
- [ ] **Docker + compose** (2h)
  - Dockerfile frontend; `docker-compose.yml` sobe os dois serviços
  - `docker-compose up` reproduz o fluxo local
- [ ] **Docs + README** (1h)
  - atualizar `README.md` (como rodar, testar)

**Fim Dia 3 (fecha o MVP)**: tarja interativa por palavra-chave, múltiplos padrões,
contagem por tipo, sobe via `docker-compose`, docs atualizadas.

> Seleção visual (RF08b) foi movida para o **Dia 4** (dedicado), por causa da complexidade
> de coordenadas. O MVP fecha no Dia 3 sem ela.

---

## Dia 4 (8h) — Seleção visual (viewport) + revisão

Objetivo: fechar a feature **cortável** de tarja por seleção visual (RF08b) e revisar
tudo que foi feito nos Dias 1–3. É pós-MVP: só entra depois do core estável.

- [ ] **Spike de mapeamento viewport → PDF** (1h) — **risco alto, fazer primeiro**
  - com pdf.js, converter coordenada do canvas para pontos do PDF (`viewport.convertToPdfPoint`)
  - validar nos casos que quebram: **página rotacionada**, `CropBox`/`MediaBox` deslocado, zoom
  - se o mapeamento não fechar de forma confiável, **reavaliar/cortar** a feature aqui
- [ ] **Backend — regions** (2h)
  - `use_cases/redact_regions.py` + `POST /redact/regions` — aceita **lista (batch)** de `Region` → `fitz.Rect`
  - redação real; remove texto que **intersecta** o retângulo
  - `test_redact_regions.py` — texto sob a região some do `get_text()`; testar página rotacionada
- [ ] **Frontend — desenhar região no preview** (2.5h)
  - `components/RegionRedact.tsx` — modo seleção: desenhar retângulo sobre o canvas do preview
  - acumular regiões (batch) e enviar em uma chamada; atualizar preview + contagem
  - avisar que a tarja pega o texto que **intersecta** a região
- [ ] **Revisão geral (Dias 1–4)** (2.5h)
  - rerodar toda a suíte (`pytest` + Playwright); conferir critérios de "pronto" do MVP
  - revisar segurança: redação real em **todos** os caminhos (auto, palavra-chave, região)
  - **Conselho** (4 personas) sobre o resultado; registrar decisões no Obsidian
  - limpar dívidas/TODOs, atualizar docs onde divergiu da implementação

**Fim Dia 4**: seleção visual funcional (ou decisão consciente de cortar) + base revisada e estável.

---

## Dia 5 (8h) — Deploy web (fase 2)

Objetivo: sair do local e subir a aplicação na web, com storage, persistência e auth.
Sai do modelo puramente stateless/cliente para uma arquitetura web com backing services.

- [ ] **Storage de arquivos** (1.5h)
  - **MinIO/S3** para os PDFs (em vez do blob no cliente); upload/download via URLs assinadas
  - política de retenção curta / expiração (privacidade)
- [ ] **Banco + persistência** (2h)
  - **Supabase/Postgres + SQLAlchemy**; audit-log de redações (o quê, quando, contagem — **sem** o dado sensível)
  - migrations
- [ ] **Autenticação** (1.5h)
  - **JWT**; proteger os endpoints de redação; escopo por usuário
- [ ] **Deploy + infra** (2.5h)
  - **Docker** em produção; **Traefik** (TLS/roteamento) na **Azure**
  - variáveis de ambiente/secrets; CORS de produção; `NEXT_PUBLIC_API_URL` apontando pro backend web
  - rate-limit e limites de tamanho no edge

**Fim Dia 5**: aplicação acessível na web, com storage, banco, auth e HTTPS.

> ⚠️ O Dia 5 muda premissas do MVP (stateless, sem banco). Rever `requirements.md` (RNF02)
> e `stack.md` antes de começar — a privacidade agora depende de retenção curta + audit sem PII.

---

## Critérios de "pronto" (MVP)

- 100% dos CPFs válidos de teste tarjados e **não extraíveis** na saída (zero falso-negativo).
- 0 falso-positivo em `111.111.111-11` / sequências inválidas.
- PDF < 20 páginas processado em poucos segundos.
- Preview reflete fielmente o PDF de saída.
- Tarja interativa (palavra-chave) aplica redação real (não extraível).
- Novo padrão adicionado **sem alterar** o use case.
- `docker-compose up` roda o fluxo ponta a ponta.
