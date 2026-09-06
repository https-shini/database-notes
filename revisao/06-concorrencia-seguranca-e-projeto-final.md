# 06 — Concorrência, Segurança e Projeto Final

## 1. Objetivos de aprendizagem

Ao final deste arquivo você será capaz de:

- Entender e demonstrar bloqueios (locks) e controle de concorrência com `FOR UPDATE`.
- Reconhecer os principais níveis de isolamento e o comportamento padrão do Oracle.
- Criar usuários, papéis (`ROLE`) e conceder/revogar privilégios (`GRANT`/`REVOKE`).
- Consolidar todo o projeto evolutivo em uma visão final e propor melhorias de arquitetura.

Pré-requisito: arquivos 01 a 05 executados.

---

## 2. Conceitos teóricos

### 2.1 Concorrência e locks

Quando duas sessões tentam alterar a mesma linha, o Oracle bloqueia a segunda até a primeira confirmar (`COMMIT`) ou desfazer (`ROLLBACK`). Isso evita **lost updates** (uma alteração sobrescrevendo outra sem saber).

- `SELECT ... FOR UPDATE`: bloqueia explicitamente as linhas retornadas, para que outra sessão não as altere até você confirmar/desfazer.
- `SELECT ... FOR UPDATE NOWAIT`: falha imediatamente (`ORA-00054`) se a linha já estiver bloqueada, em vez de esperar.
- `SELECT ... FOR UPDATE SKIP LOCKED`: ignora linhas já bloqueadas, útil para filas de processamento.

### 2.2 Isolamento de transações

Oracle usa por padrão o nível **READ COMMITTED**: uma consulta nunca vê dados não confirmados de outra transação, mas pode ver dados diferentes se consultada duas vezes na mesma transação (porque outra sessão fez `COMMIT` entre as duas leituras). O nível **SERIALIZABLE** existe para cenários que exigem uma "foto" fixa da transação inteira, ao custo de mais erros de serialização sob concorrência alta.

### 2.3 Segurança e controle de acesso

- `CREATE USER usuario IDENTIFIED BY senha;`
- `GRANT privilegio TO usuario;` / `REVOKE privilegio FROM usuario;`
- `CREATE ROLE nome_papel;` — agrupa privilégios para atribuir a vários usuários de uma vez.
- Privilégios de objeto (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `EXECUTE`) vs. privilégios de sistema (`CREATE SESSION`, `CREATE TABLE`).

Princípio do menor privilégio: cada usuário/aplicação deve ter apenas o acesso estritamente necessário.

---

## 3. Evolução do banco (fechamento do projeto)

### 3.1 Papéis de acesso

```sql
-- Papel para quem só consulta relatórios (ex.: dashboards)
CREATE ROLE role_leitura_noticias;
GRANT SELECT ON vw_noticias_completas TO role_leitura_noticias;
GRANT SELECT ON vw_producao_jornalista TO role_leitura_noticias;

-- Papel para operadores da redação (podem cadastrar/publicar via package)
CREATE ROLE role_redator;
GRANT EXECUTE ON pkg_noticias TO role_redator;
GRANT SELECT ON tb_noticia TO role_redator;
GRANT SELECT ON tb_categoria TO role_redator;
GRANT SELECT ON tb_status_noticia TO role_redator;
```

**Por quê:** ao final do projeto, não faz sentido todo usuário ter `DELETE`/`UPDATE` direto nas tabelas — o acesso de escrita deve passar pelo `pkg_noticias`, que já contém as regras de negócio (visto no arquivo 05). Isso também resolve, em parte, o desafio de integridade levantado desde o arquivo 01: regras centralizadas em vez de espalhadas.

**Validação:**

```sql
SELECT role FROM user_role_privs; -- executar como o usuário que criou os papéis, ou consultar DBA_ROLES como DBA
SELECT table_name, privilege FROM role_tab_privs WHERE role = 'ROLE_REDATOR';
```

### 3.2 Visão final do modelo

Ao final dos 6 arquivos, o banco possui:

- `tb_status_noticia`, `tb_categoria` — tabelas de apoio/lookup.
- `tb_jornalista` — com `email` único.
- `tb_noticia` — com `conteudo` em `CLOB`, `data_publicacao`, `data_atualizacao`.
- `tb_noticia_jornalista` — associativa N:N.
- `tb_auditoria_noticia` — log automático via trigger.
- `pkg_noticias` — camada de regras de negócio (cadastro, publicação).
- Views `vw_noticias_completas` e `vw_producao_jornalista`.
- Papéis `role_leitura_noticias` e `role_redator`.

---

## 4. Exemplos práticos

### 4.1 Demonstrando um bloqueio com `FOR UPDATE`

**Sessão A:**

```sql
SELECT id_noticia, titulo, id_status_noticia
FROM tb_noticia
WHERE id_noticia = 1
FOR UPDATE;
-- NÃO faça COMMIT/ROLLBACK ainda — deixe a transação aberta
```

**Sessão B (em outra conexão), enquanto A está aberta:**

```sql
UPDATE tb_noticia SET titulo = 'Tentativa concorrente' WHERE id_noticia = 1;
-- Esta sessão fica BLOQUEADA aguardando a Sessão A liberar a linha
```

**Sessão A** então executa `COMMIT;` ou `ROLLBACK;` — nesse momento, a Sessão B é liberada e sua atualização prossegue (ou falha, se a linha não existir mais).

**Observação:** monitore bloqueios ativos com:

```sql
SELECT s.sid, s.serial#, o.object_name, l.locked_mode
FROM v$locked_object l
JOIN dba_objects o ON o.object_id = l.object_id
JOIN v$session s ON s.sid = l.session_id;
```

(requer privilégio para consultar essas views, geralmente concedido a um DBA)

### 4.2 `SKIP LOCKED` — fila de publicação

```sql
DECLARE
    CURSOR c_fila IS
        SELECT id_noticia FROM tb_noticia
        WHERE id_status_noticia = 1 -- Rascunho
        FOR UPDATE SKIP LOCKED;
BEGIN
    FOR rec IN c_fila LOOP
        DBMS_OUTPUT.PUT_LINE('Processando notícia ' || rec.id_noticia || ' (sem concorrência de outra sessão)');
        -- lógica de processamento aqui
    END LOOP;
    COMMIT;
END;
/
```

**Uso típico:** múltiplos processos "workers" consumindo a mesma fila de rascunhos sem disputar as mesmas linhas.

### 4.3 Criando usuário e concedendo papel

```sql
CREATE USER app_redacao IDENTIFIED BY "SenhaForte#2024";
GRANT CREATE SESSION TO app_redacao;
GRANT role_redator TO app_redacao;
```

Teste (conectado como `app_redacao`):

```sql
BEGIN
    DECLARE
        v_id NUMBER;
    BEGIN
        pkg_noticias.cadastrar('Teste de permissão', 'conteudo', 1, v_id);
        COMMIT;
    END;
END;
/
-- Deve funcionar (EXECUTE concedido no package)

DELETE FROM tb_noticia WHERE id_noticia = 1;
-- Deve falhar com ORA-01031: insufficient privileges (sem GRANT DELETE direto)
```

### 4.4 Revogando acesso

```sql
REVOKE role_redator FROM app_redacao;
```

---

## 5. Exercícios práticos

1. Reproduza o cenário 4.1 com duas sessões reais (ou simulando com comentários explicando o que aconteceria) e documente o comportamento observado.
2. Crie um papel `role_auditor` que tenha apenas `SELECT` em `tb_auditoria_noticia`, e um usuário `app_auditoria` com esse papel.
3. Escreva uma consulta usando `SELECT ... FOR UPDATE NOWAIT` e explique o que aconteceria se a linha já estivesse bloqueada por outra sessão.
4. Usando `v$session` (ou explicando conceitualmente se não tiver privilégio de DBA), descreva como identificaria qual sessão está segurando um lock que está bloqueando a sua.
5. Reescreva o exemplo 4.2 para, além de processar, também publicar automaticamente (via `pkg_noticias.publicar`) as notícias que atendem a regra de `pode_publicar`.

---

## 6. Desafios de melhoria do banco (projeto final)

Estes desafios pedem uma visão de arquitetura sobre **todo** o projeto construído nos 6 arquivos — não apenas o conteúdo deste arquivo:

1. **Normalização:** revise `tb_status_noticia` e `tb_categoria`. Considerando tudo que foi implementado (incluindo o trigger de auditoria), você ainda recomendaria mantê-las como tabelas de lookup separadas, ou migraria para `CHECK CONSTRAINT` com uma lista fixa? Justifique com prós e contras.
2. **Integridade:** identifique pelo menos duas regras de negócio do domínio de notícias que **ainda não** têm nenhuma garantia no banco (nem `CHECK`, nem trigger, nem package) e proponha como implementá-las.
3. **Performance:** com o volume crescendo, quais índices você criaria hoje, dado os padrões de consulta usados ao longo dos 6 arquivos (filtros por status, por categoria, por data, joins com a associativa)? Escreva os `CREATE INDEX` que você proporia.
4. **Segurança:** o papel `role_redator` tem `SELECT` direto nas tabelas de apoio, mas escrita só via package. Isso é suficiente, ou você reforçaria mais (por exemplo, `VPD`/Virtual Private Database, não coberto aqui, mas vale citar como próximo passo de pesquisa)?
5. **Concorrência:** o trigger `trg_auditoria_status_noticia` (arquivo 05) grava na tabela de auditoria dentro da mesma transação da atualização. Sob alta concorrência, isso pode gerar contenção na tabela de auditoria (todas as sessões inserindo na mesma tabela). Que estratégias mitigam isso (ex.: partição por data, autonomous transaction)? Pesquise `PRAGMA AUTONOMOUS_TRANSACTION` e avalie os riscos de usá-la aqui (dica: ela quebra a atomicidade entre o evento e o log em caso de rollback — isso é aceitável para auditoria?).
6. **Retrospectiva:** volte ao desafio do arquivo 01 (pergunta 1 a 5). Para cada um, explique **em qual arquivo** (02 a 06) o problema foi resolvido, parcialmente resolvido, ou ainda está em aberto.

---

## 7. Questões de revisão

1. Explique a diferença entre um lock implícito (causado por um `UPDATE` comum) e um lock explícito via `SELECT ... FOR UPDATE`.
2. Um usuário reclama que sua consulta "trava" ao rodar um relatório. Levantando hipóteses **apenas** com o que foi estudado neste arquivo, quais seriam as duas causas mais prováveis e como você investigaria cada uma?
3. Escreva os comandos `GRANT` necessários para que um novo papel `role_editor_chefe` possa executar `pkg_noticias` inteiro E também `DELETE` diretamente em `tb_noticia` (privilégio que nenhum outro papel tem).
4. Diferencie privilégio de sistema e privilégio de objeto, dando um exemplo de cada no contexto deste banco.
5. Considerando toda a evolução do banco ao longo dos 6 arquivos, escreva um parágrafo técnico (como se fosse para um README do projeto) descrevendo a arquitetura final: tabelas, camada de regras (package), auditoria, e modelo de acesso (papéis).
