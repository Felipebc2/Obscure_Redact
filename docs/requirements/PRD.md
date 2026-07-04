# PRD — Obscure-Redacted

> Product Requirements Document. Condensa requirements, stack, personas, backend e frontend.

## 1. Visão
Ferramenta web simples que aplica **tarja preta real** em dados sensíveis de PDFs.
MVP foca em **CPF**; a arquitetura permite escalar para RG, CNPJ, e-mail e telefone
apenas registrando novos padrões.

## 2. Problema
Tarjar dados sensíveis manualmente é lento e inseguro. Ferramentas que só desenham
um retângulo por cima deixam o texto **extraível** — falsa sensação de segurança e
risco de vazamento (LGPD).

## 3. Personas (resumo)
- **Analista jurídico** — anonimiza documentos rapidamente (primário).
- **DPO/LGPD** — exige redação real e não-retenção de dados.
- **Desenvolvedor** — estende padrões sem tocar no fluxo.
- **Titular do dado** — não quer seu CPF recuperável.

## 4. Objetivos e não-objetivos
**Objetivos (MVP)**: upload PDF → detectar/validar CPF → redação real → download + contagem.
**Não-objetivos (MVP)**: OCR/escaneado, login, persistência/audit, edição manual de tarjas.

## 5. Princípio de design nº 1 — Redação real
`PyMuPDF: add_redact_annot + apply_redactions` remove o texto subjacente e pinta a caixa.
**Nunca overlay.** Critério de aceite: o texto tarjado não pode ser extraível na saída.

## 6. Stack (meio-termo, 100% na stack do usuário)
Backend **FastAPI + Pydantic + PyMuPDF** · Frontend **Next.js + Tailwind** ·
Testes **Pytest + Playwright** · **Docker** · Clean Architecture.
Fase 2 (web): **MinIO/S3** + **Supabase/Postgres + SQLAlchemy** + **JWT**, deploy Azure/Traefik.

## 7. Fluxo
`UI upload → FastAPI /redact → UseCase RedactDocument → PatternRegistry (CPF) →
PyMuPDF redação real → PDF stream + X-Redaction-Count → download`.
Stateless: nada persistido no MVP.

## 8. Arquitetura
Clean Architecture. Dependências: `web/pdf → application → domain`.
Domínio (patterns) puro; PyMuPDF isolado atrás de `PdfRedactorPort`.
Extensão de padrão = 1 arquivo em `domain/patterns/` + registro no `registry`.

## 9. API
- `GET /health → {status: ok}`
- `POST /redact` (multipart `file`) → `application/pdf` + headers
  `X-Redaction-Count`, `X-Redaction-Counts`. Erros `400` (não-PDF) / `422` (sem arquivo).

## 10. Critérios de aceite (Gherkin) — ver `requirements.md`
CPF válido é tarjado e não extraível; `111.111.111-11` não é tarjado;
PDF sem CPF retorna intacto; não-PDF é rejeitado.

## 11. Roadmap (resumo) — ver `ROADMAP.md`
- **Dia 1**: backend core (CPF + redação real + endpoint).
- **Dia 2**: frontend + integração + E2E + erros.
- **Dia 3**: novos padrões + contagem por tipo + Docker/compose + docs/deploy.

## 12. Métricas de sucesso (MVP)
- 100% dos CPFs válidos de teste tarjados e **não extraíveis**.
- 0 falso-positivo em `111...`/sequências inválidas nos fixtures.
- Documento < 20 páginas processado em poucos segundos.
- Novo padrão adicionado sem alterar o use case.
