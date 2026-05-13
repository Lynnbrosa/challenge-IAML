# PrevioPLS — IA/ML

Disciplina **Inteligência Artificial & Machine Learning** da challenge Ford FIAP 2026. Entrega: notebook Jupyter completo de segmentação não-supervisionada (Base 1) + classificação supervisionada D0 (Base 2) com verificação anti data leakage.

## O dataset

Usamos o **Online Retail (UCI)** — `data/Online Retail.xlsx` — como proxy metodológico do dataset Ford sintético (ainda não disponibilizado pelo professor). Cada `CustomerID` representa um comprador, RFM cobre o comportamento pós-venda (Base 1), e features da **primeira invoice** simulam D0 (Base 2). O método é idêntico ao que será aplicado ao dataset Ford — basta substituir `load_data()` quando os arquivos chegarem.

A justificativa metodológica completa está no header do notebook + na seção 13 (anti-leakage).

## Estrutura

```
PrevioPLS-ML/
├── README.md
├── requirements.txt
├── relatorio_executivo.md          ← resumo de negócio (1 página)
├── previoPLS_ml.ipynb              ← entrega principal (76 células)
├── data/
│   └── Online Retail.xlsx
└── scripts/
    └── build_notebook.py           ← gerador do .ipynb
```

## Como rodar

```bash
cd C:\Users\User\PrevioPLS-ML
python -m venv .venv
.\.venv\Scripts\activate          # Windows
# source .venv/bin/activate         # Linux/Mac
pip install -r requirements.txt
jupyter notebook previoPLS_ml.ipynb
```

Tempo de execução completa: **~5-15 min** (KMeans + GridSearchCV em ~4k clientes é rápido; o gargalo é o XGBoost grid).

## O que o notebook entrega

### Parte 1 — Segmentação (Base 1, não-supervisionado)

- EDA completa (missing, distribuições, outliers, sazonalidade, geografia).
- Limpeza com decisões justificadas em markdown.
- **RFM** clássico + 3 features auxiliares (AOV, UniqueProducts, Tenure).
- Pré-processamento (log + RobustScaler) com justificativa.
- Seleção consciente de variáveis (NÃO joga tudo no modelo — explica por que tirar Tenure/UniqueProducts).
- **3 algoritmos comparados**: K-Means, DBSCAN, Hierarchical (Ward).
- **K escolhido por elbow + silhouette + Davies-Bouldin** combinado.
- **Mapeamento explícito dos clusters aos 4 perfis Ford** (Fiel/Abandono/Esquecido/Econômico) com heurísticas derivadas das medianas R/F/M/AOV — nunca "Cluster 0/1/2".
- **Tabela de estratégia de retenção** por perfil (oferta + canal + timing + KPI).

### Parte 2 — Classificação (Base 2, supervisionado)

- Feature engineering **exclusivamente do D0** (primeira invoice do cliente).
- **Bloco de verificação anti-leakage** que falha o notebook se algum nome banido aparecer (`recency`, `frequency`, `monetary`, `tenure`, etc).
- Split estratificado.
- **3 modelos com GridSearchCV** (Logistic, Random Forest, XGBoost) e F1 macro como métrica de seleção.
- Avaliação no test set: classification report por classe + matriz de confusão.
- **Feature importance + SHAP** pra interpretabilidade (atende LGPD Art. 20).
- Serialização (`joblib`) do modelo final para o backend Java consumir.

### Parte 3 — Leitura executiva

Seção 20 do notebook. Resumo do perfis, prioridade de retenção, performance do modelo D0, e **pipeline operacional completo** mostrando como o modelo se integra ao backend `PrevioPLS/` (Spring Boot) + app mobile `PrevioPLS-Mobile/` (React Native).

## Regenerar o notebook

Se quiser modificar células sem editar o `.ipynb` à mão:

```bash
# edita scripts/build_notebook.py (lista CELLS)
python scripts/build_notebook.py
```

Isso regenera o `previoPLS_ml.ipynb` from scratch.

## Anti-leakage — onde está provado

O notebook documenta a regra inviolável em 3 pontos:

1. **Header** — declaração da regra US02 (a regra crítica do produto).
2. **Seção 13** — tabela de equivalência mostrando que toda feature D0 deriva exclusivamente da primeira invoice + analogia com o D0 Ford (regiao, modelo, valor_compra, concessionaria_id).
3. **Seção 14** — bloco programático que cruza features candidatas contra uma lista de tokens banidos (`recency`, `frequency`, `monetary`, `tenure`, etc) e **levanta `RuntimeError`** se detectar leak. Roda toda vez que o notebook executa — não dá pra "esquecer".

## Integração com o restante do PrevioPLS

| Saída do notebook                   | Quem consome                                                                              |
|-------------------------------------|------------------------------------------------------------------------------------------|
| `ml_model.pkl` (joblib)             | `MlService.classificar()` em `PrevioPLS/` (Spring Boot) — substitui o stub SHA-256       |
| Mapeamento `cluster_id → persona`   | `MlService.scriptParaPerfil()` (já tem os scripts comerciais por perfil)                  |
| Estratégia por perfil (seção 12)    | App mobile `PrevioPLS-Mobile/` mostra o script + sugestão na visão 360 do lead             |
| Métricas de retenção                | KPIs do produto (Conversão de Risco, VIN Share, Engajamento Operacional)                  |
