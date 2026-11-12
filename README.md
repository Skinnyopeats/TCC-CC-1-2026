# TCC-CC-1-2026

Previsão da taxa de inadimplência de crédito a pessoas jurídicas no Brasil, seis meses à frente,
com dados públicos do Sistema de Informações de Crédito (SCR) do Banco Central. Trabalho de
Conclusão de Curso em Ciência da Computação, Centro Universitário IESB.

Este repositório contém apenas o necessário para reproduzir os resultados: os notebooks do
pipeline e as versões das bibliotecas. Dados, modelos treinados, tabelas e figuras são gerados
pela execução.

## Requisitos

- Python 3.11
- Cerca de 8 GB livres em disco (3,8 GB de arquivos do SCR mais os Parquets derivados)
- Acesso à internet na primeira execução (download do SCR e das séries do SGS/BCB)

```bash
python -m venv .venv
.venv/Scripts/pip install -r requirements.txt     # Linux/macOS: .venv/bin/pip
```

## Execução

Os notebooks devem ser executados em ordem, a partir da pasta `notebooks/`:

| Notebook | Etapa | Principais saídas em `data/processed/` |
|---|---|---|
| `00_coleta` | Download dos arquivos anuais do SCR (2012–2025) e das séries macroeconômicas do SGS | `data/raw/scr/*.zip`, `macro.parquet` |
| `01_preprocessamento` | Conversão dos ZIPs para Parquet, filtrando pessoas jurídicas | `scr_pj/scr_pj_<ano>.parquet` |
| `03_feature_engineering` | Agregação em segmentos (CNAE × porte × modalidade), variáveis com defasagem de publicação, alvo em t+6 | `ml_dataset.parquet`, `ml_metadata.json` |
| `04_modelos` | Regressão do nível: modelos, referenciais ingênuos, ARIMA, Diebold-Mariano, R² decomposto | `resultados_modelos.csv`, `resultados_within_segment.csv`, `resultados_diebold_mariano.csv` |
| `05_interpretabilidade` | Valores SHAP | `shap_importancias.csv` |
| `06_horizonte` | Desempenho de t+1 a t+12 | `resultados_horizonte.csv` |
| `07_classificacao` | Detecção de eventos de deterioração, DeLong, bootstrap de bloco mensal, limiar relativo, AUC mensal | `resultados_classificacao*.csv` |
| `08_features_scr` | Variáveis adicionais do SCR (ativo problemático, maturidade, efeito safra) | `ml_dataset_ext.parquet`, `resultados_features_scr*.csv` |
| `09_backtest` | Backtest de origem móvel (102 origens, 2016–2024) | `resultados_backtest*.csv` |

Execução sem interface, na raiz do repositório:

```bash
for nb in 00_coleta 01_preprocessamento 03_feature_engineering 04_modelos \
          05_interpretabilidade 06_horizonte 07_classificacao 08_features_scr 09_backtest; do
  jupyter nbconvert --to notebook --execute --inplace \
    --ExecutePreprocessor.timeout=-1 notebooks/$nb.ipynb
done
```

Os notebooks `00` e `01` são idempotentes e pulam arquivos já existentes. Do `03` em diante a
execução completa leva cerca de uma hora; o `09` é o mais demorado.

## Parâmetros do protocolo

- **Defasagem de publicação:** SCR 2 meses; Selic 0; IPCA e confiança 1; desocupação, spread,
  saldo de crédito e IBC-Br 2. Definidas no notebook `03` e gravadas em `ml_metadata.json`.
- **Embargo:** horizonte + defasagem do SCR = 8 meses, no corte único e em cada origem do backtest.
- **Corte único:** teste de 2022-01 a 2024-06.
- **Sementes:** 42 em todos os estimadores e no bootstrap.
- **Hiperparâmetros:** fixos, sem otimização; definidos nos próprios notebooks.

## Observação sobre os dados

O Banco Central republica arquivos do SCR de anos anteriores. Os resultados do trabalho foram
obtidos com os arquivos baixados em novembro de 2026; downloads em outra data podem produzir
números ligeiramente diferentes.

## Fontes

- SCR.data — arquivos `https://www.bcb.gov.br/pda/desig/planilha_<ano>.zip`
- SGS/BCB — séries 4189, 433, 24369, 20783, 4394, 24364, 4380, 20539, 20541
