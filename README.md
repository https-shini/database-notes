# database-notes

Laboratório de estudos em banco de dados relacional — anotações, scripts, exercícios e comparativos entre SGBDs.

![SQL](https://img.shields.io/badge/SQL-ANSI-blue)
![Oracle](https://img.shields.io/badge/Oracle-XE%2021c-red)
![MySQL](https://img.shields.io/badge/MySQL-8.4-orange)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791)

---

## Sobre

Repositório de estudo contínuo em banco de dados, do modelo relacional à administração básica. O foco não é acumular exercícios resolvidos, mas registrar **o que muda entre os SGBDs e por quê** — a parte que raramente está documentada de forma comparativa.

Cada tópico é estudado uma vez e implementado nos três bancos, para separar o que é SQL padrão do que é comportamento de fornecedor.

## Escopo

| SGBD | Versão | Uso no repo |
|---|---|---|
| Oracle Database XE | 21c | Referência principal (trilha de certificação) |
| MySQL | 8.4 | Comparativo — dialeto e comportamento |
| PostgreSQL | 16 | Comparativo — padrão ANSI e funções analíticas |

## Estrutura

```
database-notes/
├── docs/
│   ├── fundamentos/      # modelo relacional, álgebra, normalização
│   ├── sql/              # DDL, DML, joins, subqueries, funções analíticas
│   ├── dialetos/         # comparativos Oracle × MySQL × PostgreSQL
│   └── administracao/    # índices, transações, privilégios, tuning básico
├── sql/
│   ├── oracle/
│   ├── mysql/
│   └── postgres/
├── exercicios/           # listas resolvidas, por tema
├── projetos/             # modelagens completas end-to-end
├── datasets/             # bases de apoio para prática
└── infra/                # docker-compose dos ambientes
```

## Ambientes

Todos os bancos sobem via Docker, sem instalação na máquina:

```bash
git clone https://github.com/https-shini/db-lab.git
cd db-lab/infra
docker compose up -d
```

| Serviço | Porta | Credenciais |
|---|---|---|
| Oracle XE | 1521 | ver `infra/.env.example` |
| MySQL | 3306 | ver `infra/.env.example` |
| PostgreSQL | 5432 | ver `infra/.env.example` |

Para carregar os datasets de apoio:

```bash
./infra/seed.sh
```

## Conteúdo

### Fundamentos
- [ ] Modelo relacional e álgebra relacional
- [ ] Chaves: primária, estrangeira, candidata, composta
- [ ] Normalização — 1FN à 3FN, e quando desnormalizar
- [ ] Modelagem conceitual, lógica e física

### SQL
- [ ] `SELECT`, filtros, ordenação
- [ ] Joins — inner, outer, self, cross
- [ ] Agregações, `GROUP BY`, `HAVING`
- [ ] Subqueries e subqueries correlacionadas
- [ ] Operadores de conjunto
- [ ] DDL — `CREATE`, `ALTER`, `DROP`, tipos de dados
- [ ] Constraints
- [ ] DML e controle de transação
- [ ] Views, índices, sequences
- [ ] Funções de data, string e conversão
- [ ] Tratamento de `NULL`
- [ ] Window functions e CTEs

### Comparativos entre dialetos
- [ ] Tipos de dados equivalentes
- [ ] Funções de data e string
- [ ] Auto-incremento: `IDENTITY` × `AUTO_INCREMENT` × `SERIAL`
- [ ] Paginação: `ROWNUM` / `FETCH FIRST` × `LIMIT`
- [ ] Concatenação e tratamento de nulo
- [ ] Sintaxe de upsert

### Administração
- [ ] Planos de execução e leitura de `EXPLAIN`
- [ ] Estratégias de indexação
- [ ] Níveis de isolamento
- [ ] Usuários, roles e privilégios
- [ ] Backup e restore básicos

## Trilha de certificação

O repositório acompanha uma trilha de estudo com marcos definidos:

| Fase | Marco | Status |
|---|---|---|
| 1 | HackerRank SQL — Basic | ⬜ |
| 2 | freeCodeCamp — Relational Databases | ⬜ |
| 3 | Consolidação em SQL avançado | ⬜ |
| 4 | Oracle Database SQL Associate (1Z0-071) | ⬜ |

## Convenções

- Palavras-chave SQL em maiúsculas, identificadores em minúsculas com `snake_case`
- Um arquivo por tópico, prefixado por ordem: `01-select.sql`, `02-joins.sql`
- Todo script começa com um comentário de cabeçalho: objetivo, SGBD e pré-requisitos
- Comparativos entre dialetos ficam em `docs/dialetos/`, nunca dispersos nos scripts

## Referências

- Oracle SQL Language Reference
- MySQL 8.4 Reference Manual
- PostgreSQL 16 Documentation
- *Database System Concepts* — Silberschatz, Korth e Sudarshan

## Licença

MIT — veja [LICENSE](LICENSE).

---

Mantido por [Guilherme de Souza Cruz](https://github.com/https-shini) · [gcruz.dev.br](https://gcruz.dev.br)
