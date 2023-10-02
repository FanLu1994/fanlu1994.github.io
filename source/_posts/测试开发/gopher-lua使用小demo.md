---
title: gopher-lua使用小demo
date: 2023-05-17 09:04:08
tags: golang实验室
categories: 测试开发
---
> 模拟读者读书

## 首先新建reader类

```go
package main

import "fmt"

type Reader struct {
	Uid         uint32
	UserName    string
	ReaderCount uint8
}

func (reader *Reader) read(book string) {
	reader.ReaderCount++
	fmt.Printf("Reader:%v,Name:%v,read book %v\n", reader.Uid, reader.UserName, book)
}

```

## 将reader类注册到lua中

```go
package main

import lua "github.com/yuin/gopher-lua"

const luaPersonTypeName = "reader"

var readerMethods = map[string]lua.LGFunction{
	"read":     luaReaderRead,
	"username": readerGetSetUsername,
}

// 注册定义的类成为lua的一个元表
func registerReaderType(L *lua.LState) {
	mt := L.NewTypeMetatable(luaPersonTypeName)
	L.SetGlobal("reader", mt)
	L.SetField(mt, "new", L.NewFunction(luaNewReader))
	L.SetField(mt, "__index", L.SetFuncs(L.NewTable(), readerMethods))
}

// lua创建对象方法
func luaNewReader(L *lua.LState) int {
	reader := &Reader{
		uint32(L.CheckInt(1)),
		L.CheckString(2),
		uint8(L.CheckInt(3)),
	}
	ud := L.NewUserData()
	ud.Value = reader
	L.SetMetatable(ud, L.GetTypeMetatable(luaPersonTypeName))
	L.Push(ud)
	return 1
}

// 在lua中获取对象的重要一步
func checkReader(L *lua.LState) *Reader {
	ud := L.CheckUserData(1)
	if v, ok := ud.Value.(*Reader); ok {
		return v
	}
	L.ArgError(1, "reader expected")
	return nil
}

// 方法注册到lua中
func luaReaderRead(L *lua.LState) int {
	r := checkReader(L)
	book := L.ToString(2)
	r.read(book)
	return 1
}

// 属性的get Set方法， 注意方法名必须这样写：结构名GetSet属性名，大小写也要注意
func readerGetSetUsername(L *lua.LState) int {
	r := checkReader(L)
	if L.GetTop() == 2 {
		r.UserName = L.CheckString(2)
		return 0
	}
	L.Push(lua.LString(r.UserName))
	return 1
}
```

## 也许有一些模块需要注入到lua中

```go
package main

import (
	"fmt"
	lua "github.com/yuin/gopher-lua"
)

var modFuncs = map[string]lua.LGFunction{
	"eat":    Eat,
	"drink":  Drink,
	"record": Record,
}

func Eat(L *lua.LState) int {
	msg := L.CheckString(1)
	fmt.Println("eat:", msg)
	return 0
}

func Drink(L *lua.LState) int {
	msg := L.CheckString(1)
	fmt.Println("drink:", msg)
	return 0
}

func Record(L *lua.LState) int {
	r := checkReader(L)
	fmt.Printf("%v读完了！一共%v本书！\n", r.UserName, r.ReaderCount)
	return 1
}

func Loader(L *lua.LState) int {
	mod := L.SetFuncs(L.NewTable(), modFuncs)
	L.SetField(mod, "mymod", lua.LString("value"))
	L.Push(mod)
	return 1
}

```

## 预先定义一个lua文件

这样所有的协程可以共享这个lua文件

```lua
local mymod =require("mymod")  -- 加载注入的模块

function init()
    global_id = 1
    global_name = "test"
end

function newReader()
    r = reader.new(global_id,global_name,0)
end

-- 连续执行三次
function read(book)
    r:read(book)
    mymod.eat("面包")
    mymod.drink("雪碧")
end

function finish()
    mymod.record(r)
end
```

## 然后可以试试看啦

```go
package main

import (
	"bufio"
	"fmt"
	"github.com/yuin/gopher-lua"
	"github.com/yuin/gopher-lua/parse"
	"math/rand"
	"os"
	"strconv"
	"sync"
	"time"
)

// TODO: 加载lua代码执行
// TODO: 多线程

var wg sync.WaitGroup

func main() {
	books := []string{
		"活着", "白鹿原", "春秋战国", "兄弟", "许三观卖血记", "丰乳肥臀",
	}

	luaPath := "./main/test.lua"
	luaProto, err := compileFile(luaPath)
	if err != nil {
		fmt.Println(err)
		return
	}

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go DoRead(luaProto, uint32(i), "Reader"+strconv.Itoa(i), books)
	}

	wg.Wait()

}

// 机器人主流程
func DoRead(luaProto *lua.FunctionProto, id uint32, name string, books []string) {
	fmt.Println(id)
	L := lua.NewState()
	defer L.Close()
	registerReaderType(L)
	L.PreloadModule("mymod", Loader)          // 注入自己的模块
	lFunc := L.NewFunctionFromProto(luaProto) // 从字节码解析得到
	L.Push(lFunc)
	L.PCall(0, lua.MultRet, nil)

	// init
	if err := L.CallByParam(lua.P{
		Fn:      L.GetGlobal("init"),
		NRet:    0,
		Protect: true,
	}, lua.LNil); err != nil {
		fmt.Println(err)
	}

	// 新建机器人
	L.SetGlobal("global_id", lua.LNumber(id))
	L.SetGlobal("global_name", lua.LString(name))
	if err := L.CallByParam(lua.P{
		Fn:      L.GetGlobal("newReader"),
		NRet:    0,
		Protect: true,
	}, lua.LNil); err != nil {
		fmt.Println(err)
	}

	// 读书
	for i := 0; i < 3; i++ {
		book := books[rand.Int()%len(books)]
		if err := L.CallByParam(lua.P{
			Fn:      L.GetGlobal("read"),
			NRet:    0,
			Protect: true,
		}, lua.LString(book)); err != nil {
			fmt.Println(err)
		}
		time.Sleep(time.Second)
	}

	// 结束
	if err := L.CallByParam(lua.P{
		Fn:      L.GetGlobal("finish"),
		NRet:    0,
		Protect: true,
	}, lua.LNil); err != nil {
		fmt.Println(err)
	}
	wg.Done()
}

// 解析文件变成lua字节码
func compileFile(filePath string) (*lua.FunctionProto, error) {
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

```



