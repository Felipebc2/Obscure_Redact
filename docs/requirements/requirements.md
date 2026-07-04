# Requirements — Obscure-Redacted

## Visão

Protótipo funcional que recebe um PDF, aplica **tarja preta real** sobre dados sensíveis
(começando por **CPF**), permite ao usuário **revisar e complementar as tarjas
interativamente** e devolve o PDF tarjado para download. Escopo inicial: MVP local.

Fluxo do usuário:
`upload → tarja automática (CPF) → preview do resultado → tarjas interativas
(clique e/ou palavra-chave) → preview novamente → download`.

## Requisitos Funcionais (RF)

| ID | Requisito | Prioridade (MVP) |
|----|-----------|------------------|
| RF01 | Usuário faz upload de um arquivo PDF pela interface web | Must |
| RF02 | Sistema detecta CPFs no texto do PDF (formatado ou não) | Must |
| RF03 | Sistema valida os dígitos verificadores do CPF para reduzir falso-positivo | Must |
| RF04 | Sistema aplica **redação real** (remove o texto + pinta caixa preta) sobre cada match | Must |
| RF05 | Usuário **visualiza o PDF tarjado** (preview) antes de baixar, para conferir incoerências | Must |
| RF06 | Usuário baixa o PDF tarjado | Must |
| RF07 | Interface exibe a **contagem de tarjas** aplicadas | Should |
| RF08 | Usuário tarja **texto adicional interativamente** após a tarja automática | Should |
| RF08a | — por **palavra-chave** (lista/batch) que o usuário define; opções `whole_word` e normalização de acento | Should |
| RF08b | — por **seleção visual no PDF** (desenho de região no preview) | Could (Dia 4, pós-MVP) |
| RF09 | Cada tarja interativa é **redação real** (mesmo critério de não-extração do RF04) | Must (se RF08) |
| RF10 | Sistema suporta múltiplos padrões via registry (RG, CNPJ, e-mail, telefone) | Could (Dia 3) |
| RF11 | Interface mostra contagem por tipo de dado | Could (Dia 3) |

> **Decisão pendente (RF08):** o método da tarja interativa não está fechado. Opções em
> `frontend.md` (palavra-chave vs. seleção visual). Recomendação: entregar **palavra-chave
> primeiro** (mais simples, sem mapeamento de coordenadas de viewport) e **seleção visual**
> em seguida. Ambos reusam a mesma redação real do backend.

## Requisitos Não-Funcionais (RNF)

| ID | Requisito |
|----|-----------|
| RNF01 | **Redação real, nunca overlay** — o texto sensível não pode ser extraível no PDF de saída |
| RNF02 | **Stateless / privacidade** — nada persistido no servidor. A tarja interativa usa o **PDF de trabalho mantido no cliente** (blob); cada etapa reenvia os bytes atuais e recebe os atualizados. Ações interativas devem ser **agrupadas (batch)** por chamada, não uma request por clique |
| RNF03 | Apenas PDFs com **camada de texto real** (sem OCR no MVP) |
| RNF04 | Processar um documento típico (< 20 páginas) em poucos segundos |
| RNF05 | Portátil via Docker — mesmo comportamento local e em deploy web futuro |
| RNF06 | Clean Architecture — domínio independente de framework/lib |
| RNF07 | Extensível — novo padrão = novo arquivo no registry, sem alterar o caso de uso |
| RNF08 | **Limite de tamanho de upload** (ex.: 20 MB) — arquivos maiores são rejeitados (`413`) |
| RNF09 | **Robustez contra PDF malicioso** — limite de páginas e timeout de processamento; abrir com `fitz` de forma defensiva (proteção contra PDF-bomb) |

## Critérios de Aceite (Gherkin)

```gherkin
Funcionalidade: Tarjar CPF em PDF

  Cenário: PDF com um CPF válido é tarjado
    Dado um PDF com texto contendo o CPF "529.982.247-25"
    Quando o usuário envia o arquivo para redação
    Então o PDF de saída contém uma caixa preta sobre o CPF
    E o texto "529.982.247-25" não é mais extraível do PDF de saída
    E a contagem de tarjas é 1

  Cenário: Todos os CPFs válidos presentes são tarjados (sem falso-negativo)
    Dado um PDF com 3 CPFs válidos em posições diferentes
    Quando o usuário envia o arquivo para redação
    Então nenhum dos 3 CPFs é extraível do PDF de saída
    E a contagem de tarjas é 3

  Cenário: Sequência de 11 dígitos inválida não é tarjada
    Dado um PDF com texto contendo "111.111.111-11"
    Quando o usuário envia o arquivo para redação
    Então nenhuma tarja é aplicada
    E a contagem de tarjas é 0

  Cenário: PDF sem CPF permanece intacto
    Dado um PDF sem nenhum CPF
    Quando o usuário envia o arquivo para redação
    Então a contagem de tarjas é 0
    E o PDF de saída é retornado normalmente

  Cenário: Arquivo que não é PDF é rejeitado
    Dado um arquivo que não é PDF
    Quando o usuário tenta enviar o arquivo
    Então o sistema retorna um erro de formato inválido

  Cenário: Arquivo acima do limite de tamanho é rejeitado
    Dado um PDF maior que o limite configurado
    Quando o usuário tenta enviar o arquivo
    Então o sistema retorna erro 413 (payload muito grande)

Funcionalidade: Preview antes do download

  Cenário: Usuário confere o resultado antes de baixar
    Dado um PDF já tarjado automaticamente
    Quando o usuário abre o preview
    Então ele vê o documento com as caixas pretas aplicadas
    E o botão de download fica disponível

Funcionalidade: Tarja interativa por palavra-chave

  Cenário: Usuário tarja uma ou mais palavras escolhidas (batch)
    Dado um PDF em preview contendo a palavra "Confidencial" 2 vezes
    Quando o usuário informa a lista de palavras-chave ["Confidencial"]
    Então as 2 ocorrências recebem redação real
    E o texto "Confidencial" não é mais extraível do PDF de saída
    E a contagem de tarjas aumenta em 2

  Cenário: whole_word evita tarjar substring
    Dado um PDF contendo "Banana"
    Quando o usuário informa a palavra-chave "Ana" com whole_word ativo
    Então nenhuma tarja é aplicada em "Banana"

Funcionalidade: Tarja interativa por seleção visual

  Cenário: Usuário tarja uma região selecionada
    Dado um PDF em preview
    Quando o usuário desenha uma região sobre um trecho da página
    Então todo texto cujo bbox intersecta a região recebe redação real
    E esse texto não é mais extraível do PDF de saída
```

## Fora de escopo (MVP)

- OCR / PDFs escaneados (imagem).
- Autenticação de usuário.
- Persistência / histórico / audit-log (previsto na fase 2 web).
- Histórico completo de desfazer/refazer. O cliente guarda **1 checkpoint** (PDF original ou pós-tarja-automática) para "resetar" — não um histórico de N versões.
