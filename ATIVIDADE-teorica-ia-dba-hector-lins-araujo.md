# Atividade Teórica: Usuários Especialistas, IA e Distribuição Segura de Dados

**Alunos:** Celso Hector, Daniel Lins, Nicolas Araujo <br>
**Turma:** Banco de Dados 2026 - G2 <br>
**Data:** 10/08/2026 <br>
**Repositório Git:** https://github.com/hectorex/atividade-bd

## Resumo Executivo

Este trabalho analisa os impactos e os riscos da utilização de ferramentas de Inteligência Artificial generativa por usuários especialistas no acesso direto a um SGBD PostgreSQL. Com o uso de assistentes virtuais para a criação de consultas SQL complexas sem a mediação da equipe de desenvolvimento, surgem ameaças reais à organização, como a exposição de dados sensíveis, a degradação da performance por queries ineficientes, vazamentos de informações via prompts e erros de lógica nas transações.

Diante desse cenário, defendemos que a distribuição segura dos dados depende centralmente do fortalecimento da governança do DBA. A abordagem proposta baseia-se na aplicação rigorosa do princípio do menor privilégio, na restrição do acesso por meio de views e roles customizadas, no controle de tempo de execução e na auditoria constante de logs. Conclui-se que a IA deve ser adotada como uma ferramenta de produtividade, desde que o banco de dados imponha as travas técnicas necessárias para garantir a segurança, a integridade e a conformidade de todo o ambiente.

## 1. Desenvolvimento Teórico

### 1.1 O que é o DBA e quais suas funções?
O **DBA** é o profissional responsável por gerenciar, manter, otimizar e garantir a segurança e integridade de um SGBD. No contexto em que usuários especialistas utilizam Inteligência Artificial generativa para criar consultas SQL, as funções clássicas do DBA tornam-se ainda mais cruciais:

* **Definição do Esquema:** O DBA define as tabelas, atributos, tipos de dados e relacionamentos.
  * *Relação com a IA:* Como ferramentas de IA geram código SQL com base nos metadados do banco, um esquema bem definido e documentado evita que a IA cometa erros de interpretação de regra de negócio ao criar consultas complexas.

* **Estrutura de Dados e Métodos de Acesso:** O DBA decide como os dados são armazenados fisicamente e cria estruturas como índices.
  * *Relação com a IA:* Consultas geradas por IA frequentemente contêm *JOINs* ineficientes ou varreduras completas em tabelas. O DBA precisa criar índices adequados para mitigar a degradação da performance do PostgreSQL trazida por essas consultas automáticas.

* **Autorização de Acesso:** O DBA controla quem pode acessar o banco e o que cada usuário ou aplicação pode executar.
  * *Relação com a IA:* Usuários especialistas usando assistentes de IA não devem ter acesso total. O DBA deve aplicar o Princípio do Menor Privilégio para impedir que um prompt incorreto ou malicioso exponha dados sensíveis ou modifique tabelas críticas.

* **Regras de Integridade:** O DBA estabelece limites para garantir que os dados permaneçam corretos e consistentes.
  * *Relação com a IA:* A IA pode sugerir comandos de alteração (`UPDATE`/`DELETE`) ou inserção incorretos. As regras de integridade configuradas pelo DBA atuam como uma camada final de defesa no PostgreSQL para rejeitar transações inválidas geradas por erros de lógica da IA.

### 1.2 Perfis de usuários de banco de dados
Os usuários de banco de dados são divididos em quatro perfis principais:

* **Programadores de aplicações:** São os desenvolvedores que constroem os sistemas da empresa. 
  * *Vantagens:* Criam telas e rotinas automatizadas, tratando os dados com segurança no código da aplicação antes de chegar ao banco.
  * *Limitações:* Qualquer mudança na regra de negócio ou novo relatório exige tempo de desenvolvimento e alteração no código do sistema.

* **Usuários navegantes:** São os funcionários que usam os sistemas prontos no dia a dia.
  * *Vantagens:* Não precisam saber nada de SQL ou TI. Interagem com telas simples e intuitivas, com risco quase zero de danificar a estrutura do banco.
  * *Limitações:* Ficam totalmente presos às funções pré-programadas do sistema. Se precisarem de um dado diferente, dependem da TI.

* **Usuários sofisticados:** São analistas, cientistas de dados ou gestores que possuem conhecimento intermediário/avançado de SQL e ferramentas de análise.
  * *Vantagens:* Conseguem extrair dados complexos e montar relatórios sem depender da equipe de desenvolvimento.
  * *Limitações:* Podem montar consultas ineficientes se não conhecerem os índices e a estrutura física do banco.

* **Usuários especialistas:** São profissionais focados estritamente na regra de negócio. 
  * *Vantagens:* Têm domínio profundo dos dados e dos problemas da empresa, sabendo exatamente qual resposta precisam obter.
  * *Limitações:* Historicamente não dominam a linguagem SQL nem as boas práticas de banco de dados.

### 1.3 Riscos do uso de IA por usuários especialistas
Usar IA generativa pode ajudar bastante um profissional no dia a dia, mas isso não elimina a necessidade de entender banco de dados. A IA pode gerar consultas, explicar erros e até sugerir soluções, porém ela também pode interpretar alguma coisa de forma errada. Se a pessoa não tiver conhecimento suficiente para perceber isso, pode acabar executando um comando sem entender direito o que ele vai fazer. A partir disso, o grupo identificou cinco riscos principais que podem afetar a segurança e os dados de uma empresa.

**1. Consulta incorreta por erro de lógica.**
Quando um comando está com erro de sintaxe, normalmente o próprio banco acusa o erro e não deixa executar. O problema maior é quando o comando está correto, mas a lógica está errada. Nesse caso, ele pode rodar normalmente e apresentar um resultado que parece certo, mesmo não sendo.

Um exemplo seria um `JOIN` feito usando a coluna errada. Isso pode duplicar algumas linhas e fazer com que uma venda de R$ 600 apareça como R$ 1.200 em um relatório. Os dados originais continuam corretos dentro do banco, mas o relatório apresenta uma informação errada e alguém pode tomar uma decisão com base nela.

O problema pode ser ainda maior quando envolve alteração ou exclusão de dados. Um `UPDATE` ou `DELETE` sem a cláusula `WHERE`, por exemplo, pode acabar afetando todas as linhas da tabela quando a intenção era modificar apenas algumas. Dependendo do caso, pode ser necessário recuperar os dados usando um backup. Por isso, esse tipo de erro afeta principalmente a integridade dos dados e pode ser difícil de perceber logo de início.

**2. Exposição de dados sensíveis.**
Outro risco acontece quando a IA gera consultas muito amplas. Um `SELECT *`, por exemplo, pode retornar todas as colunas de uma tabela, incluindo informações como CPF, endereço e telefone, mesmo que o funcionário não precise desses dados para realizar o trabalho dele.

Isso entra em conflito com o princípio da necessidade previsto na LGPD, segundo o qual cada pessoa deve ter acesso somente aos dados necessários para cumprir sua função. O fato de o funcionário trabalhar na empresa ou não ter intenção de causar algum problema não significa que ele deveria ter acesso a qualquer informação.

Nesse caso, o principal impacto é na confidencialidade. Depois que um dado aparece na tela, é colocado em uma planilha ou enviado para outra pessoa, simplesmente retirar a permissão depois não resolve completamente o problema, porque a informação já foi visualizada ou copiada.

Além da questão de segurança, também existe a parte jurídica. A LGPD prevê sanções que podem chegar a 2% do faturamento da empresa no Brasil, limitadas a R$ 50 milhões por infração, dependendo do caso. Também podem existir outras consequências jurídicas além das medidas aplicadas pela ANPD.

**3. Degradação de performance.**
Em uma empresa, várias partes do sistema podem estar usando o mesmo banco de dados ao mesmo tempo. O caixa da loja pode estar registrando vendas, o estoque sendo atualizado, o site recebendo acessos e algum funcionário gerando relatórios.

A IA pode criar uma consulta que funciona normalmente, mas que não foi pensada para aquele banco específico. Ela pode não conhecer quais colunas possuem índices, o tamanho real das tabelas ou o custo de cruzar uma quantidade muito grande de registros.

Com isso, uma consulta que deveria ser simples pode acabar lendo uma tabela inteira e consumindo muito processamento, memória e acesso ao disco. Se isso acontecer enquanto outras pessoas estão usando o sistema, as demais operações podem ficar mais lentas ou até parar de responder por algum tempo.

Nesse caso, não necessariamente existe vazamento ou alteração dos dados. O problema está na disponibilidade do sistema. Se isso acontecer em um horário de pico, por exemplo, a empresa pode até perder vendas porque o sistema ficou lento. Uma das medidas que podem ajudar é configurar um `statement_timeout`, fazendo com que consultas que ultrapassem determinado tempo sejam canceladas automaticamente.

**4. Vazamento de informações por prompts.**
Esse risco acontece fora do próprio banco de dados. Para conseguir uma resposta mais precisa da IA, uma pessoa pode acabar copiando e colando informações reais no prompt, como parte de um relatório, registros de clientes, resultados de consultas ou até códigos internos da empresa.

A partir desse momento, aquela informação saiu do ambiente da empresa e foi enviada para um serviço externo. Dependendo da ferramenta utilizada, podem existir políticas diferentes sobre armazenamento, processamento e localização desses dados.

Um caso conhecido aconteceu com a Samsung em 2023, quando funcionários utilizaram ferramentas de IA generativa e acabaram inserindo informações internas, incluindo código e conteúdo relacionado ao trabalho. Depois desses episódios, a empresa restringiu o uso dessas ferramentas em seus dispositivos e sistemas internos.

Esse risco é complicado porque não acontece diretamente dentro do banco. Se alguém copiar dados e colocar em um prompt, as permissões configuradas no PostgreSQL não conseguem impedir isso. Os logs do banco também não vão mostrar que aquela informação foi enviada para uma IA. Se houver dados pessoais envolvidos, ainda pode configurar tratamento indevido de dados pessoais segundo a LGPD.

**5. Escalada de privilégios.**
Também pode acontecer de um usuário tentar executar alguma operação e receber uma mensagem dizendo que não possui permissão. Sem saber exatamente o motivo daquela restrição, ele pode copiar o erro e perguntar para uma IA como resolver.

A IA pode interpretar aquilo apenas como um problema técnico e sugerir comandos como `GRANT ALL` ou mudanças nas roles do banco. O problema é que ela não conhece necessariamente as regras de segurança daquela empresa e não sabe se aquela permissão foi limitada de propósito pelo DBA.

Se o usuário entende bem da área em que trabalha, mas não possui conhecimento suficiente sobre administração de banco de dados, pode executar a sugestão acreditando que está apenas corrigindo um erro. Na prática, ele pode estar removendo uma proteção importante.

Por isso, alterações de permissões devem ficar restritas a usuários responsáveis pela administração do banco. Dessa forma, mesmo que uma IA sugira aumentar privilégios, um usuário comum não terá autorização para executar esse tipo de mudança.

Esses exemplos mostram que o uso de IA pode trazer riscos diferentes para um banco de dados. Um erro de lógica pode comprometer a **integridade** das informações, o acesso indevido a dados pessoais pode afetar a **confidencialidade**, e consultas muito pesadas podem prejudicar a **disponibilidade** do sistema.

Por esse motivo, não existe uma única configuração capaz de resolver todos esses problemas. É necessário utilizar diferentes medidas de segurança em conjunto, criando várias camadas de proteção. É justamente essa ideia de proteção em camadas que será abordada na próxima seção.


### 1.4 Distribuição segura de dados
### 1.4 Distribuição segura de dados

Depois de entender os principais riscos do uso de IA com bancos de dados, o próximo passo é pensar em formas de diminuir esses problemas. O próprio banco oferece vários recursos de segurança que podem ser usados para isso. A ideia do grupo é trabalhar com essas medidas em conjunto, como diferentes camadas de proteção. Nenhuma delas resolve todos os problemas sozinha, mas uma pode proteger justamente onde a outra não consegue.

Um ponto importante é que essa proteção deve estar dentro do próprio banco de dados, e não depender da IA. Em vez de confiar que a IA sempre vai gerar uma consulta correta e segura, o banco deve estar preparado para bloquear uma operação quando o usuário não tiver autorização para realizá-la.

**1. Princípio do menor privilégio.**
O princípio do menor privilégio é relativamente simples: cada usuário deve ter acesso somente ao que realmente precisa para realizar o seu trabalho.

Mesmo que várias pessoas utilizem o mesmo banco de dados, isso não significa que todas precisam enxergar as mesmas informações. Um funcionário do caixa, por exemplo, pode precisar registrar uma venda e consultar o preço de um produto, mas provavelmente não precisa ter acesso ao CPF ou ao endereço dos clientes para fazer isso. Não significa que a empresa desconfia daquele funcionário, apenas que essas informações não são necessárias para a função dele.

No PostgreSQL, isso pode ser feito retirando permissões e depois liberando apenas aquilo que cada usuário realmente precisa, usando comandos como `REVOKE ALL` e `GRANT SELECT`.

Isso é especialmente importante quando existe uso de IA, porque a segurança não fica dependendo da consulta que ela criou. Se um usuário possui somente permissão de leitura e a IA sugerir um `UPDATE`, por exemplo, o banco não deve permitir a alteração. Da mesma forma, se o usuário tentar acessar uma tabela com dados pessoais para a qual não possui autorização, a consulta será bloqueada. O mesmo vale para comandos que alteram permissões: se apenas o DBA pode concedê-las, uma sugestão da IA nesse sentido não será executada.

**2. Uso de views para limitar colunas e linhas.**
Uma view pode ser entendida como uma espécie de janela para uma tabela. Em vez de entregar acesso à tabela completa, é possível criar uma view mostrando somente as informações que determinado usuário realmente precisa.

Esse recorte pode acontecer tanto nas colunas quanto nas linhas. Dados como CPF, telefone e endereço podem simplesmente não aparecer na view. Também é possível mostrar apenas determinados registros. Uma condição como `WHERE ativo = TRUE`, por exemplo, permite mostrar somente os registros que estão ativos.

No PostgreSQL, a view é criada a partir de uma consulta `SELECT` e recebe um nome. Assim, o usuário pode trabalhar com aquela visão limitada dos dados sem necessariamente ter acesso direto à tabela original.

Isso também ajuda bastante quando existe IA envolvida. Se o usuário trabalha somente com uma view que não possui CPF, por exemplo, uma consulta ampla feita sobre essa view continuará sem retornar o CPF, porque essa informação não está disponível naquele acesso.

Essa medida também pode diminuir o risco de vazamento por prompts. Se o usuário trabalha com informações que já foram limitadas anteriormente, existe uma chance menor de ele acabar copiando dados pessoais desnecessários e enviando para uma ferramenta externa.

**3. Roles customizadas por perfil.**
Em vez de configurar as permissões de cada funcionário uma por uma, é possível criar roles de acordo com as funções existentes dentro da empresa. Por exemplo: uma role para caixa, outra para analista e outra para gestor.

Cada pessoa continua utilizando seu próprio login, mas recebe as permissões relacionadas à função que exerce. No PostgreSQL, uma role desse tipo pode ser criada como `NOLOGIN`. Isso significa que ninguém entra diretamente no banco usando aquela role. Ela funciona como um conjunto de permissões que depois é associado aos usuários.

Isso também diminui a possibilidade de erro do próprio DBA. Imagine uma empresa com dezenas ou centenas de usuários. Se cada um tiver suas permissões configuradas manualmente, aumenta a chance de alguém receber um acesso que não deveria ter por engano.

Com as roles, a configuração fica mais organizada. Se alguma regra mudar, o DBA pode alterar a role e aplicar a mudança aos usuários ligados a ela. Se uma pessoa mudar de função ou sair da empresa, também fica mais fácil remover aquele acesso.

Como cada funcionário continua usando seu login individual, ainda é possível identificar nos registros quem realizou determinada operação. No caso do uso de IA, isso também ajuda a evitar que uma consulta aproveite alguma permissão que ficou liberada por engano.

**4. Controle de tempo de execução e concorrência.**
Nem todo problema está relacionado a acessar informações que não deveria. Uma consulta pode ser completamente autorizada e, mesmo assim, causar problemas por ser pesada demais.

Imagine uma consulta criada por IA que fique vários minutos lendo tabelas muito grandes. Durante esse período, ela pode consumir processamento, memória e acesso ao disco, enquanto outras operações do sistema também precisam utilizar o banco.

No PostgreSQL, uma forma de reduzir esse problema é utilizar o `statement_timeout`. Com ele, pode ser definido um limite para o tempo de execução de uma consulta. Se o limite for de 30 segundos, por exemplo, uma consulta que ultrapassar esse tempo será cancelada automaticamente.

Também é possível controlar a quantidade de conexões utilizadas por determinados usuários ou perfis. Isso ajuda a evitar uma situação em que vários analistas executem relatórios pesados ao mesmo tempo e ocupem recursos que também são necessários para operações importantes da empresa.

Essa proteção está ligada principalmente à disponibilidade. Em uma loja, por exemplo, uma consulta muito pesada não deveria conseguir prejudicar o funcionamento do caixa enquanto existem clientes esperando para serem atendidos.

**5. Auditoria e rastreamento de operações.**
Mesmo com várias proteções, ainda é importante saber o que aconteceu dentro do banco. É aí que entra a auditoria.

A ideia é manter registros das operações realizadas, permitindo identificar informações como usuário, comando executado e horário. Diferentemente das medidas anteriores, a auditoria não necessariamente impede que alguma coisa aconteça. Ela serve principalmente para investigar o que aconteceu depois.

No PostgreSQL, recursos de registro podem ajudar nesse acompanhamento. O `log_statement`, por exemplo, pode registrar comandos executados, enquanto a extensão `pg_stat_statements` permite acompanhar estatísticas das consultas, ajudando a identificar aquelas que são executadas com muita frequência ou que possuem maior custo.

Com essas informações, o DBA consegue investigar consultas que estão prejudicando o desempenho, perceber comportamentos fora do normal e entender melhor como o banco está sendo utilizado. Em caso de incidente envolvendo dados pessoais, os registros também podem ajudar na investigação do que ocorreu.

Porém, existe uma limitação importante. O banco só consegue registrar aquilo que acontece dentro dele. Se um usuário autorizado fizer uma consulta normal, copiar o resultado e depois colocar essas informações em uma ferramenta de IA externa, o banco não consegue enxergar essa segunda ação. Para ele, ocorreu apenas uma consulta autorizada. Por isso, o vazamento por prompts continua sendo um problema que não pode ser resolvido somente com configurações do PostgreSQL.

**6. Conformidade com a LGPD desde o desenho.**
A LGPD também precisa ser considerada desde o momento em que o banco de dados está sendo planejado. Diferentemente das outras medidas, isso não significa executar um único comando específico. É uma ideia que deve orientar várias decisões de segurança.

Se determinado funcionário não precisa visualizar o CPF de um cliente para realizar sua função, por exemplo, o ideal é que essa informação nem chegue até ele. Quando o DBA cria uma view sem determinados dados pessoais ou limita permissões de acordo com a necessidade de cada usuário, essa preocupação já está sendo colocada em prática.

Uma abordagem menos segura seria liberar um acesso muito amplo e simplesmente orientar os funcionários a não consultarem determinadas informações. O problema é que, mesmo com a orientação, os dados continuam disponíveis. Se alguém cometer um erro ou fizer uma consulta inadequada, essas informações ainda podem aparecer.

Por isso, considerar a LGPD desde o desenho do banco transforma uma obrigação que poderia parecer apenas jurídica em uma preocupação também técnica. Views, roles e permissões podem ser planejadas levando em consideração quais informações cada usuário realmente precisa acessar.

Quando observamos todas essas medidas juntas, fica mais fácil perceber por que é importante trabalhar com várias camadas de segurança. O menor privilégio e as views ajudam a controlar quais informações podem ser acessadas. As roles facilitam a organização dessas permissões e diminuem a chance de erros de configuração. O controle de execução ajuda a manter o sistema disponível, enquanto a auditoria permite investigar o que aconteceu quando alguma coisa foge do esperado. A preocupação com a LGPD ajuda a orientar todas essas decisões.

Mesmo assim, ainda existe uma limitação: o banco não consegue controlar totalmente aquilo que o usuário faz com uma informação depois que recebeu acesso legítimo a ela. Se alguém copiar um resultado e enviar para uma IA externa, por exemplo, as configurações do PostgreSQL não conseguem impedir isso sozinhas.

Por esse motivo, a distribuição segura dos dados não depende apenas das configurações técnicas. Também é necessário acompanhar como esses recursos estão sendo utilizados e orientar os usuários sobre os riscos. Nesse ponto, o papel do DBA continua sendo importante, como será abordado na próxima seção.


### 1.5 Atuação do DBA no cenário de IA
Monitoramento, políticas de acesso, auditoria, orientação aos usuários,
performance e backups.

### 1.6 Análise crítica: qual a melhor abordagem?

A posição do grupo é que **não existe uma medida única suficiente**: a melhor abordagem para distribuir dados com segurança quando usuários especialistas usam IA é a **defesa em profundidade** (segurança em camadas), com o **próprio banco de dados** — e não a ferramenta de IA nem a aplicação — atuando como a última linha de controle.

O raciocínio central é o seguinte: no cenário tradicional, a segurança podia se apoiar na aplicação, porque era o programador quem escrevia o SQL. Com a IA generativa, o **usuário especialista passa a gerar SQL diretamente**, sem mediação. Isso significa que o grupo **não tem controle sobre o prompt** que o usuário escreve nem sobre o que a IA devolve. Confiar na "boa consulta" da IA seria transferir a segurança para um componente que erra, alucina e pode expor dados. Por isso, o controle precisa estar **na camada dos dados**, onde é o PostgreSQL que decide o que cada usuário pode ver e executar, independentemente do que o prompt pediu.

A partir dessa premissa, o grupo defende a combinação das seguintes camadas, em ordem de importância:

1. **Menor privilégio como fundação.** Cada usuário nasce sem acesso e recebe apenas o mínimo necessário. A IA pode gerar `SELECT * FROM clientes`, mas se a role não tem permissão na tabela base, a consulta simplesmente falha.
2. **Views como interface obrigatória.** Os usuários e a IA nunca tocam nas tabelas reais; só enxergam views que já removem colunas com dados pessoais (CPF, endereço) e filtram linhas. Assim, mesmo uma consulta "perfeita" da IA não consegue devolver o que não está na view. Para filtro de **linhas** mais robusto — por exemplo, cada analista só enxergar clientes da sua própria região — o PostgreSQL ainda oferece o **Row Level Security (RLS)** como complemento às views.
3. **Roles customizadas por perfil**, agrupando permissões por função (analista, gestor, auditor) em vez de configurar usuário por usuário — o que reduz o erro humano do próprio DBA.
4. **Controle de recursos** (`statement_timeout`, limite de conexões) para conter consultas pesadas geradas por IA que degradariam a performance.
5. **Auditoria contínua** (logs e `pg_stat_statements`), porque nenhuma barreira é perfeita: é preciso rastrear quem executou o quê e detectar abuso.
6. **Conformidade com a LGPD embutida no desenho**, e não como remendo: dado pessoal que o analista não precisa ver simplesmente não deve chegar até ele.

Há ainda um ponto específico da IA que reforça a escolha por views: o **vazamento por prompt**. Quando o usuário cola dados reais numa ferramenta de IA externa para "pedir ajuda", esses dados saem da organização. Se o usuário só trabalha sobre views já mascaradas, o que ele eventualmente enviar para a IA externa **já não contém dado pessoal identificável**, reduzindo o risco na origem.

Por fim, o grupo entende que a tecnologia sozinha não fecha o problema: é indispensável o **DBA como auditor humano** e a **orientação de uso** aos especialistas. A IA gera; o DBA governa. A melhor abordagem, portanto, é técnica **e** organizacional: banco configurado para negar por padrão, mais um DBA que monitora, revisa e educa.

## 2. Exemplos e Casos

### 2.1 View `clientes_visiveis` (limita colunas e linhas)

A view abaixo expõe apenas o que um analista precisa e **omite dados pessoais** (CPF, e-mail, endereço completo, telefone), além de restringir as linhas a clientes ativos:

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

> **Por que a view funciona mesmo sem acesso à tabela base?** No PostgreSQL, uma view é executada com os privilégios do **dono da view** (mecanismo conhecido como *ownership chaining*), e não com os do usuário que a consulta. Por isso é possível liberar a view para o analista e, ao mesmo tempo, manter a tabela `clientes` totalmente inacessível para ele. Em views usadas como barreira de segurança, recomenda-se ainda declará-las com `WITH (security_barrier)`, para impedir que funções injetadas no `WHERE` "vazem" linhas que deveriam ter sido filtradas.

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
- Dados pessoais simplesmente **não estão nas views**, então nem a consulta da IA nem um eventual prompt enviado a uma ferramenta externa expõem CPF ou endereço.

**Resultado:** os analistas continuam autônomos e produtivos com a IA, mas dentro de um "cercado" seguro definido pelo DBA — conciliando produtividade, segurança, performance e conformidade.

## 3. Referências

- SILBERSCHATZ, A.; KORTH, H. F.; SUDARSHAN, S. **Sistema de Banco de Dados**. 6. ed. Rio de Janeiro: Elsevier, 2012. (origem da definição de Administrador de Banco de Dados e do controle centralizado dos dados).

- ELMASRI, R.; NAVATHE, S. B. **Sistemas de Banco de Dados**. 7. ed. São Paulo: Pearson, 2018.

- POSTGRESQL GLOBAL DEVELOPMENT GROUP. **PostgreSQL Documentation** — seções *CREATE ROLE*, *GRANT*, *CREATE VIEW* (incluindo `security_barrier`) e *Row Security Policies* (RLS). Disponível em: https://www.postgresql.org/docs/. Acesso em: ago. 2026.

- BRASIL. **Lei nº 13.709, de 14 de agosto de 2018** — Lei Geral de Proteção de Dados Pessoais (LGPD), com destaque ao Art. 5º, I (dados pessoais, categoria que abrange CPF, e-mail e endereço) e ao Art. 5º, II (dados pessoais sensíveis).

- CAFEGEEK. **Perfis de Usuários de Banco de Dados**. Jan. 2026.

## 4. Conclusões

O estudo mostrou que a chegada da IA generativa **muda de lugar o risco**, mas não cria uma ameaça sem solução. Ao permitir que usuários especialistas gerem SQL diretamente, a IA remove a camada de proteção que antes existia no código da aplicação — o que obriga a organização a **reforçar a segurança dentro do próprio banco de dados**.

O principal aprendizado do grupo é que **segurança de dados com IA é uma questão de arquitetura, não de confiança na ferramenta**. Não se deve esperar que a IA "gere consultas seguras"; deve-se garantir que, mesmo que ela gere uma consulta perigosa, o banco a impeça. Isso se alcança com camadas que se reforçam: menor privilégio, views que filtram colunas e linhas, roles por perfil, controle de execução e auditoria contínua.

Refletimos também que a **LGPD deixa de ser um detalhe jurídico e vira requisito técnico**: se o dado pessoal não precisa ser visto, ele não deve sequer chegar ao usuário — e, por consequência, não chega aos prompts enviados a ferramentas externas.

Por fim, o ponto mais observado pelo grupo é o **papel insubstituível do DBA**. A IA acelera a geração de consultas, mas quem define o esquema, desenha as políticas de acesso, monitora consultas abusivas e orienta o uso responsável continua sendo um profissional humano. A conclusão se resume em uma frase: **a IA gera as consultas; o DBA governa os dados**.

## [Link do Repositório Git](https://github.com/hectorex/atividade-bd/blob/main/ATIVIDADE-teorica-ia-dba-hector-lins-araujo.md)
