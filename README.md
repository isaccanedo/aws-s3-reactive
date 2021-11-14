# AWS S3 com Java - Suporte reativo


# 1. Introdução
A AWS oferece muitos serviços por meio de suas muitas APIs que podemos acessar de Java usando seu SDK oficial. Até recentemente, porém, este SDK não oferecia suporte para operações reativas e tinha apenas suporte limitado para acesso assíncrono.

Com o lançamento do SDK da AWS para Java 2.0, agora podemos usar essas APIs no modo de I/O totalmente sem bloqueio, graças à adoção do padrão Reactive Streams.

Neste tutorial, vamos explorar esses novos recursos implementando uma API REST de armazenamento de blob simples no Spring Boot que usa o conhecido serviço S3 como back-end de armazenamento.

# 2. Visão geral das operações AWS S3
Antes de mergulhar na implementação, vamos fazer uma rápida visão geral do que queremos alcançar aqui. Um serviço de armazenamento de blob típico expõe operações CRUD que um aplicativo front-end consome para permitir que um usuário final carregue, liste, baixe e exclua vários tipos de conteúdo, como áudio, imagens e documentos.

Um problema comum com o qual as implementações tradicionais devem lidar é como lidar com arquivos grandes ou conexões lentas com eficiência. Nas primeiras versões (pré-servlet 3.0), tudo que a especificação JavaEE tinha a oferecer era uma API de bloqueio, portanto, precisávamos de um encadeamento para cada cliente de armazenamento de blob simultâneo. Este modelo tem a desvantagem de exigir mais recursos do servidor (ou seja, máquinas maiores) e torná-los mais vulneráveis ​​a ataques do tipo DoS:

<img src="rective-upload.png">

<img src="thread-per-client.png">

Usando uma pilha reativa, podemos tornar nosso serviço muito menos intensivo em recursos para o mesmo número de clientes. A implementação do reator usa um pequeno número de threads que são despachados em resposta a eventos de conclusão de I/O, como a disponibilidade de novos dados para ler ou a conclusão de uma gravação anterior.

Isso significa que o mesmo thread continua tratando desses eventos - que podem se originar de qualquer uma das conexões de cliente ativas - até que não haja mais trabalho disponível para fazer. Esta abordagem reduz muito o número de mudanças de contexto - uma operação bastante cara - e permite o uso muito eficiente dos recursos disponíveis:

<img src="rective-upload.png">

# 3. Configuração do projeto
Nosso projeto de demonstração é um aplicativo Spring Boot WebFlux padrão que inclui as dependências de suporte usuais, como Lombok e JUnit.

Além dessas bibliotecas, precisamos trazer as dependências do SDK da AWS para Java V2:

```
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>software.amazon.awssdk</groupId>
            <artifactId>bom</artifactId>
            <version>2.10.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>s3</artifactId>
        <scope>compile</scope>
    </dependency>

    <dependency>
        <artifactId>netty-nio-client</artifactId>
        <groupId>software.amazon.awssdk</groupId>
        <scope>compile</scope>
    </dependency>
</dependencies>
```

O SDK da AWS fornece um BOM que define as versões necessárias para todas as dependências, portanto, não precisamos especificá-las na seção de dependências de nosso arquivo POM.

Adicionamos a biblioteca cliente S3, que trará outras dependências centrais do SDK. Também precisamos da biblioteca cliente Netty, necessária já que usaremos APIs assíncronas para interagir com a AWS.

A documentação oficial da AWS contém mais detalhes sobre os transportes disponíveis.

# 4. Criação de cliente AWS S3
O ponto de entrada para operações S3 é a classe S3AsyncClient, que usaremos para iniciar novas chamadas de API.

Como precisamos apenas de uma única instância dessa classe, vamos criar uma classe @Configuration com um método @Bean que a constrói, para que possamos injetá-la onde precisarmos:

```
@Configuration
@EnableConfigurationProperties(S3ClientConfigurarionProperties.class)
public class S3ClientConfiguration {
    @Bean
    public S3AsyncClient s3client(S3ClientConfigurarionProperties s3props, 
      AwsCredentialsProvider credentialsProvider) {
        SdkAsyncHttpClient httpClient = NettyNioAsyncHttpClient.builder()
          .writeTimeout(Duration.ZERO)
          .maxConcurrency(64)
          .build();
        S3Configuration serviceConfiguration = S3Configuration.builder()
          .checksumValidationEnabled(false)
          .chunkedEncodingEnabled(true)
          .build();
        S3AsyncClientBuilder b = S3AsyncClient.builder().httpClient(httpClient)
          .region(s3props.getRegion())
          .credentialsProvider(credentialsProvider)
          .serviceConfiguration(serviceConfiguration);

        if (s3props.getEndpoint() != null) {
            b = b.endpointOverride(s3props.getEndpoint());
        }
        return b.build();
    }
}
```

Para esta demonstração, estamos usando uma classe @ConfigurationProperties mínima (disponível em nosso repositório) que contém as seguintes informações necessárias para acessar os serviços S3:

- Região: um identificador de região AWS válido, como us-east-1;
- AccessKeyId/secretAccessKey: Nossa chave e identificador de API da AWS;
- Endpoint: Um URI opcional que podemos usar para substituir o endpoint de serviço padrão do S3. O principal caso de uso é usar o código demo com soluções alternativas de armazenamento que oferecem uma API compatível com S3 (minio e localstack são exemplos);
- Bucket: nome do bucket onde armazenaremos os arquivos carregados.

Existem alguns pontos que vale a pena mencionar sobre o código de inicialização do cliente. Primeiro, estamos desabilitando o tempo limite de gravação e aumentando a simultaneidade máxima, para que os uploads possam ser concluídos mesmo em situações de baixa largura de banda.

Em segundo lugar, estamos desabilitando a validação da soma de verificação e habilitando a codificação em partes. Estamos fazendo isso porque queremos começar a enviar dados para o intervalo assim que os dados chegarem ao nosso serviço por streaming.

Por fim, não estamos tratando da criação do bucket em si, pois presumimos que já foi criado e configurado por um administrador.

Quanto às credenciais, fornecemos um AwsCredentialsProvider personalizado que pode recuperar as credenciais das propriedades do Spring. Isso abre a possibilidade de injetar esses valores por meio da abstração de ambiente do Spring e todas as suas implementações PropertySource com suporte, como Vault ou Config Server:

# 5. Visão geral do serviço de upload
Agora vamos implementar um serviço de upload, que estará acessível no caminho/inbox.

Um POST para este caminho de recurso armazenará o arquivo em nosso balde S3 sob uma chave gerada aleatoriamente. Armazenaremos o nome do arquivo original como uma chave de metadados, para que possamos usá-lo para gerar os cabeçalhos de download HTTP apropriados para navegadores.

Precisamos lidar com dois cenários distintos: uploads simples e de várias partes. Vamos criar um @RestController e adicionar métodos para lidar com esses cenários:

```
@RestController
@RequestMapping("/inbox")
@Slf4j
public class UploadResource {
    private final S3AsyncClient s3client;
    private final S3ClientConfigurarionProperties s3config;

    public UploadResource(S3AsyncClient s3client, S3ClientConfigurarionProperties s3config) {
        this.s3client = s3client;
        this.s3config = s3config;        
    }
    
    @PostMapping
    public Mono<ResponseEntity<UploadResult>> uploadHandler(
      @RequestHeader HttpHeaders headers, 
      @RequestBody Flux<ByteBuffer> body) {
      // ... see section 6
    }

    @RequestMapping(
      consumes = MediaType.MULTIPART_FORM_DATA_VALUE,
      method = {RequestMethod.POST, RequestMethod.PUT})
    public Mono<ResponseEntity<UploadResult>> multipartUploadHandler(
      @RequestHeader HttpHeaders headers,
      @RequestBody Flux<Part> parts ) {
      // ... see section 7
    }
}
```

As assinaturas do manipulador refletem a principal diferença entre os dois casos: no caso simples, o corpo contém o próprio conteúdo do arquivo, enquanto no caso de várias partes pode ter várias “partes”, cada uma correspondendo a um arquivo ou dados de formulário.

Para sua conveniência, ofereceremos suporte a uploads de várias partes usando os métodos POST ou PUT. A razão para isso é que algumas ferramentas (cURL, notavelmente) usam o último por padrão ao enviar arquivos com a opção -F.

Em ambos os casos, retornaremos um UploadResult contendo o resultado da operação e as chaves de arquivo geradas que um cliente deve usar para recuperar os arquivos originais - mais sobre isso depois!

# 6. Upload de arquivo único
Nesse caso, os clientes enviam conteúdo em uma operação POST simples com o corpo da solicitação contendo dados brutos. Para receber este conteúdo em um aplicativo Reactive Web, tudo o que temos que fazer é declarar um método @PostMapping que recebe um argumento Flux <<ByteBuffer>>.

O streaming desse fluxo para um novo arquivo S3 é simples neste caso.

Tudo o que precisamos é construir um PutObjectRequest com uma chave gerada, tamanho de arquivo, tipo de conteúdo MIME e passá-lo para o método putObject() em nosso cliente S3:

```
@PostMapping
public Mono<ResponseEntity<UploadResult>> uploadHandler(@RequestHeader HttpHeaders headers,
  @RequestBody Flux<ByteBuffer> body) {
    // ... some validation code omitted
    String fileKey = UUID.randomUUID().toString();
    MediaType mediaType = headers.getContentType();

    if (mediaType == null) {
        mediaType = MediaType.APPLICATION_OCTET_STREAM;
    }
    CompletableFuture future = s3client
      .putObject(PutObjectRequest.builder()
        .bucket(s3config.getBucket())
        .contentLength(length)
        .key(fileKey.toString())
        .contentType(mediaType.toString())
        .metadata(metadata)
        .build(), 
      AsyncRequestBody.fromPublisher(body));

    return Mono.fromFuture(future)
      .map((response) -> {
        checkResult(response);
        return ResponseEntity
          .status(HttpStatus.CREATED)
          .body(new UploadResult(HttpStatus.CREATED, new String[] {fileKey}));
        });
}
```

O ponto-chave aqui é como estamos passando o Flux de entrada para o método putObject().

Este método espera um objeto AsyncRequestBody que fornece conteúdo sob demanda. Basicamente, é um Publicador normal com alguns métodos de conveniência extras. Em nosso caso, tiraremos proveito do método fromPublishe() para converter nosso Flux no tipo necessário.

Além disso, supomos que o cliente enviará o cabeçalho HTTP Content-Length com o valor correto. Sem essas informações, a chamada falhará, pois este é um campo obrigatório.

Os métodos assíncronos no SDK V2 sempre retornam um objeto CompletableFuture. Nós o pegamos e o adaptamos para um Mono usando seu método de fábrica fromFuture(). Isso é mapeado para o objeto UploadResult final.

# 7. Upload de vários arquivos
Lidar com um upload multipart/form-data pode parecer fácil, especialmente ao usar bibliotecas que lidam com todos os detalhes para nós. Então, podemos simplesmente usar o método anterior para cada arquivo carregado? Bem, sim, mas isso tem um preço: armazenamento em buffer.

Para usar o método anterior, precisamos do comprimento da parte, mas as transferências de arquivos em partes nem sempre incluem essas informações. Uma abordagem é armazenar a peça em um arquivo temporário e, em seguida, enviá-lo para a AWS, mas isso diminuirá o tempo total de upload. Isso também significa armazenamento extra para nossos servidores.

Como alternativa, usaremos aqui um upload de várias partes da AWS. Este recurso permite que o upload de um único arquivo seja dividido em vários blocos que podemos enviar em paralelo e fora de ordem.

As etapas são as seguintes, precisamos enviar:

- A solicitação createMultipartUpload - AWS responde com um uploadId que usaremos nas próximas chamadas;
- Pedaços de arquivo contendo o uploadId, número de sequência e conteúdo - AWS responde com um identificador ETag para cada parte
- Uma solicitação completeUpload contendo o uploadId e todas as ETags recebidas
Observação: vamos repetir essas etapas para cada FilePart recebido!

### 7.1. Pipeline de nível superior
O multipartUploadHandler em nossa classe @Controller é responsável por lidar, não surpreendentemente, com uploads de arquivos multipartes. Nesse contexto, cada parte pode ter qualquer tipo de dado, identificado por seu tipo MIME. O framework Reactive Web entrega essas partes ao nosso manipulador como um Fluxo de objetos que implementam a interface Part, que processaremos em seguida:

```
return parts
  .ofType(FilePart.class)
  .flatMap((part)-> saveFile(headers, part))
  .collect(Collectors.toList())
  .map((keys)-> new UploadResult(HttpStatus.CREATED, keys)));
```

Esse pipeline começa filtrando as partes que correspondem a um arquivo carregado real, que sempre será um objeto que implementa a interface FilePart. Cada parte é então passada para o método saveFile, que lida com o upload real de um único arquivo e retorna a chave do arquivo gerada.

Coletamos todas as chaves em uma lista e, finalmente, construímos o UploadResult final. Estamos sempre criando um novo recurso, portanto, retornaremos um status CREATED mais descritivo (202) em vez de um OK normal.

### 7.2 Lidando com um único upload de arquivo
Já descrevemos as etapas necessárias para carregar um arquivo usando o método multipartes da AWS. Porém, há um porém: o serviço S3 requer que cada parte, exceto a última, tenha um tamanho mínimo determinado - 5 MBytes, atualmente.

Isso significa que não podemos simplesmente pegar os pedaços recebidos e enviá-los imediatamente. Em vez disso, precisamos armazená-los em buffer localmente até atingirmos o tamanho mínimo ou o final dos dados. Como também precisamos de um local para rastrear quantas peças enviamos e os resultados CompletedPart resultantes, criaremos uma classe interna UploadState simples para manter este estado:

```
class UploadState {
    String bucket;
    String filekey;
    String uploadId;
    int partCounter;
    Map<Integer, CompletedPart> completedParts = new HashMap<>();
    int buffered = 0;
    // ... getters/setters omitted
    UploadState(String bucket, String filekey) {
        this.bucket = bucket;
        this.filekey = filekey;
    }
}
```

Dadas as etapas necessárias e o armazenamento em buffer, terminamos com a implementação pode parecer um pouco intimidante à primeira vista:

```
Mono<String> saveFile(HttpHeaders headers,String bucket, FilePart part) {
    String filekey = UUID.randomUUID().toString();
    Map<String, String> metadata = new HashMap<String, String>();
    String filename = part.filename();
    if ( filename == null ) {
        filename = filekey;
    }       
    metadata.put("filename", filename);    
    MediaType mt = part.headers().getContentType();
    if ( mt == null ) {
        mt = MediaType.APPLICATION_OCTET_STREAM;
    }
    UploadState uploadState = new UploadState(bucket,filekey);     
    CompletableFuture<CreateMultipartUploadResponse> uploadRequest = s3client
      .createMultipartUpload(CreateMultipartUploadRequest.builder()
        .contentType(mt.toString())
        .key(filekey)
        .metadata(metadata)
        .bucket(bucket)
        .build());

    return Mono
      .fromFuture(uploadRequest)
      .flatMapMany((response) -> {
          checkResult(response);              
          uploadState.uploadId = response.uploadId();
          return part.content();
      })
      .bufferUntil((buffer) -> {
          uploadState.buffered += buffer.readableByteCount();
          if ( uploadState.buffered >= s3config.getMultipartMinPartSize() ) {
              uploadState.buffered = 0;
              return true;
          } else {
              return false;
          }
      })
      .map((buffers) -> concatBuffers(buffers))
      .flatMap((buffer) -> uploadPart(uploadState,buffer))
      .reduce(uploadState,(state,completedPart) -> {
          state.completedParts.put(completedPart.partNumber(), completedPart);              
          return state;
      })
      .flatMap((state) -> completeUpload(state))
      .map((response) -> {
          checkResult(response);
          return  uploadState.filekey;
      });
}
```

Começamos coletando alguns metadados de arquivo e usando-os para criar um objeto de solicitação exigido pela chamada de API createMultipartUpload(). Essa chamada retorna um CompletableFuture, que é o ponto de partida para nosso pipeline de streaming.

Vamos revisar o que cada etapa deste pipeline faz:

- Após receber o resultado inicial, que contém o uploadId gerado pelo S3, salvamos no objeto upload state e iniciamos o streaming do corpo do arquivo. Observe o uso de flatMapMany aqui, que transforma o Mono em um Flux;
- Usamos bufferUntil () para acumular o número necessário de bytes. O pipeline neste ponto muda de um Fluxo de objetos DataBuffer para um Fluxo de objetos List <<DataBuffer>>;
- Converta cada List <<DataBuffer>> em um ByteBuffer
- Envie o ByteBuffer para o S3 (consulte a próxima seção) e retorne o valor CompletedPart resultante do downstream;
- Reduza os valores CompletedPart resultantes no uploadState
- Sinaliza S3 que concluímos o upload (mais sobre isso depois);
- Retorne a chave do arquivo gerada.

### 7.3. Carregando partes do arquivo
Mais uma vez, vamos deixar claro que uma "parte do arquivo" aqui significa uma parte de um único arquivo (por exemplo, os primeiros 5 MB de um arquivo de 100 MB), não uma parte de uma mensagem que passa a ser um arquivo, como está em o stream de nível superior!

O pipeline de upload de arquivo chama o método uploadPart() com dois argumentos: o estado de upload e um ByteBuffer. A partir daí, construímos uma instância UploadPartRequest e usamos o método uploadPart() disponível em nosso S3AsyncClient para enviar os dados:

```
private Mono<CompletedPart> uploadPart(UploadState uploadState, ByteBuffer buffer) {
    final int partNumber = ++uploadState.partCounter;
    CompletableFuture<UploadPartResponse> request = s3client.uploadPart(UploadPartRequest.builder()
        .bucket(uploadState.bucket)
        .key(uploadState.filekey)
        .partNumber(partNumber)
        .uploadId(uploadState.uploadId)
        .contentLength((long) buffer.capacity())
        .build(), 
        AsyncRequestBody.fromPublisher(Mono.just(buffer)));
    
    return Mono
      .fromFuture(request)
      .map((uploadPartResult) -> {              
          checkResult(uploadPartResult);
          return CompletedPart.builder()
            .eTag(uploadPartResult.eTag())
            .partNumber(partNumber)
            .build();
      });
}
```

Aqui, usamos o valor de retorno da solicitação uploadPart() para construir uma instância CompletedPart. Este é um tipo de SDK da AWS de que precisaremos mais tarde, ao construir a solicitação final que fecha o upload.

### 7.4 Concluindo o upload
Por último, mas não menos importante, precisamos terminar o upload do arquivo de várias partes enviando uma solicitação completeMultipartUpload() para S3. Isso é muito fácil, pois o pipeline de upload passa todas as informações de que precisamos como argumentos:

```
private Mono<CompleteMultipartUploadResponse> completeUpload(UploadState state) {        
    CompletedMultipartUpload multipartUpload = CompletedMultipartUpload.builder()
        .parts(state.completedParts.values())
        .build();
    return Mono.fromFuture(s3client.completeMultipartUpload(CompleteMultipartUploadRequest.builder()
        .bucket(state.bucket)
        .uploadId(state.uploadId)
        .multipartUpload(multipartUpload)
        .key(state.filekey)
        .build()));
}
```

# 8. Baixando arquivos da AWS
Em comparação com uploads de várias partes, baixar objetos de um bucket do S3 é uma tarefa muito mais simples. Nesse caso, não precisamos nos preocupar com pedaços ou algo parecido. A API SDK fornece o método getObject() que leva dois argumentos:

- Um objeto GetObjectRequest contendo o intervalo solicitado e a chave do arquivo;
- Um AsyncResponseTransformer, que nos permite mapear uma resposta de streaming de entrada para outra coisa.

O SDK fornece algumas implementações do último que tornam possível adaptar o fluxo a um Flux, mas, novamente, com um custo: eles armazenam dados em buffer internamente em um buffer de array. Como esse armazenamento em buffer resulta em um tempo de resposta ruim para um cliente de nosso serviço de demonstração, implementaremos nosso próprio adaptador - o que não é grande coisa, como veremos.

### 8.1 Baixar controlador
Nosso controlador de download é um Spring Reactive @RestController padrão, com um único método @GetMapping que lida com solicitações de download. Esperamos a chave do arquivo por meio de um argumento @PathVariable e retornaremos um ResponseEntity assíncrono com o conteúdo do arquivo:

```
@GetMapping(path="/{filekey}")
Mono<ResponseEntity<Flux<ByteBuffer>>> downloadFile(@PathVariable("filekey") String filekey) {    
    GetObjectRequest request = GetObjectRequest.builder()
      .bucket(s3config.getBucket())
      .key(filekey)
      .build();
    
    return Mono.fromFuture(s3client.getObject(request,new FluxResponseProvider()))
      .map(response -> {
        checkResult(response.sdkResponse);
        String filename = getMetadataItem(response.sdkResponse,"filename",filekey);            
        return ResponseEntity.ok()
          .header(HttpHeaders.CONTENT_TYPE, response.sdkResponse.contentType())
          .header(HttpHeaders.CONTENT_LENGTH, Long.toString(response.sdkResponse.contentLength()))
          .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + filename + "\"")
          .body(response.flux);
      });
}
```

Aqui, getMetadataItem() é apenas um método auxiliar que procura uma determinada chave de metadados na resposta sem fazer distinção entre maiúsculas e minúsculas.

Este é um detalhe importante: o S3 retorna informações de metadados usando cabeçalhos HTTP especiais, mas esses cabeçalhos não diferenciam maiúsculas de minúsculas (consulte RFC 7230, seção 3.2). Isso significa que as implementações podem mudar o caso para um determinado item à vontade - e isso realmente acontece ao usar o MinIO.

### 8.2. Implementação FluxResponseProvider
Nosso FluxReponseProvider deve implementar a interface AsyncResponseTransformer, que tem apenas quatro métodos:

- Prepare(), onde podemos fazer qualquer configuração necessária;
- onResponse(), chamado quando S3 retorna o status de resposta e metadados;
- onStream() chamado quando a resposta possui um corpo, sempre após onResponse();
- exceptionOccurred() chamada no caso de algum erro.

A tarefa desse provedor é lidar com esses eventos e criar uma instância FluxResponse, contendo a instância GetObjectResponse fornecida e o corpo da resposta como um fluxo:

```
class FluxResponseProvider implements AsyncResponseTransformer<GetObjectResponse,FluxResponse> {    
    private FluxResponse response;
    @Override
    public CompletableFuture<FluxResponse> prepare() {
        response = new FluxResponse();
        return response.cf;
    }

    @Override
    public void onResponse(GetObjectResponse sdkResponse) {            
        this.response.sdkResponse = sdkResponse;
    }

    @Override
    public void onStream(SdkPublisher<ByteBuffer> publisher) {
        response.flux = Flux.from(publisher);
        response.cf.complete(response);            
    }

    @Override
    public void exceptionOccurred(Throwable error) {
        response.cf.completeExceptionally(error);
    }
}
```

Finalmente, vamos dar uma olhada rápida na classe FluxResponse:

```
class FluxResponse {
    final CompletableFuture<FluxResponse> cf = new CompletableFuture<>();
    GetObjectResponse sdkResponse;
    Flux<ByteBuffer> flux;
}
```

# 9. Conclusão
Neste tutorial, cobrimos os fundamentos do uso das extensões reativas disponíveis na biblioteca AWS SDK V2. Nosso foco aqui foi o serviço AWS S3, mas podemos estender as mesmas técnicas a outros serviços reativos, como o DynamoDB.