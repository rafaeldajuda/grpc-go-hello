
# GRPC-GO-HELLO

Aplicação simples em GRPC feito em Golang.

## Referências

Aplicação e o tutorial foram feitos em cima das seguintes referências:

* https://grpc.io/docs/languages/go/basics/
* https://dev.to/thenicolau/introducao-ao-grpc-golang-210f

## Hello.proto

Arquivos `.proto` servem como esquemas para gerar as funções do GRPC. Neles conterão as estruturas, os serviços e as funções dos serviços.

`estruturas`
(contracts/hello.proto)
```proto
syntax = "proto3";

message HelloRequest {
    string name = 1;
}

message HelloResponse {
    string msg = 1;
}
```

**OBS:** o número seguido do tipo (string) e do nome da variável (name ou msg) é a ordem que o campo que vai se encontrar na estrutura.

Dentro dos serviços declaramos as funções que os serviços terão. Nas funções definimos como será a entrada e a sáida da função, porém como a função executará os dados vai ser definido no server.

`serviço e função`
(contracts/hello.proto)
```proto
option go_package = "./pb";

service HelloService {
    rpc Hello(HelloRequest) returns (HelloResponse) {};
}
```

(contracts/hello.proto)
```proto
syntax = "proto3";

message HelloRequest {
    string name = 1;
}

message HelloResponse {
    string msg = 1;
}

option go_package = "./pb";

service HelloService {
    rpc Hello(HelloRequest) returns (HelloResponse) {};
}
```

## Protobuf

* Vamos instalar o compilador do Protobuf que irá gerar nosso código. Caso esteja usando Linux, você pode usar o apt ou o apt-get, por exemplo:
```cmd
apt install -y protobuf-compiler
protoc --version
```

* Após instalar o compilador é preciso instalar os plugins do GO para compilador de protocolo 
```cmd
$ go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
$ go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2
```

* Atualize seu PATH para que o compilador protoc possa encontrar os plugins:
```
$ export PATH="$PATH:$(go env GOPATH)/bin"
```

* Feito isso vamos rodar o seguinte comando para gerar os arquivos `.pb`:
```cmd
protoc --go_out=./pb --go_opt=paths=source_relative   --go-grpc_out=./pb --go-grpc_opt=paths=source_relative  contracts/*.proto
```

## Server

O server é onde iremos iniciar o servidor do GRPC e registrar quais serviços iremos utilizar.

(server/main.go)
```golang
package main

import (
	"context"
	"grpc-go/pb"
	"log"
	"net"

	"google.golang.org/grpc"
)

type Server struct {
	pb.HelloServiceServer
}

func (s *Server) Hello(ctx context.Context, request *pb.HelloRequest) (*pb.HelloResponse, error) {
	return &pb.HelloResponse{Msg: "Hello " + request.GetName()}, nil
}

func main() {

	listen, err := net.Listen("tcp", "0.0.0.0:9000")
	if err != nil {
		log.Fatalf("Failed to listen: %v", err)
	}

	grpcServer := grpc.NewServer()
	pb.RegisterHelloServiceServer(grpcServer, &Server{})

	if err := grpcServer.Serve(listen); err != nil {
		log.Fatalf("Failed to serve: %v", err)
	}
}
```

A estrutura `Server` irá conter a estrutura `pb.HelloServiceServer` que foi gerada a partir do arquivo `hello.proto`. A função `Hello`, que pertece a estrutura `Server`, representa a função Hello do arquivo `hello.proto`. É aqui que definimos como a função será executada.
```golang
type Server struct {
	pb.HelloServiceServer
}

func (s *Server) Hello(ctx context.Context, request *pb.HelloRequest) (*pb.HelloResponse, error) {
	return &pb.HelloResponse{Msg: "Hello " + request.GetName()}, nil
}
```

Para criar um server no GRPC é preciso definir uma conexão `tcp`:
```golang
listen, err := net.Listen("tcp", "0.0.0.0:9000")
if err != nil {
	log.Fatalf("Failed to listen: %v", err)
}
```

Depois iniciamos um server GRPC sem definições de conexão e serviços:
```golang
grpcServer := grpc.NewServer()
```

Registramos o serviço no server e como ele passmos uma instância do `Server`. Lembrando que o `Server` possui a estrutura que foi criado pelo `hello.proto`:
```golang
pb.RegisterHelloServiceServer(grpcServer, &Server{})
```

E finalmente iremos iniciar o servidor:
```golang
if err := grpcServer.Serve(listen); err != nil {
	log.Fatalf("Failed to serve: %v", err)
}
```

## Client

É onde iremos conectar em alguma conexão GRPC e consumir os serviços.

(client/main.go)
```golang
package main

import (
	"context"
	"grpc-go/pb"
	"log"

	"google.golang.org/grpc"
)

func main() {
	connection, err := grpc.Dial("localhost:9000", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("Failed to connect: %v", err)
	}
	defer connection.Close()

	client := pb.NewHelloServiceClient(connection)

	request := &pb.HelloRequest{Name: "Rafael"}
	response, err := client.Hello(context.Background(), request)
	if err != nil {
		log.Fatalf("Failed to call: %v", err)
	}

	log.Printf("Response: %v", response.Msg)

}
```

Antes de consumir o serviço é preciso conectar ao server:
```golang
connection, err := grpc.Dial("localhost:9000", grpc.WithInsecure())
if err != nil {
	log.Fatalf("Failed to connect: %v", err)
}
defer connection.Close()
```

Iniciamos um client com a conexão criada. O client terá acesso as funções criadas pelo `hello.proto`.
```golang
client := pb.NewHelloServiceClient(connection)
```

Iremos criar uma variável chamada request do tipo `pb.HelloRequest`, passando um nome qualquer para o campo `Name`:
```golang
request := &pb.HelloRequest{Name: "Rafael"}
```

Agora iremos chamar a função `client.Hello`, passando como parâmetros um contexto e a variável request:
```golang
response, err := client.Hello(context.Background(), request)
if err != nil {
	log.Fatalf("Failed to call: %v", err)
}
```

Caso não ocorra erro a resposta será armazenada na variável response e printada logo em seguida.
```golang
log.Printf("Response: %v", response.Msg)
```

## Teste

Para testar é precisa iniciar o Server:
```cmd
go run server/main.go
```

Depois iniciar o Client:
```cmd
go run client/main.go
```

Caso não ocorra nenhum erro, o Client irá printar a seguinte mensagem:
```cmd
2023/04/21 09:25:04 Response: Hello <name>
```
