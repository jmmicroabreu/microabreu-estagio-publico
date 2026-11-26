# Postgres documentação

Documentação pública sobre **PostgreSQL e alta disponibilidade**, feita durante um estágio.
O conteúdo é genérico: não inclui endereços, configurações reais nem informação interna de nenhuma empresa.

## Wiki

Toda a documentação está na **[wiki deste repositório](https://github.com/jmmicroabreu/microabreu-estagio-publico/wiki)**:

| Página | Para quem | O que encontra |
| :--- | :--- | :--- |
| [Manual de Introdução ao PostgreSQL](https://github.com/jmmicroabreu/microabreu-estagio-publico/wiki/Manual-de-Introdu%C3%A7%C3%A3o-ao-PostgreSQL) | Qualquer pessoa | O que é uma base de dados e o PostgreSQL, explicado de forma simples |
| [Pgpool-II](https://github.com/jmmicroabreu/microabreu-estagio-publico/wiki/Pgpool%E2%80%90II) | Técnico | O que é o Pgpool-II, funcionalidades, vantagens e desvantagens |
| [PgBouncer](https://github.com/jmmicroabreu/microabreu-estagio-publico/wiki/PgBouncer) | Técnico | O que é o PgBouncer, modos de pooling, instalação e configuração |
| [Comparação e Recomendação](https://github.com/jmmicroabreu/microabreu-estagio-publico/wiki/Compara%C3%A7%C3%A3o-e-Recomenda%C3%A7%C3%A3o) | Técnico e gestão | Quatro formas de dar alta disponibilidade ao PostgreSQL e quando escolher cada uma |
| [Guia de Implementação em Laboratório](https://github.com/jmmicroabreu/microabreu-estagio-publico/wiki/Guia-de-Implementa%C3%A7%C3%A3o-em-Laborat%C3%B3rio) | Técnico | Passo a passo: 3 servidores PostgreSQL com Pgpool-II e failover automático |

## Documentos

| Documento | Para quem | O que encontra |
| :--- | :--- | :--- |
| [Backups com pgBackRest](docs/backups-pgbackrest.md) | Técnico | Backups automáticos (completo semanal + incremental de hora a hora + WAL), PITR e restore de teste |
| [Teste de restore numa VM](docs/teste-restore-vm.md) | Técnico | Criar uma VM local e repor lá os backups, para o teste mensal |

## Autores

- Ricardo Ferreira
- José Machado

Orientação: Eduardo Amaral (formador)
