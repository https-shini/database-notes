# 05 — Packages, Transações e Triggers

## 1. Objetivos de aprendizagem

Ao final deste arquivo você será capaz de:

- Agrupar procedures e functions relacionadas em `PACKAGE`s (especificação e corpo).
- Controlar transações explicitamente com `COMMIT`, `ROLLBACK` e `SAVEPOINT`.
- Criar `TRIGGER`s para auditoria e para reforço de regras de negócio.
- Tratar exceções de forma estruturada.

Pré-requisito: arquivos 01 a 04 executados.

---

## 2. Conceitos teóricos

### 2.1 Packages

Um package tem duas partes:

- **Especificação (spec):** a interface pública — o que outros blocos podem chamar.
- **Corpo (body):** a implementação, que pode conter também elementos privados (não visíveis na spec).

Vantagens: organização lógica, encapsulamento (esconder implementação), variáveis/estado de pacote (persistem durante a sessão), e melhor performance (o pacote inteiro é carregado em memória uma vez).

### 2.2 Transações

Uma transação é uma unidade lógica de trabalho: ou todas as operações são confirmadas, ou nenhuma é.

- `COMMIT`: confirma permanentemente as alterações da transação atual.
- `ROLLBACK`: desfaz todas as alterações não confirmadas.
- `SAVEPOINT nome`: marca um ponto intermediário; `ROLLBACK TO nome` desfaz só até ali, mantendo o que veio antes.

Regra de ouro: cada unidade de negócio completa deve ter exatamente um `COMMIT` (ou `ROLLBACK`) no fim — não em cada procedure isolada (retomando o desafio 2 do arquivo 04).

### 2.3 Triggers

Um trigger é um bloco PL/SQL executado automaticamente em resposta a um evento (`INSERT`, `UPDATE`, `DELETE`) em uma tabela.

| Momento | Granularidade | Uso típico |
|---|---|---|
| `BEFORE` | `ROW` | Validar/ajustar valores antes de gravar |
| `AFTER` | `ROW` | Auditoria, replicar dados, disparar ações |
| `BEFORE`/`AFTER` | `STATEMENT` | Ações únicas por comando, não por linha |

Dentro de triggers `ROW`, `:NEW` e `:OLD` referenciam os valores da linha depois e antes da operação, respectivamente (`:OLD` não existe em `INSERT`, `:NEW` não existe em `DELETE`).

### 2.4 Tratamento de exceções

```sql
BEGIN
    ...
EXCEPTION
    WHEN NO_DATA_FOUND THEN ...
    WHEN DUP_VAL_ON_INDEX THEN ...
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE; -- repropaga o erro após log/rollback
END;
```

`WHEN OTHERS` sem `RAISE`/log é uma má prática clássica — "engole" erros silenciosamente.

---

## 3. Evolução do banco

### 3.1 Tabela de auditoria

Para suportar os triggers de log, criamos uma tabela nova:

```sql
CREATE TABLE tb_auditoria_noticia (
    id_auditoria     NUMBER(10)     NOT NULL,
    id_noticia       NUMBER(8)      NOT NULL,
    acao             VARCHAR2(10)   NOT NULL,
    status_anterior  NUMBER(4),
    status_novo      NUMBER(4),
    data_evento      DATE           DEFAULT SYSDATE NOT NULL,
    CONSTRAINT pk_auditoria_noticia PRIMARY KEY (id_auditoria)
);

CREATE SEQUENCE seq_auditoria_noticia START WITH 1 INCREMENT BY 1;
```

**Por quê:** o desafio do arquivo 01 apontou a ausência de rastro de auditoria — agora resolvemos isso de forma automática via trigger, em vez de depender de código de aplicação lembrar de gravar o log.

**Validação:**

```sql
SELECT table_name FROM user_tables WHERE table_name = 'TB_AUDITORIA_NOTICIA';
```

---

## 4. Exemplos práticos

### 4.1 Package — operações de notícia

**Especificação:**

```sql
CREATE OR REPLACE PACKAGE pkg_noticias AS
    PROCEDURE cadastrar (
        p_titulo     IN  tb_noticia.titulo%TYPE,
        p_conteudo   IN  tb_noticia.conteudo%TYPE,
        p_categoria  IN  tb_noticia.id_categoria%TYPE,
        p_novo_id    OUT tb_noticia.id_noticia%TYPE
    );

    PROCEDURE publicar (p_id_noticia IN tb_noticia.id_noticia%TYPE);

    FUNCTION pode_publicar (p_id_noticia IN tb_noticia.id_noticia%TYPE) RETURN BOOLEAN;
END pkg_noticias;
/
```

**Corpo:**

```sql
CREATE OR REPLACE PACKAGE BODY pkg_noticias AS

    FUNCTION pode_publicar (p_id_noticia IN tb_noticia.id_noticia%TYPE) RETURN BOOLEAN IS
        v_qtd NUMBER;
    BEGIN
        SELECT COUNT(*) INTO v_qtd
        FROM tb_noticia_jornalista
        WHERE id_noticia = p_id_noticia;

        RETURN v_qtd > 0;
    END pode_publicar;

    PROCEDURE cadastrar (
        p_titulo     IN  tb_noticia.titulo%TYPE,
        p_conteudo   IN  tb_noticia.conteudo%TYPE,
        p_categoria  IN  tb_noticia.id_categoria%TYPE,
        p_novo_id    OUT tb_noticia.id_noticia%TYPE
    ) IS
    BEGIN
        INSERT INTO tb_noticia (id_noticia, titulo, conteudo, id_status_noticia, id_categoria)
        VALUES (seq_noticia.NEXTVAL, p_titulo, p_conteudo, 1, p_categoria)
        RETURNING id_noticia INTO p_novo_id;
        -- sem COMMIT aqui: quem orquestra a transação decide quando confirmar
    EXCEPTION
        WHEN OTHERS THEN
            ROLLBACK;
            RAISE;
    END cadastrar;

    PROCEDURE publicar (p_id_noticia IN tb_noticia.id_noticia%TYPE) IS
    BEGIN
        IF NOT pode_publicar(p_id_noticia) THEN
            RAISE_APPLICATION_ERROR(-20001, 'Não é possível publicar notícia sem jornalista.');
        END IF;

        UPDATE tb_noticia
        SET id_status_noticia = 2, data_publicacao = SYSDATE
        WHERE id_noticia = p_id_noticia;
    END publicar;

END pkg_noticias;
/
```

**Uso, com transação orquestrada pelo chamador:**

```sql
DECLARE
    v_id tb_noticia.id_noticia%TYPE;
BEGIN
    SAVEPOINT antes_cadastro;

    pkg_noticias.cadastrar('Notícia via package', 'Conteúdo...', 2, v_id);

    BEGIN
        pkg_noticias.publicar(v_id); -- vai falhar: ainda sem jornalista
    EXCEPTION
        WHEN OTHERS THEN
            DBMS_OUTPUT.PUT_LINE('Aviso: ' || SQLERRM);
            ROLLBACK TO antes_cadastro;
    END;

    COMMIT;
END;
/
```

**Explicação:** o `SAVEPOINT` permite desfazer só o cadastro problemático sem afetar outras alterações feitas antes na mesma sessão — essa é a essência de controlar transações de forma granular.

### 4.2 Trigger de auditoria (AFTER UPDATE)

```sql
CREATE OR REPLACE TRIGGER trg_auditoria_status_noticia
AFTER UPDATE OF id_status_noticia ON tb_noticia
FOR EACH ROW
BEGIN
    INSERT INTO tb_auditoria_noticia (
        id_auditoria, id_noticia, acao, status_anterior, status_novo
    ) VALUES (
        seq_auditoria_noticia.NEXTVAL, :NEW.id_noticia, 'UPDATE',
        :OLD.id_status_noticia, :NEW.id_status_noticia
    );
END;
/
```

Teste:

```sql
UPDATE tb_noticia SET id_status_noticia = 3 WHERE id_noticia = 2;
COMMIT;

SELECT * FROM tb_auditoria_noticia ORDER BY data_evento DESC;
```

**Resultado esperado:** uma nova linha em `tb_auditoria_noticia` refletindo a mudança.

### 4.3 Trigger de validação (BEFORE INSERT)

```sql
CREATE OR REPLACE TRIGGER trg_valida_data_publicacao
BEFORE INSERT OR UPDATE OF data_publicacao ON tb_noticia
FOR EACH ROW
BEGIN
    IF :NEW.data_publicacao IS NOT NULL AND :NEW.data_publicacao < :NEW.data_criacao THEN
        RAISE_APPLICATION_ERROR(-20002, 'Data de publicação não pode ser anterior à data de criação.');
    END IF;
END;
/
```

**Erro comum:** triggers `BEFORE`/`AFTER` `FOR EACH ROW` que fazem `SELECT` ou `DML` na **mesma tabela** que disparou o trigger podem causar `ORA-04091: mutating table` — evite isso; se precisar, use uma abordagem com tabela de estado auxiliar ou trigger `STATEMENT`-level com uma coleção acumulada.

### 4.4 Tratamento estruturado de exceções com exceção nomeada

```sql
DECLARE
    v_id_invalido EXCEPTION;
    v_qtd NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_qtd FROM tb_noticia WHERE id_noticia = 9999;

    IF v_qtd = 0 THEN
        RAISE v_id_invalido;
    END IF;
EXCEPTION
    WHEN v_id_invalido THEN
        DBMS_OUTPUT.PUT_LINE('Notícia 9999 não existe.');
END;
/
```

---

## 5. Exercícios práticos

1. Adicione ao `pkg_noticias` uma procedure `arquivar(p_id_noticia)` que muda o status para "Arquivada", só se a notícia estiver "Publicada" (caso contrário, levante um erro customizado com `RAISE_APPLICATION_ERROR`).
2. Crie um trigger `BEFORE INSERT` em `tb_jornalista` que impeça `data_contratacao` no futuro (`SYSDATE`).
3. Crie um trigger `AFTER INSERT` em `tb_noticia_jornalista` que grave em `tb_auditoria_noticia` uma linha com `acao = 'VINCULO'` sempre que um jornalista for associado a uma notícia.
4. Escreva um bloco que cadastre 3 notícias em sequência usando `pkg_noticias.cadastrar`, com um `SAVEPOINT` entre cada uma, e simule uma falha na terceira, revertendo apenas ela.
5. Adicione tratamento de exceção em `pkg_noticias.publicar` para capturar o erro customizado (`-20001`) e registrá-lo em `tb_auditoria_noticia` como uma tentativa falha (`acao = 'FALHA_PUBLICACAO'`).

---

## 6. Desafios de melhoria do banco

1. O package `pkg_noticias.cadastrar` não faz `COMMIT`. Isso significa que quem chama a procedure é responsável por confirmar. Isso é uma boa prática de design? Que problema aconteceria se **todas** as procedures do sistema fizessem `COMMIT` internamente, especialmente em fluxos que chamam várias procedures em sequência?
2. `trg_auditoria_status_noticia` só dispara em `UPDATE`. Ele cobre o cenário de uma notícia ser **criada** já com status diferente de "Rascunho"? Deveria?
3. Avalie o risco de mutating table: se você precisasse, em um trigger de `tb_noticia`, validar uma regra que depende de contar linhas da própria `tb_noticia` (ex.: "no máximo 5 rascunhos simultâneos por jornalista"), como resolveria sem cair em `ORA-04091`?
4. A tabela `tb_auditoria_noticia` cresce indefinidamente. Isso é um problema de longo prazo? Que estratégias (particionamento, arquivamento periódico) você pesquisaria para mitigar, sem implementar agora?
5. `RAISE_APPLICATION_ERROR` usa códigos como `-20001`, `-20002` "mágicos" espalhados pelo código. Proponha uma forma de centralizar esses códigos (por exemplo, constantes no package spec).

---

## 7. Questões de revisão

1. Explique a diferença entre declarar uma function como parte pública (na spec) versus privada (só no body) de um package. Dê um exemplo do `pkg_noticias` para cada caso.
2. O código abaixo tem um problema de transação. Identifique-o:
   ```sql
   BEGIN
       pkg_noticias.cadastrar('A', 'conteudo', 1, v_id1);
       COMMIT;
       pkg_noticias.cadastrar('B', 'conteudo', 1, v_id2);
       ROLLBACK; -- intenção: desfazer só a segunda notícia
   END;
   ```
3. Escreva um trigger `BEFORE DELETE` em `tb_jornalista` que impeça a exclusão de um jornalista que ainda tenha notícias associadas, com uma mensagem de erro clara (mesmo sabendo que a FK já impediria — pratique a mensagem customizada).
4. O que é `ORA-04091` e por que ele acontece? Dê um exemplo hipotético no contexto de `tb_noticia` que causaria esse erro.
5. Analise um cenário: dois usuários chamam `pkg_noticias.publicar` para a mesma notícia quase simultaneamente. Antes de estudarmos concorrência formalmente (arquivo 06), levante uma hipótese sobre o que poderia dar errado.
