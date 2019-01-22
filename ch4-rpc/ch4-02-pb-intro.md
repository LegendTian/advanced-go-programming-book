# 4.2 Protobuf

Protobuf是Protocol Buffers的简称，它是Google公司开发的一种数据描述语言，并于2008年对外开源。Protobuf刚开源时的定位类似于XML、JSON等数据描述语言，通过附带工具生成代码并实现将结构化数据序列化的功能。但是我们更关注的是Protobuf作为接口规范的描述语言，可以作为设计安全的跨语言PRC接口的基础工具。

## 4.2.1 Protobuf入门

对于没有用过Protobuf的读者，建议先从官网了解下基本用法。这里我们尝试将Protobuf和RPC结合在一起使用，通过Protobuf来最终保证RPC的接口规范和安全。Protobuf中最基本的数据单元是message，是类似Go语言中结构体的存在。在message中可以嵌套message或其它的基础数据类型的成员。

首先创建hello.proto文件，其中包装HelloService服务中用到的字符串类型：

```protobuf
syntax = "proto3";

package main;

message String {
	string value = 1;
}
```

开头的syntax语句表示采用proto3的语法。第三版的Protobuf对语言进行了提炼简化，所有成员均采用类似Go语言中的零值初始化（不再支持自定义默认值），因此消息成员也不再需要支持required特性。然后package指令指明当前是main包（这样可以和Go的包名保持一致，简化例子代码），当然用户也可以针对不同的语言定制对应的包路径和名称。最后message关键字定义一个新的String类型，在最终生成的Go语言代码中对应一个String结构体。String类型中只有一个字符串类型的value成员，该成员编码时用1编号代替名字。

在XML或JSON等数据描述语言中，一般通过成员的名字来绑定对应的数据。但是Protobuf编码却是通过成员的唯一编号来绑定对应的数据，因此Protobuf编码后数据的体积会比较小，但是也非常不便于人类查阅。我们目前并不关注Protobuf的编码技术，最终生成的Go结构体可以自由采用JSON或gob等编码格式，因此大家可以暂时忽略Protobuf的成员编码部分。

Protobuf核心的工具集是C++语言开发的，在官方的protoc编译器中并不支持Go语言。要想基于上面的hello.proto文件生成相应的Go代码，需要安装相应的插件。首先是安装官方的protoc工具，可以从 https://github.com/google/protobuf/releases 下载。然后是安装针对Go语言的代码生成插件，可以通过`go get github.com/golang/protobuf/protoc-gen-go`命令安装。

然后通过以下命令生成相应的Go代码：

```
$ protoc --go_out=. hello.proto
```

其中`go_out`参数告知protoc编译器去加载对应的protoc-gen-go工具，然后通过该工具生成代码，生成代码放到当前目录。最后是一系列要处理的protobuf文件的列表。

这里只生成了一个hello.pb.go文件，其中String结构体内容如下：

```go
type String struct {
	Value string `protobuf:"bytes,1,opt,name=value" json:"value,omitempty"`
}

func (m *String) Reset()         { *m = String{} }
func (m *String) String() string { return proto.CompactTextString(m) }
func (*String) ProtoMessage()    {}
func (*String) Descriptor() ([]byte, []int) {
	return fileDescriptor_hello_069698f99dd8f029, []int{0}
}

func (m *String) GetValue() string {
	if m != nil {
		return m.Value
	}
	return ""
}
```

生成的结构体中还会包含一些以`XXX_`为名字前缀的成员，我们已经隐藏了这些成员。同时String类型还自动生成了一组方法，其中ProtoMessage方法表示这是一个实现了proto.Message接口的方法。此外Protobuf还为每个成员生成了一个Get方法，Get方法不仅可以处理空指针类型，而且可以和Protobuf第二版的方法保持一致（第二版的自定义默认值特性依赖这类方法）。

基于新的String类型，我们可以重新实现HelloService服务：

```go
type HelloService struct{}

func (p *HelloService) Hello(request *String, reply *String) error {
	reply.Value = "hello:" + request.GetValue()
	return nil
}
```

其中Hello方法的输入参数和输出的参数均改用Protobuf定义的String类型表示。因为新的输入参数为结构体类型，因此改用指针类型作为输入参数，函数的内部代码同时也做了相应的调整。

至此，我们初步实现了Protobuf和RPC组合工作。在启动RPC服务时，我们依然可以选择默认的gob或手工指定json编码，甚至可以重新基于protobuf编码实现一个插件。虽然做了这么多工作，但是似乎并没有看到什么收益！

回顾第一章中更安全的RPC接口部分的内容，当时我们花费了极大的力气去给RPC服务增加安全的保障。最终得到的更安全的RPC接口的代码本身就非常繁琐的使用手工维护，同时全部安全相关的代码只适用于Go语言环境！既然使用了Protobuf定义的输入和输出参数，那么RPC服务接口是否也可以通过Protobuf定义呢？其实用Protobuf定义语言无关的RPC服务接口才是它真正的价值所在！

下面更新hello.proto文件，通过Protobuf来定义HelloService服务：

```protobuf
service HelloService {
	rpc Hello (String) returns (String);
}
```

但是重新生成的Go代码并没有发生变化。这是因为世界上的RPC实现有千万种，protoc编译器并不知道该如何为HelloService服务生成代码。

不过在protoc-gen-go内部已经集成了一个名字为`grpc`的插件，可以针对gRPC生成代码：

```
$ protoc --go_out=plugins=grpc:. hello.proto
```

在生成的代码中多了一些类似HelloServiceServer、HelloServiceClient的新类型。这些类型是为gRPC服务的，并不符合我们的RPC要求。

不过gRPC插件为我们提供了改进的思路，下面我们将探索如何为我们的RPC生成安全的代码。


## 4.2.2 定制代码生成插件

Protobuf的protoc编译器是通过插件机制实现对不同语言的支持。比如protoc命令出现`--xxx_out`格式的参数，那么protoc将首先查询是否有内置的xxx插件，如果没有内置的xxx插件那么将继续查询当前系统中是否存在protoc-gen-xxx命名的可执行程序，最终通过查询到的插件生成代码。对于Go语言的protoc-gen-go插件来说，里面又实现了一层静态插件系统。比如protoc-gen-go内置了一个gRPC插件，用户可以通过`--go_out=plugins=grpc`参数来生成gRPC相关代码，否则只会针对message生成相关代码。

参考gRPC插件的代码，可以发现generator.RegisterPlugin函数可以用来注册插件。插件是一个generator.Plugin接口：

```go
// A Plugin provides functionality to add to the output during
// Go code generation, such as to produce RPC stubs.
type Plugin interface {
	// Name identifies the plugin.
	Name() string
	// Init is called once after data structures are built but before
	// code generation begins.
	Init(g *Generator)
	// Generate produces the code generated by the plugin for this file,
	// except for the imports, by calling the generator's methods P, In,
	// and Out.
	Generate(file *FileDescriptor)
	// GenerateImports produces the import declarations for this file.
	// It is called after Generate.
	GenerateImports(file *FileDescriptor)
}
```

其中Name方法返回插件的名字，这是Go语言的Protobuf实现的插件体系，和protoc插件的名字并无关系。然后Init函数是通过g参数对插件进行初始化，g参数中包含Proto文件的所有信息。最后的Generate和GenerateImports方法用于生成主体代码和对应的导入包代码。

因此我们可以设计一个netrpcPlugin插件，用于为标准库的RPC框架生成代码：

```go
import (
	"github.com/golang/protobuf/protoc-gen-go/generator"
)

type netrpcPlugin struct{ *generator.Generator }

func (p *netrpcPlugin) Name() string                { return "netrpc" }
func (p *netrpcPlugin) Init(g *generator.Generator) { p.Generator = g }

func (p *netrpcPlugin) GenerateImports(file *generator.FileDescriptor) {
	if len(file.Service) > 0 {
		p.genImportCode(file)
	}
}

func (p *netrpcPlugin) Generate(file *generator.FileDescriptor) {
	for _, svc := range file.Service {
		p.genServiceCode(svc)
	}
}
```

首先Name方法返回插件的名字。netrpcPlugin插件内置了一个匿名的`*generator.Generator`成员，然后在Init初始化的时候用参数g进行初始化，因此插件是从g参数对象继承了全部的公有方法。其中GenerateImports方法调用自定义的genImportCode函数生成导入代码。Generate方法调用自定义的genServiceCode方法生成每个服务的代码。

目前，自定义的genImportCode和genServiceCode方法只是输出一行简单的注释：

```go
func (p *netrpcPlugin) genImportCode(file *generator.FileDescriptor) {
	p.P("// TODO: import code")
}

func (p *netrpcPlugin) genServiceCode(svc *descriptor.ServiceDescriptorProto) {
	p.P("// TODO: service code, Name = " + svc.GetName())
}
```

要使用该插件需要先通过generator.RegisterPlugin函数注册插件，可以在init函数中完成：

```go
func init() {
	generator.RegisterPlugin(new(netrpcPlugin))
}
```

因为Go语言的包只能静态导入，我们无法向已经安装的protoc-gen-go添加我们新编写的插件。我们将重新克隆protoc-gen-go对应的main函数：

```go
package main

import (
	"io/ioutil"
	"os"

	"github.com/golang/protobuf/proto"
	"github.com/golang/protobuf/protoc-gen-go/generator"
)

func main() {
	g := generator.New()

	data, err := ioutil.ReadAll(os.Stdin)
	if err != nil {
		g.Error(err, "reading input")
	}

	if err := proto.Unmarshal(data, g.Request); err != nil {
		g.Error(err, "parsing input proto")
	}

	if len(g.Request.FileToGenerate) == 0 {
		g.Fail("no files to generate")
	}

	g.CommandLineParameters(g.Request.GetParameter())

	// Create a wrapped version of the Descriptors and EnumDescriptors that
	// point to the file that defines them.
	g.WrapTypes()

	g.SetPackageNames()
	g.BuildTypeNameMap()

	g.GenerateAllFiles()

	// Send back the results.
	data, err = proto.Marshal(g.Response)
	if err != nil {
		g.Error(err, "failed to marshal output proto")
	}
	_, err = os.Stdout.Write(data)
	if err != nil {
		g.Error(err, "failed to write output proto")
	}
}
```

为了避免对protoc-gen-go插件造成干扰，我们将我们的可执行程序命名为protoc-gen-go-netrpc，表示包含了nerpc插件。然后用以下命令重新编译hello.proto文件：

```
$ protoc --go-netrpc_out=plugins=netrpc:. hello.proto
```

其中`--go-netrpc_out`参数告知protoc编译器加载名为protoc-gen-go-netrpc的插件，插件中的`plugins=netrpc`指示启用内部唯一的名为netrpc的netrpcPlugin插件。在新生成的hello.pb.go文件中将包含增加的注释代码。

至此，手工定制的Protobuf代码生成插件终于可以工作了。

## 4.2.3 自动生成完整的RPC代码

在前面的例子中我们已经构建了最小化的netrpcPlugin插件，并且通过克隆protoc-gen-go的主程序创建了新的protoc-gen-go-netrpc的插件程序。现在开始继续完善netrpcPlugin插件，最终目标是生成RPC安全接口。

首先是自定义的genImportCode方法中生成导入包的代码：

```go
func (p *netrpcPlugin) genImportCode(file *generator.FileDescriptor) {
	p.P(`import "net/rpc"`)
}
```

然后要在自定义的genServiceCode方法中为每个服务生成相关的代码。分析可以发现每个服务最重要的是服务的名字，然后每个服务有一组方法。而对于服务定义的方法，最重要的是方法的名字，还有输入参数和输出参数类型的名字。

为此我们定义了一个ServiceSpec类型，用于描述服务的元信息：

```go
type ServiceSpec struct {
	ServiceName string
	MethodList  []ServiceMethodSpec
}

type ServiceMethodSpec struct {
	MethodName     string
	InputTypeName  string
	OutputTypeName string
}
```

然后我们新建一个buildServiceSpec方法用来解析每个服务的ServiceSpec元信息：

```go
func (p *netrpcPlugin) buildServiceSpec(
	svc *descriptor.ServiceDescriptorProto,
) *ServiceSpec {
	spec := &ServiceSpec{
		ServiceName: generator.CamelCase(svc.GetName()),
	}

	for _, m := range svc.Method {
		spec.MethodList = append(spec.MethodList, ServiceMethodSpec{
			MethodName:     generator.CamelCase(m.GetName()),
			InputTypeName:  p.TypeName(p.ObjectNamed(m.GetInputType())),
			OutputTypeName: p.TypeName(p.ObjectNamed(m.GetOutputType())),
		})
	}

	return spec
}
```

其中输入参数是`*descriptor.ServiceDescriptorProto`类型，完整描述了一个服务的所有信息。然后通过`svc.GetName()`就可以获取Protobuf文件中定义的服务的名字。Protobuf文件中的名字转为Go语言的名字后，需要通过`generator.CamelCase`函数进行一次转换。类似的，在for循环中我们通过`m.GetName()`获取方法的名字，然后再转为Go语言中对应的名字。比较复杂的是对输入和输出参数名字的解析：首先需要通过`m.GetInputType()`获取输入参数的类型，然后通过`p.ObjectNamed`获取类型对应的类对象信息，最后获取类对象的名字。

然后我们就可以基于buildServiceSpec方法构造的服务的元信息生成服务的代码：

```go
func (p *netrpcPlugin) genServiceCode(svc *descriptor.ServiceDescriptorProto) {
	spec := p.buildServiceSpec(svc)

	var buf bytes.Buffer
	t := template.Must(template.New("").Parse(tmplService))
	err := t.Execute(&buf, spec)
	if err != nil {
		log.Fatal(err)
	}

	p.P(buf.String())
}
```

为了便于维护，我们基于Go语言的模板来生成服务代码，其中tmplService是服务的模板。

在编写模板之前，我们先查看下我们期望生成的最终代码大概是什么样子：

```go
type HelloServiceInterface interface {
	Hello(in String, out *String) error
}

func RegisterHelloService(srv *rpc.Server, x HelloService) error {
	if err := srv.RegisterName("HelloService", x); err != nil {
		return err
	}
	return nil
}

type HelloServiceClient struct {
	*rpc.Client
}

var _ HelloServiceInterface = (*HelloServiceClient)(nil)

func DialHelloService(network, address string) (*HelloServiceClient, error) {
	c, err := rpc.Dial(network, address)
	if err != nil {
		return nil, err
	}
	return &HelloServiceClient{Client: c}, nil
}

func (p *HelloServiceClient) Hello(in String, out *String) error {
	return p.Client.Call("HelloService.Hello", in, out)
}
```

其中HelloService是服务名字，同时还有一系列的方法相关的名字。

参考最终要生成的代码可以构建如下模板：

```go
const tmplService = `
{{$root := .}}

type {{.ServiceName}}Interface interface {
	{{- range $_, $m := .MethodList}}
	{{$m.MethodName}}(*{{$m.InputTypeName}}, *{{$m.OutputTypeName}}) error
	{{- end}}
}

func Register{{.ServiceName}}(
	srv *rpc.Server, x {{.ServiceName}}Interface,
) error {
	if err := srv.RegisterName("{{.ServiceName}}", x); err != nil {
		return err
	}
	return nil
}

type {{.ServiceName}}Client struct {
	*rpc.Client
}

var _ {{.ServiceName}}Interface = (*{{.ServiceName}}Client)(nil)

func Dial{{.ServiceName}}(network, address string) (
	*{{.ServiceName}}Client, error,
) {
	c, err := rpc.Dial(network, address)
	if err != nil {
		return nil, err
	}
	return &{{.ServiceName}}Client{Client: c}, nil
}

{{range $_, $m := .MethodList}}
func (p *{{$root.ServiceName}}Client) {{$m.MethodName}}(
	in *{{$m.InputTypeName}}, out *{{$m.OutputTypeName}},
) error {
	return p.Client.Call("{{$root.ServiceName}}.{{$m.MethodName}}", in, out)
}
{{end}}
`
```

当Protobuf的插件定制工作完成后，每次hello.proto文件中RPC服务的变化都可以自动生成代码。也可以通过更新插件的模板，调整或增加生成代码的内容。在掌握了定制Protobuf插件技术后，你将彻底拥有这个技术。
