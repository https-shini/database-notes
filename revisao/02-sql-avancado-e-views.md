# 02 — SQL Avançado e Views

## 1. Objetivos de aprendizagem

Ao final deste arquivo você será capaz de:

- Combinar dados de múltiplas tabelas com `JOIN` (INNER, LEFT).
- Usar subqueries e funções de agregação com `GROUP BY`/`HAVING`.
- Criar e usar `VIEW`s para simplificar consultas recorrentes.
- Evoluir a estrutura do banco com `ALTER TABLE`, corrigindo parte das lacunas identificadas no arquivo 01.

Pré-requisito: ter executado todos os scripts do arquivo `01-modelagem-e-sql.md`.

---

## 2. Conceitos teóricos

### 2.1 JOINs

- **INNER JOIN:** retorna apenas linhas com correspondência em ambas as tabelas.
- **LEFT JOIN:** retorna todas as linhas da tabela à esquerda, mesmo sem correspondência (colunas da direita vêm `NULL`).

Uso típico: `tb_noticia` LEFT JOIN `tb_noticia_jornalista` para encontrar notícias **sem** jornalista associado.

### 2.2 Subqueries

Uma subquery é uma consulta dentro de outra. Pode aparecer em `WHERE`, `FROM` ou `SELECT`. É **correlacionada** quando referencia uma coluna da consulta externa (é reavaliada linha a linha) e **não correlacionada** quando é independente (avaliada uma vez).

### 2.3 Funções de agregação e GROUP BY / HAVING

`COUNT`, `SUM`, `AVG`, `MAX`, `MIN` resumem grupos de linhas definidos por `GROUP BY`. `HAVING` filtra **grupos** (depois da agregação), enquanto `WHERE` filtra **linhas** (antes da agregação).

### 2.4 Views

Uma `VIEW` é uma consulta salva com nome, tratada como uma tabela virtual. Não armazena dados fisicamente (por padrão) — a cada `SELECT` na view, a query subjacente é reexecutada. Usos: simplificar consultas complexas recorrentes e restringir o acesso a colunas sensíveis.

---

## 3. Evolução do banco

### 3.1 O que muda e por quê

Vamos corrigir parte das lacunas identificadas no desafio do arquivo 01:

1. **Ampliar `conteudo`** para `CLOB` (texto longo sem limite prático de 4000 caracteres).
2. **Adicionar `UNIQUE` em `email`** de `tb_jornalista` — evita e-mails duplicados.
3. **Adicionar coluna `data_publicacao`** em `tb_noticia`, para diferenciar criação de publicação efetiva.
4. **Adicionar `data_atualizacao`** com `DEFAULT SYSDATE`, iniciando o rastro de auditoria.

```sql
-- 1) Ampliar conteudo para CLOB
ALTER TABLE tb_noticia MODIFY (conteudo CLOB);

-- 2) Garantir e-mail único
ALTER TABLE tb_jornalista
    ADD CONSTRAINT uq_jornalista_email UNIQUE (email);

-- 3) Nova coluna de data de publicação (pode ser nula até a notícia ser publicada)
ALTER TABLE tb_noticia ADD (data_publicacao DATE);

-- 4) Coluna de auditoria simples
ALTER TABLE tb_noticia ADD (data_atualizacao DATE DEFAULT SYSDATE NOT NULL);
```

**Como validar:**

```sql
SELECT column_name, data_type, nullable
FROM user_tab_columns
WHERE table_name = 'TB_NOTICIA'
ORDER BY column_id;

SELECT constraint_name, constraint_type
FROM user_constraints
WHERE table_name = 'TB_JORNALISTA';
```

Resultado esperado: `TB_NOTICIA` com as novas colunas `DATA_PUBLICACAO` e `DATA_ATUALIZACAO`; `TB_JORNALISTA` com uma constraint do tipo `U` (unique) para `email`.

> Note que ainda não resolvemos todos os pontos do desafio anterior (como o `CHECK` de status/categoria) — isso será tratado quando estudarmos triggers e regras de negócio mais adiante, onde faz mais sentido pedagogicamente.

---

## 4. Exemplos práticos

### 4.1 INNER JOIN — notícia com nome da categoria e status

```sql
SELECT n.titulo, c.categoria, s.status
FROM tb_noticia n
INNER JOIN tb_categoria c ON c.id_categoria = n.id_categoria
INNER JOIN tb_status_noticia s ON s.id_status_noticia = n.id_status_noticia
ORDER BY n.titulo;
```

### 4.2 JOIN com tabela associativa (N:N)

```sql
SELECT n.titulo, j.nome AS jornalista
FROM tb_noticia n
JOIN tb_noticia_jornalista nj ON nj.id_noticia = n.id_noticia
JOIN tb_jornalista j ON j.id_jornalista = nj.id_jornalista
ORDER BY n.titulo, j.nome;
```

### 4.3 LEFT JOIN — notícias sem jornalista associado

```sql
SELECT n.id_noticia, n.titulo
FROM tb_noticia n
LEFT JOIN tb_noticia_jornalista nj ON nj.id_noticia = n.id_noticia
WHERE nj.id_noticia IS NULL;
```

**Explicação:** quando não há correspondência, todas as colunas vindas de `tb_noticia_jornalista` ficam `NULL` — por isso o filtro `IS NULL` identifica a ausência de vínculo.

### 4.4 Subquery correlacionada — jornalistas com mais de 1 notícia

```sql
SELECT j.nome
FROM tb_jornalista j
WHERE (
    SELECT COUNT(*)
    FROM tb_noticia_jornalista nj
    WHERE nj.id_jornalista = j.id_jornalista
) > 1;
```

### 4.5 GROUP BY / HAVING — categorias com mais de 2 notícias publicadas

```sql
SELECT c.categoria, COUNT(*) AS qtd_publicadas
FROM tb_noticia n
JOIN tb_categoria c ON c.id_categoria = n.id_categoria
WHERE n.id_status_noticia = 2 -- Publicada
GROUP BY c.categoria
HAVING COUNT(*) > 2;
```

**Boas práticas:** todo `SELECT` com colunas agregadas e não agregadas precisa listar as não agregadas no `GROUP BY`, senão o Oracle lança `ORA-00979`.

### 4.6 Criando Views

```sql
CREATE OR REPLACE VIEW vw_noticias_completas AS
SELECT
    n.id_noticia,
    n.titulo,
    c.categoria,
    s.status,
    n.data_criacao,
    n.data_publicacao
FROM tb_noticia n
JOIN tb_categoria c ON c.id_categoria = n.id_categoria
JOIN tb_status_noticia s ON s.id_status_noticia = n.id_status_noticia;

CREATE OR REPLACE VIEW vw_producao_jornalista AS
SELECT
    j.id_jornalista,
    j.nome,
    COUNT(nj.id_noticia) AS total_noticias
FROM tb_jornalista j
LEFT JOIN tb_noticia_jornalista nj ON nj.id_jornalista = j.id_jornalista
GROUP BY j.id_jornalista, j.nome;
```

Uso:

```sql
SELECT * FROM vw_noticias_completas WHERE status = 'Publicada';
SELECT * FROM vw_producao_jornalista ORDER BY total_noticias DESC;
```

**Observação sobre erros:** `CREATE OR REPLACE VIEW` falha se você tentar mudar o número/nome de colunas de uma view referenciada por outros objetos dependentes com `WITH READ ONLY`, ou se a nova definição remover uma coluna usada por outra view. Prefira recriar (`DROP` + `CREATE`) em caso de mudanças estruturais grandes.

---

## 5. Exercícios práticos

1. Escreva um `LEFT JOIN` que liste todas as categorias e a quantidade de notícias em cada uma, incluindo categorias sem nenhuma notícia (dica: `LEFT JOIN` de `tb_categoria` para `tb_noticia`).
2. Escreva uma consulta com subquery que retorne os jornalistas que **nunca** escreveram nenhuma notícia.
3. Crie uma view `vw_noticias_por_categoria` que agregue a contagem de notícias por categoria e por status simultaneamente.
4. Usando `HAVING`, encontre jornalistas contratados há mais de 1 ano que escreveram menos de 2 notícias (combine `WHERE`, `JOIN`, `GROUP BY` e `HAVING`).
5. Atualize `data_publicacao` de todas as notícias com status "Publicada" para a mesma data de `data_criacao` (isso simula preencher um histórico).

---

## 6. Desafios de melhoria do banco

1. A view `vw_producao_jornalista` usa `LEFT JOIN`. O que mudaria no resultado se fosse `INNER JOIN`? Teste as duas versões e compare.
2. `conteudo` virou `CLOB`. Isso tem implicações em `ORDER BY conteudo` ou `WHERE conteudo = '...'`. Investigue e explique por que Oracle restringe certas operações em colunas `CLOB`.
3. Proponha uma consulta de performance: como você encontraria, sem escrever a lógica em PL/SQL ainda, quais notícias estão em "Rascunho" há mais de 30 dias (indicando possível esquecimento)?
4. As views criadas aqui não têm `WITH READ ONLY`. Pesquise essa cláusula e avalie se ela seria adequada para `vw_producao_jornalista`. Justifique.
5. Seu banco hoje não tem nenhum índice explícito além dos criados automaticamente para PK/UNIQUE. Que coluna(s) você indexaria para acelerar filtros comuns (ex.: `id_status_noticia`, `data_criacao`)? Não implemente ainda — apenas justifique.

---

## 7. Questões de revisão

1. Explique com suas palavras a diferença entre `WHERE` e `HAVING`, usando um exemplo da tabela `tb_noticia`.
2. A consulta abaixo retorna um erro. Identifique-o e corrija:
   ```sql
   SELECT c.categoria, n.titulo, COUNT(*)
   FROM tb_noticia n
   JOIN tb_categoria c ON c.id_categoria = n.id_categoria
   GROUP BY c.categoria;
   ```
3. Escreva uma consulta que liste, para cada categoria, o título da notícia mais recente (dica: subquery com `MAX(data_criacao)` ou análise de correlação).
4. Explique em que cenário uma `VIEW` melhora a manutenibilidade de um sistema, e em que cenário ela pode mascarar problemas de performance.
5. Um analista sugere transformar `vw_noticias_completas` em uma **tabela materializada** para acelerar relatórios. Quais seriam os trade-offs dessa decisão (atualização dos dados vs. velocidade de leitura)?
