# PRD — Obscure-Redacted

> Product Requirements Document. Condensa requirements, stack, personas, backend e frontend.

## 1. Visão
Ferramenta web simples que aplica **tarja preta real** em dados sensíveis de PDFs.
MVP foca em **CPF** automático; o usuário então **revisa o resultado (preview)** e pode
**complementar as tarjas interativamente** (por palavra-chave e/ou seleção visual) antes
de baixar. A arquitetura permite escalar para RG, CNPJ, e-mail e telefone apenas
registrando novos padrões.

## 2. Problema
Tarjar dados sensíveis manualmente é lento e inseguro. Ferramentas que só desenham
um retângulo por cima deixam o texto **extraível** — falsa sensação de segurança e
risco de vazamento (LGPD). Além disso, a detecção automática nunca é 100%: o usuário
precisa **conferir** e **tarjar trechos extras** que o sistema não pegou.

## 3. Personas (resumo)
- **Analista jurídico** — anonimiza documentos rapidamente e revisa/complementa as tarjas (primário).
- **DPO/LGPD** — exige redação real e não-retenção de dados.
- **Desenvolvedor** — estende padrões sem tocar no fluxo.
- **Titular do dado** — não quer seu CPF recuperável.

## 4. Objetivos e não-objetivos
**Objetivos (MVP)**: upload PDF → detectar/validar CPF → redação real → **preview** →
**tarja interativa** (palavra-chave; seleção visual como próximo passo) → download + contagem.
**Não-objetivos (MVP)**: OCR/escaneado, login, persistência/audit, desfazer no servidor.

## 5. Princípio de design nº 1 — Redação real
`PyMuPDF: add_redact_annot + apply_redactions` remove o texto subjacente e pinta a caixa.
**Nunca overlay.** Vale para a tarja automática **e** para a interativa.
Critério de aceite: o texto tarjado não pode ser extraível na saída.

## 6. Stack (meio-termo, 100% na stack do usuário)
Backend **FastAPI + Pydantic + PyMuPDF** · Frontend **Next.js + Tailwind + pdf.js** (preview) ·
Testes **Pytest + Playwright** · **Docker** · Clean Architecture.
Fase 2 (web): **MinIO/S3** + **Supabase/Postgres + SQLAlchemy** + **JWT**, deploy Azure/Traefik.

## 7. Fluxo
`UI upload → /redact (CPF, redação real) → PDF de trabalho no cliente → preview (pdf.js) →
laço interativo (/redact/keyword | /redact/regions) → preview → download`.
**Stateless**: nada persistido; o PDF de trabalho vive no cliente e é reenviado por etapa.

## 8. Arquitetura
Clean Architecture. Dependências: `web/pdf → application → domain`.
Domínio (patterns) puro; PyMuPDF isolado atrás de `PdfRedactorPort` (métodos
`redact_patterns` / `redact_keyword` / `redact_regions`).
Extensão de padrão = 1 arquivo em `domain/patterns/` + registro no `registry`.

## 9. API
- `GET /health → {status: ok}`
- `POST /redact` (multipart `file`) → `application/pdf` + `X-Redaction-Count(s)`. Auto (patterns).
- `POST /redact/keyword` (multipart `file` + `keyword`) → PDF atualizado. Tarja por palavra.
- `POST /redact/regions` (multipart `file` + `regions` JSON) → PDF atualizado. Tarja por seleção.
- Erros: `400` (não-PDF/corrompido) · `413` (grande demais) · `422` (parâmetro ausente).

## 10. Critérios de aceite (Gherkin) — ver `requirements.md`
CPF válido é tarjado e não extraível; **todos** os CPFs presentes são tarjados (sem
falso-negativo); `111.111.111-11` não é tarjado; PDF sem CPF retorna intacto; não-PDF e
arquivo grande são rejeitados; preview mostra o resultado; palavra-chave/região recebem
redação real.

## 11. Roadmap (resumo) — ver `ROADMAP.md`
MVP core nos Dias 1–3; Dia 4 e 5 são pós-MVP.
- **Dia 1**: backend core (CPF + redação real + endpoint + spike de coordenadas + hardening).
- **Dia 2**: frontend + integração + **preview (pdf.js)** + E2E + erros.
- **Dia 3** (fecha o MVP): **tarja interativa por palavra-chave** + novos padrões + contagem por tipo + Docker/compose + docs.
- **Dia 4** (pós-MVP): **seleção visual por viewport** (RF08b, cortável) + revisão geral dos Dias 1–4.
- **Dia 5** (fase 2): **deploy web** — MinIO/S3, Supabase/Postgres + SQLAlchemy, JWT, Azure/Traefik.

## 12. Métricas de sucesso (MVP)
- 100% dos CPFs válidos de teste tarjados e **não extraíveis** (zero falso-negativo nos fixtures).
- 0 falso-positivo em `111...`/sequências inválidas nos fixtures.
- Documento < 20 páginas processado em poucos segundos.
- Preview reflete fielmente o PDF de saída.
- Tarja interativa aplica redação real (não extraível).
- Novo padrão adicionado sem alterar o use case.
