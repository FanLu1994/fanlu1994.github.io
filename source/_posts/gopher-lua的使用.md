---
title: gopher-lua的使用
date: 2023-05-17 09:03:13
tags:
---
#golang #lua #压测
> https://github.com/yuin/gopher-lua#usage

## 简单使用
1. 首先声明一个lua虚拟机： L := lua.NewState()  返回一个LState Struct
2. 然后可以执行lua格式的字符串或者File
	- lua.DoString(`print("hello")`)
	- lua.DoFile(lua脚本的路径)

LState定义如下：

```go
type LState struct {
	G       *Global
	Parent  *LState
	Env     *LTable
	Panic   func(*LState)
	Dead    bool
	Options Options

	stop         int32
	reg          *registry
	stack        callFrameStack
	alloc        *allocator
	currentFrame *callFrame
	wrapped      bool
	uvcache      *Upvalue
	hasErrorFunc bool
	mainLoop     func(*LState, *callFrame)
	ctx          context.Context
}
```

- Get方法  获取栈中的变量
- 

## 数据模型
gopher-lua中的说有变量值都是一个LValue, 是go语言中的interface，包含两个方法：
- String（）string
- Type() LValueType
```go
type LValue interface {  
   String() string  
   Type() LValueType   
   assertFloat64() (float64, bool)  
   assertString() (string, bool)  
   assertFunction() (*LFunction, bool)  
}
```

该接口的实现包括如下类：

| Type name   | Go type        | Type() value | Constants         |
| ----------- | -------------- | ------------ | ----------------- |
| `LNilType`  | (constants)    | `LTNil`      | `LNil`            |
| `LBool`     | (constants)    | `LTBool`     | `LTrue`, `LFalse` |
| `LNumber`   | float64        | `LTNumber`   | `-`               |
| `LString`   | string         | `LTString`   | `-`               |
| `LFunction` | struct pointer | `LTFunction` | `-`               |
| `LUserData` | struct pointer | `LTUserData` | `-`               |
| `LState`    | struct pointer | `LTThread`   | `-`               |
| `LTable`    | struct pointer | `LTTable`    | `-`               |
| `LChannel`  | chan LValue    | `LTChannel`  | `-`               |

- lv.Type()可以获取类型
- 原表不可用；没有错误捕捉

## Callstack & Registry size

LState的调用栈的大小控制着脚本中Lua函数的最大调用深度（Go函数的调用不算在内）。

LState的注册表实现了对调用函数（包括Lua和Go函数）和表达式中的临时变量的栈存储。它的存储需求将随着调用堆栈的使用和代码的复杂性而增加。

注册表和调用堆栈都可以被设置为固定大小或自动大小。

```go
 L := lua.NewState(lua.Options{
    RegistrySize: 1024 * 20,         // this is the initial size of the registry
    RegistryMaxSize: 1024 * 80,      // this is the maximum size that the registry can grow to. If set to `0` (the default) then the registry will not auto grow
    RegistryGrowStep: 32,            // this is how much to step up the registry by each time it runs out of space. The default is `32`.
 })
defer L.Close()
```



## API

### 从lua中调用go函数

```go
func Double(L *lua.LState) int {
    lv := L.ToInt(1)             /* get argument */
    L.Push(lua.LNumber(lv * 2)) /* push result */
    return 1                     /* number of results */
}

func main() {
    L := lua.NewState()
    defer L.Close()
    L.SetGlobal("double", L.NewFunction(Double)) /* Original lua_setglobal uses stack... */
   	if err := L.DoString(`print(double(20))`); err != nil {
		panic(err)
	}
}
```

注册为lua函数之后，会变成一个LGFunction类型；

支持协程中运行；

### 加载lua内置库的函数

```go
func main() {
    L := lua.NewState(lua.Options{SkipOpenLibs: true})
    defer L.Close()
    for _, pair := range []struct {
        n string
        f lua.LGFunction
    }{
        {lua.LoadLibName, lua.OpenPackage}, // Must be first
        {lua.BaseLibName, lua.OpenBase},
        {lua.TabLibName, lua.OpenTable},
    } {
        if err := L.CallByParam(lua.P{
            Fn:      L.NewFunction(pair.f),
            NRet:    0,
            Protect: true,
        }, lua.LString(pair.n)); err != nil {
            panic(err)
        }
    }
    if err := L.DoFile("main.lua"); err != nil {
        panic(err)
    }
}
```

### 在go中创建一个lua的模块

1. 首先定义一组方法  类型为map[string]lua.LGFuntion

2. 然后调用SetFuncs  将函数表分配给一个lua table，作为一个模块，获取到一个LTable

3. 然后将模块push到栈

   ```go
   func Loader(L *lua.LState) int {
       // register functions to the table
       mod := L.SetFuncs(L.NewTable(), exports)
       // register other stuff
       L.SetField(mod, "name", lua.LString("value"))
       // returns the module
       L.Push(mod)
       return 1
   }
   ```

4. 通过PreLoadModule（name,  注册方法）将模块注册到虚拟机中

   ```go
   L.PreloadModule("mymodule", mymodule.Loader)
   ```

### 在go中调用lua方法

```go
if err := L.CallByParam(lua.P{
    Fn: L.GetGlobal("double"),		// lua方法名
    NRet: 1,					// 
    Protect: true,
    }, lua.LNumber(10)); err != nil {
    panic(err)
}
```

- CallByParam方法 第一个参数 lua.P结构； 第二个参数 参数
- 通过lua.P 结构进行调用
- 实际使用中 函数参数也可以使用提前设置全局变量的方式来实现

### 自定义类型

支持在Go中自定义新类型

```go
type Person struct {
    Name string
}

const luaPersonTypeName = "person"

// 注册类型
func registerPersonType(L *lua.LState) {
    mt := L.NewTypeMetatable(luaPersonTypeName)  // 新建一个元表
    L.SetGlobal("person", mt)					// 元表设置为全局变量
    // static attributes
    L.SetField(mt, "new", L.NewFunction(newPerson)) // 注册方法到元表中 静态放啊
    // methods
    L.SetField(mt, "__index", L.SetFuncs(L.NewTable(), personMethods)) // 注册方法到元表
}

// Constructor
func newPerson(L *lua.LState) int {			// go方法
    person := &Person{L.CheckString(1)}
    ud := L.NewUserData()
    ud.Value = person
    L.SetMetatable(ud, L.GetTypeMetatable(luaPersonTypeName))
    L.Push(ud)
    return 1
}

// Checks whether the first lua argument is a *LUserData with *Person and returns this *Person.
func checkPerson(L *lua.LState) *Person {  // 检查类型
    ud := L.CheckUserData(1)
    if v, ok := ud.Value.(*Person); ok {
        return v
    }
    L.ArgError(1, "person expected")
    return nil
}

var personMethods = map[string]lua.LGFunction{		// 方法表
    "name": personGetSetName,
}

// Getter and setter for the Person#Name
func personGetSetName(L *lua.LState) int {			// 属性的Getter和Setter 在lua中通过p:name()调用
    p := checkPerson(L)
    if L.GetTop() == 2 {
        p.Name = L.CheckString(2)
        return 0
    }
    L.Push(lua.LString(p.Name))
    return 1
}

func main() {
    L := lua.NewState()
    defer L.Close()
    registerPersonType(L)
    if err := L.DoString(`						
        p = person.new("Steeve")	
        print(p:name("新名字")) --  
		print(p:name())
        p:name("Alice")
        print(p:name()) -- "Alice"
    `); err != nil {
        panic(err)
    }
}
```

### 共享lua字节代码

调用DoFile将加载一个Lua脚本，将其编译为字节码，并在一个LState中运行字节码。

如果你有多个LState，它们都需要运行同一个脚本，你可以在它们之间共享字节码，这将节省内存。共享字节码是安全的，因为它是只读的，不能被lua脚本所改变。

```go
// CompileLua reads the passed lua file from disk and compiles it.
func CompileLua(filePath string) (*lua.FunctionProto, error) {
    file, err := os.Open(filePath)
    defer file.Close()
    if err != nil {
        return nil, err
    }
    reader := bufio.NewReader(file)
    chunk, err := parse.Parse(reader, filePath)
    if err != nil {
        return nil, err
    }
    proto, err := lua.Compile(chunk, filePath)
    if err != nil {
        return nil, err
    }
    return proto, nil
}

// DoCompiledFile takes a FunctionProto, as returned by CompileLua, and runs it in the LState. It is equivalent
// to calling DoFile on the LState with the original source file.
func DoCompiledFile(L *lua.LState, proto *lua.FunctionProto) error {
    lfunc := L.NewFunctionFromProto(proto)
    L.Push(lfunc)
    return L.PCall(0, lua.MultRet, nil)
}
```

### go协程

LState不是goroutine-safe。建议每个goroutine使用一个LState，并通过使用通道在goroutine之间通信。

通道在GopherLua中由通道对象表示。而一个通道表提供了执行通道操作的函数。

有些对象不能通过通道发送，因为它本身有非goroutine安全的对象。

一个线程(state)
一个函数
一个用户数据
一个有元数据的表

