# Personas

## P1 — Analista Jurídico / Paralegal (usuário primário)
- **Contexto**: precisa anonimizar peças, contratos e petições antes de compartilhar.
- **Dor**: tarjar CPF manualmente (imprimir, riscar, escanear) é lento e falho.
- **Objetivo**: subir o PDF, receber a versão tarjada em segundos.
- **Sucesso**: nenhum CPF permanece extraível no documento entregue.

## P2 — DPO / Encarregado LGPD (stakeholder de conformidade)
- **Contexto**: responde por vazamento de dado pessoal (art. 5º LGPD).
- **Dor**: overlay falso dá sensação de segurança mas o dado continua extraível.
- **Objetivo**: garantir **redação real** e processo sem retenção de dados.
- **Sucesso**: auditoria confirma que o texto sensível foi removido, não coberto.

## P3 — Desenvolvedor (mantém/estende o sistema)
- **Contexto**: precisa adicionar novos padrões (RG, CNPJ, e-mail, telefone).
- **Dor**: acoplar detecção à lib de PDF torna cada novo padrão arriscado.
- **Objetivo**: adicionar um padrão registrando uma classe `Pattern`, sem tocar no fluxo.
- **Sucesso**: novo padrão = 1 arquivo novo + 1 linha no registry + teste.

## P4 — Cidadão / Titular do dado (beneficiário indireto)
- **Contexto**: é a pessoa cujo CPF aparece no documento.
- **Objetivo**: que seu dado não vaze quando o documento circula.
- **Sucesso**: seu CPF não é recuperável do arquivo publicado.
