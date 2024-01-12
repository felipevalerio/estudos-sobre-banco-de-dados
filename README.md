# estudos-sobre-banco-de-dados

Just a bunch of annotations about DMBS based on the CMU Database Courses and the book Database Internals.
Everything is in Portuguese, I will rewrite the in a less crazy and more "formal" way in English.









A tarefa principal de um banco de dados (sistema de gerenciamento de banco de dados) é armazenar dados de uma forma segura e torná-los disponíveis para os usuários.

Banco de dados é um sistema modular composto por: uma camada de transporte (transport layer) para aceitar requisições, processador de query (query processor) que determina a maneira mais eficiente de executar uma query, um motor de execuções (execution engine) que toma conta das execuções das requisições e um motor de armazenamento (storage engine) que será discutido abaixo.


Storage Engines: é um componente de um banco de dados responsável por armazenar, recuperar e organizar/gerenciar dados em memória e em disco de uma maneira persistente. A storage engine fornece uma API que permite criar, editar, deletar e selecionar dados armazenados. 
Alguns bancos de dados (MySQL) podem ter várias storage engines para serem escolhidas. Elas são “plugáveis”, podendo ser trocadas a qualquer momento. Por exemplo: InnoDB, MyRocks e MyISAM.

OLTP: esse tipo de banco de dados tem a capacidade de lidar com um alto número de requisições e transações do usuário. As queries são mais “simples” e curtas.

OLAP: esse tipo de banco de dados é capaz de executar queries complexas, para que sejam realizadas análises nos dados armazenados.

HTAP: combina os dois tipos anteriores.


Existem muitos outros termos e classificações: armazenamentos de valores-chave (key value store), bancos de dados relacionais (relational databases), armazenamentos orientados a documentos (document databases), bancos de dados gráficos (graph databases) e banco de dados não relacional (noSQL).






![Screenshot 2024-01-11 at 23-25-18 Anotações sobre Databases](https://github.com/felipevalerio/estudos-sobre-banco-de-dados/assets/33168249/f12b264d-e3e6-4e34-b530-db69c0bdd19d)






Arquitetura de um banco de dados



A storage engine é por sua vez, composta por diversos componentes:

Transaction Manager: realiza o agendamento de transações e garante que elas não possam sair do banco de dados em um estado logicamente inconsistente.

Lock Manager: realiza o bloqueio de objetos do banco de dados para que as transações sejam executadas, garantindo que as operações simultâneas não violem a integridade física dos dados.

Access Methods (Storage Structure): uma estrutura que é responsável por organizar os dados em disco, como por exemplo: B-Tree ou LSM Trees.

Buffer Manager: responsável por realizar o cache das páginas na memória.

Recovery Manager: mantém um log das operações e é responsável por recuperar o estado do sistema em caso de falha.

In memory databases: armazena dados “em memória” e usa o disco para recuperação e logging.

Disk bases databases: armazena os dados em disco e usa a memória para cache ou como um armazenamento temporário.


Leitura de Dados de maneira sequencial ou de maneira aleatória

Leitura de maneira aleatória em memórias não voláteis (HDD, SDD), é quase sempre muito mais lenta do que uma leitura de maneira sequencial.

O Banco de dados sempre irá tentar maximizar a leitura de dados de maneira sequencial.

I/O em disco é uma ação cara e se for realizada diversas vezes para somente uma operação/transação, pode acabar afetando a performance do banco.


Exemplo parcial de um banco baseado em disco (HDD)
























File Storage

Armazena um ou mais arquivos em disco em um formato de arquivo proprietário.

Storage Manager é responsável por manter os dados em disco. Alguns fazem a leitura e escrita dos arquivos de maneira esporádica.

Organiza os arquivos como uma coleção de páginas.

A página é um bloco de dados de tamanho fixo:
contém diversos tipos de dados: tuplas, logs, indexes, metadados…
maioria dos bancos mantém um tipo de dados de uma página de maneira específica
cada página tem um ID específico, usado pelo banco para mapear as páginas em disco

Quando falamos de páginas, podemos nos referir a três definições:
Hardware Page (geralmente 4KB)
OS Page (4KB)
Database Page (512B-32KB)


Organização das páginas em disco

Diferentes formas de organizar páginas em disco, com o objetivo de organizar e saber como encontrar as páginas:
Heap File Organization
Tree File Organization
Sequential/Sorted File Organization
Hashing File Organization














Heap File Organization

Coleção de páginas de maneira desorganizada com as páginas sendo armazenadas como tuplas. Suporta create/get/write/delete page (API) assim como iteração pelas páginas.

offset: descolamento. No caso, o deslocamento será descoberto se o número da página (ID?) for multiplicado pelo tamanho da mesma.




O Diretório de páginas contém todas as páginas de arquivos que foram criadas, suas posições, tamanhos etc…



O diretório precisa ser um reflexo dos arquivos que estão armazenados de maneira física, todas as informações (tamanho, posições) devem ser iguais.

O diretório também guarda informações (metadados) que dizem respeito a quanto espaço tem de sobra, páginas vazias etc…

Um arquivo (database file) para diversas tabelas (como duckdb): primary_db.duckdb

Ou diversos arquivos para diversas tabelas (postgresql)














Estrutura de uma página



Toda página contém um header de metadados sobre o conteúdo da página:

Tamanho da página
Checksum -> para checar a integridade dos dados
Versão do banco
Compression/Encoding dos metadados
Informação do schema
Sumário dos dados


Layout de uma página

Como os dados são organizados dentro de uma página.

Organizados como tuplas
Organizados como estruturas de logs
Organizados via index


Exemplo de dados organizados em tuplas. Um dado com 5 atributos será armazenado dessa forma:

{01, “Felipe”, 28, “Programador”, “Morretes”, “Paraná”}

Strawman Idea: manter um contador com a quantidade de tuplas em uma página e adicionar uma nova tupla no fim.

Isso é útil para pegar o número da página, multiplicar pelo tamanho da mesma e o resultado será o offset para indicar onde escrever a página.



Problemas dessa ideia:
Deletar uma tupla se torna um problema, pois nós poderíamos querer usar o espaço liberado pela tupla deletada, mas isso irá violar a regra da ideia que diz que uma nova tupla deve ser adicionada no fim, ou seja, em uma posição após todas as tuplas já existentes na página. A tupla seria adicionada no local recém liberado, tornando assim as tuplas fora de ordem.




E se as tuplas não tiverem um tamanho fixo? Exemplo: endereços de email tem tamanhos diferentes. Então a tupla não poderia caber na página (que tem tamanho fixo)




Slotted Pages: existe um header, mas junto com ele existe um array de slots que contém as posições das tuplas.
O header mantém as informações sobre o número de slots usados e a localização do offset (deslocamento) do último slot usado.



Caso um novo dado seja adicionado, uma nova tupla, o slot array vai conter a posição da nova tupla e o array irá crescer do começo para o fim. Enquanto a tupla será adicionada do fim ao começo.



Caso uma tupla seja deletada, é preciso somente mover uma tupla, por exemplo a tupla #4, para o local (offset) vago e atualizar o slot 4 com o novo offset, indicando que a tupla #4 está agora no local (offset) que era antes vago. 


Resumindo: um banco de dados, na sua camada de storage, contém um diretório com todas as páginas de arquivos criadas em disco. Esse diretório contém diversas informações sobre as páginas: IDs, tamanho, posição e outros metadados. Quando for necessário ler, deletar, criar ou atualizar as páginas, o banco cria uma cópia desse diretório em memória (buffer pool), e recebe da execution engine qual página é necessária para fazer a ação. O diretório então calcula o tamanho da página x o número dela (ID?) para conseguir o offset que é o deslocamento que a página está localizada. Pense em offset como o resultado de uma operação que está dizendo, a página de ID X tem o tamanho Y, o offset é uma referência que vai do começo até atingir o tamanho final da página, por isso em português offset é um deslocamento. É o descolamento da página em disco. Isso então retorna um ponteiro de 64 bits para a execution engine, para que ela possa fazer a ação requisitada pelo usuário. Se forem feitas alterações (update), as mesmas são feitas na página in-memory, para que então os dados sejam replicados na página que está armazenada de maneira física, em disco. 

A página também contém diversas informações sobre ela mesma. No header estão presentes o tamanho da página, checksum, versão do banco, compression/encoding dos metadados, informação do schema, sumário dos dados.
A página também utiliza de offsets para saber onde se encontram os dados armazenados na página (armazenados como tuplas).

Como identificar as tuplas?


