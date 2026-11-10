##  O Motor Temporal e Otimização Quantitativa (Projeto 2)

Neste projeto, elevamos a Engenharia de Dados ao nível de **Séries Temporais** e **Análise Quantitativa**, resolvendo o problema crítico de *Data Leakage* (Vazamento de Dados) e enfrentando a eficiência financeira das casas de apostas.

### Extração e o "Escudo Temporal"
Para evitar que o modelo previsse o passado usando médias do futuro, construímos um Pipeline de ETL com Web Scraping (via *Wayback Machine*) para extrair o diário cronológico de jogos (Match Logs). 
* **Feature Engineering:** Criamos um Laço de Repetição Temporal que viaja rodada a rodada calculando a **Média Móvel de 5 jogos** para o Ataque (xG) e Defesa (xGA) de cada equipe, estritamente *antes* de a bola rolar.

### O Paradoxo da Acurácia vs. ROI
Focamos o *Random Forest* no mercado de Gols (Over/Under 2.5). Focamos o *Random Forest* no mercado de Gols (Over/Under 2.5). O modelo classificava a maior parte dos jogos com alta confiança, mas o Backtest Financeiro cruzado com as Odds da Bet365 revelou um **ROI negativo**. 
* **O Diagnóstico:** acurácia não vence o *Juice* (margem da casa) e, para um modelo probabilístico, acurácia nem é a métrica certa. O que importa é ter probabilidade mais bem calibrada que o mercado. O modelo apostava em todos os jogos com falsa confiança, comprando odds esmagadas sem Valor Esperado positivo.

### Calibração de Probabilidades e Radar EV+
Para reverter o prejuízo, implementamos técnicas de fundos quantitativos:
1. **Calibração:** Usamos o `CalibratedClassifierCV` (Scikit-Learn) para corrigir a superconfiança do Random Forest, forçando a IA a gerar probabilidades matemáticas reais.
2. **Otimização de Threshold:** Construímos um "Radar Quantitativo" em Python que simulou o campeonato dezenas de vezes para encontrar a margem de segurança perfeita. 
* **O Resultado:** o modelo de 12 variáveis chegou a um ROI muito próximo (52,30%, também com 10 entradas), mas com threshold ótimo diferente (1,29 contra 1,27) e hiperparâmetros diferentes, não foi uma comparação controlada.
* * **A Observação:** o filtro de EV+ alto reduz drasticamente o volume, restando pouquíssimas entradas por temporada. Com 10 operações não é possível distinguir habilidade preditiva de variância, nem concluir nada sobre a eficiência estrutural do mercado.

## Feature Engineering e a Barreira da Multicolinearidade

Após alcançar 52.30% de ROI com médias de xG (Expected Goals), a hipótese testada foi: *"Dar mais profundidade tática à máquina aumentará o volume de apostas lucrativas?"*
**Importante:** este threshold foi selecionado varrendo o conjunto de teste e escolhendo o valor de maior ROI. O número é, portanto, o teto otimista do modelo, não uma estimativa honesta de desempenho. Ver Limitações Metodológicas.

* **A Engenharia:** Expandimos o Motor Temporal extraindo dados do *football-data.co.uk*. Passamos de 4 variáveis para 12 dimensões táticas de Ataque e Defesa: `xG`, `xGA`, `Remates`, `Remates no Alvo`, `Cantos` e `Faltas`.
* **O Resultado:** o modelo de 12 variáveis chegou a um ROI muito próximo (52,30%, também com 10 entradas), mas com threshold ótimo diferente (1,29 contra 1,27) e hiperparâmetros diferentes — não foi uma comparação controlada.
* **A Observação:** o filtro de EV+ alto reduz drasticamente o volume, restando pouquíssimas entradas por temporada. Com 10 operações não é possível distinguir habilidade preditiva de variância, nem concluir nada sobre a eficiência
estrutural do mercado.
* **O Diagnóstico:** a verificação mostrou colinearidade moderada entre as variáveis de finalização (VIF entre X e Y), mas não redundância completa. A ausência de ganho no modelo de 12 variáveis não é totalmente explicada por multicolinearidade, o volume de apostas, restrito a 10 entradas, é a explicação mais provável.

## Limitações Metodológicas Conhecidas 

**1. O threshold de EV+ foi otimizado sobre o próprio conjunto de teste.** O "Radar Quantitativo" varre limiares de 1.00 a 1.35 e reporta o que maximiza o ROI no teste. Isso é seleção de hiperparâmetro em cima do conjunto de avaliação: o valor reportado não é a performance do modelo, é o máximo de aproximadamente trinta estimativas ruidosas. O procedimento correto é separar um conjunto de validação para escolher o threshold e reportar o resultado no teste, intocado. 

**2. Volume de operações baixo demais para inferência.** Dez entradas no Random Forest e cinco no XGBoost. Nesse volume, o intervalo de confiança do ROI cobre desde prejuízo relevante até lucro alto. Nenhum dos dois resultados é estatisticamente distinguível de acaso. 

**3. O threshold ótimo do XGBoost (1.32) está colado no limite superior da busca (1.35).** Ótimo na borda do espaço de busca é sinal clássico de que o procedimento está perseguindo ruído, e não encontrando um ponto de operação real. 

**4. Vazamento temporal na calibração.** `CalibratedClassifierCV(cv=5)` usa `StratifiedKFold` por padrão, que embaralha as observações. Em série temporal isso significa calibrar as probabilidades de um jogo usando partidas posteriores a ele. O motor de features foi construído com cuidado para evitar vazamento, mas ele reentrou pela calibração. A correção é `cv=TimeSeriesSplit(5)` ou um conjunto de calibração cronologicamente separado. 

**5. A calibração foi aplicada, mas nunca verificada.** Não há Brier score antes e depois, nem reliability diagram. Sem isso, não há evidência de que a "lente corretiva" tenha de fato corrigido a superconfiança do modelo. 

**6. O merge com as odds usa `how='inner'` sem verificação de contagem.** Se algum nome de equipe escapar do dicionário de padronização, as linhas correspondentes desaparecem silenciosamente da base. Nenhum ponto do pipeline imprime o número de linhas sobreviventes.
