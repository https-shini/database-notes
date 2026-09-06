# 04 — Procedures, Functions e Cursores

## 1. Objetivos de aprendizagem

Ao final deste arquivo você será capaz de:

- Criar e chamar `PROCEDURE`s com parâmetros `IN`, `OUT` e `IN OUT`.
- Criar `FUNCTION`s que retornam valores utilizáveis em SQL e PL/SQL.
- Declarar e manipular cursores explícitos, incluindo cursores parametrizados.
- Usar `RETURNING INTO` para capturar valores gerados por `INSERT`.

Pré-requisito: arquivos 01 a 03 executados.

---

## 2. Conceitos teóricos

### 2.1 Procedures vs Functions

| | Procedure | Function |
|---|---|---|
| Retorno | Não retorna valor (mas pode ter parâmetros `OUT`) | Retorna exatamente um valor via `RETURN` |
| Uso em SQL | Não pode ser chamada dentro de um `SELECT` | Pode (se determinística/sem efeitos colaterais problemáticos) |
| Uso típico | Ações (inserir, atualizar, orquestrar) | Cálculos, validações, transformação de dados |

### 2.2 Modos de parâmetro

- `IN` (padrão): valor de entrada, somente leitura dentro do bloco.
- `OUT`: valor de saída, a procedure escreve nele; o valor inicial passado é ignorado.
- `IN OUT`: entrada e saída — a procedure lê e pode sobrescrever.

### 2.3 Cursores explícitos

Um cursor explícito dá controle total sobre o ciclo de vida de uma consulta multi-linha:

```sql
DECLARE
    CURSOR c_noticias IS SELECT titulo FROM tb_noticia;
    v_titulo tb_noticia.titulo%TYPE;
BEGIN
    OPEN c_noticias;
    LOOP
        FETCH c_noticias INTO v_titulo;
        EXIT WHEN c_noticias%NOTFOUND;
        DBMS_OUTPUT.PUT_LINE(v_titulo);
    END LOOP;
    CLOSE c_noticias;
END;
/
```

Atributos úteis: `%FOUND`, `%NOTFOUND`, `%ROWCOUNT`, `%ISOPEN`.

### 2.4 Cursores parametrizados

```sql
CURSOR c_por_categoria (p_id_categoria NUMBER) IS
    SELECT titulo FROM tb_noticia WHERE id_categoria = p_id_categoria;
```

Permite reutilizar a mesma definição de cursor com valores diferentes a cada `OPEN`.

### 2.5 `RETURNING INTO`

Captura, no mesmo comando, um valor gerado (como uma PK vinda de sequence) sem precisar de um `SELECT` adicional:

```sql
INSERT INTO tb_noticia (id_noticia, titulo, conteudo, id_status_noticia, id_categoria)
VALUES (seq_noticia.NEXTVAL, 'Título X', 'Conteúdo...', 1, 1)
RETURNING id_noticia INTO v_novo_id;
```

---

## 3. Evolução do banco

Sem alterações estruturais. A evolução aqui é de **camada de código**: em vez de scripts SQL soltos, passamos a encapsular regras em procedures e functions reutilizáveis — preparando o terreno para packages no arquivo 05.

---

## 4. Exemplos práticos

### 4.1 Procedure com parâmetro IN — cadastrar jornalista

```sql
CREATE OR REPLACE PROCEDURE sp_cadastrar_jornalista (
    p_nome              IN tb_jornalista.nome%TYPE,
    p_email             IN tb_jornalista.email%TYPE,
    p_telefone          IN tb_jornalista.telefone%TYPE,
    p_data_contratacao  IN tb_jornalista.data_contratacao%TYPE
) IS
BEGIN
    INSERT INTO tb_jornalista (id_jornalista, nome, email, telefone, data_contratacao)
    VALUES (seq_jornalista.NEXTVAL, p_nome, p_email, p_telefone, p_data_contratacao);

    COMMIT;
END sp_cadastrar_jornalista;
/
```

Chamada:

```sql
BEGIN
    sp_cadastrar_jornalista(
        'Camila Rocha', 'camila.rocha@redacao.com',
        '11966665555', TO_DATE('05/02/2024', 'DD/MM/YYYY')
    );
END;
/
```

**Erro comum:** se `email` já existir (constraint `uq_jornalista_email` do arquivo 02), a procedure lança `ORA-00001: unique constraint violated` — trate isso com `EXCEPTION` se quiser uma mensagem amigável (veremos tratamento robusto de exceções no arquivo 05).

### 4.2 Procedure com OUT — cadastrar notícia retornando o ID

```sql
CREATE OR REPLACE PROCEDURE sp_cadastrar_noticia (
    p_titulo     IN  tb_noticia.titulo%TYPE,
    p_conteudo   IN  tb_noticia.conteudo%TYPE,
    p_categoria  IN  tb_noticia.id_categoria%TYPE,
    p_novo_id    OUT tb_noticia.id_noticia%TYPE
) IS
BEGIN
    INSERT INTO tb_noticia (id_noticia, titulo, conteudo, id_status_noticia, id_categoria)
    VALUES (seq_noticia.NEXTVAL, p_titulo, p_conteudo, 1, p_categoria) -- status 1 = Rascunho
    RETURNING id_noticia INTO p_novo_id;

    COMMIT;
END sp_cadastrar_noticia;
/
```

Chamada:

```sql
DECLARE
    v_id tb_noticia.id_noticia%TYPE;
BEGIN
    sp_cadastrar_noticia('Manchete de teste', 'Conteúdo de teste', 3, v_id);
    DBMS_OUTPUT.PUT_LINE('Notícia criada com ID ' || v_id);
END;
/
```

### 4.3 Function — contar notícias de um jornalista

```sql
CREATE OR REPLACE FUNCTION fn_contar_noticias_jornalista (
    p_id_jornalista IN tb_jornalista.id_jornalista%TYPE
) RETURN NUMBER IS
    v_total NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_total
    FROM tb_noticia_jornalista
    WHERE id_jornalista = p_id_jornalista;

    RETURN v_total;
END fn_contar_noticias_jornalista;
/
```

Uso em SQL puro e em PL/SQL:

```sql
SELECT nome, fn_contar_noticias_jornalista(id_jornalista) AS total
FROM tb_jornalista;
```

```sql
BEGIN
    DBMS_OUTPUT.PUT_LINE(fn_contar_noticias_jornalista(1));
END;
/
```

**Boas práticas:** functions chamadas dentro de `SELECT` devem evitar `COMMIT`/`DML` (podem lançar `ORA-14551`) — mantenha-as puramente de leitura/cálculo.

### 4.4 Cursor parametrizado — listar notícias por categoria

```sql
DECLARE
    CURSOR c_por_categoria (p_id_categoria NUMBER) IS
        SELECT titulo, data_criacao
        FROM tb_noticia
        WHERE id_categoria = p_id_categoria
        ORDER BY data_criacao DESC;

    v_rec c_por_categoria%ROWTYPE;
BEGIN
    OPEN c_por_categoria(1); -- categoria 1 = Política
    LOOP
        FETCH c_por_categoria INTO v_rec;
        EXIT WHEN c_por_categoria%NOTFOUND;
        DBMS_OUTPUT.PUT_LINE(v_rec.titulo || ' — ' || v_rec.data_criacao);
    END LOOP;
    CLOSE c_por_categoria;
END;
/
```

### 4.5 Function que valida regra de negócio (uso em IF)

```sql
CREATE OR REPLACE FUNCTION fn_pode_publicar (
    p_id_noticia IN tb_noticia.id_noticia%TYPE
) RETURN BOOLEAN IS
    v_qtd_jornalistas NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_qtd_jornalistas
    FROM tb_noticia_jornalista
    WHERE id_noticia = p_id_noticia;

    RETURN v_qtd_jornalistas > 0; -- regra: só pode publicar se tiver autor
END fn_pode_publicar;
/
```

```sql
DECLARE
    v_id tb_noticia.id_noticia%TYPE := 2;
BEGIN
    IF fn_pode_publicar(v_id) THEN
        UPDATE tb_noticia SET id_status_noticia = 2 WHERE id_noticia = v_id;
        COMMIT;
        DBMS_OUTPUT.PUT_LINE('Notícia publicada.');
    ELSE
        DBMS_OUTPUT.PUT_LINE('Não é possível publicar: sem jornalista associado.');
    END IF;
END;
/
```

**Nota:** funções `RETURN BOOLEAN` não podem ser chamadas diretamente em `SELECT` (SQL não tem tipo booleano nativo) — apenas em PL/SQL.

---

## 5. Exercícios práticos

1. Crie uma procedure `sp_atualizar_status_noticia(p_id_noticia, p_novo_status)` que atualiza o status e faz `COMMIT`.
2. Crie uma function `fn_media_noticias_por_jornalista` que retorne a média de notícias por jornalista ativo (com pelo menos 1 notícia).
3. Crie um cursor parametrizado que receba um intervalo de datas (`p_data_inicio`, `p_data_fim`) e liste as notícias criadas nesse período.
4. Crie uma procedure que receba `p_id_jornalista IN OUT` fictício para praticar o modo `IN OUT` — por exemplo, recebendo um ID e "normalizando" para um valor padrão caso seja inválido (não precisa ser realista, é para fixar a sintaxe).
5. Combine cursor + function: percorra todas as notícias em "Rascunho" e, usando `fn_pode_publicar`, publique automaticamente as que já têm jornalista associado.

---

## 6. Desafios de melhoria do banco

1. A procedure `sp_cadastrar_jornalista` (4.1) não trata a exceção de e-mail duplicado — ela simplesmente propaga o erro do banco. Isso é aceitável para este estágio do projeto? Que tipo de tratamento você adicionaria (sem implementar ainda, apenas descreva a estratégia)?
2. Cada procedure de escrita (4.1, 4.2) faz seu próprio `COMMIT`. Isso é uma boa prática quando múltiplas procedures precisam ser chamadas em sequência como parte de uma única operação de negócio? Qual problema isso pode causar (veremos a solução formal com transações no arquivo 05)?
3. A function `fn_contar_noticias_jornalista` é chamada em `SELECT` para cada linha de `tb_jornalista` (exemplo 4.3) — isso é conhecido como problema de performance N+1 quando o volume cresce. Como você reescreveria essa consulta usando `JOIN`/`GROUP BY` para evitar chamar a função linha a linha?
4. Avalie a regra de `fn_pode_publicar`: ela está "hardcoded" dentro da function. Se a regra de negócio mudar (por exemplo, exigir 2 jornalistas), isso quebra alguma outra parte do sistema? Como isolar melhor essa regra?
5. Cursores explícitos abertos sem `CLOSE` em caso de exceção vazam recursos. Reescreva o exemplo 4.4 garantindo que o cursor seja fechado mesmo se ocorrer um erro no meio do loop (dica: bloco `EXCEPTION` com `CLOSE`, ou `%ISOPEN`).

---

## 7. Questões de revisão

1. Explique por que uma `FUNCTION` que executa `INSERT`/`UPDATE`/`COMMIT` pode causar problemas ao ser chamada dentro de um `SELECT`.
2. O código abaixo não compila. Identifique o erro:
   ```sql
   CREATE OR REPLACE PROCEDURE sp_teste (p_valor OUT NUMBER) IS
   BEGIN
       p_valor := p_valor + 1;
   END;
   /
   ```
3. Escreva uma function `fn_dias_desde_publicacao(p_id_noticia)` que retorne o número de dias desde `data_publicacao` (trate o caso de a notícia ainda não ter sido publicada).
4. Compare, em termos de legibilidade e controle, usar um cursor `FOR ... IN (SELECT ...)` (visto no arquivo 03) versus um cursor explícito com `OPEN/FETCH/CLOSE`. Quando cada abordagem é preferível?
5. Descreva um cenário do sistema de notícias em que um parâmetro `IN OUT` seria genuinamente mais adequado que separar em dois parâmetros (`IN` e `OUT`).
