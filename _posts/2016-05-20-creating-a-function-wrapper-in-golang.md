---
layout: post
title: creating a function wrapper in golang
date: '2016-05-20T15:53:09-0500'
author: arges
tags:
- golang

---

Here is a snippet that allows one to create a 'wrapper' function that you can
input a generic function and parameters too in golang. `callme` takes a
function and a variadic set of parameters. By using the reflect library we can
properly check number of parameters a function has as well as value types.
Finally we use `Call` to call the function with the processed parameters. A use
case for this could be calling a function and allowing for retries or
additional checking.

~~~go
package main

import (
	"fmt"
	"reflect"
	"strconv"
)

func callme(fn interface{}, params ...interface{}) (result []reflect.Value) {
	f := reflect.ValueOf(fn)
	if f.Type().NumIn() != len(params) {
		panic("incorrect number of parameters!")
	}
	inputs := make([]reflect.Value, len(params))
	for k, in := range params {
		inputs[k] = reflect.ValueOf(in)
	}
	return f.Call(inputs)
}

func hello(i int) {
	fmt.Println("hello " + strconv.Itoa(i))
}

func hiya(name string) {
	fmt.Println("hiya " + name)
}

func awesome(i int, name string) {
	fmt.Println("high " + strconv.Itoa(i) + ", " + name)
}

func main() {
	callme(hello, 1)
	callme(hiya, "buddy")
	callme(awesome, 5, "dude")
}
~~~
