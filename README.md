# Case XPCourses

Este projeto é um estudo de caso prático de análise de dados, focando na plataforma de produtos digitais fictícia XPCourses.

### Conjunto de dados:

<img width="640" height="362" alt="image" src="https://github.com/user-attachments/assets/6464ae4e-6e12-4002-bf41-65da0c6ae881" />

---

O objetivo principal foi utilizar a linguagem SQL para responder a questões de negócio, com ênfase em métricas de desempenho.

Antes de iniciar as análises, foi realizado um processo de preparação dos dados para garantir a qualidade e integridade das informações. As seguintes ações foram tomadas:

#### Tabela 'Sales'
- Linhas duplicadas foram removidas para assegurar a precisão dos dados de vendas.

#### Tabela ‘Product’

- A coluna 'deletion_date' apresentava datas em mais de um formato, incluindo DD/MM/AAAA, diferente do padrão adotado (AAAA-MM-DD). Todas as datas foram ajustadas para o mesmo formato (AAAA-MM-DD).
- A coluna 'niche' continha duplicidade de grafias para o nicho 'Tecnologia e Inovação', aparecendo tanto como 'Tecnologia-e-Informação' quanto como 'Tecnologia e Inovação'. Os registros foram padronizados.

---

## **Based on the available data ‘SQL datasets.zip”. We would like to know:**

### 1. The top 10 products that sold the most in each niche with deactivated membership area and activated recovery.

Objetivo: Retornar os **10 produtos mais vendidos em cada nicho** considerando apenas aqueles com **área de membros desativada** e **recuperação ativa**.
Para isso, a query calcula o total de vendas por produto e, em seguida, gera um ranking dentro de cada nicho.

**CTE `product_sales`**
- São selecionados os dados da tabela de vendas (`sales`) relacionados aos produtos (`product`).
- O filtro aplicado garante que apenas produtos com `member_area_active = FALSE` e `recovery_active = TRUE` sejam considerados.
- O `COUNT(s.product_id)` calcula o total de vendas de cada produto.
- O `GROUP BY s.product_id, p.niche` garante que o resultado esteja agregado por produto e nicho.
    
**CTE `product_ranking`**
- A partir da CTE `product_sales`, aplica-se a função  `RANK()` para gerar a posição de cada produto dentro do seu nicho.
- A cláusula `PARTITION BY psl.niche ORDER BY psl.total_sales DESC` faz com que o ranking seja **reiniciado para cada nicho**, ordenando os produtos do mais vendido para o menos vendido.
- O uso de `RANK()` (em vez de `ROW_NUMBER()`) garante que produtos com o mesmo número de vendas fiquem empatados na mesma posição no ranking.
    
```
WITH product_sales AS
  (SELECT s.product_id,
          p.niche,
          COUNT(s.product_id) AS total_sales
   FROM sales s
   JOIN product p ON p.product_id = s.product_id
   WHERE p.member_area_active = FALSE
     AND p.recovery_active = TRUE
   GROUP BY s.product_id,
            p.niche),
     product_ranking AS
  (SELECT psls.*,
          RANK() OVER(PARTITION BY psls.niche
                      ORDER BY psls.total_sales DESC) AS rnk
   FROM product_sales psls)
SELECT prkn.*
FROM product_ranking prkn
WHERE rnk <= 10
ORDER BY niche,
         rnk
```

<img width="279" height="558" alt="image" src="https://github.com/user-attachments/assets/d2e1bf87-91bd-4a2b-8325-2e78fab262c9" />

---

### 2. The top 10 producers who joined XPCourses from 2020 onwards and achieved the highest sales using recovery.

Objetivo: Obter os **10 produtores que mais venderam utilizando a funcionalidade de recovery**, considerando apenas aqueles que se registraram na plataforma **a partir de 2020**.

#### CTE `producers_sales`
- Nessa etapa, são contabilizadas as vendas por produtor.
- O `JOIN` entre `sales` e `product` conecta as vendas aos produtos, e consequentemente, aos produtores.
- O filtro `pdt.recovery_active = TRUE` garante que apenas produtos com **recovery ativo** sejam considerados.
- O `COUNT(s.purchase_id)` soma o número de vendas de cada produtor.
- O `GROUP BY pdt.producer_id` agrega os dados por produtor.

### CTE `producers_rnk`
- A partir da CTE `producers_sales`, aplica-se a função `RANK()`.
- O `ORDER BY ps.total_sales DESC` gera o ranking dos produtores do maior para o menor em vendas.
- O `JOIN` com a tabela `producer` adiciona informações dos produtores e aplica o filtro de data considerando apenas produtores que se registraram em **2020 ou depois**.

```
WITH producers_sales AS
  (SELECT pdt.producer_id,
          COUNT(s.purchase_id) AS total_sales
   FROM sales s
   JOIN product pdt ON pdt.product_id = s.product_id
   WHERE pdt.recovery_active = TRUE
   GROUP BY pdt.producer_id),
     producers_rnk AS
  (SELECT ps.*,
          RANK() OVER(
                      ORDER BY ps.total_sales DESC) AS rnk
   FROM producers_sales ps
   JOIN producer pdr ON pdr.producer_id = ps.producer_id
   WHERE DATE_TRUNC('YEAR', pdr.registry_date) >= '2020-01-01')
SELECT pr.*
FROM producers_rnk pr
ORDER BY rnk
```

<img width="200" height="137" alt="image" src="https://github.com/user-attachments/assets/ee357d78-63c8-4e04-ab1d-e1e51834ad53" />

---

## 3. How much more a producer with the recovery feature activated is likely to sell in each niche? Consider only producers who registered from 2020 onwards.

Neste caso, não é possível estimar de forma confiável quanto a mais um produtor com a funcionalidade de recuperação ativada tende a vender em cada nicho, considerando apenas produtores cadastrados a partir de 2020. O motivo é duplo:

- Todos os produtores registrados a partir de 2020 possuem apenas produtos com a funcionalidade de recuperação ativada, o que impede a criação de um grupo de controle para comparação.
- Uma comparação justa exigiria dados de vendas dos mesmos produtos, pelos mesmos produtores, nas mesmas condições, com e sem a funcionalidade ativada, algo que só seria possível por meio de um teste A/B. No entanto, esses dados não estão disponíveis no conjunto de dados atual.

Portanto, qualquer tentativa de quantificar essa diferença poderia gerar conclusões enviesadas ou enganosas, e por isso não foi proposta nenhuma query ou cálculo alternativo.

```
SELECT pr.producer_id,
       p.niche,
       pr.registry_date,
       p.recovery_active
FROM producer AS pr
JOIN product AS p ON pr.producer_id = p.producer_id
WHERE pr.registry_date >= '2020-01-01'
ORDER BY p.niche,
         p.registry_date
```

<img width="424" height="442" alt="image" src="https://github.com/user-attachments/assets/faaf471a-061e-467e-8f42-47939bc441a3" />

---

## 4. The product niche(s) with the highest number of cancellations and refunds.

Objetivo: Identificar o(s) **nicho(s) de produto com maior quantidade de cancelamentos e reembolsos**.

### CTE `fail_purchases`
- Nesta etapa, são contabilizados os cancelamentos e reembolsos por nicho.
- O `JOIN` entre `sales` e `product` conecta cada venda ao seu produto e, consequentemente, ao nicho correspondente.
- O filtro `WHERE cancelled = TRUE OR refund = TRUE` garante que apenas vendas canceladas ou reembolsadas sejam consideradas.
- O `COUNT(s.purchase_id)` calcula o total de ocorrências de falha por nicho.
- O `GROUP BY p.niche` agrega os dados por nicho.

### CTE `ranked_niches`
- A partir da tabela `fail_purchases`, aplica-se a função `RANK()` para gerar o ranking dos nichos de acordo com a quantidade de cancelamentos e reembolsos.
- A cláusula `ORDER BY total DESC` garante que o nicho com mais ocorrências fique na primeira posição.
- O uso de `RANK()` (em vez de `ROW_NUMBER()`) permite que **nichos empatados** com o mesmo número de falhas tenham a mesma posição no ranking.

```
WITH fail_purchases AS
  (SELECT p.niche,
          COUNT(s.purchase_id) AS total
   FROM sales s
   JOIN product p ON p.product_id = s.product_id
   WHERE cancelled = TRUE
     OR refund = TRUE
   GROUP BY p.niche),
     ranked_niches AS
  (SELECT niche,
          total,
          RANK() OVER(
                      ORDER BY total DESC) AS rnk
   FROM fail_purchases)
SELECT *
FROM ranked_niches
WHERE rnk <= 3;
```
<img width="229" height="115" alt="image" src="https://github.com/user-attachments/assets/1cef2a96-41da-4d57-9a41-cb8c89745b1f" />

---

## 5. Calculate the total money lost by producer due to cancellations and refunds. Is there any difference for producers considering products with the recovery tool activated?
Objetivo: Calcular o **total de dinheiro perdido por cada produtor** devido a cancelamentos e reembolsos, distinguindo entre produtos com a **funcionalidade de recuperação ativada ou desativada**, para avaliar possíveis diferenças de impacto.

### Consulta principal

- Foi feito um `JOIN` entre as tabelas `sales` e `product` para relacionar cada venda ao seu produtor e ao status de recuperação do produto.
- Para cada produtor, foram calculadas somas condicionais (`SUM(CASE WHEN ...)`) separando:
    - **Reembolsos em produtos com recovery ativo:** `total_refund_recovery_active`
    - **Reembolsos em produtos com recovery desativado:** `total_refund_recovery_disabled`
    - **Cancelamentos em produtos com recovery ativo:** `total_cancel_recovery_active`
    - **Cancelamentos em produtos com recovery desativado:** `total_cancel_recovery_disabled`
    - **Total perdido:** `total_lost` (soma de todas as perdas independentemente do estado de recovery)
  
### Análise dos resultados

- O produtor com maior perda foi o **producer_id = 1**, que acumulou **US$ 10,316.00** em cancelamentos e reembolsos, todos em produtos com recovery ativo.
- Para a maioria dos produtores, todas as perdas aconteceram **exclusivamente em produtos com recovery ativo**, resultando em valores **zerados** para produtos sem recovery.
- O **único** produtor que possui produtos com recovery desativado e apresentou perdas nesses casos foi o **`producer_id = 8`**, que perdeu **US$ 9,924.00** no total.

```
SELECT p.producer_id,
       SUM(CASE WHEN s.refund = TRUE
                    AND p.recovery_active = TRUE THEN s.product_price ELSE 0 END) AS total_refund_recovery_active,
       SUM(CASE WHEN s.refund = TRUE
                    AND p.recovery_active = FALSE THEN s.product_price ELSE 0 END) AS total_refund_recovery_disabled,
       SUM(CASE WHEN s.cancelled = TRUE
                    AND p.recovery_active = TRUE THEN s.product_price ELSE 0 END) AS total_cancel_recovery_active,
       SUM(CASE WHEN s.cancelled = TRUE
                    AND p.recovery_active = FALSE THEN s.product_price ELSE 0 END) AS total_cancel_recovery_disabled,
       SUM(CASE WHEN s.cancelled = TRUE
                    OR s.refund = TRUE THEN s.product_price ELSE 0 END) AS total_lost
FROM sales s
JOIN product p ON s.product_id = p.product_id
GROUP BY p.producer_id
ORDER BY p.producer_id;
```

<img width="917" height="282" alt="image" src="https://github.com/user-attachments/assets/ccecb5a7-3e9c-42c4-8979-79c8773fceb3" /> <br>

Para entender melhor o caso do producer_id = 8, foi feita uma consulta destacando suas perdas com e sem recovery:

```
SELECT producer_id,
       SUM(CASE WHEN (cancelled = TRUE
                      OR refund = TRUE)
                    AND p.recovery_active = TRUE THEN s.product_price ELSE 0 END) AS total_lost_with_recovery,
       SUM(CASE WHEN (cancelled = TRUE
                      OR refund = TRUE)
                    AND p.recovery_active = FALSE THEN s.product_price ELSE 0 END) AS total_lost_without_recovery,
       SUM(CASE WHEN (cancelled = TRUE
                      OR refund = TRUE) THEN s.product_price ELSE 0 END) AS total_lost
FROM sales s
JOIN product p ON s.product_id = p.product_id
WHERE p.producer_id = 8
GROUP BY producer_id;
```
<img width="489" height="61" alt="image" src="https://github.com/user-attachments/assets/6e46f714-bc63-4107-9f18-778187403e36" /> <br><br>

- O produtor **8** perdeu **$9,924.00 no total**.
- Desse valor, **71,66%** das perdas ($7,112.00) ocorreram em **produtos sem recovery**, enquanto apenas **28,34%** ($2,812.00) foram em produtos com recovery ativo.
- Esse é o único caso em que é possível comparar de forma direta o impacto da ferramenta de recuperação.

---

## 6. If you need to create a ranking of the top creators of 2022, which variables you consider crucial for ranking them? You can also create variables from the data. You must explain your reasoning and your choice of variables and show how this reflect in your SQL code.

Objetivo: Criar um ranking de produtores com base em seu desempenho de vendas durante o ano de 2022.

**Variáveis consideradas:**

**`total_amount`**: Essa variável, que calcula a soma do preço de todos os produtos vendidos, é a métrica principal. Ela mostra o **volume financeiro** gerado pelo produtor, sendo essencial para identificar quem realmente traz mais receita para o negócio. É a base do nosso ranking (`ORDER BY total_amount DESC`).

**`total_purchases`**: O número de compras totais ajuda a entender o perfil de vendas do produtor. Um alto faturamento combinado com um alto volume de compras pode indicar produtos de menor valor com grande aceitação, enquanto um alto faturamento com poucas compras pode indicar produtos de alto valor (ticket alto).

**`refund_rate`**: Esta é uma métrica crucial para avaliar a **qualidade e a satisfação do cliente**. Mesmo com muitas vendas, um alto índice de reembolso pode sinalizar que o produto não está entregando o que promete, levando à insatisfação do cliente. A taxa é calculada dividindo o número total de reembolsos (`total_refunds`) pelo número total de compras (`total_purchases`).


- A consulta SQL une as tabelas `sales` e `product` para associar cada venda ao seu produtor. 
- O filtro `WHERE s.cancelled = false` e `DATE_PART('year', s.purchase_date) = 2022` garante que apenas vendas válidas do ano de 2022 sejam consideradas. 
- Finalmente, a consulta agrupa os resultados por `producer_id` para agregar as métricas de cada produtor.

```
SELECT DISTINCT p.producer_id,
                SUM(s.product_price) AS total_amount,
                COUNT(DISTINCT s.purchase_id) AS total_purchases,
                COUNT(CASE
                          WHEN s.refund = TRUE THEN 1
                          ELSE NULL
                      END) AS total_refunds,
                ROUND(COUNT(CASE
                                WHEN s.refund = TRUE THEN 1
                                ELSE NULL
                            END)::NUMERIC / COUNT(DISTINCT s.purchase_id), 2) AS refund_rate
FROM sales s
JOIN product p ON p.product_id = s.product_id
WHERE s.cancelled = FALSE
  AND DATE_PART('year', s.purchase_date) = 2022
GROUP BY p.producer_id
ORDER BY total_amount DESC
```

<img width="453" height="280" alt="Captura de Tela 2025-08-21 às 20 24 53" src="https://github.com/user-attachments/assets/64f588d8-2c9d-43cf-bbd0-c23c71cb529b" /> <br/>

- O **`producer_id = 1`** é o líder de faturamento, com uma receita de $63.882,00 e 86 vendas, e mantém uma taxa de reembolso muito baixa (1%).
- Produtores com os **`ID's 8, 7 e 6`** se destacam por terem uma taxa de reembolso de 0%, indicando uma ótima aceitação de seus produtos e alta satisfação do cliente, mesmo com volumes de venda consideráveis.
- O **`producer_id = 2`**, apesar de estar bem posicionado em faturamento, apresenta a maior taxa de reembolso da lista (17%). Isso pode sinalizar uma oportunidade de melhoria na qualidade dos produtos, no alinhamento de expectativas ou no suporte ao cliente.
