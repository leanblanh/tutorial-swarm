# Tutorial Docker Swarm
# Tutorial 1 - Primeiros Passos

Para seguir esse tutorial você vai precisar de uma máquina com a Docker Engine Instalada. https://docs.docker.com/desktop/setup/install/linux/ubuntu/

## Step #1
Ativar o Swarm na instância que tem o docker engine instalada. Chamado Node.
No Swarm, o Node é a máquina onde o Docker está instalado. Quando ativamos o modo Swarm, esse Node assume uma função:

**Manager**: O "cérebro". Ele decide onde os containers rodam e mantém o estado desejado da aplicação.

**Worker**: O "braço". Ele apenas executa as tarefas que o Manager envia.
Como estamos em um único node, seu Ubuntu será Manager e Worker simultaneamente.

Ação no Terminal
Execute o comando abaixo para transformar seu host em um Manager:
<br><br>
`docker swarm init --advertise-addr 127.0.0.1` <br><br>
Nota: O parâmetro --advertise-addr indica qual IP os outros nodes usariam para falar com este Manager. Como este tutorial é em um ambiente de estudo local (localhost) e único, usamos o loopback (127.0.0.1). <br>

### Retorno
**Swarm initialized**: O Docker agora criou uma infraestrutura interna de PKI (criptografia) para comunicação segura.

**Join Token**: Ele exibirá um comando longo começando com docker swarm join. Ignore-o por enquanto, já que não vamos adicionar outras máquinas.

**Raft Consensus**: O Manager agora mantém um banco de dados interno (Raft) para garantir que, se você pedir 3 réplicas de um serviço, ele saiba que precisa manter exatamente 3.

**Verificação**
Para confirmar que o node está pronto e operacional, digite:

`docker node ls`

Será exibida uma tabela com o HOSTNAME da máquina, o STATUS como Ready e a MANAGER STATUS como Leader.

Se precisar reverter o Docker para o modo padrão (standalone) ou "limpar" as configurações do orquestrador, o comando é simples:

`docker swarm leave --force`

**O que esse comando faz?**
**Remoção de Serviços**: Todos os serviços, stacks e containers gerenciados pelo Swarm são interrompidos e removidos imediatamente.

**Limpeza de Redes**: As redes do tipo overlay (que veremos no próximo passo) são excluídas.

**Certificados e Segurança**: As chaves de criptografia e o banco de dados interno (Raft) que o Swarm criou no Passo 1 são apagados do disco.

**Estado do Node**: O Docker Engine volta a operar apenas com docker run e docker-compose, sem as capacidades de replicação e auto-recuperação do Swarm.

**Nota**
Em um ambiente com múltiplos nodes, um Worker pode sair pacificamente apenas com o comando `docker swarm leave`.
Porém, em um node Manager exige o `--force` para confirmar que você está ciente de que está destruindo a "gerência" do cluster.

## Step #2
Agora que o seu node é um Manager, precisamos preparar o terreno para que a API, o Nginx e o Postgres conversem entre si. No Docker comum (standalone), usamos redes do tipo bridge. No Swarm, o conceito evolui para a Overlay Network.

**O Conceito**

A rede Overlay é um túnel virtual que conecta os containers (agora chamados de Tasks) em uma rede privada e isolada do host.

- Service Discovery: No Swarm, você não precisa saber o IP de cada container. Se o Nginx precisa falar com a API, ele simplesmente chama por http://api. O Swarm tem um servidor DNS interno que resolve o nome do serviço para o IP correto.

- Isolamento: Somente os serviços que você colocar explicitamente nesta rede conseguirão "se enxergar". O Postgres ficará protegido do mundo externo, acessível apenas pela API.

Ação no Terminal
Vamos criar uma rede chamada `app_network`. Execute:

`docker network create --driver overlay app_network`

O comando retornará um ID longo (o identificador da rede). Para confirmar que a rede foi criada com o driver correto, use:

`docker network ls`

Irá exbir a rede `app_network com a coluna SCOPE definido como swarm. Isso é crucial, pois redes local não funcionam para serviços do Swarm.

**Conceito chave**: A rede Overlay é o "backbone" de comunicação do seu cluster Swarm, oferecendo DNS interno e isolamento de tráfego.

## Step #3

Vamos usar o Docker Secrets nativo para as senhas do Postgres, pois ele já introduz o conceito de "entrega segura de dados" que o Docker comum não faz nativamente (sem Compose).

### 1. Persistência (Volume)
No Swarm, se o container do Postgres morrer e subir em outro lugar (ou reiniciar), os dados somem se não houver um volume.

`docker volume create pg_data`

### 2. Segredo (Secret)

Em vez de passar a senha do banco em uma variável de ambiente (visível no docker inspect), vamos criar um objeto de segredo no banco de dados do Swarm.

`echo "minha_senha_super_secreta" | docker secret create db_password -`

O Swarm pegou essa string, criptografou-a e guardou no nó Manager. Quando subirmos o Postgres, o Swarm vai "teletransportar" esse segredo para dentro do container, montando um sistema de arquivos temporário (em RAM) que o banco lerá.

**Conceito chave**: Segredos no Swarm não são variáveis de ambiente expostas; são arquivos montados em memória, protegidos pelo orquestrador.

### Conceito importante sobre a gestão de secrets no Docker Swarm
No Docker comum (docker run), é comum passarmos senhas via variáveis de ambiente (ex: `-e POSTGRES_PASSWORD=123`). O problema? Qualquer um com acesso ao terminal que der um `docker inspect` ou um ps aux verá a senha em texto claro.

**No Docker Swarm Secrets:**

- Entrega Segura: O segredo viaja do Manager para o Worker (mesmo que sejam a mesma máquina no seu caso) através de um túnel TLS criptografado.

- Sistema de Arquivos em RAM (tmpfs): O Swarm monta um "disco virtual" na memória RAM do container em `/run/secrets/<nome_do_segredo>`.

- Volatilidade: O segredo nunca toca o disco rígido (HDD/SSD) do node onde o container está rodando. Se você desligar a máquina da tomada, o segredo "some" da memória daquele container. Ele só existe enquanto o processo do container estiver vivo.

### O que acontece se o Swarm Manager cair?
Aqui entra a importância do Raft Log (o banco de dados do Swarm).

- Se o serviço do Docker cair e voltar: O Manager tem o segredo salvo em um banco de dados interno no disco (criptografado em repouso por padrão no Docker moderno). Ao reiniciar, ele descriptografa o banco e "entrega" o segredo novamente para os serviços.

- Se o Node explodir (perda total de disco): Sim, você perde o segredo. Como ele foi criado via CLI (`echo ... | docker secret create`), ele reside apenas no banco de dados do Swarm. Se você não tiver o comando original ou um backup do diretório `/var/lib/docker/swarm`, terá que recriá-lo.

### E se você sair do Swarm Mode (`docker swarm leave --force`)?
Todas as chaves morrem. Ao executar esse comando, você está instruindo o Docker a apagar toda a estrutura de orquestração daquela máquina, incluindo:

1. O banco de dados Raft (onde os segredos estão guardados).

2. Os certificados de criptografia.

3. As definições de rede.

**Conclusão**: Para um ambiente produtivo, você deve ter os seus arquivos de definição (YAML) e os valores dos segredos guardados em um cofre externo (como o Hashicorp Vault ou outro) ou em um gerenciador de configurações. O Swarm é o veículo de entrega e armazenamento temporário, não o lugar de "backup definitivo" dos seus segredos.

## Step #4
Chegou o momento de consolidar tudo o que preparamos até aqui: a Rede, o Volume e o Segredo. No Docker Swarm, não usamos o comando `docker-compose up`. Utilizamos o conceito de Stack.

Uma Stack (pilha) é um conjunto de serviços interdependente que compartilha o mesmo ciclo de vida. Enquanto o `docker-compose` gerencia containers isolados, a Stack gerencia **Serviços** no orquestrador.

A grande diferença é a seção `deploy. É nela que dizemos ao Swarm quantas réplicas queremos e como ele deve se comportar se um container falhar.

### Ação: Criando o arquivo de configuração
Crie um arquivo chamado `stack.yml`:
```
version: '3.8'

services:
  # O Banco de Dados
  db:
    image: postgres:alpine
    networks:
      - app_network
    volumes:
      - pg_data:/var/lib/postgresql/data
    environment:
      # Aqui dizemos ao Postgres para ler a senha do ARQUIVO em RAM
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

  # A API (usaremos uma imagem leve para teste)
  api:
    image: hashicorp/http-echo
    command: ["-text", "API respondendo via Swarm"]
    networks:
      - app_network
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure

  # O Nginx (Proxy Reverso)
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - app_network
    # O Nginx aqui simularia o roteamento para a API
    deploy:
      placement:
        constraints: [node.role == manager]

networks:
  app_network:
    external: true

volumes:
  pg_data:
    external: true

secrets:
  db_password:
    external: true
```
**Nota**:


`constraints: [node.role == manager]` (Nginx): Como só temos um node, essa regra é indiferente (o único node já é manager). Em clusters multi-node, usamos isso para garantir que o Nginx rode apenas em máquinas com IP público conhecido.

**Nota**:

Nunca use replicas: > 1 para bancos de dados SQL no Swarm sem uma estratégia de clusterização específica da aplicação (como o Repmgr ou Patroni). Para o nosso tutorial, mantenha o Postgres com 1 réplica.

É fundamental distinguir disponibilidade de container de replicação de dados:

**- Réplicas do Swarm**: São cópias idênticas do processo. Útil para APIs (que são stateless ou "sem estado"), onde tanto faz qual réplica responde.

**- Cluster de Banco de Dados (Stateful)**: Exige uma arquitetura de Primary/Replica (ou Master/Slave). Para ter um cluster real, você precisaria de:

1. Uma instância de Escrita (Primary).

2. Uma ou mais instâncias de Leitura (Replica), que recebem cópias dos dados via rede.

3. Um software de gerenciamento (como o Patroni ou Stolon) para orquestrar quem é o mestre

### **Executando o Deploy**

Para subir toda a sua infraestrutura de uma vez, execute:

`docker stack deploy -c stack.yml meu_projeto`

O que observar no terminal?
**1. Creating service meu_projeto_db**: O Swarm cria a definição do serviço.

**2. Creating service meu_projeto_api**: Note que ele criará 2 instâncias (réplicas) conforme solicitado no YAML.

**3. Creating service meu_projeto_nginx**: Ele mapeia a porta 80 do seu Ubuntu para o container.

**Por que declaramos as redes/volumes como `external: true`?**

Recomendo essa prática para evitar que, ao remover a Stack, o Swarm apague acidentalmente seu volume de dados ou sua rede principal. Nós os criamos manualmente nos passos anteriores para garantir o controle total (perfeito para auditoria de recursos).

**Verificação de Estado**

Para ver se o orquestrador conseguiu subir tudo (especialmente o Postgres com o segredo), use:

`docker stack services meu_projeto`

Você verá uma coluna `REPLICAS`. Se estiver `1/1` ou `2/2`, o Swarm teve sucesso em alocar os recursos.

**Conceito chave**: A Stack é o seu "contrato" com o Swarm. Você descreve o estado desejado, e ele trabalha para manter esse estado vivo.

## Step #5
Vamos testar a resiliência do cluster. Na arquitetura que estamos usando, o Nginx está funcionando como proxy reverso e deve ser a única "cara" da aplicação para o mundo externo, protegendo a API e o Banco de Dados dentro da rede isolada.

No Docker Swarm, o Nginx usará o DNS interno para encontrar as réplicas da API.

### Proxy Reverso e Escalabilidade
Para este passo, precisamos de duas ações:

1. Criar uma configuração para o Nginx.

2. Atualizar a Stack para aplicar essa configuração.

**Criando a Configuração do Nginx**

Crie um arquivo chamado `nginx.conf` na mesma pasta do `stack.yml`:

```
events {}

http{
    upstream minha_api {
        # O Swarm resolve 'api' para os IPs das réplicas automaticamente
        server api:3000;
    }

    server {
        listen 80;
        location / {
            proxy_pass http://minha_api;
        }
    }
}
```
**Nota**: O nome `api` no `server` do bloco `upstream` é o nome do serviço que definimos no YAML. O Swarm faz o Load Balancing nativo entre as réplicas da API.

### Atualizando o `stack.yml`

Vamos usar o conceito de **Configs** do Swarm. Assim como os *Secrets*, as *Configs* são objetos de primeira classe no Swarm para arquivos não sensíveis.

Atualize seu `stack.yml` para incluir o arquivo de configuração:

```
services:
  # O Nginx (Proxy Reverso)
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"      
    networks:
      - app_network
    # Inclusão das Configs do Nginx
    configs:
      - source: nginx_config
        target: /etc/nginx/nginx.conf
    # O Nginx aqui simularia o roteamento para a API
    deploy:
      placement:
        constraints: [node.role == manager]
    

  # A API (usaremos uma imagem leve para teste)
  api:
    image: hashicorp/http-echo
    command: ["-listen=:3000", "-text", "Resposta da Réplica da API"]
    networks:
      - app_network
    deploy:
      replicas: 3

  # O Banco de Dados
  db:
    image: postgres:alpine
    environment:
      POSTGRES_DB: mydatabase
      POSTGRES_USER: user
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - app_network
    secrets:
      - db_password

volumes:
  pg_data:
    external: true

networks:
  app_network:
    external: true

secrets:
  db_password:
    external: true

configs:
  nginx_config:
    file: ./nginx.conf
```

### Aplicando a atualização
Execute o comando de deploy novamente. O Swarm é inteligente: ele não destrói tudo, ele apenas aplica o Diff (as diferenças).

`docker stack deploy -c stack.yml meu_projeto`

**Nota**: O Docker Swarm trata `configs` e `secrets` como imutáveis. Se você apenas alterar o arquivo `nginx.conf` no disco e rodar o `stack deploy`, o Swarm não atualizará o serviço porque o nome da config (`nginx_config) ainda é o mesmo.

**Para forçar a atualização, temos duas opções**:

1. Remover a stack e subir de novo (mais rápido para este tutorial):

```
docker stack rm meu_projeto
# Aguarde alguns segundos para a limpeza
docker stack deploy -c stack.yml meu_projeto
```

2. Ou versionar a config no YAML (ex: nginx_config_v2).
```
configs:
  nginx_config_v2: # Novo nome aqui também
    file: ./nginx.conf
```
**O que o Swarm fará:**
- Ele detecta que nginx_config_v2 é um objeto novo.
- Ele cria essa config no banco de dados Raft.
- Ele inicia um **Rolling Update** no serviço Nginx: cria um novo container com a nova config e, só se o novo subir com sucesso, ele mata o antigo.

Se simplesmente deletassemos a stack e subisse de novo, perderíamos o histórico do que estava rodando. Versionando as configs:

**1. Você mantém a rastreabilidade**: sabe exatamente qual versão da configuração estava ativa em determinado horário.

**2. Zero Downtime**: O serviço nunca para de responder durante a atualização.

**3. Rollback**: Se a versão v2 estiver errada, você altera o YAML de volta para v1 e o Swarm reverte o ambiente instantaneamente para o estado anterior conhecido e estável.

**Conceito chave**: Auditoria no Swarm é feita via service logs e inspect de tasks. Mudanças de configuração exigem novos nomes para garantir a imutabilidade e a segurança do deploy.

### O Teste de Resiliência (A Magia do Swarm)

Agora que temos 3 réplicas da API atrás do Nginx, vamos fazer o seguinte:

**1. Acesse no navegador**: `http://127.0.0.1. Você verá a mensagem da API.

**2. Verifique as tarefas**: 

`docker service ps meu_projeto_api `

Veremos 3 "Tasks" Rodando

**Nota**: Se quiser listar todos os services ativos no node rode o comando `docker service ls`. Para listar as stacks ativas no seu Swarm use `dokcer stack ls

Para remover um service use o comando `docker service rm <nome_do_service>`.

Para remover uma stack inteira use `docker stack rm <nome_da_stack>`

**3. Simule uma falha**:

Liste os containers com o comando:

`docker ps

Pegue o ID de um dos containers da API com `docker ps e force a remoção dele:

`docker rm -f <ID_DO_CONTAINER_API>`

**4. Observe**: Rode `docker service ps meu_projeto_api` novamente de imediato. Você verá que o Swarm percebeu que o estado atual (2 réplicas) não condiz com o desejado (3 réplicas) e subiu um novo container automaticamente.
Você verá a Task antiga com o status Shutdown ou Failed. Verá uma nova Task com status Running.

**Conceito chave**: O Swarm atua como um "vigilante". Você define o contrato (3 réplicas + config do Nginx) e ele garante a autorrecuperação (Self-healing).

### Visualização de Logs
No Swarm, você não precisa caçar o container individual para ver os logs. Você audita o Serviço.

**Ver os logs do serviço (Agregado)**
Este comando agrupa os logs de todas as réplicas (as 3 instâncias da API) em um único fluxo, identificando cada uma:

`docker service logs -f meu_projeto_api`

**Investigando a "Queda"**

Para entender por que um container morreu ou por que o Swarm decidiu subir outro, o comando mais importante é o `inspect focado na Task:

1. Liste as tasks (incluindo as que falharam):
   `docker service ps meu_projeto_api`

2. Inspecione a task que deu erro: Pegue o ID da task que aparece como `Shutdown` ou `Failed`e rode:
`docker inspect <ID_DA_TASK>`. Procure a seção "Status". Lá você encontrará o ContainerStatus, o ExitCode e o Error. Se o container foi morto manualmente (como fizemos com docker rm), o Swarm registrará isso.

**Conceito chave**: Auditoria no Swarm é feita via service logs e inspect de tasks. 

# Tutorial 2 - Entendendo Builds De Imagens Docker com Swarm

O ponto cego de muitos profissionais ao migrar para o Swarm é: "O Swarm não possui o comando build". No docker-compose, o Docker faz o build e sobe o container em um único fluxo. No Swarm, o Build e o Deploy são etapas totalmente separadas. Vamos simular uma pequena API em Node.js.

### Crie uma pasta para este tutorial
Crie uma pasta para esse tutorial e entre nela
```
mkdir tutorial-build && cd tutorial-build
mkdir api
```

Agora crie um arquivo chamado api/Dockerfile

```
FROM alpine
# Apenas um comando que fica rodando e imprimindo a versão
CMD ["sh", "-c", "echo 'API Versão 1.0 rodando' && sleep 3600"]
```
**O Build Manual (Obrigatório no Swarm)**


Como o Swarm não sabe ler a instrução `build`:, precisamos "preparar o estoque" de imagens antes de pedir para o orquestrador trabalhar.

Execute o build dando um nome (tag) claro para sua imagem:

`docker build -t minha-api-local:v1 ./api`

Quando você rodar a Stack, o Swarm vai procurar no seu Docker Engine uma imagem chamada `minha-api-local:v1`. Se ela não existir, ele tentará baixar do Docker Hub e falhará.

**O arquivo da Stack (`stack.yml`)**


Agora, crie o arquivo de deploy. Note que não existe a cláusula build.

**O Deploy e a Falha Comum**

Tente rodar:

`docker stack deploy -c stack.yml tutorial_v2`

Rode `docker service ps tutorial_v2_web`.

**O que observar**: Como estamos em um node único, o Swarm vai encontrar a imagem `minha-api-local:v1` que você acabou de buildar e vai subir os containers com sucesso.

**O Problema do "Update" (O Racional)**


Imagine que você alterou o código da sua API. No `docker-compose`, você faria `docker-compose up --build`. No Swarm, o fluxo é:

Alterar o Dockerfile (mude o texto de 'Versão 1.0' para 'Versão 2.0').

Novo Build: `docker build -t minha-api-local:v2 ./api` (sempre mude a tag!).

Atualizar o Serviço:

`docker service update --image minha-api-local:v2 tutorial_v2_web`

**Nota**: Evite a tag `:latest`. Se você buildar uma nova imagem com o mesmo nome `minha-api-local:latest`, o Swarm pode achar que a imagem é a mesma que ele já tem e não atualizar o container. Usar versões (v1, v2) garante a imutabilidade e facilita o rollback.

**Conceito chave**: 
- No Compose: `Código` -> `Up` (Build automático).

- No Swarm: `Código` -> `Build Manual/pipeline` -> `Stack Deploy/Service Update`.

# Tutorial 3 - Transformando seu Manager em um Registry Privado

Neste cenário, o Registry será um Serviço do Swarm. Isso é elegante porque, se o Registry cair, o próprio Swarm o reinicia.

## Passo 1: Criar o Serviço de Registry
Diferente de um container comum, vamos publicar o Registry como um serviço global no cluster. Execute:

```
docker service create --name registry-local \
--publish published=5000,target=5000 --mount \ type=volume, source=registry_data,target=/var/lib/registry registry:2
```

**Nota**: Por que o volume? Se o serviço reiniciar, não vamos perder as imagens já armazenadas. O registry_data garante a persistência.

## Passo 2: Ajustar o HTTPS
Por padrão, o Docker exige HTTPS para se comunicar com um Registry. Como estamos em ambiente local (`127.0.0.1` ou IP da rede local), precisamos avisar ao Docker que ele pode confiar no nosso Registry sem SSL (por enquanto).

1. Edite o arquivo de configuração do Docker:

`sudo nano /etc/docker/daemon.json`

2. Adicione (ou crie) o seguinte conteúdo (substitua pelo seu IP se for usar outros nodes):

```
{
  "insecure-registries" : ["127.0.0.1:5000"]
}
```
3. Reinicie o Docker para aplicar:

`sudo systemctl restart docker`

## Passo 3: O Workflow de "Tag e Push"
Agora o fluxo de trabalho muda. Você não apenas builda; você carimba e envia.

**1. Build:**

`docker build -t minha-api:v1 ./api`

**2. Tag**

Precisamos dizer ao Docker que essa imagem pertence ao nosso Registry local.

`docker tag minha-api:v1 127.0.0.1:5000/minha-api:v1`

**3. Push**

`docker push 127.0.0.1:5000/minha-api:v1`


## Passo 4: Atualizar o seu `stack.yml`

Agora, no arquivo de deploy, a imagem não será mais o nome local, mas o endereço completo do Registry:

```
services:
  api:
    image: 127.0.0.1:5000/minha-api:v1
    deploy:
      replicas: 3

```

## Passo 5: Deploy com Autenticação

Ao rodar o deploy, usamos a flag que garante que todos os nodes (mesmo que só tenhamos um agora) saibam como buscar a imagem:

`docker stack deploy -c stack.yml meu_projeto --with-registry-auth`

Ao usar este método, ganhamos:

- Rastreabilidade: Você consegue listar o que está no seu registry (curl http://127.0.0.1:5000/v2/_catalog).

- Padronização: O mesmo comando que você usa no Ubuntu (Minha Máquina) funcionará na AWS ou Azure, mudando apenas o endereço do Registry.

- Isolamento: Se a internet cair, seu deploy continua funcionando porque as imagens estão "dentro de casa".