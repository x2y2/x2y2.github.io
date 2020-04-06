---
layout: post
title: go字符串处理
subtitle: 字符串拼接、分隔、子串判断
date: 2020-04-06
categories: Go
cover: 'assets/img/road_green_trees_forest.jpg'
tags: go string 
---

在日常开发工作中对于字符串的处理应该是最常见的场景。比如，我想要构建这样的字符串\<type\>:\<client id\>:\<id\>
比如，需要截取字符串、判断字符串中是否包含其他子串等。
## 1. 字符串拼接
### "+"拼接
```
key := itemType + ':' + clientId + ":" + id
```
### string.Join方式

```
key := string.Join([]string{itemType,clientId,id},":")
```
### fmt.Sprintf方式
```
key := fmt.Sprintf("%s:%s:%s",itemType,clientId,id)
```
### bytes.Buffer方式
```
l := len(itemType) + len(clientId) + len(id) + 2
buf := make([]byte,0,l)
w := bytes.NewBuffer(buf)
w.WriteString(itemType)
w.WriteRune(':')
w.WriteString(clientId)
w.WriteRune(':')
w.WriteString(id)
key := w.String()
```
## 2. 判断字符串中是否包含其他子串
### strings.Contains
```
str1 := "the lucky day!"
str2 := "lucky"
if strings.Contains(str1,str2) {
        fmt.Println("found")
    }
```
### strings.Index
返回值为-1说明没有该字符串，大于-1说明含有该字符串
```
str1 := "the lucky day!"
str2 := "lucky"
if i := string.Index(str1,str2); i != -1{
    fmt.Printf("found")
}
```
### strings.HasPrefix
以xxx字符串开头
```
str1 := "the lucky day!"
str2 := "lucky"
strings.HasPrefix(str1,"the")
```
### strings.HasSuffix(str1,"!")
以xxx字符串结尾
```
str1 := "the lucky day!"
str2 := "lucky"
strings.HasSuffix(str1,"!")
```
### strings.ContainsAny
子串总任何一个字符在主串中,即返回true
```
strings.ContainsAny(str1,"tl")
```
### strings.IndexFunc
返回子串第一次出现的索引
```
str := "the lucky day!"
strings.IndexFunc(str,func(c rune) bool{return c == 'd' || c == 'a'})
```
## 3. 字符串分隔
### strings.Split
以分隔符分隔
```
strings.Split("a,b,c",",")
```
### strings.Fields
以空白字符分隔
```
str := "the lucky day!"
fmt.Println(strings.Fields(str))
```
### re := regexp.MustCompile();re.Split(str,n)
以正则表达式作为参数，比如: `[#|!]`,以\#或\|或！作为分隔符,Split中的n >= 0时，表示最多分隔为n个子串，n = -1表示不限定子串数。
```
str := "what#a#lucky|day!"
re := regexp.MustCompile(`[#|!]`)
fmt.Println(re.Split(str,-1))
```
## 3. 删除空白字符
### strings.TrimSpace
删除首尾空白字符
```
fmt.Println(strings.TrimSpace("  Good night \n \t "))
```
### strings.Trim
删除首尾指定字符
```
fmt.Println(strings.Trim("#Good night!!!","#!"))
```
### strings.TrimLeft
删除首部指定字符
```
fmt.Println(strings.TrimLeft("#Good night","#"))
```
### strings.TrimRight
删除尾部指定字符
```
fmt.Println(strings.TrimRight("Good night!!!","!"))
```
### strings.Replace
替换字符串中任意位置的字符
```
fmt.Println(strings.Replace("Good #night#","#","",-1))
```