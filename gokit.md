# Gokit

Spring boot like framework for [[golang]]. It has 3 major Components

## Service Layer

- Innermost layer where business logic resides.
- Modeled as services
- Oblivious to Endpoint/Transport Layers
- Can be used by multiple Transports (grpc/json/http)

## Endpoint Layer

- Represents a single RPC Method
- Service exposed as an Endpoint
- Endpoint can be exposed by multiple Transports

## Transport Layer

- Exposes various Transports
  - grpc
  - http

# Building a `pastebin` clone

## Define a service blueprint interface

```go
// PbService provides storage capabilities
type PbService interface {
	Create(content string, ctx context.Context) (string, error)
	Delete(key string, ctx context.Context) (string, error)
	Get(key string, ctx context.Context) (string, error)
}
```

## Make a new struct to define the PasteBin Service

This struct is used to group together the functionalities of pastebin service

```go
type pbService struct {
	memory map[uuid.UUID]string
}

// NewPbService make a new PbService
func NewPbService() PbService {
	return pbService{
		memory: make(map[uuid.UUID]string),
	}
}
```

## Implement the PbService Interface on the struct

In [[golang]] we do not have a key word to define that this structs implements a specific interface like the `implements` in Java.

They way we enforce contracts is by implementing all the methods of the contract interface in our case here its the `PbService` interface.

Since our `NewPbService` method returns the type of `PbService` the go compiler will ensure that `NewPbService` confirms to the `PbService` interface.

```go
//Create: Here we store the content and return a uuid
func (s pbService) Create(ctx context.Context, content string) (string, error) {
	id := uuid.New()
	s.memory[id] = content
	return id.String(), nil
}

//Delete: Here we use the key to find and delete the content stored
func (s pbService) Delete(ctx context.Context, key string) (string, error) {
	id, err := uuid.Parse(key)
	if err != nil {
		return "", errors.New("Invalid Uuid Format")
	}
	delete(s.memory, id)
	return "ok", nil
}

//Get: Here we use the key to find and return the content stored
func (s pbService) Get(ctx context.Context, key string) (string, error) {
	id, err := uuid.Parse(key)
	if err != nil {
		return "", errors.New("Invalid Uuid Format")
	}
	content, exists := s.memory[id]
	if exists {
		return content, nil
	}
	return "", errors.New("Invalid Uuid")
}
```

## Request and Response

In Go kit, the primary messaging pattern is RPC.

So, every method in our interface will be modeled as a remote procedure call. For each method, we define request and response structs, capturing all of the input and output parameters respectively.

### Create Request Response

```go
type createPbRequest struct {
	content string `json:"content"`
}

type createPbResponse struct {
	key string `json:"key"`
	Err string `json:"err,omitempty"` // errors don't JSON-marshal, so we use a string
}
```

### Delete Request Response

```go
type deletePbRequest struct {
	key string `json:"key"`
}

type deletePbResponse struct {
	status string `json:"status"`
	Err    string `json:"err,omitempty"` // errors don't JSON-marshal, so we use a string
}
```

### Get Request Response

```go
type getPbRequest struct {
	key string `json:"key"`
}

type getPbResponse struct {
	content string `json:"content"`
	Err     string `json:"err,omitempty"` // errors don't JSON-marshal, so we use a string
}
```

## Define Endpoints

An endpoint represents a single RPC, which is a single method in our service.

### Create Endpoint

```go
func createPbEndpoint(svc PbService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(createPbRequest)
		key, err := svc.Create(ctx, req.Content)
		if err != nil {
			return createPbResponse{key, err.Error()}, nil
		}
		return createPbResponse{key, ""}, nil
	}
}
```

### Delete Endpoint

```go
func deletePbEndpoint(svc PbService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(deletePbRequest)
		status, err := svc.Delete(ctx, req.Key)
		if err != nil {
			return deletePbResponse{status, err.Error()}, nil
		}
		return deletePbResponse{status, ""}, nil
	}
}
```

### Get Endpoint

```go
func getPbEndpoint(svc PbService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(getPbRequest)
		content, err := svc.Get(ctx, req.Key)
		if err != nil {
			return getPbResponse{content, err.Error()}, nil
		}
		return getPbResponse{content, ""}, nil
	}
}
```
## Define Transport

Since this trivial example used JSON over HTTP we would have to decode the JSON to structs that our service can understand

### Create Requester Decoder 

```go
func decodeCreatePbRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request createPbRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}
```

### Delete Request Decoder

```go
func decodeDeletePbRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request deletePbRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}
```

### Get Request Decoder

```go
func decodeGetPbRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request getPbRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}
```
### Response Encoder 

This method would accept an `interface` type and convert it JSON, this allows it to accept `createPbResponse`,`deletePbResponse`,`getPbResponse` as an `interface{}` and encode it as json using the annotations in the struct definition.

```go
func encodeResponse(_ context.Context, w http.ResponseWriter, response interface{}) error {
	return json.NewEncoder(w).Encode(response)
}
```

## Main 

```go
import (
	"context"
	"encoding/json"
	"errors"
	"log"
	"net/http"

	"github.com/go-kit/kit/endpoint"
	"github.com/google/uuid"

	httptransport "github.com/go-kit/kit/transport/http"
)

func main() {
	svc := NewPbService()
	createPbHandler := httptransport.NewServer(
		createPbEndpoint(svc),
		decodeCreatePbRequest,
		encodeResponse,
	)

	deletePbHandler := httptransport.NewServer(
		deletePbEndpoint(svc),
		decodeDeletePbRequest,
		encodeResponse,
	)
	
	getPbHandler := httptransport.NewServer(
		getPbEndpoint(svc),
		decodeGetPbRequest,
		encodeResponse,
	)
	
	http.Handle("/create", createPbHandler)
	http.Handle("/delete", deletePbHandler)
	http.Handle("/get", getPbHandler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```
## Divide and Conquer
At this point the `main.go` has a lot of code so lets move to different files so that we have separation of concerns.

### `service.go` 

```go
package main

import (
	"context"
	"errors"

	"github.com/google/uuid"
)

// PbService provides storage capabilities
type PbService interface {
	Create(ctx context.Context, content string) (string, error)
	Delete(ctx context.Context, key string) (string, error)
	Get(ctx context.Context, key string) (string, error)
}

type pbService struct {
	memory map[uuid.UUID]string
}

// NewPbService make a new PbService
func NewPbService() PbService {
	return pbService{
		memory: make(map[uuid.UUID]string),
	}
}

//Create: Here we store the content and return a uuid
func (s pbService) Create(ctx context.Context, content string) (string, error) {
	id := uuid.New()
	s.memory[id] = content
	return id.String(), nil
}

//Get: Here we use the key to find and return the content stored
func (s pbService) Get(ctx context.Context, key string) (string, error) {
	id, err := uuid.Parse(key)
	if err != nil {
		return "", errors.New("Invalid Uuid Format")
	}
	content, exists := s.memory[id]
	if exists {
		return content, nil
	}
	return "", errors.New("Invalid Uuid")
}

//Delete: Here we use the key to find and delete the content stored
func (s pbService) Delete(ctx context.Context, key string) (string, error) {
	id, err := uuid.Parse(key)
	if err != nil {
		return "", errors.New("Invalid Uuid Format")
	}
	delete(s.memory, id)
	return "ok", nil
}
```

### `transport.go`

```go
package main

import (
	"context"
	"encoding/json"
	"net/http"

	"github.com/go-kit/kit/endpoint"
)

type createPbRequest struct {
	Content string `json:"content"`
}

type createPbResponse struct {
	Key string `json:"key"`
	Err string `json:"err,omitempty"` // errors don't JSON-marshal, so we use a string
}

type getPbRequest struct {
	Key string `json:"key"`
}

type getPbResponse struct {
	Content string `json:"content"`
	Err     string `json:"err,omitempty"` // errors don't JSON-marshal, so we use a string
}

type deletePbRequest struct {
	Key string `json:"key"`
}

type deletePbResponse struct {
	Status string `json:"status"`
	Err    string `json:"err,omitempty"` // errors don't JSON-marshal, so we use a string
}

func createPbEndpoint(svc PbService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(createPbRequest)
		key, err := svc.Create(ctx, req.Content)
		if err != nil {
			return createPbResponse{key, err.Error()}, nil
		}
		return createPbResponse{key, ""}, nil
	}
}

func deletePbEndpoint(svc PbService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(deletePbRequest)
		status, err := svc.Delete(ctx, req.Key)
		if err != nil {
			return deletePbResponse{status, err.Error()}, nil
		}
		return deletePbResponse{status, ""}, nil
	}
}

func getPbEndpoint(svc PbService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(getPbRequest)
		content, err := svc.Get(ctx, req.Key)
		if err != nil {
			return getPbResponse{content, err.Error()}, nil
		}
		return getPbResponse{content, ""}, nil
	}
}

func decodeCreatePbRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request createPbRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func decodeGetPbRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request getPbRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func decodeDeletePbRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request deletePbRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}
```

### `main.go`

```go
func main() {
	svc := NewPbService()
	createPbHandler := httptransport.NewServer(
		createPbEndpoint(svc),
		decodeCreatePbRequest,
		encodeResponse,
	)

	deletePbHandler := httptransport.NewServer(
		deletePbEndpoint(svc),
		decodeDeletePbRequest,
		encodeResponse,
	)
	getPbHandler := httptransport.NewServer(
		getPbEndpoint(svc),
		decodeGetPbRequest,
		encodeResponse,
	)
	http.Handle("/create", createPbHandler)
	http.Handle("/delete", deletePbHandler)
	http.Handle("/get", getPbHandler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```
## Logging Middleware

All applications need to log information, this can be enabled by adding a logging middleware that we create in a file called `logging.go`

Middleware in go-kit work on `Endpoint`

The interface definition is `type Middleware func(Endpoint) Endpoint`, which means it is a function that takes in an endpoint and returns an endpoint

We can create the `loggingMiddleware` so that it adheres to the `PbService` by implementing the `Create` `Delete` `Get` methods.
```go
type loggingMiddleware struct {
	logger log.Logger
	next   PbService
}
```

### Create 
```go
func (m loggingMiddleware) Create(ctx context.Context, content string) (output string, err error) {
	// This defered function would be invoked just before the retuen statement
	defer func(begin time.Time) {
		m.logger.Log(
			"method", "CreatePb",
			"input", content,
			"output", output,
			"err", err,
			"took", time.Since(begin),
		)
	}(time.Now())
	output, err = m.next.Create(ctx, content)
	return
}
```

### Delete
```go
 func (m loggingMiddleware) Delete(ctx context.Context, key string) (output string, err error) {
	defer func(begin time.Time) {
		m.logger.Log(
			"method", "DeletePb",
			"input", key,
			"output", output,
			"err", err,
			"took", time.Since(begin),
		)
	}(time.Now())
	output, err = m.next.Delete(ctx, key)
	return
}
```

### Get 
```go
func (m loggingMiddleware) Get(ctx context.Context, key string) (output string, err error) {
	defer func(begin time.Time) {
		m.logger.Log(
			"method", "GetPb",
			"input", key,
			"output", output,
			"err", err,
			"took", time.Since(begin),
		)
	}(time.Now())
	output, err = m.next.Get(ctx, key)
	return
}
```

## Wiring the Middleware

In order to wire the middleware in all we have to do is link it up with the service that we have defined

```go
package main

import (
	"context"
	"encoding/json"
	"net/http"
	"os"

	"github.com/go-kit/kit/log"

	httptransport "github.com/go-kit/kit/transport/http"
)

func main() {
	// Use the global logger
	logger := log.NewLogfmtLogger(os.Stderr)
	var svc PbService
	svc = NewPbService()
	// Wire the middleware and thats it
	svc = loggingMiddleware{logger, svc}

	createPbHandler := httptransport.NewServer(
		createPbEndpoint(svc),
		decodeCreatePbRequest,
		encodeResponse,
	)

	deletePbHandler := httptransport.NewServer(
		deletePbEndpoint(svc),
		decodeDeletePbRequest,
		encodeResponse,
	)

	getPbHandler := httptransport.NewServer(
		getPbEndpoint(svc),
		decodeGetPbRequest,
		encodeResponse,
	)
	http.Handle("/create", createPbHandler)
	http.Handle("/delete", deletePbHandler)
	http.Handle("/get", getPbHandler)
	logger.Log("msg", "HTTP", "addr", ":8080")
	logger.Log("err", http.ListenAndServe(":8080", nil))
}

func encodeResponse(_ context.Context, w http.ResponseWriter, response interface{}) error {
	return json.NewEncoder(w).Encode(response)
}
```

## PasteBin
### Client 
```sh 

$ curl localhost:8080/create -XPOST -d '{"content":"THIS IS SPARTA"}'

{"key":"c449250a-d74c-4d23-acbb-6785b0bd822a"} 

$ curl localhost:8080/get -XPOST -d '{"key":"c449250a-d74c-4d23-acbb-6785b0bd822a"}'

{"content":"THIS IS SPARTA"}

$ curl localhost:8I00/delete -XPOST -d '{"key":"c449250a-d74c-4d23-acbb-6785b0bd822a"}'

{"status":"ok"}

$ curl localhost:8080/get -XPOST -d '{"key":"c449250a-d74c-4d23-acbb-6785b0bd822a"}'

{"content":"","err":"Invalid Uuid"}

```

### Server
```sh
$ ./pastebin-II 

msg=HTTP addr=:8080

method=CreatePb input="THIS IS SPARTA" output=c449250a-d74c-4d23-acbb-6785b0bd822a err=null took=67.92µs

method=GetPb input=c449250a-d74c-4d23-acbb-6785b0bd822a output="THIS IS SPARTA" err=null took=1.675µs

method=DeletePb input=c449250a-d74c-4d23-acbb-6785b0bd822a output=ok err=null took=1.45µs

method=GetPb input=c449250a-d74c-4d23-acbb-6785b0bd822a output= err="Invalid Uuid" took=803ns
``` 

[//begin]: # "Autogenerated link references for markdown compatibility"
[golang]: golang "Golang"
[//end]: # "Autogenerated link references"
