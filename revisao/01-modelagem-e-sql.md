# 01 — Modelagem e SQL Básico

## 1. Objetivos de aprendizagem

Ao final deste arquivo você será capaz de:

- Traduzir um modelo entidade-relacionamento (ER) em `CREATE TABLE` no Oracle.
- Usar corretamente `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK` e `NOT NULL`.
- Inserir, atualizar e apagar dados com `INSERT`, `UPDATE`, `DELETE`.
- Manipular datas com `TO_DATE` e `SYSDATE`.
- Escrever `SELECT`s básicos com filtros, ordenação e agregações simples.
- Identificar problemas de modelagem em um banco existente.

Este é o **primeiro arquivo de um projeto evolutivo**: o banco criado aqui será reutilizado e evoluído nos arquivos 02 a 06. Nada aqui é descartável.

---

## 2. Conceitos teóricos

### 2.1 Modelagem relacional

Um banco relacional organiza dados em **tabelas** (relações), onde cada linha é uma **tupla** e cada coluna um **atributo**. As regras de integridade mais comuns são:

- **Chave primária (PK):** identifica unicamente cada linha. Não pode ser nula nem repetida.
- **Chave estrangeira (FK):** referencia a PK de outra tabela, garantindo integridade referencial (não é possível inserir um filho apontando para um pai inexistente).
- **UNIQUE:** garante que uma coluna (ou conjunto de colunas) não se repita, mesmo não sendo PK.
- **CHECK:** restringe os valores válidos de uma coluna via uma expressão booleana.
- **NOT NULL:** impede valores ausentes.

### 2.2 Tipos de dados Oracle mais usados

| Tipo | Uso |
|---|---|
| `NUMBER(p,s)` | Números inteiros ou decimais (p = precisão, s = escala) |
| `VARCHAR2(n)` | Texto de tamanho variável até `n` caracteres |
| `DATE` | Data e hora (sempre com componente de hora, mesmo que não usado) |
| `CHAR(n)` | Texto de tamanho fixo (raramente necessário) |

Oracle não possui `INT` como tipo nativo — é um sinônimo de `NUMBER(38)`. Também não existe `VARCHAR` puro recomendado: usa-se sempre `VARCHAR2`.

### 2.3 Datas: `SYSDATE` e `TO_DATE`

- `SYSDATE` retorna a data/hora atual do servidor.
- `TO_DATE('valor', 'formato')` converte uma string em `DATE` usando uma máscara, por exemplo `TO_DATE('15/03/2024', 'DD/MM/YYYY')`.

Erro comum: comparar uma `DATE` armazenada (que sempre tem componente de hora, mesmo que `00:00:00`) com um valor sem considerar a hora — isso pode causar resultados inesperados em filtros de "dia inteiro".

---

## 3. Estrutura do banco

### 3.1 Domínio escolhido

Vamos construir e evoluir um **sistema de gestão de notícias e jornalistas** — uma pequena redação digital, onde notícias têm categoria e status, e são escritas por um ou mais jornalistas.

### 3.2 Modelo inicial (versão 1)

Este é o ponto de partida do projeto. É uma modelagem **propositalmente simples**: você vai evoluí-la (e criticá-la) nos próximos arquivos.

```sql
-- Ordem de execução: de cima para baixo (tabelas-pai antes das tabelas-filho)

CREATE TABLE tb_status_noticia (
    id_status_noticia  NUMBER(4)      NOT NULL,
    status              VARCHAR2(40)  NOT NULL,
    CONSTRAINT pk_status_noticia PRIMARY KEY (id_status_noticia)
);

CREATE TABLE tb_categoria (
    id_categoria  NUMBER(4)      NOT NULL,
    categoria     VARCHAR2(40)   NOT NULL,
    CONSTRAINT pk_categoria PRIMARY KEY (id_categoria)
);

CREATE TABLE tb_jornalista (
    id_jornalista      NUMBER(6)      NOT NULL,
    nome               VARCHAR2(80)   NOT NULL,
    email              VARCHAR2(80)   NOT NULL,
    telefone           VARCHAR2(20),
    data_contratacao   DATE           NOT NULL,
    CONSTRAINT pk_jornalista PRIMARY KEY (id_jornalista)
);

CREATE TABLE tb_noticia (
    id_noticia         NUMBER(8)      NOT NULL,
    titulo             VARCHAR2(120)  NOT NULL,
    conteudo           VARCHAR2(4000) NOT NULL,
    data_criacao       DATE           DEFAULT SYSDATE NOT NULL,
    id_status_noticia  NUMBER(4)      NOT NULL,
    id_categoria       NUMBER(4)      NOT NULL,
    CONSTRAINT pk_noticia PRIMARY KEY (id_noticia),
    CONSTRAINT fk_noticia_status FOREIGN KEY (id_status_noticia)
        REFERENCES tb_status_noticia (id_status_noticia),
    CONSTRAINT fk_noticia_categoria FOREIGN KEY (id_categoria)
        REFERENCES tb_categoria (id_categoria)
);

CREATE TABLE tb_noticia_jornalista (
    id_noticia_jornalista  NUMBER(10)  NOT NULL,
    id_noticia             NUMBER(8)   NOT NULL,
    id_jornalista          NUMBER(6)   NOT NULL,
    CONSTRAINT pk_noticia_jornalista PRIMARY KEY (id_noticia_jornalista),
    CONSTRAINT fk_nj_noticia FOREIGN KEY (id_noticia)
        REFERENCES tb_noticia (id_noticia),
    CONSTRAINT fk_nj_jornalista FOREIGN KEY (id_jornalista)
        REFERENCES tb_jornalista (id_jornalista)
);
```

**Validação:**

```sql
SELECT table_name FROM user_tables
WHERE table_name LIKE 'TB_%'
ORDER BY table_name;
```

Resultado esperado: as 5 tabelas listadas.

> **Nota de projeto:** este modelo tem lacunas intencionais (colunas sem tamanho ideal, sem `UNIQUE` em `email`, sem `CHECK` de valores de status/categoria pré-definidos, títulos limitados). Você vai analisá-las no desafio da seção 6 — não é erro seu se notar isso.

### 3.3 Sequences para geração de IDs

Oracle não tem `AUTO_INCREMENT`. Usamos `SEQUENCE`:

```sql
CREATE SEQUENCE seq_status_noticia START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE seq_categoria START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE seq_jornalista START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE seq_noticia START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE seq_noticia_jornalista START WITH 1 INCREMENT BY 1;
```

---

## 4. Exemplos práticos

### 4.1 Inserindo dados de apoio (status e categoria)

```sql
INSERT INTO tb_status_noticia (id_status_noticia, status)
VALUES (seq_status_noticia.NEXTVAL, 'Rascunho');

INSERT INTO tb_status_noticia (id_status_noticia, status)
VALUES (seq_status_noticia.NEXTVAL, 'Publicada');

INSERT INTO tb_status_noticia (id_status_noticia, status)
VALUES (seq_status_noticia.NEXTVAL, 'Arquivada');

INSERT INTO tb_categoria (id_categoria, categoria) VALUES (seq_categoria.NEXTVAL, 'Política');
INSERT INTO tb_categoria (id_categoria, categoria) VALUES (seq_categoria.NEXTVAL, 'Esportes');
INSERT INTO tb_categoria (id_categoria, categoria) VALUES (seq_categoria.NEXTVAL, 'Tecnologia');

COMMIT;
```

**Explicação:** `seq.NEXTVAL` gera o próximo valor da sequence a cada chamada. `COMMIT` confirma as alterações (será aprofundado no arquivo 05).

**Boas práticas:** sempre popule primeiro as tabelas "pai" (referenciadas por FK) antes das "filhas".

### 4.2 Inserindo jornalistas com datas

```sql
INSERT INTO tb_jornalista (id_jornalista, nome, email, telefone, data_contratacao)
VALUES (seq_jornalista.NEXTVAL, 'Marina Alves', 'marina.alves@redacao.com',
        '11988887777', TO_DATE('10/01/2021', 'DD/MM/YYYY'));

INSERT INTO tb_jornalista (id_jornalista, nome, email, telefone, data_contratacao)
VALUES (seq_jornalista.NEXTVAL, 'Bruno Castro', 'bruno.castro@redacao.com',
        '11977776666', TO_DATE('22/07/2022', 'DD/MM/YYYY'));

COMMIT;
```

**Erro comum:** usar `TO_DATE` com formato divergente do valor (`TO_DATE('2021-01-10', 'DD/MM/YYYY')`) — o Oracle lança `ORA-01861: literal não corresponde à string de formato`.

### 4.3 Inserindo notícias e o relacionamento N:N

```sql
INSERT INTO tb_noticia (id_noticia, titulo, conteudo, id_status_noticia, id_categoria)
VALUES (seq_noticia.NEXTVAL, 'Nova lei de trânsito entra em vigor',
        'Texto completo da notícia sobre a nova legislação...',
        2, 1); -- status 2 = Publicada, categoria 1 = Política

INSERT INTO tb_noticia_jornalista (id_noticia_jornalista, id_noticia, id_jornalista)
VALUES (seq_noticia_jornalista.NEXTVAL, 1, 1);

COMMIT;
```

**Observação:** aqui usamos os IDs "chutados" (1, 2...) para simplificar — em código real, capture o valor gerado com `RETURNING id_noticia INTO :variavel` (retomaremos isso no PL/SQL).

### 4.4 Atualizando e apagando dados

```sql
-- Atualizar o status de uma notícia
UPDATE tb_noticia
SET id_status_noticia = 3   -- Arquivada
WHERE id_noticia = 1;

-- Remover um vínculo notícia-jornalista
DELETE FROM tb_noticia_jornalista
WHERE id_noticia = 1 AND id_jornalista = 1;

COMMIT;
```

**Cuidado:** `DELETE FROM tb_jornalista WHERE id_jornalista = 1;` falharia com `ORA-02292` se existir alguma notícia vinculada a ele — isso é a integridade referencial protegendo o banco.

### 4.5 Consultas básicas

```sql
-- Todas as notícias publicadas
SELECT titulo, data_criacao
FROM tb_noticia
WHERE id_status_noticia = 2
ORDER BY data_criacao DESC;

-- Notícias criadas nos últimos 30 dias
SELECT titulo
FROM tb_noticia
WHERE data_criacao >= SYSDATE - 30;

-- Quantidade de notícias por categoria
SELECT id_categoria, COUNT(*) AS qtd
FROM tb_noticia
GROUP BY id_categoria;
```

---

## 5. Exercícios práticos

1. Insira mais 2 categorias e 2 status de sua escolha.
2. Cadastre 3 novos jornalistas, com datas de contratação diferentes, usando `TO_DATE`.
3. Cadastre 4 novas notícias, distribuindo entre categorias e status diferentes, e associe cada uma a pelo menos 1 jornalista (algumas com 2).
4. Escreva um `SELECT` que liste o nome dos jornalistas contratados há mais de 2 anos (use `SYSDATE`).
5. Escreva um `UPDATE` que mude o status de todas as notícias de uma categoria específica para "Arquivada".
6. Escreva um `SELECT` que conte quantas notícias cada jornalista já escreveu (dica: você vai precisar de `JOIN`, mas se ainda não viu, pode usar subquery — o `JOIN` formal vem no próximo arquivo).

---

## 6. Desafios de melhoria do banco

Analise o modelo criado na seção 3.2 e responda (não corrija ainda — apenas identifique e justifique):

1. A coluna `email` em `tb_jornalista` deveria ter alguma restrição adicional além de `NOT NULL`? Qual e por quê?
2. `titulo VARCHAR2(120)` é suficiente para uma manchete? E `conteudo VARCHAR2(4000)` para o corpo de uma notícia longa? O que aconteceria se você tentasse inserir mais que isso?
3. Hoje, `status` e `categoria` são strings livres nas tabelas de apoio — nada impede cadastrar "Publicada" e "publicada" como registros diferentes. Isso é um problema? Como você mitigaria?
4. A tabela `tb_noticia_jornalista` impede que o mesmo jornalista seja associado duas vezes à mesma notícia? Teste e explique o resultado.
5. Existe algum dado sensível de auditoria faltando (por exemplo, quando um registro foi alterado pela última vez)? Proponha, sem implementar ainda, o que adicionaria.

Guarde suas respostas — vamos revisitar e implementar correções reais nos arquivos seguintes.

---

## 7. Questões de revisão

1. Explique a diferença entre `PRIMARY KEY` e `UNIQUE` no contexto da tabela `tb_jornalista`. Em que cenário você usaria cada uma?
2. O comando abaixo falha. Identifique o erro e explique por quê:
   ```sql
   INSERT INTO tb_noticia (id_noticia, titulo, id_status_noticia, id_categoria)
   VALUES (seq_noticia.NEXTVAL, 'Título de teste', 99, 1);
   ```
3. Escreva uma consulta que retorne o título das notícias e o nome da categoria correspondente, sem usar `JOIN` (use subquery correlacionada ou não correlacionada).
4. Um colega sugere trocar `DATE` por `VARCHAR2(10)` na coluna `data_criacao` para "facilitar a formatação". Avalie essa proposta: quais problemas isso traria para consultas e ordenações?
5. Dado o cenário: você precisa impedir que uma notícia seja inserida com uma categoria que não existe em `tb_categoria`. Isso já está garantido no modelo atual? Justifique tecnicamente.
