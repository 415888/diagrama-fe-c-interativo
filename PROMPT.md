# Prompt de especificação — Diagrama Fe-C Interativo

> Este documento descreve, de forma completa e independente de linguagem, o software que existe hoje como `index.html`. Use-o como prompt para uma IA (ou como spec para você mesmo) recriar o projeto em qualquer linguagem/stack — Python, C#, C++, etc. Ele cobre: o que o programa faz, as equações e modelos físicos exatos, os dados/tabelas usados, e os requisitos de interface. Copie este arquivo inteiro como prompt inicial.

## 1. Objetivo do software

Criar uma simulação interativa do diagrama de fases Fe-C (ferro-carbono) acoplada a uma simulação de microestrutura por autômato celular/Monte Carlo, cobrindo:

1. **Diagrama Fe-C arrastável** — o usuário escolhe teor de carbono (%C) e duas temperaturas (austenitização `Ta` e transformação `Tt`) arrastando pontos sobre o diagrama.
2. **Simulação de microestrutura** em uma grade 2D (360×240 células, 2 µm/célula):
   - Crescimento de grão austenítico (modelo de Potts/Monte Carlo).
   - Transformação controlada por difusão do carbono (leis de Fick): ferrita ou cementita pró-eutetoide, depois perlita.
   - Têmpera: conversão de austenita residual em martensita + austenita retida (Koistinen–Marburger), com Ms dependente do %C local.
3. **Módulo de cementação**: solução analítica da 2ª lei de Fick (função erro) para perfil de carbono em peça cementada, com tabelas de erf e de difusividade, e "problemas inversos" (profundidade de camada, tempo necessário).
4. **Tabela de aços SAE/AISI** posicionados no diagrama, com faixas de temperatura de tratamento térmico calculadas automaticamente.
5. **Visualização em escala atômica** (células CFC da austenita e CCC da ferrita) e gráficos de fração de fase × tempo, crescimento de grão, e perfil de carbono ao longo de uma linha da micrografia.

O programa é 100% determinístico dado um seed de RNG, parametrizado por sliders, e roda em tempo real (loop de animação).

## 2. Constantes e dados físicos usados

```
A1 = 727 °C         (temperatura eutetoide)
CE = 0.76 %C        (composição eutetoide)
CA = 0.022 %C       (solubilidade máx. de C na ferrita, a A1)
CCM = 6.70 %C       (composição da cementita, Fe3C)
RCAL = 1.987 cal/(mol·K)   (constante dos gases, unidades cal)
RJ = 8.314 J/(mol·K)       (constante dos gases, unidades SI)
CELL = 2 µm         (tamanho de cada célula da grade)
GRID = 360 x 240 células
```

### 2.1 Linhas do diagrama Fe-C (coordenadas %C, °C)

- **Liquidus**: (0, 1538) → (0.53, 1493) → (4.30, 1147)
- **Solidus δ**: (0, 1538) → (0.09, 1493)
- **Peritética**: (0.09, 1493) → (0.53, 1493)
- **δ/γ**: (0, 1394) → (0.09, 1493) e (0, 1394) → (0.17, 1493)
- **Solidus γ**: (0.17, 1493) → (2.14, 1147)
- **Eutética**: (2.14, 1147) → (6.70, 1147)
- **Acm**: de (0.76, 727) até (2.14, 1147), extrapolável abaixo de 727 °C por `TAcm(C) = A1 + 304.3*(C - CE)`
- **A3** (tabela de pontos, interpolação linear):
  `[[0,912],[0.1,880],[0.2,852],[0.3,828],[0.4,806],[0.5,787],[0.6,768],[0.7,744],[0.76,727]]`
  Acima de 0.76 %C, extrapolar como `A1 - 300*(C - CE)`.
- **A1 (eutetoide)**: reta horizontal em 727 °C de (0.022, 727) a (6.70, 727)
- **Solvus α**: (0, 912) → (0.022, 727) → (0.006, 400)
- **Linha da cementita**: vertical em C = 6.70, de 1252 °C até o limite inferior do gráfico
- **Extrapolações abaixo de A1** (tracejadas): a partir de (0.76, 727), para A3 estendida: `(CE + (A1-T)/300, T)`; para Acm estendida: `(CE - (A1-T)/304.3, T)`

### 2.2 Regra da alavanca / fração de fase pró-eutetoide em equilíbrio

```
Abaixo de A1 (equilíbrio final):
  hipoeutetoide (C < 0.76): W_ferrita,pró = (0.76 - C) / (0.76 - 0.022)
  hipereutetoide (C > 0.76): W_cementita,pró = (C - 0.76) / (6.70 - 0.76)

Entre A1 e A3/Acm (duas fases, equilíbrio local):
  hipo: W_pró = (Cg_A3(T) - C) / (Cg_A3(T) - Calpha(T))
  hiper: W_pró = (C - Cg_Acm(T)) / (6.70 - Cg_Acm(T))

onde:
  Calpha(T) = 0.022 * (912-T)/(912-A1)              se T >= A1
            = 0.022 * clamp((T-400)/(A1-400), 0, 1)  se T < A1
  Cg_A3(T)  = interpolação inversa na tabela A3 (composição da austenita sobre a linha A3 em T, entre 727 e 912 °C)
  Cg_Acm(T) = 0.76 + (T - 727)/304.3
```

### 2.3 Difusão do carbono (1ª e 2ª leis de Fick)

```
1ª lei (fluxo):        J = -D * dC/dx
2ª lei (forma diferencial): dC/dt = D * ∇²C

Difusividade do C na austenita (fonte: aula, unidades cal):
  D_C(T) = 0.12 * exp(-32000 / (1.987 * (T + 273.15)))     [cm²/s]

Difusividade do C na austenita (Callister, alternativa):
  D0 = 2.3e-5 m²/s, Q = 148 kJ/mol

Difusividade do C na ferrita (Callister):
  D0 = 1.1e-6 m²/s, Q = 87 kJ/mol

Difusividade do Fe na austenita/ferrita e do Mn na austenita — ver tabela completa na seção 2.7.

Na simulação em grade: para cada passo de tempo dt,
  r = D(T) * dt / dx²          (dx = CELL em cm)
  dt escolhido para manter D*dt/dx² ≤ 0.2 (estabilidade, diferenças finitas explícitas)
  Fluxo entre células vizinhas von Neumann (4-conectividade para difusão; 8-conectividade para o autômato de fase):
    J(c→j) = r * (Cg[j] - Cg[c])   aplicado simetricamente (ganha um, perde o outro)
```

### 2.4 Crescimento do grão austenítico (modelo de Potts / Monte Carlo)

```
Grade inicial: diagrama de Voronoi com ~520 sementes aleatórias (grão fino inicial).
A cada Monte Carlo Step (MCS), para cada célula (amostragem aleatória, NC tentativas por MCS):
  1. Escolher célula c com orientação/ID de grão v = aus[c]
  2. Escolher vizinho aleatório j (8-conectividade), com orientação nv = aus[j]
  3. Se nv == v ou nv < 0 (partícula), pular
  4. Energia local = número de vizinhos (dos 8) com orientação diferente da própria
  5. dE = energiaA(c, nv) - energiaA(c, v)
  6. Se dE < 0, ou (dE == 0 e sorteio < 0.5): aceitar a mudança (c assume orientação nv)

Mobilidade do contorno (depende da temperatura de austenitização Ta):
  mobility(Ta) = clamp(exp((Ta - 860)/55), 0.1, 15)

Número total de MCS simulados = min(1500, round(tempo_de_encharque * mobility(Ta)))

Partículas ancoradoras de contorno (Zener pinning): fração configurável de blocos 2x2 marcados
como "partícula" (fase especial, imóvel, bloqueia o crescimento de grão) — representa carbonetos/
nitretos de microligantes (Nb, Ti, V, Al) em aços HSLA.
```

### 2.5 Nucleação e crescimento da fase pró-eutetoide (ferrita ou cementita)

```
Nucleia preferencialmente nos contornos de grão austenítico (pagB[c] = true se célula tem vizinho
de orientação austenítica diferente).

Taxa de nucleação (por célula de contorno, por segundo):
  s3 = clamp((T_crítica - Tt) / 150, 0.05, 1)      onde T_crítica = TA3(C) ou TAcm(C)
  rate = 2e-4 * 10^(1.5*s3) * (3 se cementita, senão 1) * fator_ajuste_N

Crescimento: a célula acumula uma fração de fase φ (0 a 1); ao completar 1, a célula "congela"
na fase pró-eutetoide. A taxa de avanço depende da supersaturação local:
  drive = Cg_eq(Tt) - Cg[c]           (ferrita: austenita mais pobre que o equilíbrio)
  drive = Cg[c] - Cg_eq(Tt)           (cementita: austenita mais rica que o equilíbrio)
  dphi = min(0.25, 1.2 * r * drive / |Cg_eq - Cp| * (nº vizinhos da mesma fase) * fator_geométrico)

O carbono rejeitado (ferrita) ou consumido (cementita) é redistribuído para as células austeníticas
vizinhas "à frente" da interface (não ainda enriquecidas), simulando o "morro" de carbono rejeitado
que avança por difusão.
```

### 2.6 Nucleação e crescimento da perlita

```
Só ocorre na "janela pseudoeutetoide": a austenita vizinha precisa ter composição dentro de
  [CE - (A1-T)/304.3, CE + (A1-T)/300]
(extrapolação das linhas A3/Acm abaixo de A1).

Velocidade de crescimento linear G(T), tabela de Mehl e Dube (interpolação linear em T, mm/s):
  T(°C): 560   600   620   640   660   680   690   700   715   727
  log10(G): -2.15 -2.15 -2.2  -2.3  -2.5  -3.0  -3.2  -3.5  -4.0  -6

Taxa de nucleação N(T):
  s = clamp((A1 - T)/160, 0.04, 1)
  rateN = 10^(-4.2 + 3*s) * fator_ajuste_N

Nucleia preferencialmente em contornos/interfaces (ou em qualquer ponto, conforme opção do usuário).
Crescimento modelado com sub-passos para garantir G*dt ≤ 0.5 célula; opcionalmente anisotrópico
(frente "lamelar" orientada, com parâmetro de forma ajustável — perlita lisa vs. ramificada/fractal).
Dimensão fractal da frente de crescimento pode ser medida por box-counting.
Fração de perlita formada é comparada ao Avrami: ln(-ln(1-X)) vs ln(t), inclinação = expoente n.
```

### 2.7 Têmpera — transformação martensítica

```
Temperatura de início da martensita Ms, em função do %C (tabela, Krauss; concorda com
Andrews Ms = 539 - 423*C até ~0.6 %C):
  %C:  0    0.2   0.4   0.6   0.8   1.0   1.2   1.4   1.6   2.0
  Ms:  540  460   380   310   250   200   160   130   100   60
(interpolação linear entre pontos; abaixo de 2.0, Ms = 60 - 80*(C-2))

Fração de martensita formada — equação de Koistinen–Marburger:
  f_M(C, T) = 1 - exp[-0.011 * (Ms(C) - T)]     se Ms(C) > T, senão 0

O que não transforma em martensita fica como austenita retida.
Cada célula de austenita vira martensita usando o SEU teor de carbono local (herdado do campo Cg
resolvido por difusão), não o C0 nominal — por isso regiões com acúmulo/depleção de carbono geram
martensita + austenita retida heterogêneas.

Morfologia: dentro de cada grão austenítico prévio, dividir em "pacotes" (Voronoi secundário) com
4 variantes de orientação por pacote; dentro de cada pacote, desenhar ripas/placas com espaçamento
lamelar dependente do %C local:
  lamM(C) = 2.6                      se C < 0.6
          = 6.5                      se C > 1.0
          = 2.6 + (C-0.6)/0.4 * 3.9   interpolado entre 0.6 e 1.0
(ripas finas em baixo carbono, placas largas em alto carbono/austenita retida)

Dureza aproximada de 100% martensita em função do %C (tabela, Aula 8):
  %C:   0.1  0.2  0.3  0.4  0.5  0.6  0.7  0.8  1.0  1.4
  HRC:  39   46   51   56   60   63   64   65   66   66
(interpolação linear)

Opções de têmpera: direta (da temperatura de austenitização), ou após espera em Tt por um tempo
configurável (têmpera incompleta — ferrita/perlita já formada não retransforma); meio de têmpera
até 25 °C ou subzero (-196 °C, para reduzir austenita retida).
```

### 2.8 Cementação — solução analítica da 2ª lei de Fick (função erro)

```
Para um sólido semi-infinito com D constante, superfície mantida em Cs, núcleo em C0:
  C(x,t) = Cs - (Cs - C0) * erf( x / (2*sqrt(D*t)) )

erf(z) aproximação numérica (Abramowitz & Stegun 7.1.26):
  t = 1/(1 + 0.3275911*|z|)
  erf(z) = sign(z) * [1 - (a1*t + a2*t² + a3*t³ + a4*t⁴ + a5*t⁵) * exp(-z²)]
  a1=0.254829592, a2=-0.284496736, a3=1.421413741, a4=-1.453152027, a5=1.061405429

Também fornecer uma TABELA de erf(z) para interpolação linear manual (valores de z de 0 a 2.8,
ver código-fonte `ERF` para a lista completa de 35 pontos) — importante para reproduzir o método
didático de "ler na tabela e interpolar" usado na disciplina.

Parâmetros de entrada: Cs (teor na superfície), temperatura de cementação, tempo de cementação,
teor que define a "camada cementada" (ex. 0.4 %C), e dois "problemas inversos":
  (a) dado um teor-alvo Cx, achar a profundidade x onde ele ocorre;
  (b) dado um teor-alvo Cx e uma profundidade x, achar o tempo necessário.
Ambos resolvidos invertendo a função erro (por tabela, com interpolação).

Resultado pode ser usado como condição inicial de carbono não-uniforme na simulação principal.
```

### 2.9 Tabela de difusividades (D0, Q) para referência

```
Espécie | Meio   | D0 (m²/s) | Q (kJ/mol) | Fonte
C       | Fe-γ   | 1.2e-5    | 133.9      | Aula (0.12 cm²/s; 32000 cal/mol)
C       | Fe-γ   | 2.3e-5    | 148        | Callister
C       | Fe-α   | 1.1e-6    | 87         | Callister
Fe      | Fe-γ   | 5.0e-5    | 284        | Callister
Fe      | Fe-α   | 2.8e-4    | 251        | Callister
Mn      | Fe-γ   | 1.78e-5   | 264        | literatura (aproximado)

D(T) = D0 * exp(-Q*1000 / (8.314 * (T + 273.15)))   [resultado em m²/s; multiplicar por 1e4 para cm²/s]
```

### 2.10 Escala atômica

```
Parâmetro de rede da austenita (CFC), em Å, função do %C:
  a_γ(C) = 3.555 + 0.044*C

Parâmetro de rede da ferrita (CCC): a_α = 2.866 Å (constante)

Raio do átomo de carbono: 0.77 Å (não cabe em nenhum sítio octaédrico sem distorcer a rede —
daí a diferença de solubilidade entre austenita de 2.14 %C e ferrita de 0.022 %C).

Espaçamento interlamelar da perlita (aproximação empírica para o eutetoide):
  S(T) = 8.02 / (A1 - T)   [µm], válido para T < A1 - 2
```

### 2.11 Tabela de aços SAE/AISI

```
Nome    | %C min | %C max | Ligado? | Aplicação típica
1006    | 0.02   | 0.08   | não     | Chapas para estampagem profunda
1010    | 0.08   | 0.13   | não     | Chapas, arames, peças para cementar
1020    | 0.18   | 0.23   | não     | Estruturas, eixos leves, peças cementadas
1030    | 0.28   | 0.34   | não     | Peças forjadas, parafusos
1045    | 0.43   | 0.50   | não     | Eixos, engrenagens, virabrequins
4140    | 0.38   | 0.43   | sim     | Eixos e engrenagens temperados (Cr-Mo)
4340    | 0.38   | 0.43   | sim     | Peças de alta resistência, grandes seções (Ni-Cr-Mo)
1060    | 0.55   | 0.65   | não     | Molas, rodas ferroviárias
8660    | 0.55   | 0.65   | sim     | Molas e peças temperadas (Ni-Cr-Mo)
1070    | 0.65   | 0.75   | não     | Molas, lâminas
1080    | 0.75   | 0.88   | não     | Arames de alta resistência, trilhos
1095    | 0.90   | 1.03   | não     | Molas, ferramentas de corte
52100   | 0.98   | 1.10   | sim     | Rolamentos (1.45% Cr)

Classes por %C: Baixo C [0,0.25) · Médio C [0.25,0.60) · Alto C [0.60,1.00) ·
                Ultra-alto C [1.00,2.14) · Ferros fundidos [2.14,6.70]

Faixas de tratamento térmico (regras práticas, Aula 7 / Krauss-ASM):
  Recozimento pleno / têmpera: hipo → A3+30 a A3+50 °C ; hiper → A1+30 a A1+50 °C
  Normalização:                hipo → A3+50 a A3+80 °C ; hiper → Acm+50 a Acm+80 °C
  Esferoidização: 700–720 °C          Recristalização: 600–700 °C          Alívio de tensões: 550–650 °C
```

## 3. Requisitos de interface (UI/UX)

1. **Painel de simulação** (canvas principal, 360×240 "pixels" = células):
   - Renderização colorida por fase (austenita, ferrita, cementita, perlita com lamelas, martensita, austenita retida, partículas).
   - Modo alternativo "micrografia Nital" (tons de cinza, como em microscopia óptica real).
   - Opção de mapa de calor de %C na austenita.
   - Rótulo de fase/estágio atual, barra de escala, legenda dinâmica.
   - Estatísticas ao vivo: tempo, diâmetro de grão, % fase pró-eutetoide, % perlita, dimensão fractal, Hall-Petch (limite de escoamento estimado), % martensita, % austenita retida, dureza aproximada.

2. **Diagrama Fe-C interativo** (canvas, arrastável):
   - Dois modos de zoom: diagrama completo (0–6.70 %C, 400–1600 °C) e região dos aços (0–1.7 %C, 0–1150 °C).
   - Dois marcadores arrastáveis (ponto de austenitização, ponto de transformação) ligados a sliders de %C, Ta, Tt.
   - Camadas opcionais: faixas de aços SAE/AISI sobre o eixo de composição; faixas de tratamento térmico (normalização, recozimento, esferoidização, recristalização, alívio de tensões) coloridas por região.
   - Linha de amarração (tie-line) desenhada na temperatura de transformação atual.
   - Texto de "veredito" dinâmico explicando o estado previsto (classe do aço, fases esperadas, percentuais).
   - Botões de presets rápidos (ex: "1020", "Eutetoide", "Têmpera 1045", "Austenita retida", etc.) que ajustam todos os sliders de uma vez.

3. **Painel de parâmetros** (sliders): tempo de encharque, partículas ancoradoras, forma da frente da perlita, ajustes de nucleação/crescimento, modo de nucleação (contornos vs. aleatório), modo e parâmetros de têmpera, paleta de cores, unidade de comprimento (µm/nm/Å), velocidade de animação; botões play/pause, reiniciar, nova amostra (novo seed).

4. **Gráficos** (2 lado a lado): fração de fase × tempo (com linha de equilíbrio tracejada e marcador de têmpera); crescimento do grão austenítico × MCS.

5. **Perfil de carbono**: gráfico de %C ao longo de uma linha horizontal da micrografia (clicável/arrastável para escolher a linha), com faixas e linhas de referência (C0, composição de equilíbrio na interface, janela pseudoeutetoide).

6. **Painel de cementação**: sliders de Cs, temperatura, tempo, seletor de fonte de D; gráfico do perfil erf; passo a passo textual do cálculo (equações com números substituídos); tabelas de erf e de difusividades com linhas destacadas conforme o cálculo atual; checkbox para usar esse perfil como condição inicial da simulação principal.

7. **Tabela de aços**: lista clicável (ao clicar, leva os sliders ao aço escolhido), barra de classificação por %C, explicação da nomenclatura SAE/AISI e das regras de temperatura usadas.

8. **Escala atômica**: "escada" de comprimentos característicos (célula da simulação, grão, espaçamento interlamelar, parâmetro de rede, distâncias de difusão de C e Mn) e desenhos em escala das células CFC (austenita) e CCC (ferrita) com o átomo de carbono e o sítio octaédrico.

## 4. Arquitetura sugerida por linguagem (para reimplementação)

- **Python**: núcleo numérico em NumPy (grade como array 2D, operações vetorizadas para difusão; loop do Potts pode ficar em Python puro ou usar Numba/Cython se precisar de mais velocidade); UI em PyQt/PySide (QGraphicsView ou QPainter para o canvas) ou Streamlit+Plotly (mais simples, roda no navegador local).
- **C#**: núcleo numérico em C# puro ou com arrays nativos; UI em WPF (Canvas/WriteableBitmap para a grade) com data binding para os sliders; gráficos com um pacote como OxyPlot ou LiveCharts.
- **C++**: núcleo em C++ puro; UI com Dear ImGui + SDL2/OpenGL (leve, bom para prototipagem) ou Qt (mais robusto, widgets nativos); renderizar a grade como textura atualizada por frame.
- **C**: recomendado só como "motor" (biblioteca compilada) exposto via bindings para Python/C#/C++, não como linguagem de UI.

Em qualquer linguagem, a parte "difícil" de portar é a seção 2 inteira (as equações e os algoritmos de autômato celular) — a interface é secundária e pode ser simplificada se necessário (ex: trocar os componentes de UI por uma versão mais simples, desde que o núcleo físico-numérico da seção 2 seja preservado fielmente).

## 5. Fontes citadas no material original

Aula 2 (diagrama Fe-C, regra da alavanca), Aula 3 (difusão, ferrita alotriomórfica, cementita pró-eutetoide, Avrami, Mehl e Dube), Aula 7 (mapa de tratamentos térmicos, Krauss/ASM), Aula 8 (crescimento de grão austenítico, dureza × carbono, têmpera e temperabilidade), Lista 1 e Lista 3 (SMM0561 — Tratamentos Térmicos de Aços). Dados complementares de Callister (difusividades) e de literatura aproximada para o Mn. Consultar ASM Handbook vol. 4 (Heat Treating) e normas ASTM/SAE para valores de projeto em aços ligados.
