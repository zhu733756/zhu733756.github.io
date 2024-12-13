---
title: '聊聊MongoDB BSON数据格式'
tags: ['MongoDB', 'bson', 'bsondump']
categories: ['数据库', '实战', 'golang', 'demo']
series: ['MongoDB 知识汇总']
author: ['zhu733756']
date: 2024-12-13T15:52:11+08:00
---

## 前戏

> 小白：老花，最近我在研究`MongoDB`，听说它使用`BSON`作为数据格式，这`BSON`到底是个啥玩意儿？

> 老花：哈哈，小白，`BSON`是`Binary JSON`的缩写，它是一种类`JSON`的二进制编码格式。`BSON`的设计目的, 是为了在`MongoDB`中存储和交换文档数据。它继承了`JSON`的灵活性和可读性，同时增加了一些额外的数据类型，比如日期、二进制数据和代码等。

> 小白：那为啥要用`BSON`这种数据存储格式呢？

> 老花：有几个原因。首先，`BSON`是二进制格式，这意味着它比`JSON`更紧凑，传输效率更高。其次，`BSON`支持更多的数据类型，这使得它能够存储更复杂的数据结构。再者，`BSON`是自描述的，每个字段都有类型信息，这使得解析`BSON`数据时更加方便。

小白：听起来挺酷的，那我们怎么解析`BSON`数据呢？

## BSON 数据格式

解析`BSON`数据其实很简单。

在 Go 语言中，`MongoDB`的官方驱动提供了很好的支持。你可以使用`go.mongodb.org/mongo-driver/bson`这个包来处理`BSON`数据。

比如，你可以这样解析一个`BSON`文档：

```go
var doc bson.M
err := bson.Unmarshal(data, &doc)
if err != nil {
    log.Fatal(err)
}
fmt.Println(doc)
```

这里的`data`是你从 MongoDB 获取的 BSON 格式的字节切片，`bson.M`是一个可以存储任何类型值的 map，非常适合用来解析`BSON`文档。

## 如何实现 BSON 模糊搜索?

> 小白：那如果我想对 `BSON` 数据进行模糊搜索，比如搜索名字中包含某个字符串的文档？

老花：这个也好办。在 `MongoDB` 中，你可以使用正则表达式来进行模糊搜索。在 `Go` 语言中，你可以这样构建查询：

```go
import "go.mongodb.org/mongo-driver/bson"

// 假设我们要搜索名字中包含"haha"的文档
filter := bson.M{"name": bson.M{"$regex": "haha", "$options": "i"}} // "i"代表忽略大小写

// 然后使用这个 filter 来查询数据库
cursor, err := collection.Find(context.TODO(), filter)
if err != nil {
  log.Fatal(err)
}

// 遍历结果
for cursor.Next(context.TODO()) {
  var result bson.M
  err := cursor.Decode(&result)
  if err != nil {
  log.Fatal(err)
}

fmt.Println(result)
```

这段代码会查询所有 `name` 字段中包含`haha`的文档，并且忽略大小写。

在 MongoDB 中，除了$regex 算子用于模糊搜索外，还有一些其他的算子可以用于不同的搜索场景：

- `$regex`: 用于执行正则表达式匹配，进行模糊搜索。
- `$text`: 用于全文搜索。它与$regex 不同，$text 是专门设计用来搜索文本字段中的关键词，而不是匹配模式。
  ```go
  db.articles.createIndex( { subject: "text" } )
  db.articles.insert(
   [
     { _id: 1, subject: "coffee", author: "xyz", views: 50 },
     { _id: 2, subject: "Coffee Shopping", author: "efg", views: 5 },
     { _id: 3, subject: "Baking a cake", author: "abc", views: 90  },
     { _id: 4, subject: "baking", author: "xyz", views: 100 },
     { _id: 5, subject: "Café Con Leche", author: "abc", views: 200 },
     { _id: 6, subject: "Сырники", author: "jkl", views: 80 },
     { _id: 7, subject: "coffee and cream", author: "efg", views: 10 },
     { _id: 8, subject: "Cafe con Leche", author: "xyz", views: 10 }
   ]
  )
  db.articles.find( { $text: { $search: "coffee" } } )
  ```
- `$exists`：用于搜索存在或不存在某个字段的文档。
  ```go
  db.records.find( { b: { $exists: false } } )
  db.inventory.find( { qty: { $exists: true, $nin: [ 5, 15 ] } } )
  ```
- `$in`：用于搜索字段值在指定数组中的文档。
  ```go
  db.inventory.update(
    { tags: { $in: ["appliances", "school"] } },
    { $set: { sale: true } }
  )
  ```
- `$nin`：用于搜索字段值不在指定数组中的文档。
- `$all`：用于搜索字段值是数组，并且数组包含所有指定元素的文档。
- `$size`：用于搜索数组字段中元素个数等于指定值的文档。
- `$mod`：用于搜索字段值除以指定数字后余数为指定值的文档。
- `$where`：用于使用 JavaScript 表达式进行更复杂的查询。
  ```go
  db.players.find({$where: function() {
    return (hex_md5(this.name) == "9b53e667f30cd329dca1ec9e6a83e994")
  }});
  ```
- ...

## BSON 全文搜索

> 小白：哇，老花你太厉害了，这一下子就把 BSON 的解析和模糊搜索讲清楚了。可是 BSON 对全文搜索是不是有一定限制, 比如只能在某个字段上构建索引, 如何实现任意返回的文档也能支持全文搜索?

老花：这是个好问题!

### 使用`$text`和`$search`进行全文搜索

要对一个或多个字段进行全文搜索，你需要在这些字段上创建一个文本索引。例如，如果你想对 content 字段进行全文搜索，你可以执行以下命令：

```javascript
db.collection.createIndex({ content: 'text' });
```

如果你想要对多个字段进行全文搜索，可以这样做：

```javascript
db.collection.createIndex({ field1: 'text', field2: 'text' });
```

创建了文本索引之后，你可以使用`$text`和`$search`算子来执行全文搜索。$text算子指定了要搜索的字段，而$search 算子包含了实际的搜索字符串。例如：

```javascript
db.collection.find({
	$text: {
		$search: 'search string',
	},
});
```

这个查询会返回所有 content 字段中包含“search string”文本的文档。

### bsondump

`bsondump` 是 `MongoDB` 提供的一个命令行工具，用于将 `BSON` 数据转换为 `JSON` 格式，或者将 `JSON` 格式转换为 `BSON` 格式。这个工具在处理和调试 `BSON` 数据时非常有用，因为它可以让你查看 `BSON` 数据的结构和内容，或者在不同的格式之间进行转换。

```bash
bsondump --type=json < yourfile.bson
bsondump --type=bson < yourfile.json
```

### 其他考虑

不考虑构建索引的情况, 我们可以吧`bson`解析成字符串, 然后模糊搜索。

这样其实性能较差, 我们还可以折中, 在反序列化的时候, 通过反射直接命中搜索的字段即可, 搜索效率略微提升。

我们现看下有序的`BSON`文档, 在`golang`中用`D`和`E`代表的是有序容器, `M`是无序容器, `A`是有序数组:

```go
// bson.D{{"foo", "bar"}, {"hello", "world"}, {"pi", 3.14159}}
type D []E

// E represents a BSON element for a D. It is usually used inside a D.
type E struct {
	Key   string
	Value interface{}
}

type M map[string]interface{}
type A []interface{}
```

假设一个文档包含多个字段, 最外层应该是一个`D`, 包含多个`Key-Value`(也就是一个`E`), 这个`E`可能是`D`, 也可能是`E`, 还有可能是`A`, 还有可能是`bson`的自定义类型:

```go
func BsonSearchD(bd bson.D, targetKey string) bool {
	for _, e := range bd {
		if BsonSearchE(e, targetKey) {
			return true
		}
	}

	return false
}

func BsonSearchE(be bson.E, targetKey string) bool {
	if be.Key == targetKey {
		return true
	}

	switch v := be.Value.(type) {
	case bson.D:
		b, _ := MarshalJSOND(v)
		return strings.Contains(string(b), targetKey)
	case bson.E:
		b, _ := MarshalJSONE(v)
		return strings.Contains(string(b), targetKey)
	case primitive.Binary:
		bs, _ := binaryToUUID(v)
		return strings.Contains(bs, targetKey)
	case primitive.Timestamp:
		b, _ := jsoniter.MarshalToString(TimestampToTime(v))
		return strings.Contains(b, targetKey)
	case primitive.DateTime:
		b, _ := jsoniter.MarshalToString(DateTimeToTime(v))
		return strings.Contains(b, targetKey)
	case primitive.A:
		b, _ := MarshalJSONA(v)
		return strings.Contains(string(b), targetKey)
	default:
		vv, _ := jsoniter.Marshal(v)
		if strings.Contains(string(vv), targetKey) {
			return true
		}
	}

	return false
}
```

反射之后的不同类型没有更好的办法, 直接序列号得了:

```go
func MarshalJSOND(data bson.D) ([]byte, error) {
	var b []byte
	buf := bytes.NewBuffer(b)
	buf.WriteRune('{')

	for i, val := range data {
		b, e := MarshalJSONE(val)
		if e != nil {
			return nil, e
		}
		buf.Write(b)

		// write delimiter
		if i+1 < len(data) {
			buf.WriteRune(',')
		}
	}

	buf.WriteRune('}')
	return buf.Bytes(), nil
}

func MarshalJSONA(data bson.A) ([]byte, error) {
	var b []byte
	buf := bytes.NewBuffer(b)
	buf.WriteRune('[')
	for index, a := range data {
		switch o := a.(type) {
		case bson.D:
			b, e := MarshalJSOND(o)
			if e != nil {
				return nil, e
			}
			buf.Write(b)
		case bson.E:
			b, e := MarshalJSONE(o)
			if e != nil {
				return nil, e
			}
			buf.Write(b)
		default:
			b, e := jsoniter.Marshal(o)
			if e != nil {
				return nil, e
			}
			buf.Write(b)
		}

		if index < len(data)-1 {
			buf.WriteRune(',')
		}
	}

	buf.WriteRune(']')
	return buf.Bytes(), nil
}

func MarshalJSONE(data bson.E) ([]byte, error) {
	var b []byte
	buf := bytes.NewBuffer(b)

	// write key
	b, e := jsoniter.Marshal(data.Key)
	if e != nil {
		return nil, e
	}
	buf.Write(b)

	// write delimiter
	buf.WriteRune(':')

	// write value
	switch v := data.Value.(type) {
	case bson.D:
		b, e = MarshalJSOND(v)
		if e != nil {
			return nil, e
		}
		buf.Write(b)
	case bson.E:
		b, e = MarshalJSONE(v)
		if e != nil {
			return nil, e
		}
		buf.Write(b)
	case primitive.Binary:
		var bs string
		switch v.Subtype {
		case 0:
			bs = base64.StdEncoding.EncodeToString(v.Data)

		case 4:
			bs, e = binaryToUUID(v)
			if e != nil {
				return nil, e
			}
		default:
			bs, e = binaryToUUID(v)
			if e != nil {
				return nil, e
			}
		}
		b, e = jsoniter.Marshal(bs)
		if e != nil {
			return nil, e
		}
		buf.Write(b)
	case primitive.Timestamp:
		b, e = jsoniter.Marshal(TimestampToTime(v))
		if e != nil {
			return nil, e
		}
		buf.Write(b)
	case primitive.DateTime:
		b, e = jsoniter.Marshal(DateTimeToTime(v))
		if e != nil {
			return nil, e
		}
		buf.Write(b)
	case primitive.A:
		b, e = MarshalJSONA(v)
		if e != nil {
			return nil, e
		}
		buf.Write(b)
	default:
		b, e = jsoniter.Marshal(v)
		if e != nil {
			return nil, e
		}
		buf.Write(b)
	}

	return buf.Bytes(), nil
}

func binaryToUUID(bin primitive.Binary) (string, error) {
	uuidBytes := make([]byte, 16)
	copy(uuidBytes, bin.Data)
	uuid, err := uuid.FromBytes(uuidBytes)
	if err != nil {
		return "", err
	}

	return uuid.String(), nil
}

func TimestampToTime(timestamp primitive.Timestamp) time.Time {
	t := time.Unix(int64(timestamp.T), int64(timestamp.I)*int64(time.Millisecond))
	return t
}

func DateTimeToTime(dateTime primitive.DateTime) time.Time {
	t := time.Unix(0, int64(dateTime)*int64(time.Millisecond)*int64(time.Second))
	return t
}
```

测试代码:

```go
func TestXxx(t *testing.T) {
	log := bson.D{
		{
			Key:   "createUser",
			Value: "admin",
		},
		{
			Key:   "pwd",
			Value: "12343567",
		},
	}

	rest := BsonSearchD(log, "admin")
	assert.True(t, rest)
}
```

## 小尾巴

> 老花: 今天关于`BSON`数据的解析就到这里了, 我们下期再见!
