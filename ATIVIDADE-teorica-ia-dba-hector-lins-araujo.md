# Atividade Teórica: Usuários Especialistas, IA e Distribuição Segura de Dados

**Alunos:** Celso Hector, Daniel Lins, Nicolas Araujo <br>
**Turma:** Banco de Dados 2026 - G2 <br>
**Data:** 10/08/2026 <br>
**Repositório Git:** https://github.com/hectorex/atividade-bd

## Resumo Executivo

Breve descrição do tema e da posição adotada pelo grupo.

## 1. Desenvolvimento Teórico

### 1.1 O que é o DBA e quais suas funções?
Definição de DBA e suas funções: definição do esquema, estrutura de dados e
acesso, autorização de acesso, regras de integridade.

### 1.2 Perfis de usuários de banco de dados
Programadores de aplicações, usuários sofisticados, usuários especialistas,
usuários navegantes — vantagens e limitações de cada perfil.

### 1.3 Riscos do uso de IA por usuários especialistas
Consulta incorreta, exposição de dados sensíveis, degradação de performance,
vazamento por prompts — impactos na segurança e na integridade.

### 1.4 Distribuição segura de dados
Menor privilégio, views, roles customizadas, controle de execução, auditoria,
conformidade (LGPD).

### 1.5 Atuação do DBA no cenário de IA
Monitoramento, políticas de acesso, auditoria, orientação aos usuários,
performance e backups.

### 1.6 Análise crítica: qual a melhor abordagem?

A posição do grupo é que **não existe uma medida única suficiente**: a melhor abordagem para distribuir dados com segurança quando usuários especialistas usam IA é a **defesa em profundidade** (segurança em camadas), com o **próprio banco de dados** — e não a ferramenta de IA nem a aplicação — atuando como a última linha de controle.

O raciocínio central é o seguinte: no cenário tradicional, a segurança podia se apoiar na aplicação, porque era o programador quem escrevia o SQL. Com a IA generativa, o **usuário especialista passa a gerar SQL diretamente**, sem mediação. Isso significa que o grupo **não tem controle sobre o prompt** que o usuário escreve nem sobre o que a IA devolve. Confiar na "boa consulta" da IA seria transferir a segurança para um componente que erra, alucina e pode expor dados. Por isso, o controle precisa estar **na camada dos dados**, onde é o PostgreSQL que decide o que cada usuário pode ver e executar, independentemente do que o prompt pediu.

A partir dessa premissa, o grupo defende a combinação das seguintes camadas, em ordem de importância:

1. **Menor privilégio como fundação.** Cada usuário nasce sem acesso e recebe apenas o mínimo necessário. A IA pode gerar `SELECT * FROM clientes`, mas se a role não tem permissão na tabela base, a consulta simplesmente falha.
2. **Views como interface obrigatória.** Os usuários e a IA nunca tocam nas tabelas reais; só enxergam views que já removem colunas sensíveis (CPF, endereço) e filtram linhas. Assim, mesmo uma consulta "perfeita" da IA não consegue devolver o que não está na view.
3. **Roles customizadas por perfil**, agrupando permissões por função (analista, gestor, auditor) em vez de configurar usuário por usuário — o que reduz o erro humano do próprio DBA.
4. **Controle de recursos** (`statement_timeout`, limite de conexões) para conter consultas pesadas geradas por IA que degradariam a performance.
5. **Auditoria contínua** (logs e `pg_stat_statements`), porque nenhuma barreira é perfeita: é preciso rastrear quem executou o quê e detectar abuso.
6. **Conformidade com a LGPD embutida no desenho**, e não como remendo: dado sensível que o analista não precisa ver simplesmente não deve chegar até ele.

Há ainda um ponto específico da IA que reforça a escolha por views: o **vazamento por prompt**. Quando o usuário cola dados reais numa ferramenta de IA externa para "pedir ajuda", esses dados saem da organização. Se o usuário só trabalha sobre views já mascaradas, o que ele eventualmente enviar para a IA externa **já não contém dado pessoal identificável**, reduzindo o risco na origem.

Por fim, o grupo entende que a tecnologia sozinha não fecha o problema: é indispensável o **DBA como auditor humano** e a **orientação de uso** aos especialistas. A IA gera; o DBA governa. A melhor abordagem, portanto, é técnica **e** organizacional: banco configurado para negar por padrão, mais um DBA que monitora, revisa e educa.

## 2. Exemplos e Casos

### 2.1 View `clientes_visiveis` (limita colunas e linhas)

A view abaixo expõe apenas o que um analista precisa e **omite dados sensíveis** (CPF, e-mail, endereço completo, telefone), além de restringir as linhas a clientes ativos:

```sql
CREATE VIEW clientes_visiveis AS
SELECT
    id,
    nome,
    cidade,
    estado,
    segmento
FROM clientes
WHERE ativo = TRUE;   -- limita LINHAS (apenas clientes ativos)
-- CPF, e-mail, endereço e telefone ficam de fora -> limita COLUNAS
```

Quando a identificação parcial for realmente necessária (ex.: conferência), usa-se **mascaramento** em vez de expor o dado bruto:

```sql
CREATE VIEW clientes_mascarados AS
SELECT
    id,
    nome,
    '***.***.***-' || RIGHT(cpf, 2) AS cpf_mascarado,  -- revela só os 2 últimos dígitos
    cidade,
    estado
FROM clientes
WHERE ativo = TRUE;
```

### 2.2 Exemplo de role e permissão

Aqui aplicamos o menor privilégio: cria-se uma role de grupo por perfil, concede-se acesso **somente à view** (nunca à tabela base) e vincula-se o usuário individual a essa role.

```sql
-- 1. Role de grupo para o perfil "analista que usa IA" (sem login próprio)
CREATE ROLE analista_ia NOLOGIN;

-- 2. Somente leitura, e apenas na view — a tabela real permanece inacessível
GRANT SELECT ON clientes_visiveis TO analista_ia;
REVOKE ALL ON clientes FROM analista_ia;

-- 3. Controle de execução: consultas pesadas são abortadas após 30s
ALTER ROLE analista_ia SET statement_timeout = '30s';

-- 4. Usuário individual, vinculado à role de grupo
CREATE ROLE maria LOGIN PASSWORD 'senha_forte';
GRANT analista_ia TO maria;
```

Assim, se a IA gerar `SELECT cpf FROM clientes` para a usuária Maria, o comando **falha por falta de permissão**; e uma consulta que varra a tabela por tempo demais é **interrompida** pelo `statement_timeout`.

### 2.3 Caso real: sistema de vendas

Considere a empresa do contexto, que usa PostgreSQL para **clientes, vendas e operações**. A equipe de marketing começou a usar um gerador de SQL por IA para montar relatórios sozinha, sem passar por um programador.

**Antes dos controles:** um analista pede à IA "lista completa de clientes com dados de contato". A IA gera `SELECT * FROM clientes`, retornando **CPF, e-mail e endereço** de milhares de pessoas — violação direta da LGPD. Em outro momento, um `JOIN` mal formulado pela IA varre milhões de linhas e **trava o banco** em horário de pico.

**Depois da atuação do DBA:**

- Os analistas recebem a role `analista_ia`, com `SELECT` apenas nas views `clientes_visiveis` e `vendas_resumo`;
- O `statement_timeout` impede que consultas pesadas derrubem a performance;
- `log_statement` e a extensão `pg_stat_statements` registram e revelam as consultas mais lentas/frequentes, permitindo ao DBA detectar abuso;
- Dados pessoais sensíveis simplesmente **não estão nas views**, então nem a consulta da IA nem um eventual prompt enviado a uma ferramenta externa expõem CPF ou endereço.

**Resultado:** os analistas continuam autônomos e produtivos com a IA, mas dentro de um "cercado" seguro definido pelo DBA — conciliando produtividade, segurança, performance e conformidade.

## 3. Referências

Fontes consultadas (livros, artigos, documentação oficial do PostgreSQL,
materiais do curso).

## 4. Conclusões

Aprendizados, reflexões e principais pontos observados pelo grupo.

## [Link do Repositório Git](https://github.com/hectorex/atividade-bd/blob/main/ATIVIDADE-teorica-ia-dba-hector-lins-araujo.md)