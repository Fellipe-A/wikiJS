# WikiJS

Esse documento tem o objetivo de demonstrar o passo a passo para a criação de um *Dockerfile* para a criação de uma imagem, e o *deployment* do *WikiJS*, um projeto *open-source* de *Wiki* escrito em *Javascript*. 

## Repositório
Estaremos utilizando o [repositório](https://github.com/Requarks/wiki) oficial do projeto. Nesse repositório encontramos vários arquivos com diferentes finalidades, tais como: Garantir a segurança da imagem, gerenciamento de *pipelines*, dependências da imagem, licença e etc. Para obter o repositório, utilizaremos o git abaixo.

    git clone https://github.com/Requarks/wiki.git

## Dockerfile
O arquivo que focaremos nesse documento será o *Dockerfile*, responsável pela possibilidade da criação da imagem em qualquer ambiente, tornando a aplicação bastante portátil. 
O documento responsável pela criação da imagem para ambientes em produção encontra-se situa-se em /wiki/dev/build/Dockerfile. No mesmo diretório encontramos também o *config.yml*, necessário para descrever as configurações básicas da aplicação, como os parâmetros do banco de dados, configurações de encriptação e etc.
O arquivo oficial utiliza a criação de imagens em multi-estágios. Técnica utilizada para a otimização de *Dockerfiles*, mantendo um arquivo de fácil leitura e manutenção.  Além disso, ocorre uma maximização do *cache* das camadas da imagem.
 
    # ====================
    
    # --- Build Assets ---
    
    # ====================
    
    FROM node:14-alpine AS assets
    
    RUN apk add yarn g++ make python --no-cache
    
    WORKDIR /wiki
    
    COPY ./client ./client
    
    COPY ./dev ./dev
    
    COPY ./package.json ./package.json
    
    COPY ./.babelrc ./.babelrc
    
    COPY ./.eslintignore ./.eslintignore
    
    COPY ./.eslintrc.yml ./.eslintrc.yml
    
    RUN yarn cache clean
    
    RUN yarn --frozen-lockfile --non-interactive
    
    RUN yarn build
    
    RUN rm -rf /wiki/node_modules
    
    RUN yarn --production --frozen-lockfile --non-interactive
    
    # ===============
    
    # --- Release ---
    
    # ===============
    
    FROM node:14-alpine
    
    LABEL maintainer="requarks.io"
    
    RUN apk add bash curl git openssh gnupg sqlite --no-cache && \
    
    mkdir -p /wiki && \
    
    mkdir -p /logs && \
    
    mkdir -p /wiki/data/content && \
    
    chown -R node:node /wiki /logs
    
    WORKDIR /wiki
    
    COPY --chown=node:node --from=assets /wiki/assets ./assets
    
    COPY --chown=node:node --from=assets /wiki/node_modules ./node_modules
    
    COPY --chown=node:node ./server ./server
    
    COPY --chown=node:node --from=assets /wiki/server/views ./server/views
    
    COPY --chown=node:node ./dev/build/config.yml ./config.yml
    
    COPY --chown=node:node ./package.json ./package.json
    
    COPY --chown=node:node ./LICENSE ./LICENSE
    
    USER node
    
    VOLUME ["/wiki/data/content"]
    
    EXPOSE 3000
    
    EXPOSE 3443
    
    # HEALTHCHECK --interval=30s --timeout=30s --start-period=30s --retries=3 CMD curl -f http://localhost:3000/healthz
    
    CMD ["node", "server"]


### *Build Assets*
A primeira etapa, descrita como *Build assets* é responsável pela instalação dos requisitos básicos. Primeiramente, a partir da imagem do Node criada "em cima" da distribuição Alpine (recomendado pela remoção de arquivos não necessários, acarretando numa redução do tamanho da imagem e menor brecha para falhas, tendo em vista que apenas os pacotes essenciais estão presentes), instalamos os pacotes básicos para o funcionamento da aplicação. Em seguida os arquivos da aplicação são copiados para dentro da imagem de criação do container. 

    FROM node:14-alpine AS assets
    
    RUN apk add yarn g++ make python --no-cache
    
    WORKDIR /wiki
    
    COPY ./client ./client
    
    COPY ./dev ./dev
    
    COPY ./package.json ./package.json
    
    COPY ./.babelrc ./.babelrc
    
    COPY ./.eslintignore ./.eslintignore
    
    COPY ./.eslintrc.yml ./.eslintrc.yml

Por fim, por meio do yarn (gerenciador de pacotes Javascript mais eficiente que o npm) é realizado a instalação das dependências e o *build* da aplicação.   

    RUN yarn cache clean
    
    RUN yarn --frozen-lockfile --non-interactive
    
    RUN yarn build
    
    RUN rm -rf /wiki/node_modules
    
    RUN yarn --production --frozen-lockfile --non-interactive

### Release

A segunda etapa da criação da imagem é a criação dos diretórios que não serão estáticos, como os diretórios de *logs* e conteúdos. 

    FROM node:14-alpine
    
    LABEL maintainer="requarks.io"
    
    RUN apk add bash curl git openssh gnupg sqlite --no-cache && \
    
    mkdir -p /wiki && \
    
    mkdir -p /logs && \
    
    mkdir -p /wiki/data/content && \


Em seguida,  definimos o diretório onde os comando serão aplicados dentro do *container*, e então é realizada a alteração do proprietário do arquivo para node.  Na sequência declaramos o usuário que será o responsável pelo processo. Tal mudança de extrema importância no que tange à segurança, fazendo com que o processo não funcione com os privilégios de *root*. 


    chown -R node:node /wiki /logs
    
    WORKDIR /wiki
    
    COPY --chown=node:node --from=assets /wiki/assets ./assets
    
    COPY --chown=node:node --from=assets /wiki/node_modules ./node_modules
    
    COPY --chown=node:node ./server ./server
    
    COPY --chown=node:node --from=assets /wiki/server/views ./server/views
    
    COPY --chown=node:node ./dev/build/config.yml ./config.yml
    
    COPY --chown=node:node ./package.json ./package.json
    
    COPY --chown=node:node ./LICENSE ./LICENSE
    
    USER node
    
Então, declaramos o volume a ser montado no *host* e persistido, para que seja possível utilizar em outros *containers*, e definimos as portas à serem escutadas pelo *container*. Por fim, há um comando para checar a saúde da aplicação e então é definido o comando padrão a ser utilizado na inicialização do *container*.

    VOLUME ["/wiki/data/content"]
    
    EXPOSE 3000
    
    EXPOSE 3443
    
    # HEALTHCHECK --interval=30s --timeout=30s --start-period=30s --retries=3 CMD curl -f http://localhost:3000/healthz
    
    CMD ["node", "server"]




## Construindo a imagem

Para construir a imagem, utilizamos o  *docker build* no diretório raiz do projeto "/wiki".

    docker build -t wiki -f /dev/build/Dockerfile .
Após execução do comando acima, o processo de criação da imagem será iniciado. (Em um dos processos do *build*, me surgiu dúvida a respeito das mensagens de warning). A imagem será criada com a tag "wiki".

Como boa prática, realizamos o um escaneamento de vulnerabilidades na imagem. Para tal, uma ferramenta nativa do Docker será utilizada.

    docker scan "image-name"

![Docker scan](https://i.ibb.co/r5F18bW/docker-scan.png)

Em nosso caso, apesar dos warnings apresentados durante a criação da imagem, nenhuma vulnerabilidade foi encontrada na imagem.
