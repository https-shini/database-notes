# 03 — PL/SQL Fundamentos

## 1. Objetivos de aprendizagem

Ao final deste arquivo você será capaz de:

- Escrever blocos PL/SQL anônimos com variáveis, `%TYPE` e `%ROWTYPE`.
- Usar estruturas condicionais (`IF`, `CASE WHEN`) e de repetição (`LOOP`, `WHILE`, `FOR`).
- Trabalhar com coleções (VARRAY, nested table, tabela associativa).
- Manipular o banco evolutivo dentro de blocos PL/SQL, não apenas em SQL puro.

Pré-requisito: arquivos 01 e 02 executados.

---

## 2. Conceitos teóricos

### 2.1 Estrutura de um bloco PL/SQL

```
DECLARE
    -- declarações de variáveis, cursores, tipos
BEGIN
    -- lógica procedural
EXCEPTION
    -- tratamento de erros
END;
/
```

`DECLARE` e `EXCEPTION` são opcionais; `BEGIN...END` é obrigatório. O `/` no SQL*Plus/SQLcl executa o bloco.

### 2.2 `%TYPE` e `%ROWTYPE`

- `variavel tb_noticia.titulo%TYPE;` — a variável herda o tipo exato da coluna, evitando dessincronia se o tipo da coluna mudar.
- `registro tb_noticia%ROWTYPE;` — a variável vira um "registro" com todas as colunas da tabela.

**Boa prática:** sempre prefira `%TYPE`/`%ROWTYPE` a tipos fixos (`VARCHAR2(120)`), pois o código se adapta automaticamente a mudanças de schema (relevante, já que este banco está evoluindo).

### 2.3 Condicionais

```sql
IF condicao THEN ... ELSIF condicao2 THEN ... ELSE ... END IF;

CASE variavel
    WHEN valor1 THEN ...
    WHEN valor2 THEN ...
    ELSE ...
END CASE;
```

`CASE` também existe como expressão (retorna um valor), útil dentro de `SELECT`.

### 2.4 Loops

- `LOOP ... EXIT WHEN condicao; ... END LOOP;` — loop genérico.
- `WHILE condicao LOOP ... END LOOP;` — testa a condição antes de cada iteração.
- `FOR i IN 1..10 LOOP ... END LOOP;` — faixa numérica; `FOR i IN REVERSE 1..10` inverte a ordem.
- `FOR reg IN (SELECT ...) LOOP ... END LOOP;` — cursor implícito, muito comum (veremos cursores explícitos no próximo arquivo).

### 2.5 Coleções

| Tipo | Característica |
|---|---|
| `VARRAY` | Tamanho máximo fixo, índices densos a partir de 1 |
| Nested Table | Tamanho dinâmico, pode ter "buracos" após `DELETE` |
| Tabela associativa (`INDEX BY`) | Indexada por número ou string, não precisa de tipo de objeto no banco |

---

## 3. Evolução do banco

Não há alteração estrutural neste arquivo — o foco é usar o banco existente dentro de PL/SQL. Isso também é uma lição prática: **nem toda evolução é `ALTER TABLE`**; muitas vezes você evolui a *camada de acesso* aos dados.

---

## 4. Exemplos práticos

### 4.1 Bloco anônimo com `%TYPE` e `IF`

```sql
DECLARE
    v_titulo   tb_noticia.titulo%TYPE;
    v_qtd_jorn NUMBER;
BEGIN
    SELECT titulo INTO v_titulo
    FROM tb_noticia
    WHERE id_noticia = 1;

    SELECT COUNT(*) INTO v_qtd_jorn
    FROM tb_noticia_jornalista
    WHERE id_noticia = 1;

    IF v_qtd_jorn = 0 THEN
        DBMS_OUTPUT.PUT_LINE('Notícia "' || v_titulo || '" sem jornalista associado.');
    ELSIF v_qtd_jorn = 1 THEN
        DBMS_OUTPUT.PUT_LINE('Notícia "' || v_titulo || '" tem 1 jornalista.');
    ELSE
        DBMS_OUTPUT.PUT_LINE('Notícia "' || v_titulo || '" tem ' || v_qtd_jorn || ' jornalistas.');
    END IF;
END;
/
```

**Resultado esperado:** uma linha impressa no console (lembre de `SET SERVEROUTPUT ON` antes de rodar).

**Erro comum:** `SELECT ... INTO` que não retorna nenhuma linha lança `NO_DATA_FOUND`; se retornar mais de uma, lança `TOO_MANY_ROWS`. Sempre garanta que a consulta retorna exatamente 1 linha, ou trate a exceção.

### 4.2 `%ROWTYPE` e `CASE`

```sql
DECLARE
    v_noticia tb_noticia%ROWTYPE;
    v_faixa   VARCHAR2(20);
BEGIN
    SELECT * INTO v_noticia
    FROM tb_noticia
    WHERE id_noticia = 1;

    v_faixa := CASE
        WHEN v_noticia.data_criacao >= SYSDATE - 7  THEN 'Recente'
        WHEN v_noticia.data_criacao >= SYSDATE - 30 THEN 'Últimas semanas'
        ELSE 'Antiga'
    END;

    DBMS_OUTPUT.PUT_LINE('Notícia "' || v_noticia.titulo || '" classificada como: ' || v_faixa);
END;
/
```

### 4.3 Loop `FOR ... IN (SELECT ...)`

```sql
BEGIN
    FOR rec IN (
        SELECT n.titulo, j.nome
        FROM tb_noticia n
        JOIN tb_noticia_jornalista nj ON nj.id_noticia = n.id_noticia
        JOIN tb_jornalista j ON j.id_jornalista = nj.id_jornalista
    ) LOOP
        DBMS_OUTPUT.PUT_LINE(rec.titulo || ' — por ' || rec.nome);
    END LOOP;
END;
/
```

**Boas práticas:** este tipo de loop (cursor implícito `FOR`) é preferível a abrir/fechar cursores manualmente para leituras simples, pois o Oracle gerencia a abertura/fechamento automaticamente.

### 4.4 `WHILE` e contagem regressiva

```sql
DECLARE
    v_contador NUMBER := 5;
BEGIN
    WHILE v_contador > 0 LOOP
        DBMS_OUTPUT.PUT_LINE('Faltam ' || v_contador || ' notícias para revisar.');
        v_contador := v_contador - 1;
    END LOOP;
END;
/
```

### 4.5 Coleções — tabela associativa com nomes de categorias

```sql
DECLARE
    TYPE t_categorias IS TABLE OF tb_categoria.categoria%TYPE
        INDEX BY PLS_INTEGER;

    v_categorias t_categorias;
    v_idx        PLS_INTEGER;
BEGIN
    v_idx := 1;
    FOR rec IN (SELECT categoria FROM tb_categoria ORDER BY categoria) LOOP
        v_categorias(v_idx) := rec.categoria;
        v_idx := v_idx + 1;
    END LOOP;

    FOR i IN 1 .. v_categorias.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE('Categoria ' || i || ': ' || v_categorias(i));
    END LOOP;
END;
/
```

### 4.6 Coleções — VARRAY com limite fixo

```sql
DECLARE
    TYPE t_top_titulos IS VARRAY(3) OF VARCHAR2(120);
    v_top t_top_titulos := t_top_titulos();
BEGIN
    FOR rec IN (
        SELECT titulo FROM tb_noticia
        ORDER BY data_criacao DESC
        FETCH FIRST 3 ROWS ONLY
    ) LOOP
        v_top.EXTEND;
        v_top(v_top.LAST) := rec.titulo;
    END LOOP;

    FOR i IN 1 .. v_top.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE('Top ' || i || ': ' || v_top(i));
    END LOOP;
END;
/
```

**Observação:** um `VARRAY(3)` lança erro se você tentar `.EXTEND` além do limite (`ORA-06532: Subscript outside of limit`). Use nested table ou tabela associativa quando o tamanho não for previsível.

---

## 5. Exercícios práticos

1. Escreva um bloco que receba (via variável declarada) o `id_jornalista` e imprima o nome e a quantidade de notícias escritas, usando `IF/ELSIF` para classificar como "Iniciante" (0-1 notícias), "Ativo" (2-5) ou "Produtivo" (6+).
2. Escreva um loop `FOR i IN REVERSE 1..5` que simule uma contagem regressiva de publicação.
3. Usando uma tabela associativa indexada por `VARCHAR2` (`INDEX BY VARCHAR2(40)`), conte quantas notícias existem por status (chave = nome do status, valor = contagem).
4. Escreva um bloco que percorra todas as notícias em "Rascunho" e imprima quantos dias se passaram desde a criação de cada uma.
5. Combine `CASE` e cursor implícito para classificar cada jornalista como "Veterano" (contratado há mais de 2 anos) ou "Novo", imprimindo o resultado.

---

## 6. Desafios de melhoria do banco

1. O exemplo 4.1 pode lançar `NO_DATA_FOUND` se `id_noticia = 1` não existir (por exemplo, se você já tiver apagado essa notícia em exercícios anteriores). Reescreva o bloco tratando essa exceção com `EXCEPTION WHEN NO_DATA_FOUND THEN ...` (tratamento formal de exceções será aprofundado depois, mas already vale treinar).
2. Avalie: usar `SELECT * INTO v_noticia FROM tb_noticia ...` (exemplo 4.2) é uma boa prática se a tabela ganhar novas colunas no futuro? O que muda no comportamento do `%ROWTYPE` comparado a listar colunas manualmente?
3. Os loops que usam `DBMS_OUTPUT.PUT_LINE` para "relatórios" não escalam para milhares de notícias (saída de console). Que alternativa você usaria para gerar relatórios grandes (pense em tabelas de log ou cursores com processamento em lote — sem implementar ainda)?
4. No exemplo 4.5, se duas categorias tivessem exatamente o mesmo nome (voltando ao desafio do arquivo 01), o que aconteceria com o índice da tabela associativa? Isso reforça algum dos problemas já identificados?
5. Proponha uma melhoria: como você reescreveria o exemplo 4.6 para não ter limite fixo de "top notícias", usando nested table em vez de VARRAY?

---

## 7. Questões de revisão

1. Qual é a diferença prática entre uma tabela associativa (`INDEX BY`) e uma nested table em termos de uso de memória e persistência no banco?
2. O bloco abaixo tem um erro de lógica. Identifique-o:
   ```sql
   DECLARE
       v_contador NUMBER := 1;
   BEGIN
       WHILE v_contador <= 10 LOOP
           DBMS_OUTPUT.PUT_LINE(v_contador);
       END LOOP;
   END;
   /
   ```
3. Escreva um bloco PL/SQL que, usando `FOR ... IN (SELECT ...)`, identifique e imprima o nome do jornalista com mais notícias publicadas.
4. Explique por que `%TYPE` é preferível a declarar `VARCHAR2(120)` diretamente, no contexto de um banco que ainda está evoluindo (como o deste projeto).
5. Descreva um cenário de negócio da redação de notícias em que `VARRAY` seria mais apropriado que uma tabela associativa, e outro em que o oposto seria verdade.
