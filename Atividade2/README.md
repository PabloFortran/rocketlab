# Atividade 2 - Arquitetura Medalhão (CineData Analytics)

Projeto de ETL feito no Databricks com a base de filmes TMDB/IMDb, seguindo as camadas Bronze, Silver e Gold. No fim, a Gold está modelada em Star Schema e as perguntas de negócio foram respondidas direto nela.

Os notebooks devem ser rodados na ordem: bronze, silver e gold. Os CSVs ficam em um Volume do Databricks e as tabelas são criadas no catálogo workspace, nos schemas bronze, silver e gold.

## Bronze

Cada CSV é lido sem inferir tipos (tudo como string) e gravado em Delta no modo append, com a coluna ingestion_datetime. A cotação do dólar vem da API do Banco Central, com as datas definidas por widgets (por padrão, os últimos 7 dias). Como o modo é append, rodar o notebook mais de uma vez duplica as linhas. Foi o que aconteceu nas minhas execuções, e a Silver cuida disso.

## Silver

O que mais deu trabalho foi a sujeira da origem

Duplicados: a ingestion_datetime é a mesma para todas as linhas de uma mesma carga, então ela sozinha não decide qual versão é a mais recente. Quando há empate, fica a linha com mais colunas preenchidas depois da limpeza.

Datas: olhando os dados, as datas com barra estão em dia/mês/ano e as com hífen em mês-dia-ano (nunca há mês maior que 12 nesses campos). O que não converte em nenhum formato vira NULL. Status foi normalizado antes de traduzir, e o que não é reconhecível (por exemplo, uma data que caiu na coluna de status) virou "Não Informado".

Financeiro: alguns orçamentos vêm como "34.0M", que significa 34 milhões. Se só os símbolos fossem removidos, o valor viraria 34 dólares, então o sufixo é interpretado. Zero e negativos viram NULL. A conversão para reais usa a cotação mais recente da API. Lucro só é calculado quando orçamento e receita existem, e a margem é lucro dividido pela receita.

Deslocamento de colunas: em várias linhas há texto no lugar de números e vice-versa. Nas métricas, usei conversão segura (texto vira NULL) e notas fora de 0 a 10 também. Em linhas onde a coluna de nota do IMDb tem texto, a popularidade foi descartada, porque nesses casos ela trazia valores de outra coluna (como um ano). Nos gêneros, além de remover números, filtrei por formato e por frequência mínima, para tirar frases e caminhos de imagem que caíram ali. Nos nomes de pessoas e empresas, removi textos longos e com pontuação.

A cotação do dólar foi estendida para todos os dias do período, com os fins de semana e feriados recebendo o último valor disponível.

## Gold

As chaves substitutas são BIGINT, geradas com row_number sobre a chave natural. A fato tem uma linha por filme lançado. As tabelas ponte ligam filmes a gêneros, pessoas e produtoras, e só incluem filmes que existem na dim_movies.

Nas perguntas 5 e 6, a data limite é o lançamento mais recente já ocorrido na base, que é 19/02/2026. Os últimos 2 e 5 anos são contados a partir dela.

