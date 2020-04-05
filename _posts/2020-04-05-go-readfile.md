---
layout: post
title: go读文件的几种方式
<!-- subtitle: -->
date: 2020-04-05
categories: Go
cover: 'assets/img/path.jpg'
tags: go readfile 
---

> 在golang中读本地文件有很多种方式，总结下常用的几种方式

#### 1. Read一次读取若干字节
```
package main

import (
    "os"
    "fmt"
    "io"
        )

const BufferSize = 1024

func main(){
    file, err := os.Open("/tmp/filetoread.txt")
    if err != nil{
        fmt.Println(err)
    }
    defer file.Close()

    buffer := make([]byte,BufferSize)

    for { 
        bytesread ,err := file.Read(buffer)
        if err != nil {
            if err != io.EOF {fmt.Println(err)}
            break
        } 
        fmt.Println("bytes read: ",bytesread)
        fmt.Println("bytes read: \n",string(buffer[:bytesread]))
    }
}
```
#### 2. NewScanner逐行读
```
package main

import (
    "fmt"
    "bufio"
    "os"
)

func main(){
    file , err := os.Open("/tmp/filetoread.txt")
    if err != nil{
        fmt.Println(err)
        return
    }
    defer file.Close()

    scanner := bufio.NewScanner(file)
    scanner.Split(bufio.ScanLines)
    
    

    var lines []string
    for scanner.Scan(){
        lines = append(lines,scanner.Text())
     }
     fmt.Println("read lines:")
     fmt.Println(lines)
     for _ ,line := range lines{
        fmt.Println(line)
     }
}
```
#### 3. 协程并发读
```
package main

import (
    "os"
    "fmt"
    "io"
    "sync"

)
const BufferSize = 100

type chunk struct{
    bufsize int
    offset int64
}

func main(){
    file , err := os.Open("/tmp/filetoread.txt")
    if err != nil{
        fmt.Println(err)
        return
    }
    defer file.Close()

    fileinfo,err := file.Stat()
    if err != nil{
        fmt.Println(err)
        return 
    }

    filesize := int(fileinfo.Size())
    concurrency := filesize / BufferSize

    chunksizes := make([]chunk,concurrency)

    for i := 0; i < concurrency; i++{
        chunksizes[i].bufsize = BufferSize
        chunksizes[i].offset = int64(BufferSize * i)
    }

    if remainder := filesize % BufferSize; remainder != 0{
        c := chunk{bufsize: remainder,offset: int64(concurrency * BufferSize)}
        concurrency ++
        chunksizes = append(chunksizes,c)
    }

    var wg sync.WaitGroup
    wg.Add(concurrency)

    for i := 0; i < concurrency; i++{
        go func(chunksizes []chunk, i int){
            defer wg.Done()

            chunk := chunksizes[i]
            buffer := make([]byte,chunk.bufsize)
            bytesread, err := file.ReadAt(buffer,chunk.offset)

            if err != nil && err != io.EOF{
                fmt.Println(err)
                return
            }
            fmt.Println("bytes read, string: ",bytesread)
            fmt.Println("bytes string: ",string(buffer[:bytesread]))
        }(chunksizes,i)
    }
    wg.Wait()
}
```