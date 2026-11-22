# Diagrama Fe-C Interativo

Simulação interativa do diagrama Fe-C (ferro-carbono): austenitização, transformação controlada por difusão (leis de Fick), perlita, têmpera (martensita + austenita retida, Koistinen–Marburger) e cementação (solução da 2ª lei de Fick pela função erro).

## Como usar

Basta abrir `index.html` em qualquer navegador — não precisa de servidor, build ou dependências. É um único arquivo HTML autocontido (CSS + JS embutidos).

Para publicar como site (GitHub Pages):
1. No GitHub, vá em **Settings → Pages**
2. Em "Source", selecione a branch `main` e a pasta `/ (root)`
3. Salve — o link público aparecerá em alguns instantes

## Conteúdo

- Diagrama Fe-C arrastável (temperatura de austenitização / transformação, teor de carbono)
- Simulação de microestrutura (grão austenítico, ferrita/cementita pró-eutetoide, perlita, martensita)
- Gráficos de fração de fase × tempo e crescimento de grão
- Perfil de carbono (leis de Fick) e módulo de cementação com tabelas de erf e difusividade
- Tabela de aços SAE/AISI posicionados no diagrama, com faixas de tratamento térmico

## Origem

Criado como artefato interativo no Claude, a partir do material do curso SMM0561 (Tratamentos Térmicos de Aços).
