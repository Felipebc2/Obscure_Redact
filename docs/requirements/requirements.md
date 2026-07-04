# Requirements — Obscure-Redacted

## Visão

Protótipo funcional que recebe um PDF, aplica **tarja preta real** sobre dados sensíveis
(começando por **CPF**) e devolve o PDF tarjado para download. Escopo inicial: MVP local.

## Requisitos Funcionais (RF)

| ID | Requisito | Prioridade (MVP) |
|----|-----------|------------------|
| RF01 | Usuário faz upload de um arquivo PDF pela interface web | Must |
| RF02 | Sistema detecta CPFs no texto do PDF (formatado ou não) | Must |
| RF03 | Sistema valida os dígitos verificadores do CPF para reduzir falso-positivo | Must |
| RF04 | Sistema aplica **redação real** (remove o texto + pinta caixa preta) sobre cada match | Must |
| RF05 | Usuário baixa o PDF tarjado | Must |
| RF06 | Interface exibe a **contagem de tarjas** aplicadas | Should |
| RF07 | Sistema suporta múltiplos padrões via registry (RG, CNPJ, e-mail, telefone) | Could (Dia 3) |
| RF08 | Interface mostra contagem por tipo de dado | Could (Dia 3) |

## Requisitos Não-Funcionais (RNF)

| ID | Requisito |
|----|-----------|
| RNF01 | **Redação real, nunca overlay** — o texto sensível não pode ser extraível no PDF de saída |
| RNF02 | **Stateless / privacidade** — arquivo processado em memória/temp e descartado; nada persistido no MVP |
| RNF03 | Apenas PDFs com **camada de texto real** (sem OCR no MVP) |
| RNF04 | Processar um documento típico (< 20 páginas) em poucos segundos |
| RNF05 | Portátil via Docker — mesmo comportamento local e em deploy web futuro |
| RNF06 | Clean Architecture — domínio independente de framework/lib |
| RNF07 | Extensível — novo padrão = novo arquivo no registry, sem alterar o caso de uso |

## Critérios de Aceite (Gherkin)

```gherkin
Funcionalidade: Tarjar CPF em PDF

  Cenário: PDF com um CPF válido é tarjado
    Dado um PDF com texto contendo o CPF "529.982.247-25"
    Quando o usuário envia o arquivo para redação
    Então o PDF de saída contém uma caixa preta sobre o CPF
    E o texto "529.982.247-25" não é mais extraível do PDF de saída
    E a contagem de tarjas é 1

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
```

## Fora de escopo (MVP)

- OCR / PDFs escaneados (imagem).
- Autenticação de usuário.
- Persistência / histórico / audit-log (previsto na fase 2 web).
- Edição manual das tarjas pela UI.
