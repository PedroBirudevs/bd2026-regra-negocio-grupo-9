# Sistema de Gerenciamento de Atendimentos

**Disciplina:** Banco de Dados 2026
**Grupo:** 9
**Integrantes:** João Paulo Dias; João Victor; Maykon; Saymon; Pedro Henrique
**SGBD:** PostgreSQL
**Data:** 19 de agosto de 2026

---

## 1. Objetivo geral

Desenvolver um banco de dados relacional para uma empresa que precisa organizar o
registro e o fluxo de atendimentos ao público, controlando as filas de atendimento,
os atendentes vinculados a cada fila e o histórico de atendimentos realizados.

O objetivo não é apenas armazenar os registros, mas garantir que o banco represente
corretamente a operação real. O modelo deve impedir situações impossíveis no dia a dia
da empresa — como um atendimento vinculado a um atendente que não trabalha naquela fila,
uma pessoa cadastrada duas vezes por acumular os papéis de atendente e cliente, ou um
atendimento registrado sem fila, sem atendente ou sem cliente.

## 2. Público-alvo

**Clientes.** Pessoas que buscam o atendimento e entram em uma fila. Do ponto de vista
do banco, interessa identificá-las de forma única e recuperar seu histórico de
atendimentos.

**Atendentes.** Funcionários que realizam os atendimentos. Cada atendente pode estar
habilitado em mais de uma fila, e cada fila conta com vários atendentes. Ponto central
do enunciado: **um atendente também pode ser cliente da empresa**, ou seja, a mesma
pessoa física acumula dois papéis e não deve ser cadastrada duas vezes.

**Gestores.** Acompanham o volume de atendimentos por fila e por atendente, além do
histórico de cada cliente.

## 3. Escopo

| Entidade | O que registra |
|---|---|
| **Pessoa** | Dados pessoais comuns a clientes e atendentes (CPF, nome, contato) |
| **Cliente** | Papel de quem é atendido |
| **Atendente** | Papel de quem realiza o atendimento |
| **Fila** | Diferentes filas de atendimento da empresa |
| **Atendente × Fila** | Vínculo de quais atendentes atuam em quais filas |
| **Atendimento** | Data e hora, fila, atendente responsável e cliente atendido |

## 4. Questões de modelagem identificadas

1. **Acúmulo de papéis.** Como um atendente pode ser cliente, criar duas tabelas
   independentes com CPF duplicaria a mesma pessoa. A solução prevista é separar a
   pessoa (dados pessoais) dos papéis que ela exerce.

2. **Atendente e fila.** O vínculo é de muitos para muitos: um atendente atua em várias
   filas e uma fila tem vários atendentes. Exige tabela associativa.

3. **Coerência do atendimento.** O atendente registrado em um atendimento deve estar
   vinculado à fila daquele atendimento — regra que envolve mais de uma tabela.

4. **Ninguém atende a si mesmo.** Como a mesma pessoa pode ser atendente e cliente, o
   banco precisa impedir que ela apareça nos dois papéis no mesmo atendimento.

## 5. Estrutura do repositório

```
.
├── README.md          ← este arquivo
├── ETAPAS.md          ← divisão do trabalho em 3 etapas
├── modelo/            ← modelo conceitual e lógico (DER)
└── sql/               ← scripts de criação e carga
```
