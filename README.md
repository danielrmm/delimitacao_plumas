# Manual: Delimitação Geométrica de Plumas de Contaminação (rv4)

**Processing Algorithm para QGIS 3.x**

---

## Sobre este manual

Este documento é a referência completa do aplicativo de delimitação de plumas. Está organizado em três partes independentes, cada uma com público e profundidade próprios:

- **Parte I — Guia de Uso**: para quem vai usar o aplicativo no dia a dia. Explica cada parâmetro, mostra cenários típicos com configurações recomendadas, e ensina a interpretar as saídas. Não exige conhecimento de programação ou matemática avançada.
- **Parte II — Fundamentos Teóricos**: para quem quer entender por que o aplicativo funciona. Cobre os algoritmos de geometria computacional, estatística e splines que fundamentam cada decisão técnica. Linguagem didática, partindo do básico até a matemática formal de cada algoritmo.
- **Parte III — Implementação em Python/PyQGIS**: para quem quer entender ou modificar o código. Explica a estrutura de um Processing Algorithm, como ler parâmetros, gerar saídas, chamar outros algoritmos, e organizar o código em funções modulares. Voltado a iniciantes em Python e PyQGIS.

A leitura pode ser feita em qualquer ordem. Recomenda-se começar pela Parte I para uma visão geral, depois aprofundar em II ou III conforme o interesse.

---

# Parte I — Guia de Uso

## 1. O que o aplicativo faz

O aplicativo gera polígonos de delimitação de plumas de contaminação a partir de pontos de amostragem ambiental, classificados como **contaminados** (concentração medida acima do Valor de Investigação) ou **não-contaminados** (concentração abaixo ou igual ao VI). A delimitação é puramente **geométrica e normativa** — não usa interpolação geoestatística como kriging ou IDW. Em vez disso, segue regras matemáticas baseadas na fração da distância entre pontos contaminados e não-contaminados, refletindo a prática técnica brasileira em GAC.

Para cada par de pontos contaminado-vizinho-não-contaminado identificado pela geometria da malha amostral, o aplicativo calcula um **ponto-limite** sobre o segmento que os une. Os pontos-limite formam o contorno do polígono que representa a pluma. Duas regras estão disponíveis: a **regra dos 50%** coloca o ponto-limite no meio do segmento (delimitação balanceada), e a **regra dos 75%** coloca o ponto-limite a três quartos do caminho do contaminado em direção ao não-contaminado (delimitação conservadora, próxima do não-contaminado).

## 2. Quando usar este aplicativo

O aplicativo é adequado para:

- Delimitação preliminar de plumas em investigação detalhada (ID) e investigação confirmatória (IC)
- Apresentação cartográfica em relatórios técnicos (RAS, RBC, planos de remediação)
- Análise rápida de múltiplos contaminantes em paralelo (o algoritmo processa todos os contaminantes da camada de uma vez, agrupando automaticamente)
- Comparação de cenários (regra 50% vs 75%, profundidades diferentes, etc.)

Não é adequado para:

- Modelagem quantitativa de transporte (usar MODFLOW + MT3DMS)
- Estimativa de massa contaminante (usar interpolação geoestatística)
- Modelagem com gradiente hidráulico, anisotropia ou barreiras (requer modelo conceitual do meio físico)
- Substituir avaliação técnica do responsável habilitado — produtos do aplicativo são técnicos auxiliares, não decisões finais

## 3. Instalação no QGIS

A instalação é simples e segue o procedimento padrão de scripts do Processing:

1. Abrir o QGIS 3.x (versão recomendada: 3.40 ou superior, mas funciona a partir de 3.16)
2. Abrir o painel **Processing → Toolbox** (`Ctrl+Alt+T`)
3. Clicar no ícone de Python (parece um pergaminho) no topo da Toolbox e escolher **Add Script to Toolbox**
4. Apontar para o arquivo `delimitacao_plumas_rv4.py`
5. O algoritmo aparecerá em **Áreas Contaminadas (GAC) → Delimitação geométrica de plumas (50% / 75%) — rv4**

Alternativamente, é possível abrir o arquivo no **Script Editor** do Processing, salvá-lo em uma pasta de scripts pessoais, e o algoritmo ficará disponível em todas as sessões.

## 4. Conceitos essenciais antes de começar

### 4.1. Sistema de coordenadas (CRS)

O aplicativo **exige CRS projetado em metros**. Coordenadas geográficas em graus (EPSG:4326, por exemplo) não funcionam porque distâncias em graus não têm significado geométrico real (um grau de longitude no equador equivale a ~111 km, mas no Polo Norte equivale a 0 km). Se o CRS de entrada for geográfico, o algoritmo aborta com mensagem clara.

Para o Brasil, recomenda-se usar **SIRGAS 2000 / UTM** (EPSG:31978 a 31985, dependendo da zona). Se a camada estiver em outro CRS, reprojete antes via **Vector → Data Management → Reproject Layer**.

### 4.2. Estrutura tabular dos dados

A camada de pontos deve estar em formato **longo (long format)**, ou seja, **uma linha por amostragem de contaminante**:

| ID_ponto | Contaminante | Concentração | VI    |
|----------|--------------|--------------|-------|
| PM-01    | Benzeno      | 0,012        | 0,005 |
| PM-01    | Tolueno      | 0,3          | 0,7   |
| PM-02    | Benzeno      | 0,001        | 0,005 |
| PM-02    | Tolueno      | 0,1          | 0,7   |

**Não funciona** se os contaminantes estão em colunas separadas (formato wide). Nesse caso, é preciso fazer pivot/melt antes — em R, função `pivot_longer()` do tidyverse; em Python, `pd.melt()` do pandas.

### 4.3. Concentração igual a zero

Pontos com concentração = 0 são tratados como **não-contaminados** pelo aplicativo, com base no pressuposto de que zero representa "não detectado", "abaixo do limite de detecção (LD)" ou "abaixo do limite de quantificação (LQ)". Se a sua planilha usar zero para indicar "não amostrado" ou "ausente de dados", isso vai gerar resultados incorretos — substitua esses zeros por NULL antes.

### 4.4. Agrupamento por profundidade ou campanha

Em redes amostrais com poços multiníveis (rasos e profundos no mesmo local) ou múltiplas campanhas, **é obrigatório** usar o parâmetro de campos opcionais de agrupamento. Sem isso, o algoritmo trata pontos rasos e profundos como se estivessem no mesmo plano 2D, gerando triangulações degeneradas (pontos sobrepostos com coordenadas X,Y idênticas) e plumas que misturam aquíferos diferentes.

Os campos típicos de agrupamento são:

- **Profundidade** (em metros) ou faixa de profundidade (raso/intermediário/profundo)
- **Campanha** ou data de amostragem
- **Aquífero** ou unidade hidroestratigráfica
- **Horizonte** de solo (camada amostrada)
- **Área investigada** (quando há sub-áreas no projeto)

Pode-se selecionar múltiplos campos simultaneamente — o agrupamento é pela combinação deles.

## 5. Os parâmetros explicados

A interface está organizada em duas camadas: **parâmetros básicos** (sempre visíveis) e **parâmetros avançados** (escondidos sob "Advanced parameters" no diálogo). Os defaults dos avançados são tecnicamente sensatos para a maioria dos casos.

### 5.1. Parâmetros básicos

| Parâmetro | Padrão | Descrição |
|-----------|--------|-----------|
| Camada de pontos de amostragem | — | A camada vetorial de pontos. Pode ser shapefile, GeoPackage, ou camada de memória |
| Processar apenas feições selecionadas | False | Quando ativo, processa apenas o subset selecionado |
| Campo identificador do ponto | — | Coluna com IDs únicos (PM-01, PMN02, etc.). Opcional — se não fornecido, usa o feature ID interno |
| Campo do contaminante | — | Coluna com o nome do parâmetro (Benzeno, Tolueno, etc.) |
| Campo da concentração | — | Coluna numérica com a concentração medida |
| Campo do VI | — | Coluna numérica com o valor de referência legal |
| Campos de agrupamento | (vazio) | Múltipla seleção. Profundidade, campanha, etc. |
| Regra de delimitação | 50% | "Regra 50%" ou "Regra 75%" |
| Modo de geração | Plumas separadas | "Um polígono por grupo" ou "Plumas separadas" (componentes conexas) |
| Suavizar polígono final | True | Liga/desliga a suavização do contorno |
| Método de suavização | Catmull-Rom | "Catmull-Rom" (interpolador, padrão) ou "Chaikin" (aproximador) |
| Extrapolar plumas abertas | False | Liga a detecção e extrapolação de contaminados de borda |

### 5.2. Parâmetros avançados de suavização

A suavização tem dois métodos disponíveis com parâmetros próprios. Ao escolher um método, os parâmetros do outro são ignorados.

**Catmull-Rom** (padrão, interpolador — passa pelos vértices originais):

| Parâmetro | Padrão | Faixa | Função |
|-----------|--------|-------|--------|
| Subdivisões por aresta | 16 | 6–32 | Quantos pontos interpolados entre cada par de vértices. Mais subdivisões = curva mais suave em escalas grandes |
| Tensão da curva | 0,30 | 0,0–1,0 | Controla quanto a curva "estoura" para fora. 0 = sem suavização; 0,5 = Catmull-Rom clássico (~25% expansão); 0,30 = padrão (~13–15% expansão) |

**Chaikin** (alternativa, aproximador — contrai os cantos):

| Parâmetro | Padrão | Faixa | Função |
|-----------|--------|-------|--------|
| Iterações | 2 | 1–5 | Quantas vezes o algoritmo é aplicado em cascata. Cada iteração ~ dobra os vértices |
| Offset | 0,25 | 0,05–0,45 | Onde sobre cada aresta os pontos novos são colocados. 0,25 = Chaikin clássico (contração ~25% na área) |

### 5.3. Parâmetros avançados de filtros e validações

| Parâmetro | Padrão | Faixa | Função |
|-----------|--------|-------|--------|
| Filtro de arestas longas (k×MAD) | 2,5 | 0–10 | Multiplicador para descartar arestas Delaunay muito longas. 0 desativa. 2,0 rigoroso, 3,5 permissivo |
| Validar Voronoi | True | — | Descarta pares contaminado/não-cont espúrios, onde o ponto-limite cai mais perto de outro não-contaminado |
| Método de fechamento | Angular | — | "Ordenamento angular" (rápido, padrão) ou "Concave Hull" (melhor para plumas alongadas) |
| Alpha do Concave Hull | 0,30 | 0,05–1,0 | Controla concavidade. 1,0 = convex hull; 0,30 = concavo moderado; 0,1 = muito côncavo |
| Buffer fallback | False | — | Para grupos com <3 pontos-limite, gera buffer ao redor dos contaminados |
| Buffer fator | 0,5 | 0,1–0,95 | Multiplicador da distância média (raio do buffer) |

### 5.4. Parâmetros avançados de extrapolação

Ativados quando "Extrapolar plumas abertas" estiver ligado.

| Parâmetro | Padrão | Faixa | Função |
|-----------|--------|-------|--------|
| Fator de distância da extrapolação | 1,0 | 0,5–3,0 | Multiplicador da mediana das distâncias amostrais locais. Define quão "longe" os pontos virtuais ficam |
| Gap angular mínimo (graus) | 90 | 60–180 | Threshold para considerar um contaminado como sendo de borda. 90° = "1/4 do círculo sem amostragem" |
| Gerar polígono separado | False | — | Quando ativo, gera DUAS features por pluma: parte real e parte estimada (para simbologia diferente) |

## 6. Como interpretar as saídas

O aplicativo gera **três camadas** de saída.

### 6.1. Camada de polígonos

Esta é a saída principal — uma feature por pluma identificada. Os campos:

| Campo | Tipo | Significado |
|-------|------|-------------|
| `grupo` | texto | Identificador do grupo: "Contaminante \| valor_grupo1 \| valor_grupo2..." |
| `contaminante` | texto | Nome do contaminante |
| `regra` | texto | "Regra 50%" ou "Regra 75%" |
| `fracao` | numérico | 0,5 ou 0,75 (fração usada) |
| `n_pontos_cont` | inteiro | Número de pontos contaminados na pluma |
| `n_pontos_nao_cont` | inteiro | Número de pontos não-contaminados envolvidos na delimitação |
| `n_pontos_limite` | inteiro | Número de pontos-limite que formam o contorno (incluindo virtuais se houver) |
| `conc_max` | numérico | Concentração máxima entre os contaminados da pluma |
| `ref_legal` | numérico | Valor de referência (VI) usado |
| `area_m2` | numérico | Área do polígono em metros quadrados |
| `metodo_fechamento` | texto | Combinação dos métodos: "centroide + ângulo polar + Catmull-Rom (subdivs=16, tensão=0.30)" |
| `tem_extrapolacao` | inteiro | 0 = sem extrapolação; 1 = pluma com pontos virtuais |
| `subtipo` | texto | "completa" (padrão) / "real" / "estimada" (modo polígono separado) |
| `observacao` | texto | **Coluna verbosa** com todas as decisões de descarte e avisos |

A coluna `observacao` é especialmente importante. Ela registra, em uma única string separada por " | ", todas as decisões automáticas tomadas durante o processamento daquela pluma. Exemplos do que pode aparecer:

- `Filtro k×MAD descartou 3 aresta(s) Delaunay (limite 18.8 m): PMN02→PM28B (28.7m); ...`
- `Validação Voronoi descartou 2 par(es): PMN02→PM26B (interceptado por PM26A); ...`
- `Extrapolação ativada — contaminado(s) de borda: [PMN02, PM14]`
- `EXTRAPOLAÇÃO: 6 ponto(s)-limite virtual(is) gerado(s) ao redor de 2 contaminado(s) de borda...`
- `Suavização Catmull-Rom (subdivs=16, tensão=0.30) excluiu 2 ponto(s) contaminado(s) [PMN02, PM14] — mantido polígono original.`
- `ATENÇÃO: 1 ponto(s) não-contaminado(s) interno(s) ao polígono final — revisão técnica obrigatória. IDs: [PM26A]`

Sempre revise a coluna `observacao` antes de aceitar a pluma. Em particular, se houver mensagem de "ponto não-contaminado interno", a delimitação precisa de revisão técnica manual.

### 6.2. Camada de pontos-limite

Uma feature por ponto-limite calculado. Útil para validação visual e auditoria.

| Campo | Tipo | Significado |
|-------|------|-------------|
| `grupo` | texto | Mesmo grupo do polígono |
| `contaminante` | texto | Contaminante |
| `id_cont` | texto | ID do ponto contaminado |
| `id_nao_cont` | texto | ID do ponto não-contaminado (NULL se ponto extrapolado) |
| `conc_cont` | numérico | Concentração no ponto contaminado |
| `conc_nao_cont` | numérico | Concentração no ponto não-contaminado (NULL se extrapolado) |
| `ref_legal` | numérico | VI |
| `regra` | texto | Regra usada |
| `fracao` | numérico | Fração usada (NULL se extrapolado) |
| `tipo` | texto | **"real"** (par contaminado/não-cont real) ou **"extrapolado"** (gerado virtualmente) |
| `observacao` | texto | "Sem evidência amostral" para extrapolados |

Para visualizar a parte estimada da pluma com simbologia diferente, aplique simbologia categórica baseada no campo `tipo` desta camada.

### 6.3. Camada de linhas de conexão (opcional)

Quando ativada via parâmetro avançado, gera linhas conectando cada par contaminado/não-contaminado válido. Útil para diagnóstico visual da malha de Delaunay efetivamente usada após filtros.

## 7. Cenários práticos

A seguir, cinco cenários típicos de uso, com as configurações recomendadas e o que esperar de cada um.

### 7.1. Cenário 1: Pluma típica em malha bem distribuída

**Situação**: posto de combustível com vazamento, 15 a 30 poços de monitoramento, alguns contaminados ao redor da fonte (sob ilhas de bombas ou tanques) e o restante limpos. Malha bem distribuída em 2D, com poços rasos amostrando o aquífero livre.

**Configuração**:
- Campos básicos preenchidos normalmente
- Regra 75% (mais conservadora — borda mais próxima do não-contaminado)
- Modo: Plumas separadas
- Suavizar: True; método Catmull-Rom; tensão 0,30; subdivisões 16
- Demais parâmetros nos defaults

**O que esperar**: o algoritmo gera um (ou mais) polígono representando a pluma com contornos suavizados. A área é geometricamente correta, com pequena expansão estética da Catmull-Rom. Verifique na coluna `observacao` se há avisos.

### 7.2. Cenário 2: Pluma alongada na direção do fluxo subterrâneo

**Situação**: pluma claramente elongada (aspecto de "salsicha"), tipicamente em postos antigos onde a fonte vazou por anos e o gradiente hidráulico levou os contaminantes para longe. O ordenamento angular padrão tende a "enchê-la" em direções perpendiculares ao eixo da pluma, gerando uma forma que não respeita a realidade.

**Configuração**:
- Mesma do Cenário 1, mas:
- **Método de fechamento: Concave Hull**
- Alpha: 0,30 (padrão)

**O que esperar**: o polígono passa a acompanhar o eixo alongado da pluma. Se ficar com cantos muito agressivos ou se romper em multipolígonos, aumente alpha para 0,40 ou 0,50. Se ainda estiver englobando muito espaço vazio, diminua para 0,20 ou 0,15.

### 7.3. Cenário 3: Contaminados na borda externa da malha

**Situação**: o(s) poço(s) mais externo(s) da malha amostral ainda apresentam contaminação acima do VI. Profissionalmente, isso significa que a delimitação não foi fechada e novos poços precisam ser instalados em campanhas futuras. Você quer apresentar essa pluma "aberta" no relatório, sinalizando explicitamente a parte estimada.

**Configuração**:
- Mesma do Cenário 1, mais:
- **Extrapolar plumas abertas: True**
- Fator de distância: 1,0 (conservador) ou 1,5 (alerta forte)
- Gap angular: 90° (padrão); reduzir para 70°-80° se a malha for densa
- Gerar polígono separado: True (recomendado para apresentação cartográfica)

**O que esperar**: o log do Processing vai listar, contaminado por contaminado, o gap angular medido e a decisão (detectado/não detectado). A camada de pontos-limite passa a ter pontos com `tipo = "extrapolado"`. Quando ativada a opção de polígono separado, surgem duas features para a mesma pluma, com `subtipo = "real"` e `subtipo = "estimada"` — você pode aplicar simbologia diferente (cor sólida na real, hachura na estimada) usando regras categóricas no QGIS.

### 7.4. Cenário 4: Investigação subdimensionada

**Situação**: grupo (ex: poços profundos de uma área) com apenas 1 ou 2 contaminados e 2 a 3 não-contaminados. Insuficiente para gerar polígono pelo método padrão (mínimo de 3 pontos-limite). Sem ação, o grupo é descartado silenciosamente.

**Configuração**:
- Mesma do Cenário 1, mais:
- **Buffer fallback: True**
- Buffer fator: 0,5 (raio = metade da distância média ao não-contaminado mais próximo)

**O que esperar**: para os grupos subdimensionados, o algoritmo gera um buffer (círculo) ao redor do(s) ponto(s) contaminado(s). A coluna `observacao` registra explicitamente que se trata de **representação aproximada, não polígono normativo**. Use essa saída apenas para fim cartográfico, sinalizando no relatório que a área não foi delimitada amostralmente.

### 7.5. Cenário 5: Apresentação cartográfica para relatório técnico

**Situação**: você precisa entregar mapas finais do RAS, RBC ou plano de remediação. A apresentação visual importa — cantos angulosos parecem "técnicos demais", muito mecânicos. Quer plumas com aparência mais natural, próxima ao que se vê em softwares geoestatísticos.

**Configuração** (para plumas convexas tipicamente bem amostradas):
- Suavização: True
- Método: Catmull-Rom
- Tensão: 0,30 a 0,40 (mais expansão = visual mais "redondo")
- Subdivisões: 16 a 24 (em escala 1:1.000, mais subdivisões deixam a curva mais lisa)

**Configuração** (quando a área importa juridicamente — por exemplo, para cálculo de passivo ou laudo pericial):
- Rode duas vezes:
  1. **Sem suavização** (`Suavizar: False`) — use `area_m2` para os números do relatório
  2. **Com Catmull-Rom** (tensão 0,30) — use o polígono apenas para a apresentação cartográfica

Documente a divergência no relatório metodológico.

### 7.6. Combinação de cenários

Os cenários acima não são exclusivos. Para um caso real complexo, pode-se combinar: pluma alongada (Cenário 2) com contaminados de borda (Cenário 3) e necessidade de apresentação cartográfica (Cenário 5):

```
Método de fechamento: Concave Hull (alpha 0,30)
Extrapolar plumas abertas: True (fator 1,0, gap 90°, polígono separado: True)
Suavizar: True (Catmull-Rom, tensão 0,30, subdivisões 20)
```

## 8. Solução de problemas comuns

### 8.1. "CRS geográfico detectado, aborta"

**Causa**: a camada de entrada está em coordenadas geográficas (graus).
**Solução**: reprojete via **Vector → Data Management → Reproject Layer**. No Brasil, use SIRGAS 2000 / UTM, zonas 18S a 25S (EPSG:31978–31985).

### 8.2. "Sem pontos contaminados em todos os grupos"

**Causa**: nenhum ponto da camada tem concentração > VI. Pode ser real (área limpa) ou erro de configuração.
**Verifique**:
- Os campos de concentração e VI estão apontando corretamente?
- A unidade está consistente entre concentração e VI? (Ex: ambos em ppb, ou ambos em mg/L)
- O campo de concentração é numérico, não texto?

### 8.3. "Pluma com formato muito estranho ou enviesado"

**Causa provável**: agrupamento por profundidade não foi feito, e poços rasos estão sendo misturados com profundos.
**Solução**: adicionar um campo de profundidade (ou classe de profundidade) no parâmetro de campos opcionais de agrupamento.

### 8.4. "Polígono ficou muito menor do que esperava após suavização"

**Causa**: você está usando Chaikin (aproximador, contrai os cantos).
**Solução**: mude o método para Catmull-Rom; ou reduza o offset do Chaikin para 0,10–0,15; ou desligue a suavização.

### 8.5. "Polígono ficou muito maior do que esperava após suavização"

**Causa**: você está usando Catmull-Rom com tensão alta.
**Solução**: reduza a tensão para 0,15 ou 0,20. A tensão 0,30 é o padrão equilibrado; valores acima de 0,40 geram expansão visível.

### 8.6. "Aviso: ponto não-contaminado interno ao polígono"

**Causa**: um ponto que **deveria estar fora** da pluma ficou dentro do polígono. Tipicamente acontece em malhas com clusters de contaminação separados, onde Delaunay liga pontos por cima de não-contaminados intermediários.
**Solução em ordem**:
1. Verifique se o ponto realmente é não-contaminado (concentração ≤ VI) — pode ser erro de classificação.
2. Reduza o filtro k×MAD para 2,0 (mais rigoroso) — remove arestas longas espúrias.
3. Considere o método Concave Hull com alpha menor (0,20).
4. Em casos não resolvidos automaticamente, edite manualmente o polígono final no QGIS, justificando a edição no relatório.

### 8.7. "Extrapolação não detectou os contaminados de borda que eu esperava"

**Causa**: os contaminados não têm gap angular maior que 90° entre seus vizinhos Delaunay. O log do Processing mostra o gap medido por contaminado.
**Solução**: reduzir o threshold para 70° ou 80° via parâmetro avançado de gap angular. Verificar no log quais valores de gap apareceram.

### 8.8. "Saída em Shapefile com nomes de campos cortados"

**Causa**: Shapefile tem limite de 10 caracteres no nome do campo. Campos como `tem_extrapolacao` ficam truncados para `tem_extrap`.
**Solução**: usar **GeoPackage** (.gpkg) como formato de saída, que não tem essa limitação. No diálogo de saída, escolha "Save to GeoPackage" em vez de "Save to file".

---

# Parte II — Fundamentos Teóricos

Esta parte explica os algoritmos de geometria computacional, estatística robusta e splines que fundamentam o aplicativo. A linguagem é didática, mas as fórmulas estão presentes onde necessárias para rigor técnico. Os conceitos são introduzidos do mais simples ao mais complexo, e cada um é seguido pela aplicação prática no script.

## 9. Por que delimitação geométrica?

### 9.1. Duas filosofias de delimitação

Existem duas grandes famílias de técnicas para delimitar plumas a partir de pontos amostrais. A **geoestatística** (kriging, IDW, splines de tensão, etc.) constrói um campo contínuo de concentração interpolada a partir das medições e desenha o contorno da pluma onde esse campo cruza o valor de referência. A **delimitação geométrica** (este aplicativo) constrói o polígono diretamente a partir das posições dos pontos contaminados e não-contaminados, aplicando regras de fração de distância sem nunca interpolar concentrações.

A escolha entre as duas é técnica. Geoestatística é mais sofisticada e produz resultados visualmente mais "naturais", mas exige pressupostos fortes sobre o comportamento espacial do contaminante (estacionariedade, isotropia, modelo de variograma). Em GAC, esses pressupostos raramente se verificam — concentrações em ponto-fonte são tipicamente não-estacionárias, e a malha amostral costuma ser pequena demais para ajustar um variograma confiável.

A delimitação geométrica, por outro lado, faz o mínimo de suposições estatísticas. Ela obedece a regras determinísticas baseadas exclusivamente na geometria da malha — por isso é normativa, no sentido de que reflete uma convenção técnica regulamentada (a "regra da fração da distância") em vez de um modelo probabilístico.

### 9.2. Quando a delimitação geométrica é mais adequada

Há três situações típicas em que a abordagem geométrica é defensável e às vezes preferível:

**Investigação preliminar**: na fase inicial de avaliação, com poucos poços (5 a 15), a malha não permite ajustar um variograma estatisticamente confiável. A delimitação geométrica é uma representação honesta do que se sabe a partir das amostras existentes.

**Aplicação regulatória**: agências ambientais brasileiras (CETESB, IBAMA, IEMA) frequentemente referenciam a "regra da fração da distância" como aceitável para fins normativos. A delimitação geométrica reproduz essa regra exatamente.

**Comparação determinística entre cenários**: para comparar campanhas, contaminantes ou hipóteses, a delimitação geométrica fornece resultados reprodutíveis sem a variabilidade introduzida pela escolha de modelo de variograma.

## 10. Conceitos básicos de geometria computacional

### 10.1. Pontos no plano

Um ponto no plano é um par ordenado de coordenadas reais (x, y). No script, cada ponto-amostral, contaminado ou não-contaminado, é representado por uma instância da classe `QgsPointXY`, que armazena exatamente esses dois valores em ponto flutuante (formato IEEE 754 de 64 bits, com precisão de ~15 dígitos decimais — mais do que suficiente para qualquer escala cartográfica).

A **distância euclidiana** entre dois pontos P1 = (x1, y1) e P2 = (x2, y2) é dada pelo Teorema de Pitágoras:

```
d(P1, P2) = √((x2 - x1)² + (y2 - y1)²)
```

Em Python, isso é calculado com `math.hypot(x2-x1, y2-y1)`, que tem a vantagem de ser numericamente mais estável que a fórmula direta para valores muito grandes ou muito pequenos.

### 10.2. Polígonos

Um polígono é uma região fechada do plano delimitada por uma poligonal — sequência de segmentos de reta que voltam ao ponto de partida. Em PyQGIS, o polígono é representado como uma lista de listas de `QgsPointXY`, onde cada lista interna é um anel (o primeiro é o anel externo; demais, se houver, são buracos).

Um polígono pode ser **convexo** (qualquer segmento entre dois pontos internos do polígono fica totalmente dentro dele) ou **côncavo** (existe pelo menos um segmento entre dois pontos internos que sai do polígono). Plumas reais raramente são convexas — em particular, plumas alongadas na direção do fluxo subterrâneo são frequentemente côncavas em algum trecho.

A **área** de um polígono pode ser calculada pela fórmula de Gauss (também chamada **shoelace formula**):

```
A = ½ · |Σ (xi · yi+1 - xi+1 · yi)|
```

Onde os índices são tomados em módulo n (ou seja, xn = x0 e yn = y0). A fórmula soma os "produtos cruzados" de coordenadas consecutivas.

## 11. Triangulação de Delaunay

Esta é a estrutura geométrica central do aplicativo. Praticamente todas as decisões dependem dela.

### 11.1. O conceito intuitivo

Dado um conjunto de pontos no plano, queremos criar uma rede de triângulos que (a) cubra a região delimitada pelos pontos e (b) ligue os pontos por arestas que reflitam vizinhança "natural". Existem infinitas formas de triangular o mesmo conjunto de pontos, mas uma específica se destaca: a **triangulação de Delaunay**, descrita pelo matemático russo Boris Delaunay em 1934.

### 11.2. A definição formal — propriedade do círculo vazio

Uma triangulação é de Delaunay quando, para cada triângulo dela, **o círculo que passa pelos seus três vértices (chamado circuncírculo) não contém nenhum outro ponto do conjunto no seu interior**. Esse critério, aparentemente simples, tem consequências profundas.

Em representação esquemática:

```
     A             A triangulação de Delaunay tem
    / \            a propriedade de que, para cada
   /   \           triângulo, o circuncírculo
  /  o  \          (círculo passando pelos 3 vértices)
 B-------C         não contém nenhum outro ponto.
                   
                   Aqui o triângulo ABC é Delaunay
                   se nenhum outro ponto cair dentro
                   do círculo que passa por A, B, C.
```

Equivalentemente: a triangulação de Delaunay maximiza o menor ângulo de cada triângulo. Ou seja, evita triângulos finos e estirados, preferindo triângulos "gordos". Isso é o que torna as arestas Delaunay representações **naturais** de vizinhança espacial.

### 11.3. Algoritmos para construir a triangulação

Dois algoritmos clássicos calculam Delaunay em tempo O(n log n), eficiente para até dezenas de milhares de pontos:

**Bowyer-Watson** (incremental): começa com um triângulo gigante que envolve todos os pontos. Insere pontos um por um. Para cada ponto, identifica todos os triângulos cujo circuncírculo contém o novo ponto, remove-os, e re-triangula a região vazia conectando o novo ponto às bordas. Após inserir todos os pontos, remove o triângulo gigante inicial.

**Divide and Conquer**: divide o conjunto de pontos em duas metades, triangula cada uma recursivamente, depois funde as duas triangulações. A fusão é a parte complexa: requer identificar a "tangente comum inferior" entre as duas regiões e construir triângulos entre elas.

O QGIS usa internamente o algoritmo **Delaunay Triangulation** da Computational Geometry Algorithms Library (CGAL), uma biblioteca C++ de alta performance. No script, isso é acessado via `processing.run('qgis:delaunaytriangulation', ...)`.

### 11.4. Vizinho Delaunay

Dois pontos são **vizinhos Delaunay** quando há uma aresta direta entre eles na triangulação — sem ponto intermediário. Atenção: vizinho Delaunay não é o mesmo que "ponto mais próximo" geograficamente. Um ponto pode estar a 50m de outro e não ser seu vizinho Delaunay (se houver um terceiro ponto "entre" eles topologicamente). E pode estar a 80m e ser vizinho, se nenhum outro se interpõe.

Em malhas amostrais reais, cada ponto tem tipicamente entre 4 e 8 vizinhos Delaunay, dependendo da densidade local. Pontos no interior da malha têm mais vizinhos; pontos na fronteira externa têm menos.

### 11.5. Aplicação no script

A triangulação de Delaunay é usada para **identificar quais pares contaminado-não-contaminado devem gerar pontos-limite**. Em vez de conectar todos contra todos (o que daria n×m pares e plumas absurdas), apenas os pares que compartilham aresta na triangulação são considerados. Isso garante que os pontos-limite reflitam vizinhança espacial real, não combinatória cega.

A função `_delaunay_arestas` no script faz a triangulação via algoritmo nativo do QGIS, depois extrai as arestas dos triângulos resultantes (cada triângulo tem 3 arestas; aresta interna aparece em 2 triângulos, então deduplicamos).

## 12. Diagrama de Voronoi e dualidade

### 12.1. Definição

O diagrama de Voronoi é uma estrutura geométrica complementar à triangulação de Delaunay. Ele divide o plano em regiões chamadas **células de Voronoi**, uma por ponto do conjunto: a célula do ponto Pi contém todos os locais do plano que estão **mais próximos de Pi do que de qualquer outro ponto** do conjunto.

Geometricamente, cada borda entre duas células é a **mediatriz perpendicular** do segmento que liga os dois pontos correspondentes. Em representação esquemática:

```
       |              Cada região contém um ponto.
   A   |   B          A borda entre A e B é a mediatriz
       |              do segmento AB, equidistante aos dois.
  -----+-----
       |              Pontos próximos da borda de A
   C   |   D          poderiam pertencer a B se cruzassem
       |              ligeiramente para o outro lado.
```

### 12.2. A dualidade Delaunay-Voronoi

Aqui está um dos teoremas mais elegantes da geometria computacional: **Delaunay e Voronoi são estruturas duais**. Mais precisamente:

> Dois pontos têm uma aresta Delaunay entre eles **se, e somente se**, suas células de Voronoi compartilham uma borda.

Em outras palavras, Delaunay descreve a vizinhança como "quem está conectado a quem", e Voronoi descreve a mesma vizinhança como "qual é o território de cada um". Cada aresta Delaunay atravessa exatamente uma borda de Voronoi, e elas são perpendiculares entre si.

### 12.3. Aplicação no script — validação Voronoi

A validação Voronoi é uma das melhorias estruturais do rv2. Para cada ponto-limite calculado, ela verifica: o ponto-limite está dentro da célula de Voronoi do não-contaminado do par? Se sim, o par é geometricamente consistente. Se não — ou seja, se o ponto-limite cair na célula de outro não-contaminado — significa que esse outro não-contaminado é mais próximo do ponto-limite do que o do par. Isso é sinal de que o par original era espúrio: a aresta Delaunay "atravessou" topologicamente uma região onde outro não-contaminado deveria delimitar a pluma.

Detalhe de implementação: o script **não constrói** o diagrama de Voronoi explicitamente. Em vez disso, para cada ponto-limite, calcula a distância euclidiana para todos os não-contaminados do grupo e identifica o mais próximo. Isso é matematicamente equivalente à pertinência à célula de Voronoi, mas computacionalmente mais simples (e dispensa a chamada a algoritmos de construção de Voronoi).

## 13. Convex Hull e Concave Hull

### 13.1. Convex Hull — o "elástico esticado"

O convex hull de um conjunto de pontos é o **menor polígono convexo** que contém todos eles. Imagine pregos espalhados em uma tábua e um elástico esticado em volta — o elástico forma o convex hull. Os pontos que tocam o elástico são vértices do hull; os que ficam "para dentro" não.

Algoritmos clássicos para calcular o convex hull em tempo O(n log n):

**Graham scan**: ordena os pontos pelo ângulo polar a partir do ponto mais inferior, depois percorre a sequência testando se cada novo ponto faz "curva à esquerda" ou "curva à direita". Pontos que fazem curva à direita são removidos do hull.

**Quickhull**: estratégia divide-and-conquer análoga ao quicksort. Identifica os pontos extremos, divide o conjunto em dois, e recursivamente encontra os pontos do hull em cada metade.

### 13.2. Concave Hull (Alpha Shape)

Para conjuntos de pontos com forma alongada ou irregular, o convex hull engloba muito espaço vazio nas reentrâncias. O **concave hull** generaliza o conceito permitindo concavidades. A formulação matemática rigorosa, devida a Edelsbrunner e Mücke (1983), é o **alpha shape**:

> Dado um conjunto de pontos e um valor α > 0, o alpha shape é o subconjunto da triangulação Delaunay formado pelos triângulos cujo circuncírculo tem **raio menor que 1/α**.

Quando α → 0, o limiar de raio é infinitamente grande, todos os triângulos passam, e o resultado é o convex hull. Conforme α aumenta, triângulos com circuncírculos grandes (que ficam em regiões esparsas do conjunto) são removidos, criando concavidades.

O algoritmo `native:concavehull` do QGIS implementa essa técnica. O parâmetro de entrada (chamado `ALPHA` na interface do QGIS) é normalizado entre 0 e 1, **invertido** em relação ao α matemático: `ALPHA = 1` produz convex hull e `ALPHA = 0` produz o concave mais agressivo.

### 13.3. Aplicação no script

O concave hull é oferecido como método alternativo de fechamento do polígono, indicado para plumas alongadas ou com formas côncavas (Cenário 7.2 do Guia de Uso). É construído sobre os pontos contaminados unidos aos pontos-limite, garantindo que os contaminados fiquem dentro do hull mesmo quando o parâmetro alpha for agressivo.

## 14. Estatística robusta — mediana e MAD

### 14.1. O problema com média e desvio padrão

Considere quatro conjuntos de comprimentos de aresta, em metros:

- Conjunto A: 30, 32, 35, 38, 40 → **média = 35; desvio padrão = 4,1**
- Conjunto B: 30, 32, 35, 38, 200 → **média = 67; desvio padrão = 73,7**
- Conjunto C: 30, 32, 35, 38, 1000 → **média = 227; desvio padrão = 432,6**

Conforme um único valor extremo (outlier) é introduzido, tanto a média quanto o desvio padrão **explodem**, deixando de representar o comportamento "típico" do conjunto. Em estatística, dizemos que ambos têm **ponto de ruptura zero**: basta um único outlier para os destruir.

### 14.2. Estatística robusta — a alternativa

A **mediana** é o valor que divide o conjunto ordenado pela metade. Para os mesmos conjuntos acima, a mediana é sempre 35 — totalmente imune ao outlier. Em estatística, dizemos que a mediana tem **ponto de ruptura de 50%**: até metade dos valores podem ser arbitrários sem desestabilizar o resultado.

O **MAD** (Median Absolute Deviation) é o equivalente robusto do desvio padrão. Calcula-se como a **mediana dos desvios absolutos em relação à mediana**:

```
MAD = mediana(|xi - mediana(x)|)
```

Para distribuições aproximadamente gaussianas, multiplicar MAD por aproximadamente 1,4826 produz uma estimativa robusta do desvio padrão.

### 14.3. Aplicação no script — filtro k×MAD

O filtro de arestas longas usa essa propriedade. Calcula-se a mediana e o MAD dos comprimentos das arestas Delaunay, e descartam-se as arestas com comprimento maior que `mediana + k × MAD`. Com k = 2,5 (padrão), o filtro elimina arestas cerca de 3,7 desvios padrão acima do típico — alto o suficiente para garantir que apenas os "atravessadores" topológicos genuinamente espúrios sejam removidos.

A virtude do uso de mediana + MAD em vez de média + desvio padrão é que **as próprias arestas longas espúrias não contaminam o cálculo do limiar**. Se usássemos média + desvio padrão, as arestas longas inflariam ambos, e o limiar ficaria alto demais para flagrar essas mesmas arestas — paradoxo circular evitado pela escolha robusta.

## 15. Grafos e componentes conexas

### 15.1. O que é um grafo

Em matemática discreta, um **grafo** é uma estrutura formada por **vértices** (também chamados nós) ligados por **arestas**. Quando duas arestas compartilham um vértice, dizemos que os vértices são vizinhos. Grafos são onipresentes: redes sociais, rotas de avião, mapas de metrô, e — nosso caso — pontos de monitoramento conectados por triangulação.

No contexto do aplicativo, montamos um grafo onde os vértices são os **pontos contaminados** e as arestas conectam dois contaminados se eles forem vizinhos Delaunay diretos (sem ponto intermediário). Pontos não-contaminados ficam fora deste subgrafo.

### 15.2. Componente conexa

Um **caminho** em um grafo é uma sequência de arestas conectadas. Dois vértices estão na mesma **componente conexa** se existe um caminho entre eles. Um grafo pode ter uma única componente (todos os vértices alcançáveis entre si) ou várias componentes isoladas.

Aplicado ao subgrafo de contaminados: cada componente conexa representa um **cluster espacialmente contínuo de contaminação**. Se há dois clusters de contaminados separados por um corredor de não-contaminados (pluma "fragmentada"), eles formam duas componentes conexas distintas, e o aplicativo gera duas plumas separadas.

### 15.3. BFS — Breadth-First Search

O algoritmo clássico para identificar componentes conexas é o **BFS** (busca em largura). O procedimento:

1. Marque todos os vértices como "não visitados"
2. Pegue o primeiro vértice não visitado, marque-o como visitado, e adicione-o a uma fila
3. Tire o vértice do início da fila. Para cada vizinho dele, se ainda não foi visitado, marque-o e adicione à fila
4. Continue até a fila esvaziar — os vértices alcançados formam uma componente
5. Volte ao passo 2 enquanto houver vértices não visitados — cada nova exploração descobre uma nova componente

A complexidade do BFS é O(V + E), onde V é o número de vértices e E o número de arestas. Para malhas com poucas centenas de pontos, é praticamente instantâneo.

### 15.4. Aplicação no script

A função `_identificar_plumas` implementa exatamente esse procedimento. Recebe a lista de arestas Delaunay e a lista de índices contaminados, e retorna uma lista de listas — cada lista interna contém os índices dos contaminados que formam uma componente conexa. Essas componentes viram, no fim, as plumas separadas que o aplicativo gera (quando o modo "Plumas separadas" está ativo).

## 16. Suavização de polígonos

### 16.1. Aproximadores vs interpoladores

Existem duas grandes famílias de algoritmos de suavização. A diferença é fundamental: a curva resultante **passa pelos vértices originais** (interpolador) ou **não passa** (aproximador).

**Aproximadores** geram uma curva que fica próxima dos vértices originais, mas não os atinge. Os vértices se tornam "atratores" da curva, e os cantos angulosos são cortados por dentro. O algoritmo de Chaikin é um aproximador.

**Interpoladores** geram uma curva que passa exatamente pelos vértices originais, suavizando apenas as transições entre eles. Os Splines de Catmull-Rom e Bezier são interpoladores.

### 16.2. Algoritmo de Chaikin

Criado por George Chaikin em 1974. A regra é simples e iterativa: para cada aresta Pi → Pi+1 do polígono, gere dois pontos novos:

```
Q1 = Pi + (1 - f) · (Pi+1 - Pi)
Q2 = Pi + f · (Pi+1 - Pi)
```

Onde f é o "offset" (tipicamente 0,25 — o "Chaikin clássico"). Esses dois pontos ficam a 25% e 75% do caminho entre Pi e Pi+1. A nova lista de vértices é formada por todos esses pares de pontos, descartando os vértices originais. A cada iteração, o polígono passa a ter aproximadamente o dobro de vértices, e os cantos vão sendo progressivamente arredondados.

Após muitas iterações, a sequência converge para uma **curva B-spline quadrática**, matematicamente lisa em toda parte (sem cantos descontínuos). Por que isso importa? Porque cantos angulosos são pontos onde a derivada da curva é descontínua. Cada iteração de Chaikin substitui um canto por uma "miniaresta" entre dois pontos próximos — uma descontinuidade menor. Após várias iterações, as descontinuidades ficam tão pequenas que perceptualmente desaparecem.

**Característica importante**: o Chaikin **contrai** o polígono. Os novos pontos sempre ficam dentro do "triângulo" formado pelo vértice original e seus dois vizinhos. A área final, com offset 0,25, é cerca de 75% da original.

### 16.3. Cardinal Spline (Catmull-Rom generalizado)

Para casos em que se quer preservar área (ou expandir ligeiramente, mas não contrair), usa-se um interpolador. O **Catmull-Rom Spline** é um caso particular do mais geral **Cardinal Spline**, parametrizado por uma constante c chamada **tensão**.

A fórmula do Cardinal Spline para a aresta Pi → Pi+1 usa quatro pontos consecutivos Pi-1, Pi, Pi+1, Pi+2:

```
B(t) = h1(t) · Pi + h2(t) · Pi+1
     + h3(t) · c · (Pi+1 - Pi-1)
     + h4(t) · c · (Pi+2 - Pi)
```

Onde t ∈ [0, 1] percorre o segmento, e os polinômios de Hermite são:

```
h1(t) = 2t³ - 3t² + 1
h2(t) = -2t³ + 3t²
h3(t) = t³ - 2t² + t
h4(t) = t³ - t²
```

Verificações importantes da fórmula:

- Em t = 0: h1(0) = 1, h2(0) = 0, h3(0) = 0, h4(0) = 0 → B(0) = Pi (curva passa por Pi)
- Em t = 1: h1(1) = 0, h2(1) = 1, h3(1) = 0, h4(1) = 0 → B(1) = Pi+1 (curva passa por Pi+1)

Ou seja, a curva é **interpoladora**: passa exatamente pelos vértices originais.

A constante c controla o quanto a curva se afasta da linha reta entre Pi e Pi+1:

- c = 0: a curva é a linha reta (sem suavização)
- c = 0,5: Catmull-Rom clássico
- c > 0,5: curva mais "pronunciada", se afasta mais da linha reta

### 16.4. Por que o Cardinal Spline expande

Em polígonos convexos (como pluma típica), os cantos têm ângulo interno menor que 180°, e o Cardinal Spline tende a "abrir" cada canto para fora, expandindo a área total. A magnitude da expansão depende de c:

| Tensão c   | Expansão típica em polígonos convexos |
|------------|---------------------------------------|
| 0,15       | ~8%                                   |
| 0,30       | ~13–15% (padrão recomendado)          |
| 0,50       | ~25% (Catmull-Rom clássico)           |
| 0,75       | ~35%                                  |

Por isso o aplicativo permite ajustar a tensão. Para uso técnico-cartográfico, c = 0,30 oferece um bom equilíbrio entre suavidade visível e expansão controlada.

### 16.5. Implementação prática

O Chaikin é implementado nativamente no PyQGIS via `QgsGeometry.smooth(iterations, offset, ...)`. O Cardinal Spline, ao contrário, foi implementado **manualmente em Python puro** no script (função `_suavizar_catmull_rom`), usando a fórmula acima. Isso tem a vantagem de não depender de bibliotecas externas (scipy, numpy) e funcionar em qualquer instalação do QGIS.

## 17. Operações geométricas fundamentais

O QGIS oferece um conjunto rico de operações sobre `QgsGeometry`. As que aparecem no script:

### 17.1. makeValid

Geometrias inválidas surgem por autointersecções, orientação errada de buracos, vértices duplicados, e outros problemas. O `makeValid` aplica heurísticas para reparar — se um anel se cruza, divide em dois polígonos válidos. É uma "rede de proteção" indispensável após operações que podem quebrar a topologia.

### 17.2. buffer e o truque buffer(0)

`buffer(distance, segments)` cria uma região ao redor da geometria com a distância especificada. O parâmetro `segments` controla quantos segmentos formam o quarto de círculo nos cantos curvos.

O truque **`buffer(0)`** explora um efeito colateral interessante: ao "rodar" toda a maquinaria de buffer com distância zero, autointersecções são limpas e geometria válida é gerada. É um fallback útil quando `makeValid` não funciona.

### 17.3. contains, intersects

`contains(other)` verifica se `other` está **inteiramente dentro** da geometria, sem tocar a borda. `intersects(other)` é mais permissivo: retorna `True` se houver qualquer interseção, incluindo na borda.

No script, usamos `contains OR intersects` para verificações tolerantes — ponto exatamente sobre a borda do polígono é considerado "dentro" para fins de validação de fidelidade.

### 17.4. difference, unaryUnion

`difference(other)` retorna a parte de uma geometria que **não é** sobreposta por `other`. Usado no modo polígono separado da extrapolação: subtraindo o polígono real do polígono completo, resta a parte estimada.

`unaryUnion(geometries)` une múltiplas geometrias em uma só, dissolvendo sobreposições. Usado no buffer fallback para fundir buffers de múltiplos contaminados em um único polígono.


---

# Parte III — Implementação em Python/PyQGIS

Esta parte é dedicada a quem quer entender ou modificar o código. Pressupõe conhecimento básico de Python (variáveis, funções, classes, listas, dicionários) e familiaridade mínima com QGIS. PyQGIS é introduzido do zero.

## 18. O que é um Processing Algorithm

O QGIS oferece dois grandes ambientes para automação: a **Python Console** (interativa) e o **Processing Framework** (algoritmos reutilizáveis com interface gráfica automática). O aplicativo de delimitação de plumas é um **Processing Algorithm**.

A grande vantagem do Processing Framework é que o QGIS **gera automaticamente o diálogo de entrada** a partir da declaração dos parâmetros no código. Você não precisa programar a interface gráfica — apenas declara "preciso de uma camada de pontos, um campo numérico, um valor entre 0 e 1", e o QGIS desenha o diálogo, faz validações de tipo, e passa os valores escolhidos para o seu código.

## 19. A estrutura básica de uma classe Processing Algorithm

Todo Processing Algorithm é uma classe Python que herda de `QgsProcessingAlgorithm`. A estrutura mínima:

```python
from qgis.core import QgsProcessingAlgorithm

class MeuAlgoritmo(QgsProcessingAlgorithm):

    def name(self):
        # Identificador interno (snake_case, sem espaços)
        return 'meu_algoritmo'

    def displayName(self):
        # Nome visível ao usuário no Toolbox
        return 'Meu Algoritmo de Exemplo'

    def group(self):
        # Pasta no Toolbox
        return 'Exemplos'

    def groupId(self):
        # Identificador interno do grupo
        return 'exemplos'

    def createInstance(self):
        # Retorna nova instância — sempre assim
        return MeuAlgoritmo()

    def initAlgorithm(self, config=None):
        # Aqui declaramos os parâmetros
        pass

    def processAlgorithm(self, parameters, context, feedback):
        # Aqui está a lógica principal
        return {}
```

Os métodos `name`, `displayName`, `group`, `groupId` e `createInstance` são **metadados**: o QGIS os usa para registrar e exibir o algoritmo. Os dois métodos onde o trabalho real acontece são:

- `initAlgorithm`: declara quais parâmetros o usuário vai preencher
- `processAlgorithm`: lê os parâmetros e processa

## 20. Os parâmetros — tipos disponíveis

PyQGIS oferece classes para cada tipo de entrada que um algoritmo pode receber. Os mais comuns:

| Classe | O que representa |
|--------|------------------|
| `QgsProcessingParameterFeatureSource` | Camada vetorial de entrada |
| `QgsProcessingParameterField` | Campo (coluna) de uma camada |
| `QgsProcessingParameterNumber` | Número (inteiro ou real) |
| `QgsProcessingParameterBoolean` | Booleano (caixa de seleção) |
| `QgsProcessingParameterEnum` | Enumeração (lista de opções) |
| `QgsProcessingParameterString` | Texto livre |
| `QgsProcessingParameterFile` | Arquivo do disco |
| `QgsProcessingParameterFeatureSink` | Camada vetorial de saída |
| `QgsProcessingParameterRasterDestination` | Raster de saída |

Cada parâmetro é declarado dentro do `initAlgorithm` via `self.addParameter(...)`.

## 21. Declarando parâmetros — exemplos práticos

### 21.1. Camada vetorial de pontos

```python
self.addParameter(
    QgsProcessingParameterFeatureSource(
        'INPUT',                              # ID interno
        self.tr('Camada de pontos'),          # Descrição visível
        [QgsProcessing.TypeVectorPoint],      # Aceita só pontos
    )
)
```

O ID interno (`'INPUT'` no exemplo) é como o código vai se referir ao parâmetro depois. A função `self.tr(...)` é usada para tradução automática (i18n) — o QGIS pode exibir o texto traduzido se houver arquivo de tradução. Mesmo sem traduções, é boa prática envolver strings visíveis com `tr()`.

### 21.2. Campo de uma camada

```python
self.addParameter(
    QgsProcessingParameterField(
        'CAMPO_VALOR',
        self.tr('Campo de valor'),
        parentLayerParameterName='INPUT',                    # Vincula à camada
        type=QgsProcessingParameterField.Numeric,            # Só campos numéricos
    )
)
```

A vinculação a `parentLayerParameterName='INPUT'` faz o QGIS popular o dropdown automaticamente com os campos da camada selecionada no parâmetro `INPUT`. Quando o usuário muda a camada, o dropdown atualiza.

### 21.3. Número decimal com faixa

```python
self.addParameter(
    QgsProcessingParameterNumber(
        'TENSAO',
        self.tr('Tensão da curva (0 a 1)'),
        type=QgsProcessingParameterNumber.Double,
        defaultValue=0.30,
        minValue=0.0,
        maxValue=1.0,
        optional=True,
    )
)
```

Para inteiros, use `QgsProcessingParameterNumber.Integer`. O `optional=True` significa que o usuário pode deixar em branco — nesse caso, o `defaultValue` será usado.

### 21.4. Enumeração

```python
METODOS = ['Catmull-Rom (interpolador)', 'Chaikin (aproximador)']

self.addParameter(
    QgsProcessingParameterEnum(
        'METODO',
        self.tr('Método'),
        options=METODOS,
        defaultValue=0,    # Índice do default (0 = Catmull-Rom)
    )
)
```

A leitura desse parâmetro retorna o **índice** escolhido (0, 1, 2...), não a string. Você precisa mapear o índice para a opção no seu código.

### 21.5. Marcando parâmetros como avançados

Para esconder um parâmetro sob "Advanced parameters" no diálogo:

```python
param = QgsProcessingParameterNumber(
    'PARAMETRO_TECNICO',
    self.tr('Parâmetro avançado'),
    type=QgsProcessingParameterNumber.Double,
    defaultValue=2.5,
)
param.setFlags(
    param.flags() | QgsProcessingParameterDefinition.FlagAdvanced
)
self.addParameter(param)
```

A construção `param.flags() | FlagAdvanced` usa um operador lógico OR sobre bits — preserva flags existentes e adiciona o flag novo.

### 21.6. Saídas (sinks)

```python
self.addParameter(
    QgsProcessingParameterFeatureSink(
        'OUTPUT_POLIGONOS',
        self.tr('Polígonos de delimitação'),
        type=QgsProcessing.TypeVectorPolygon,
    )
)
```

A camada de saída (sink) é onde o algoritmo vai escrever as features resultantes. O usuário pode escolher salvar em arquivo (.shp, .gpkg, etc.) ou em camada temporária na memória.

## 22. Lendo parâmetros no processAlgorithm

Os parâmetros declarados em `initAlgorithm` são acessados em `processAlgorithm` através de funções `parameterAs*`. Cada tipo de parâmetro tem sua função de leitura:

```python
def processAlgorithm(self, parameters, context, feedback):
    # Camada de entrada (retorna QgsFeatureSource)
    source = self.parameterAsSource(parameters, 'INPUT', context)

    # Campo (retorna nome do campo como string)
    campo_valor = self.parameterAsString(parameters, 'CAMPO_VALOR', context)

    # Número
    tensao = self.parameterAsDouble(parameters, 'TENSAO', context)

    # Booleano
    suavizar = self.parameterAsBool(parameters, 'SUAVIZAR', context)

    # Enum (retorna o índice)
    metodo_idx = self.parameterAsEnum(parameters, 'METODO', context)

    # Sink de saída — retorna (sink, dest_id)
    fields = QgsFields()
    fields.append(QgsField('id', QVariant.Int))
    fields.append(QgsField('valor', QVariant.Double))

    (sink, dest_id) = self.parameterAsSink(
        parameters, 'OUTPUT', context,
        fields,                              # Campos da saída
        QgsWkbTypes.Polygon,                 # Tipo geométrico
        source.sourceCrs(),                  # CRS (herdado da entrada)
    )

    # ... lógica do algoritmo ...

    # Retornar dicionário mapeando IDs de saída para destinos
    return {'OUTPUT': dest_id}
```

## 23. Validações iniciais

Antes de processar, é essencial validar os pressupostos. No script de delimitação, validações típicas:

```python
# CRS deve ser projetado (em metros, não graus)
crs = source.sourceCrs()
if crs.isGeographic():
    raise QgsProcessingException(
        'CRS geográfico não suportado. Reprojete para UTM antes.'
    )

# Camada deve ter feições
if source.featureCount() == 0:
    raise QgsProcessingException(
        'Camada de entrada está vazia.'
    )

# Campos obrigatórios devem existir
if campo_concentracao not in [f.name() for f in source.fields()]:
    raise QgsProcessingException(
        f'Campo "{campo_concentracao}" não encontrado.'
    )
```

A exceção `QgsProcessingException` é tratada automaticamente pelo Processing Framework: aborta o algoritmo e exibe a mensagem ao usuário em uma caixa de erro.

## 24. Iterando sobre features

Para extrair os dados dos pontos:

```python
# Lista para acumular registros
regs = []

# Iterar features
for f in source.getFeatures():
    geom = f.geometry()
    if geom.isEmpty():
        continue

    # Acessar coordenadas
    pt = geom.asPoint()    # QgsPointXY

    # Acessar atributos por nome
    pid = str(f[campo_id])
    contaminante = str(f[campo_contaminante])
    conc = float(f[campo_concentracao])
    vi = float(f[campo_vi])

    # Determinar se é contaminado
    contaminado = conc > vi and conc > 0

    # Estruturar como dicionário
    regs.append({
        'pid': pid,
        'pt': pt,
        'contaminante': contaminante,
        'conc': conc,
        'vi': vi,
        'contaminado': contaminado,
    })

feedback.pushInfo(f'Total de feições: {len(regs)}')
```

A construção em forma de **lista de dicionários** é uma escolha de design: facilita acesso por nome (`r['conc']`) em vez de índice numérico, deixa o código autodocumentado, e mantém todos os dados de cada feição agrupados.

## 25. Escrevendo features no sink de saída

Para cada feature de saída:

```python
# Criar nova feature
nova_feat = QgsFeature()
nova_feat.setGeometry(QgsGeometry.fromPolygonXY([anel_pontos]))
nova_feat.setAttributes([
    'grupo_xyz',     # primeiro atributo (na ordem dos fields)
    'Benzeno',
    0.5,
    1234.56,         # area
    'observacao...',
])

# Inserir no sink
sink.addFeature(nova_feat, QgsFeatureSink.FastInsert)
```

A flag `FastInsert` desabilita verificações de validade durante a inserção (mais rápido para grandes volumes). A ordem dos atributos na lista deve coincidir exatamente com a ordem dos campos declarados no `QgsFields`.

## 26. Chamando outros algoritmos do QGIS

Algumas operações do script (triangulação Delaunay, concave hull, convex hull) usam algoritmos nativos do QGIS via `processing.run`:

```python
import processing

resultado = processing.run(
    'qgis:delaunaytriangulation',           # ID do algoritmo
    {                                        # Parâmetros do algoritmo
        'INPUT': mem_layer,
        'OUTPUT': 'memory:tmp_delaunay'      # Saída em memória
    },
    context=context,
    feedback=feedback,
    is_child_algorithm=True,                 # Importante: True
)

# Obter a camada de saída
saida = resultado['OUTPUT']

# Se for string, resolver para QgsMapLayer
if isinstance(saida, str):
    saida = QgsProcessingUtils.mapLayerFromString(saida, context)

# Iterar sobre as features resultantes
for f in saida.getFeatures():
    geom = f.geometry()
    # ... processar ...
```

A flag `is_child_algorithm=True` informa ao QGIS que este `run` está dentro de outro algoritmo. Isso permite gerenciamento correto de memória, contexto e cancelamento.

Para descobrir o ID exato de um algoritmo, abra-o no Toolbox, clique nos três pontinhos e escolha "Copy Provider/Algorithm Name".

## 27. Logs e feedback ao usuário

O parâmetro `feedback` em `processAlgorithm` é um objeto que permite escrever no log do Processing e atualizar a barra de progresso:

```python
# Mensagem informativa (azul)
feedback.pushInfo('Processando grupo Benzeno...')

# Aviso (laranja)
feedback.pushWarning('Pluma com poucos pontos-limite — qualidade reduzida.')

# Erro (vermelho, mas o algoritmo continua)
feedback.reportError('Erro recuperável no grupo X.')

# Atualizar barra de progresso (0 a 100)
feedback.setProgress(50)

# Verificar se o usuário cancelou
if feedback.isCanceled():
    return {}    # Aborta de forma limpa
```

Logs são fundamentais para o usuário entender o que está acontecendo. Em algoritmos complexos como o de delimitação, os logs documentam cada decisão tomada (descartes pelo filtro k×MAD, validações Voronoi, deteções de borda, etc.). Quando algo der errado, são a principal ferramenta de diagnóstico.

## 28. Estrutura modular — funções auxiliares

Em algoritmos não-triviais, separar a lógica em **funções auxiliares** (também chamadas helpers ou métodos privados) é essencial. Convenção: nomes começando com underscore (`_funcao_privada`) sinalizam que a função é interna à classe e não deve ser chamada de fora.

No script de delimitação, as funções auxiliares principais são:

| Função | O que faz |
|--------|-----------|
| `_ler_dados` | Itera sobre as feições e cria a lista de registros |
| `_agrupar_registros` | Agrupa por contaminante e campos opcionais |
| `_delaunay_arestas` | Triangula via QGIS e extrai lista de arestas |
| `_filtrar_arestas_por_mad` | Aplica filtro mediana + k×MAD nas arestas |
| `_calcular_pontos_limite` | Para cada par cont/não-cont, calcula o ponto-limite |
| `_validar_voronoi` | Descarta pares com ponto-limite mais próximo de outro |
| `_identificar_plumas` | BFS sobre subgrafo de contaminados |
| `_construir_poligono` | Constrói e suaviza o polígono final |
| `_poligono_angular` | Ordena pontos por ângulo polar e fecha |
| `_concave_hull_qgis` | Chama o algoritmo nativo de concave hull |
| `_suavizar_chaikin` | Wrapper sobre `QgsGeometry.smooth` |
| `_suavizar_catmull_rom` | Implementa Cardinal Spline em Python puro |
| `_detectar_contaminados_borda` | rv4: critério único por gap angular |
| `_gerar_pontos_virtuais_borda` | Gera 3 pontos virtuais em arco |
| `_validar_nao_contaminados_internos` | Verifica não-cont dentro do polígono |
| `_construir_buffer_fallback` | Buffer de emergência para grupos pequenos |
| `_processar_grupo` | Orquestra o pipeline completo de um grupo |
| `_gravar_poligono` | Centraliza a inserção no sink (evita duplicação) |

Cada função tem uma responsabilidade clara e bem definida, é independente das demais, e pode ser testada/modificada isoladamente. Esse design facilita manutenção: para mudar o algoritmo de suavização, mexe-se apenas em `_suavizar_*`; para mudar o filtro estatístico, apenas em `_filtrar_arestas_por_mad`.

## 29. Tratamento de erros

Existem três níveis de erro no script:

### 29.1. Erros fatais — abortam o algoritmo

Pressupostos básicos violados que tornam impossível continuar. Levantam `QgsProcessingException`:

```python
if crs.isGeographic():
    raise QgsProcessingException('CRS geográfico não suportado.')
```

### 29.2. Erros recuperáveis — pulam o item, continuam

Falhas pontuais em um grupo ou pluma específica, que não impedem o processamento dos demais:

```python
try:
    poligono = self._construir_poligono(...)
except Exception as e:
    feedback.pushWarning(f'Erro no grupo {chave}: {e}')
    continue    # Pula para o próximo grupo
```

### 29.3. Anomalias técnicas — registradas mas não interrompem

Situações tecnicamente possíveis que merecem registro mas não são erros:

```python
if pontos_fora_apos_suavizacao:
    obs.append('Suavização excluiu N pontos contaminados — '
               'mantido polígono original.')
    # Não interrompe — apenas registra na coluna observacao
```

Essa estratificação é importante: erros fatais raros não devem mascarar problemas técnicos em grupos individuais; e anomalias técnicas em um grupo não devem abortar o processamento dos demais. O usuário recebe ao final um conjunto de saídas onde **cada feature carrega o histórico de suas próprias decisões** na coluna `observacao`, permitindo auditoria pluma por pluma.

## 30. O pipeline completo do script

Reunindo todas as peças, o pipeline geral do `processAlgorithm` é:

```
1. Ler parâmetros da interface
2. Validações iniciais (CRS, contagem de features, campos)
3. Iterar sobre features → lista de registros (regs)
4. Agrupar registros por contaminante + campos opcionais
5. Para cada grupo:
   a. Triangulação de Delaunay
   b. Filtro k×MAD nas arestas
   c. Identificar pares contaminado/não-contaminado
   d. Calcular pontos-limite
   e. Validação Voronoi
   f. (Opcional rv4) Detectar contaminados de borda
   g. Identificar plumas via BFS
   h. Para cada pluma:
      - Filtrar pontos-limite
      - (Opcional rv4) Gerar pontos virtuais de extrapolação
      - (Buffer fallback se aplicável)
      - Construir polígono base
      - Suavizar (Catmull-Rom ou Chaikin)
      - Validar (pontos contaminados dentro? não-cont fora?)
      - (Opcional rv3) Modo polígono separado
      - Gravar no sink
6. Retornar dicionário com IDs de saída
```

Cada etapa tem sua função auxiliar correspondente, e cada uma registra suas decisões no log e/ou na coluna `observacao` da feature de saída. Esse design — pipeline modular com logs verbosos — torna o aplicativo auditável: qualquer feature de saída pode ser rastreada às decisões algorítmicas que a produziram.

## 31. Próximos passos para quem está aprendendo PyQGIS

Este script de delimitação cobre a maioria dos padrões úteis em automação geográfica:

- Leitura e iteração sobre camadas vetoriais
- Tipos diversos de parâmetros (campos, números, enums, sinks)
- Chamada a algoritmos nativos do QGIS
- Operações geométricas (buffer, contains, difference, makeValid)
- Estatística e geometria computacional puras
- Logs verbosos e tratamento de erros estratificado

Para aprofundar, recomendam-se três caminhos:

1. **PyQGIS Cookbook**: documentação oficial com exemplos de cada tipo de operação. Procurar "PyQGIS Cookbook" no portal docs.qgis.org.
2. **Estudo do código-fonte de algoritmos nativos**: o repositório do QGIS no GitHub contém todos os algoritmos do Processing em `python/plugins/processing/`. Comparar com o próprio script ensina padrões idiomáticos.
3. **Implementar variações**: pegar este script e modificar para um caso específico — por exemplo, adicionar um filtro adicional, mudar o algoritmo de suavização, gerar uma camada de saída adicional. A prática de modificação ensina mais que a leitura passiva.

---

## Encerramento

Este manual cobre a versão **rv4** do aplicativo. Mudanças entre versões são documentadas no arquivo de changelog (`03_changelog_plumas.md`).

A delimitação geométrica de plumas é uma ferramenta — útil dentro de seus limites, perigosa quando aplicada sem critério. O aplicativo automatiza decisões algorítmicas que **devem ser conferidas** pelo profissional habilitado. Cada decisão automática fica registrada na coluna `observacao` da camada de saída justamente para permitir essa conferência.

A ferramenta não substitui o julgamento técnico do responsável pela investigação. Substitui apenas o trabalho mecânico de calcular pontos-limite manualmente, montar polígonos um a um, e aplicar suavização vértice por vértice — trabalho que, feito à mão, consome horas e introduz erros. Automatizando esse trabalho, libera o profissional para se concentrar no que importa: a interpretação técnica e a tomada de decisão.

