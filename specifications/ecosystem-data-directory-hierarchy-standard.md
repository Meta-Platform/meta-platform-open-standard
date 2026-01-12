# Ecosystem Data Directory Hierarchy Standard

* **binaries**: diretório que contém todos os binários necessários para que um pacote do ecossistema possa ser executado corretamente. É utilizado em ambientes onde o sistema operacional não fornece essas dependências por padrão.

* **config-files**: contém arquivos de configuração que padronizam a execução dos pacotes dentro de um ecossistema.
    * **ecosystem-defaults.json** dentro do diretório de configurações, contém metadados importantes onde estão definidos os parâmetros que padronizam todos os quesitos do ecossistema.
* **npm-dependencies**: diretório/projeto que contém as dependências mínimas necessárias para que o *package-executor* consiga carregar um pacote juntamente com suas próprias dependências.

* **environments**: quando um pacote é executado, é criado um ambiente de execução com o nome do pacote mais o hash do seu endereço no sistema de arquivos, por exemplo
  `repository-manager.cli-e9766fee7f2d40e9c498cabb6c06a9581236c7ba9d1310999b5e24d1dd126899`.
  Cada execução ocorre em um diretório isolado por pacote executado. Dentro de um ambiente de execução, encontram-se metadados da execução e o diretório `.dependencies/node_modules`. Futuramente, também poderão existir dependências de outras linguagens. Os dois metadados mais importantes encontrados em um ambiente de execução são:

  * **metadata-hierarchy.json**: contém as informações necessárias para carregar todos os pacotes exigidos para que o ambiente seja executado dentro do ecossistema. Esse arquivo é processado e, a partir dele, é criado o metadado que será executado pelo *task executor*.
  * **execution-params.json**: metadado que contém todas as unidades de execução que serão executadas pelo *task executor*, bem como suas interdependências.

* **sockets**: diretório onde ficam todos os sockets Unix criados pelos processos em execução. É utilizado para permitir comunicação entre aplicações e processos em alta velocidade, utilizando IPC com baixo acoplamento.

* **supervisor-sockets**: todo processo iniciado pelo *package-executor* pode criar um socket de supervisão, por meio do qual é possível realizar comunicação binária com o processo via gRPC. Atualmente, as seguintes operações são suportadas: `KillInstance`, `GetStartupArguments`, `GetProcessInformation`, `GetStatus`, `ListTasks`, `GetTask`, `LogStreaming` e `StatusChangeNotification`.

* **executables**: diretório onde ficam todos os executáveis que fazem parte do ecossistema e que foram instalados pelo *myrepo* ou pelo *wizard*. Nele, encontram-se aplicações, Web APIs e CLIs disponíveis para uso, de forma equivalente ao diretório `\bin`. Uma observação importante é que o diretório **executables** deve estar presente na variável de ambiente `PATH`, para que o usuário consiga utilizar as ferramentas do ecossistema.

* **repos**: contém todos os repositórios instalados no ecossistema.

* **sources.json**: contém informações sobre os repositórios disponíveis para instalação e atualização. Neste arquivo, deve estar registrada a fonte de cada repositório, permitindo que o ecossistema consiga realizar a instalação.

* **repositories.json**: metadado que registra todos os repositórios instalados e suas respectivas aplicações. Vale lembrar que, quando um repositório é instalado, ele é armazenado no diretório **repos**.

