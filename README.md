# datajud-graph-analytics
Trabalho para o Hackathon do DATAJUD - https://www.cnj-inova.com/


ANALISE DE GRAFOS APLICADO A ACOMPANHAMENTO DE PROCESSOS JUDICIAIS

Autores:
EQUIPE 18
- Mauro Tomio Saito  
- Ibson Melo dos Santos Rego  
- Yuri Jacob Lumer  


TÍTULO: ANALISE DE GRAFOS APLICADO A ACOMPANHAMENTO DE PROCESSOS JUDICIAIS  


- BREVE DESCRIÇÃO DO PROJETO  

O presente trabalho trata de uma proposta de solução para o Desafio 1 do Hackathon do CNJ INOVA, realizado no período de 09/10/2020 a 19/11/2020.  

O desafio trata da mineração do banco de dados do DATAJUD, que tem como principais objetivos, basicamente:  

a) encontrar em soluções que fomentem a celeridade processual;  
b) construir uma estratégia inteligente de controle interno de processos e alertar sobre possíveis gargalos no tempo de tramitação processual;  
c) auxiliar na construção de um diagnóstico para oportunizar medidas assertivas a fim de permitir maior eficiência dos atos.  

Tendo em vista que a análise por grafos pode ser útil para resolução de problemas relacionados a processos e que um diagrama por vezes gera mais informação que várias linhas de palavras, além de que seguem três premissas - performance, flexibilidade e agilidade - propusemos o uso da ferramenta NEO4J. 


- NEO4J  

É uma das 25 ferramentas mais utilizadas para gerenciamento de banco de dados e a mais utilizada no modelo baseado em grafos, segundo o site https://db-engines.com/en/ranking.

É um banco de dados não-relacional (nosql), orientado à grafos, e se distingue dos tradicionais bancos de dados por não possuir um esquema fixo, como linhas e colunas, sendo, por isso, chamado 'schemaless'

A linguagem utilizada pelo NEO4J é a linguagem Cypher, que é muito similar ao SQL. Ela permite que se crie, modifique e procure dados em uma estrutura baseada em um grafo de informações e relacionamentos.



- INSTALAÇÃO

A instalação em ambiente em Windows é encontrada, de forma detalhada, no próprio site oficial da NEO4J:

https://neo4j.com/docs/operations-manual/current/installation/windows


Pré-requisitos

Antes de iniciar a apliação, para que seja possível carregar os arquivos em json, fazer a instalação da biblioteca APOC na aba de 'Plugins' da ferramenta.

Além da instalação da biblioteca APOC, é necessário incluir os parâmetros abaixo no final do arquivo neo4j.conf.
O arquivo neo4j.conf também pode ser editado na aba 'Settings', ao lado da aba 'Plugins' da área de configuração.

### Parâmetros a serem incluídos ao final do arquivo neo4j.conf
apoc.trigger.enabled=true  
apoc.ttl.enabled=true  
apoc.import.file.enabled=true  



### SCRIPTS PARA CONSTRUÇÃO DA REDE DE RELACIONAMENTOS DO BANCO DE DADOS DO DATAJUD
### Orientações Iniciais
### Para a construção dos NÓS, não há necessidade de seguuir rigorosamente a sequencia abaixo.
### Para a construção das ARESTAS (vínculos), prestar atenção para a necessidade da criação antecedente dos NÓS que compõem os vínculos.
### Prestar atenção para não executar o mesmo script mais de uma vez, pois isso gera duplicidade dos elementos.

### Os arquivos JSON a serem carregados deverão estar na pasta IMPORT, cujo atalho encontra-se no atalho do browser de configuração do NEO4J.

### Para reiniciar 'DO ZERO', digitar o seguinte script: MATCH (n) DETACH DELETE n

### Para o script abaixo, utilizou-se o arquivo 'processos-tre-sp_1.json' como exemplo.
### Para utilização de outro arquivo, carregar o arquivo json na pasta IMPORT, conforme orientação acima.
### Observar que para cada elemento criado, temos as informações de metadados dos nós, dado pela cláusula SET.

##########################
### CONSTRUÇÃO DOS NÓS ###
##########################

### NÓS de 'PROCESSOS'####

CALL apoc.load.json("processos-tre-sp_1.json") YIELD value as processo  

MERGE (p:Processo {numero: toInteger(processo.dadosBasicos.numero)})  

SET
p.millisInsercao = toInteger(processo.millisInsercao),  
p.dataAjuizamento = toInteger(processo.dadosBasicos.dataAjuizamento),  
p.classeProcessual = processo.dadosBasicos.classeProcessual,  
p.nivelSigilo = processo.dadosBasicos.nivelSigilo,  
p.competencia = processo.dadosBasicos.competencia,  
p.codLocalidade = processo.dadosBasicos.codigoLocalidade,  
p.tribunal = processo.siglaTribunal,  
p.codOrgaoJulgador = processo.dadosBasicos.orgaoJulgador.codigoOrgao  


### NÓS de 'ORGÃOS JULGADORES' ####

CALL apoc.load.json("processos-tre-sp_1.json") YIELD value as processo  
MERGE (oj:  CodOrgaoJulgador {codOrgaoJulgador: processo.dadosBasicos.orgaoJulgador.codigoOrgao})  
SET  
oj.nomeOrgao = processo.dadosBasicos.orgaoJulgador.nomeOrgao,  
oj.codMunicipioIBGE = processo.dadosBasicos.orgaoJulgador.codigoMunicipioIBGE,  
oj.instancia = processo.dadosBasicos.orgaoJulgador.instancia  


### NÓS de 'MOVIMENTOS' processuais ####

CALL apoc.load.json("processos-tre-sp_1.json") YIELD value as processo  
unwind processo.movimento as mov  
CREATE (m: Movimento {dataHora: toInteger(mov.dataHora)})  
SET m.millisInsercao = processo.millisInsercao  
SET m.movimentoNacional = mov.movimentoNacional.codigoNacional  
set m.movimentoLocalCodLocal = mov.movimentoLocal.codigoMovimento  
set m.movimentoLocal.codPaiNacional = mov.movimentoLocal.codigoPaiNacional  


### NÓS de 'CLASSES PROCESSUAIS' ####

CALL apoc.load.json("processos-tre-sp_1.json") YIELD value as processo  
MERGE (c:ClasseProcessual {classeProcessual: toInteger(processo.dadosBasicos.classeProcessual)})  


#########################################
### CONSTRUÇÃO DAS ARESTAS (VINCULOS) ###
#########################################

### VÍNCULOS entre as 'CLASSES PROCESSUAIS' e os 'PROCESSOS' ###

MATCH (p:Processo), (c:ClasseProcessual)  
where p.classeProcessual = c.classeProcessual  
CREATE (p)-[:CLASSE]->(c)  


### VÍNCULOS entre 'PROCESSO' e os 'ÓRGAOS JULGADORES' ###

MATCH (p:Processo), (j:CodOrgaoJulgador)  
WHERE p.codOrgaoJulgador = j.codOrgaoJulgador  
CREATE (j)<-[:JULGADO_POR]-(p)  


### VÍNCULOS entre 'PROCESSOS' com seus 'MOVIMENTOS' processuais (tramitação) ###

match (p:Processo), (m:Movimento)  
where p.millisInsercao=m.millisInsercao  
CREATE (p)-[:MOVIMENTO]->(m)  


#########################################
### EXEMPLO DE CONSULTAS E 'INSIGHTS' ###
#########################################

### Uma vez construído toda a rede de relacionamentos entre as entidades, podemos fazer perguntas ou mesmo, o inverso, gerar respostas a perguntas, ou seja, 'insights' diversos
### A seguir, seguem alguns exemplos de scripts que podem auxiliar em diversas análises e no amadurecimento de soluções (que seriam pouco provaveis em outros modelos de bancos de dados)


### Quantas movimentações houve no PROCESSO de número 24620116260003 ###

MATCH (p:Processo), (m: Movimento), (p)-[:MOVIMENTO]->(m)  
WHERE p.numero = 24620116260003   
RETURN p.numero  


### Quais as CLASSES PROCESSUAIS existentes em cada ÓRGÃO JULGADOR ###

MATCH (o:CodOrgaoJulgador), (p:Processo),(o)<-[:JULGADO_POR]-(p)  
RETURN  o.nomeOrgao, collect(p.classeProcessual)  


### Qual a quantidade de PROCESSOS por ÓRGÃO JULGADOR ###

MATCH (o:CodOrgaoJulgador), (c:ClasseProcessual)  
RETURN  o.nomeOrgao, count(c)  
order by count(c) desc  


### Quantas MOVIMENTOS houve em cada ÓRGÃO JULGADOR ###

MATCH (o:CodOrgaoJulgador), (p:Processo), (m:Movimento),  
(o)<-[:JULGADO_POR]-(p)-[:MOVIMENTO]->(m)  
RETURN o.nomeOrgao, COUNT(m.dataHora)  
ORDER BY COUNT(m.dataHora) DESC  


### Quantos MOVIMENTOS processuais houve em cada ÓRGÃO JULGADOR?

MATCH (o:CodOrgaoJulgador), (p:Processo), (m:Movimento), (o)<-[:JULGADO_POR]-(p)-[:MOVIMENTO]->(m)  
RETURN o.nomeOrgao, count(m.dataHora)  
ORDER BY COUNT(m.dataHora) DESC  


- MELHORIAS A SEREM IMPLEMENTADAS

Tendo em vista que um dos objetivos é "alertar sobre possíveis gargalos no tempo de tramitação processual", uma rotina que inclui o tempo entre um movimento processual e a seguinte, que seria dada pela diferença entre os metadados de dataHora entre os movimentos processuais.


- AMBIENTE DE TESTES E VERIFICAÇÃO DAS FUNCIONALIDADES - provisório para o período do Hackathon: 09/10/2020 a 19/11/2020

Para possibilitar a verificação das funcionalidades e testar as consultas, um ambiente em Windows com o NEO4J instalado será disponibilizado via RDP no endereço eletrônico no seguinte site:  

www.to1000analytics.com  
Nome de usuário: Administrator  
Senha: datajud  
