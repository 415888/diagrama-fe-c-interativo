# Proveniência do projeto

Este documento registra de onde o `index.html` veio e contra quais materiais ele foi calibrado,
para que isso não se perca quando o Project do Claude ou o histórico de chat não estiverem mais
disponíveis.

## Artefato original (versão interativa "viva")

Criado no Claude.ai como artefato interativo em **2026-10-07**, dentro do Project **"Tratamentos
Térmicos"**.

- URL do artefato (editável/visualizável enquanto a conta tiver acesso):
  `https://claude.ai/artifact/WNVoD6DknpUcWSVMTpp2Xj`
- O arquivo `index.html` nesta pasta é um **snapshot estático** dessa versão, extraído em
  2026-11-22. Qualquer edição feita depois no artefato do Claude não se reflete aqui
  automaticamente — se continuar editando lá, repita a exportação.

## Disciplina de origem

SMM0561 — Tratamentos Térmicos de Aços (curso de Engenharia de Materiais, EESC-USP).

## Materiais-fonte usados para derivar o modelo físico

Estes arquivos estavam no Project do Claude ("Tratamentos Térmicos") no momento da criação e
foram a base das equações e tabelas reproduzidas no `PROMPT.md`:

- `Aula1_Introdução ao curso.pdf`
- `Aula2_Sistema Fe-C.pdf` — diagrama Fe-C, regra da alavanca
- `Aula3_Decomposição da austenita em F e P.pdf` — difusão, ferrita alotriomórfica, cementita
  pró-eutetoide, Avrami, velocidades de Mehl e Dube
- `Aula4_Decomposição da austenita em Bainita.pdf`
- `Aula5_Decomposição da austenita em M.pdf` — martensita, Ms, Koistinen–Marburger
- `Aula6_Diagramas ITT e CCT.pdf`
- `Aula7_Recozimento e Normalização.pdf` — faixas de tratamento térmico por classe de aço
- `Aula8_Tempera e Temperabilidade.pdf` — crescimento de grão austenítico, dureza × %C
- `Micrografias aços.pdf` — referência visual para as texturas de fase desenhadas
- `Lista1_SMM0561_respostas.pdf`, `Respostas_2a_lista_SMM0561.pdf`,
  `Respostas_3a_lista_SMM0561.pdf`, `Respostas_4a_lista_SMM0561.pdf`, `Lista5_SMM0561_respostas.pdf`
  — usadas para validar/calibrar números específicos (solubilidades, frações de fase, exemplos
  de cementação resolvidos pela função erro)

Esses PDFs **não estão copiados nesta pasta** (ficam no Project do Claude). Se precisar
reconferir uma fórmula contra a fonte original, procure por esses nomes de arquivo lá. Caso o
acesso ao Project se perca, o `PROMPT.md` já contém as fórmulas e tabelas extraídas por completo
— é a cópia de segurança do conteúdo relevante desses PDFs.

## Até onde o modelo foi validado

O modelo numérico (crescimento de grão, difusão, nucleação/crescimento de fase, têmpera,
cementação) foi calibrado para reproduzir ordens de grandeza e comportamentos qualitativos
corretos (ex: espessamento parabólico ∝ √t, janela pseudoeutetoide, curva Ms × %C concordando
com Andrews até 0,6 %C) — não foi validado contra dados experimentais publicados ponto a ponto.
Para uso em trabalho formal (TCC, publicação), trate os valores numéricos exibidos como
ilustrativos/didáticos, e confira contra o ASM Handbook vol. 4 e normas ASTM/SAE antes de citar.

## Convenções de nomenclatura

- Datas no formato `AAAA-MM-DD`.
- Ângulo-pai do repositório: pasta local atual
  (`Downloads\Tratamentos\diagrama-fe-c-interativo`), destinada a virar `git init` + repositório
  próprio no GitHub, independente da pasta `Tratamentos` que guarda outros materiais do curso.
