# Docker com Nginx/Angular/Node/Websphere Liberty

A demonstração a seguir segue o diagrama abaixo:

![](http://www.corretoporconstrucao.com.br/images/diagramarede.png)

Teremos um NGINX funcionando como porta de entrada para os acessos externos (porta 80) que irá fornecer aplicações Angular e rotear as chamadas para os aplicativos Node, os aplicativos Node, acionam outra rede de microserviços (em Java no Websphere Liberty) que contém novamente um Nginx como ponto de acesso, esses aplicações no Liberty acessam as bases de dados/mainframe e retornam as informações para o NodeJs que por sua vez devolve para as aplicações Angular rodando no browser do cliente.
Isso é apenas uma demonstração de como utilizar Docker e o Docker Compose para montar em pouco tempo uma estrutura complexa e com diversas camadas de rede.
A camada do Node foi dividida em 4 servidores, sendo 2 pares (espelhos), ou seja, a mesma aplicação Node está no par 1 ou no par 2, idéia semelhante ao a camada do Liberty.
  
## Observação

O arquivo docker-compose.yml completo esta dentro do projeto, os passos com os detalhes de como foi construída a solução estão nos próximos itens, mas quem quiser apenas executar passe diretamente para o item "Docker Compose".

Basicamente criei uma aplicação Angular que possui um único ponto de acesso via Nginx na porta 70, que aciona um serviço REST em uma aplicação Node.js e essa, realiza a busca em uma aplicação Java, passando por outro Nginx, rodando num Websphere Liberty e exibe a lista de livros recebida. A estrutura está com load balance e todos os passos estão descritos aqui juntamente com as fotos de demonstração. 

## Preparando a rede 

No Docker para conseguirmos comunicar entre os containeres com os nomes e não IPs, precisamos criar uma rede

```
docker network create --driver bridge devsecops
```

E quando adicionarmos os containeres, iremos passar como argumento a rede devsecops.

```
docker network list
```

Lista todas as redes criadas.

## Baixando as imagens necessárias

Para iniciarmos, vamos baixar a imagem do Nginx:

```
docker pull nginx
```
O comando pull somente realiza o download da imagem, no caso, realizamos o download da versão mais recente do Nginx.

Vamos configurar esse Nginx e iremos chamá-lo de nginx-border, dentro da pasta nginx-border temos o arquivo nginx.conf com as configurações necessárias para iniciar o servidor.

```
docker run  -d -p 70:80 --name=nginxborder -v "/DADOS/AZTEK/workspaceMicro/DockerNginxNodeLiberty/nginxborder/nginx.conf:/etc/nginx/nginx.conf" -v "/DADOS/AZTEK/workspaceMicro/DockerNginxNodeLiberty/public/:/var/www/public" --network devsecops nginx
```

Estamos apontanto a configuração e a pasta public externamente ao container, apontando para a estrutura de pastas que temos no projeto.
Dessa forma, qualquer aplicativo Angular inserido na pasta local '/DADOS/AZTEK/workspaceMicro/DockerNginxNodeLiberty/public/' já será automcaticamente disponibilizado pelo Nginx para acesso na porta 70 (abrir o browser e digitar http://localhost:70/)

Passamos para a configuração do Node, primeiro executamos:

```
docker pull node
```
O comando:
```
docker images
```
Lista todas as imagens que já foram baixadas.


Na pasta do projeto do Node vamos criar o arquivo  

```
FROM node:latest
MAINTAINER SuvacoDeCobra
ENV PORT=3000
COPY . /var/www
WORKDIR /var/www
RUN npm install
ENTRYPOINT npm start
EXPOSE 3000
```
Que esta definindo como iremos criar a imagem do container Docker para esta aplicação, a base para esta imagem é o último node, o MAINTAINER é que criou a imagem, ENV conseguimos configurar variáveis de ambiente, o COPY copia todos os arquivos do . onde está o arquivo Dockerfile para dentro da pasta /var/www do container, o WORKDIR define qual o diretório que o container irá utilizar para iniciar, o RUN para executar a instalação das dependências, o ENTRYPOINT é executado quando o container estiver pronto e o EXPOSE define a porta que será liberada para o acesso ao container.

Para gerarmos a imagem do container executamos na pasta do projeto node-rest-api:

```
docker build . -t "suvacodecobra/node-rest-api"
```

Se agora pedir a listagem de imagens, irá aparecer a nossa imagem recém-criada.


```
docker run -d --name=nodegrupo1_node1 --network devsecops suvacodecobra/node-rest-api
docker run -d --name=nodegrupo1_node2 --network devsecops suvacodecobra/node-rest-api
```

Criamos dois dockers com a mesma imagem, com os nomes que configuramos no Nginx para realizar o balanceamento corretamente.


Agora vamos subir os containeres do Liberty:

```
docker run -d --name=libertygrupo1_websphereliberty1 -v /DADOS/AZTEK/workspaceMicro/DockerNginxNodeLiberty/libertygrupo1/dropins:/config/dropins --network devsecops websphere-liberty
docker run -d --name=libertygrupo1_websphereliberty2 -v /DADOS/AZTEK/workspaceMicro/DockerNginxNodeLiberty/libertygrupo1/dropins:/config/dropins --network devsecops websphere-liberty
```

Agora precisamos subir o nosso segundo Nginx que vai rotear para os containers do Liberty

```
docker run  -d --name=nginxrouter -v "/DADOS/AZTEK/workspaceMicro/DockerNginxNodeLiberty/nginxrouter/nginx.conf:/etc/nginx/nginx.conf" --network devsecops nginx
```

Comando para visualizar os logs dos containeres quando a coisa enrosca:

```
dcoker logs <NOME_ID_CONTAINER>
```

E quando o fogo está comendo tudo, sempre podemos entrar no container:

```
docker exec -it <NOME_CONTAINER> /bin/bash
```

Para iniciar o container quando parado:

```
docker start <NOME_CONTAINER>
```



## Docker Compose

Claro que fazer todos esses passos na mão é complicado, mas vamos criar um arquivo docker compose e a partir desse arquivo montaremos toda a estrutura em poucos passos:

Com o arquivo docker-compose.yml na raiz do projeto temos a construção de todos o ambiente:

```
version: '2'
services:
    nginxborder:
        image: nginx
        container_name: nginxborder
        volumes:
            - ./nginxborder/nginx.conf:/etc/nginx/nginx.conf
            - ./public/:/var/www/public/
        ports:
            - "70:80"
        networks: 
            - devsecops
        depends_on: 
            - "nodegrupo1_node1"
            - "nodegrupo1_node2"

    nginxrouter:
        image: nginx
        container_name: nginxrouter
        volumes:
            - ./nginxrouter/nginx.conf:/etc/nginx/nginx.conf
        networks: 
            - devsecops
        ports:
            - "80"
        depends_on: 
            - "libertygrupo1_websphereliberty1"
            - "libertygrupo1_websphereliberty2"

    nodegrupo1_node1:
        build:
            dockerfile: ./apps/node/node-rest-api/nodegrupo1.dockerfile
            context: ./apps/node/node-rest-api
        image: suvacodecobra/node-rest-api
        container_name: nodegrupo1_node1
        ports:
            - "3000"
        networks: 
            - devsecops

    nodegrupo1_node2:
        build:
            dockerfile: ./apps/node/node-rest-api/nodegrupo1.dockerfile
            context: ./apps/node/node-rest-api
        image: suvacodecobra/node-rest-api
        container_name: nodegrupo1_node2
        ports:
            - "3000"
        networks: 
            - devsecops

    libertygrupo1_websphereliberty1:
        image: websphere-liberty
        container_name: libertygrupo1_websphereliberty1
        volumes:
            - ./libertygrupo1/dropins:/config/dropins 
        ports:
            - "9080"
        networks: 
            - devsecops

    libertygrupo1_websphereliberty2:
        image: websphere-liberty
        container_name: libertygrupo1_websphereliberty2
        volumes:
            - ./libertygrupo1/dropins:/config/dropins 
        ports:
            - "9080"
        networks: 
            - devsecops

networks: 
    devsecops:
        driver: bridge
```

Para executar basta rodar:


```
docker-compose up
```

Na primeira execução irá demorar um pouco, mas a partir da segunda execução em segundos o ambiente está todo montado e pronto para ser utilizado. Para desligar:


```
docker-compose down
```



## Descrição da estrutura das pastas

```
apps -> pasta com os fontes dos aplicativos
	angular
	java 
	node
libertygrupo1/dropins -> dropins do libertygrupo1, podem ser adicionadas mais aplicações ao Liberty, basta jogar nessa pasta
libertygrupo2/dropins -> dropins do libertygrupo2, podem ser adicionadas mais aplicações ao Liberty, basta jogar nessa pasta
nginxborder -> configuração do nginx para o nginxborder
nginxrouter -> configuração do nginx para o nginxrouter
public -> pasta onde devemos inserir as aplicações Angular compiladas, podem ser inseridas mais aplicações com o container rodando
docker-compose.yml -> arquivo que monta TODO esse ambiente
```

## Funcionando  :) 

Iniciando o docker compose:

![](http://www.corretoporconstrucao.com.br/images/subindoDockerCompose.png)

Exibindo os containeres rodando:

![](http://www.corretoporconstrucao.com.br/images/containersRodando.png)

Nginx listando as aplicações Angular instaladas:

![](http://www.corretoporconstrucao.com.br/images/browserNginxListagemPublic.png)

Acionando a aplicação:

![](http://www.corretoporconstrucao.com.br/images/AngularEntrada.png)

Obtendo as mensagens do Java (caminho longo até o sertão....)

![](http://www.corretoporconstrucao.com.br/images/mensagemJavaNodeAngular.png)

Fiz duas vezes e o loadBalance jogou cada uma das chamadas em um container:

![](http://www.corretoporconstrucao.com.br/images/LoadBalance.png)



## Próximos Passos

Essa foi apenas uma pequena amostra, preciso rebuscar melhor a solução, a forma de localizar os microserviços, log em uma stack ELK, mensageria com o Kafka, mas a idéia era demonstrar a utilização do Docker para a construção de um ambiente com linguagens heterogêneas e a comunicação entre eles.

