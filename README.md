我是光年实验室高级招聘经理。
我在github上访问了你的开源项目，你的代码超赞。你最近有没有在看工作机会，我们在招软件开发工程师，拉钩和BOSS等招聘网站也发布了相关岗位，有公司和职位的详细信息。
我们公司在杭州，业务主要做流量增长，是很多大型互联网公司的流量顾问。公司弹性工作制，福利齐全，发展潜力大，良好的办公环境和学习氛围。
公司官网是http://www.gnlab.com,公司地址是杭州市西湖区古墩路紫金广场B座，若你感兴趣，欢迎与我联系，
电话是0571-88839161，手机号：18668131388，微信号：echo 'bGhsaGxoMTEyNAo='|base64 -D ,静待佳音。如有打扰，还请见谅，祝生活愉快工作顺利。

## dynamo [![GoDoc](https://godoc.org/github.com/guregu/dynamo?status.svg)](https://godoc.org/github.com/guregu/dynamo) [![Circle CI](https://circleci.com/gh/guregu/dynamo.svg?style=svg)](https://circleci.com/gh/guregu/dynamo)
`import "github.com/guregu/dynamo"` 

dynamo is an expressive [DynamoDB](https://aws.amazon.com/dynamodb/) client for Go, with an API heavily inspired by [mgo](https://labix.org/mgo). dynamo integrates with the official [AWS SDK](https://github.com/aws/aws-sdk-go/).

dynamo is still under development, so the API may change rarely. However, breaking changes will be avoided and the API can be considered relatively stable.

### Example

```go
package dynamo

import (
	"time"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/guregu/dynamo"
)

// Use struct tags much like the standard JSON library,
// you can embed anonymous structs too!
type widget struct {
	UserID int       // Hash key, a.k.a. partition key
	Time   time.Time // Range key, a.k.a. sort key
	
	Msg       string   `dynamo:"Message"`
	Count     int      `dynamo:",omitempty"`
	Friends   []string `dynamo:",set"` // Sets 
	SecretKey string   `dynamo:"-"`    // Ignored
	Children  []any    // Lists
}

func main() {
	db := dynamo.New(session.New(), &aws.Config{Region: aws.String("us-west-2")})
	table := db.Table("Widgets")
	
	// put item
	w := widget{UserID: 613, Time: time.Now(), Msg: "hello"}
	err := table.Put(w).Run() 
	
	// get the same item 
	var result widget
	err = table.Get("UserID", w.UserID).
				Range("Time", dynamo.Equal, w.Time).
				Filter("'Count' = ? AND $ = ?", w.Count, "Message", w.Msg). // placeholders in expressions
				One(&result)
	
	// get all items
	var results []widget
	err = table.Scan().All(&results)
	// ...
}
```

### Expressions

dynamo will help you write expressions used to filter results in queries and scans, and add conditions to puts and deletes. 

Attribute names may be written as is if it is not a reserved word, or be escaped with single quotes (`''`). You may also use dollar signs (`$`) as placeholders for attribute names. DynamoDB has [very large amount of reserved words](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/ReservedWords.html) so it may be a good idea to just escape everything.

Question marks (`?`) are used as placeholders for attribute values. DynamoDB doesn't have value literals, so you need to substitute everything.

Please see the [DynamoDB reference on expressions](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.SpecifyingConditions.html#ConditionExpressionReference) for more information.

```go
// Using single quotes to escape a reserved word, and a question mark as a value placeholder.
// Finds all items whose date is greater than or equal to lastUpdate.
table.Scan().Filter("'Date' >= ?", lastUpdate).All(&results)

// Using dollar signs as a placeholder for attribute names. 
// Deletes the item with an ID of 42 if its score is at or below the cutoff, and its name starts with G.
table.Delete("ID", 42).If("Score <= ? AND begins_with($, ?)", cutoff, "Name", "G").Run()

// Put a new item, only if it doesn't already exist.
table.Put(item{ID: 42}).If("attribute_not_exists(ID)").Run()
```

### Encoding support

dynamo automatically handles the following interfaces:

* [`dynamo.Marshaler`](https://godoc.org/github.com/guregu/dynamo#Marshaler) and [`dynamo.Unmarshaler`](https://godoc.org/github.com/guregu/dynamo#Unmarshaler) 
* [`dynamodbattribute.Marshaler`](https://godoc.org/github.com/aws/aws-sdk-go/service/dynamodb/dynamodbattribute#Marshaler) and [`dynamodbattribute.Unmarshaler`](https://godoc.org/github.com/aws/aws-sdk-go/service/dynamodb/dynamodbattribute#Unmarshaler)
* [`encoding.TextMarshaler`](https://godoc.org/encoding#TextMarshaler) and [`encoding.TextUnmarshaler`](https://godoc.org/encoding#TextUnmarshaler)

This allows you to define custom encodings and provides built-in support for types such as `time.Time`. 

### Compatibility with the official AWS library

dynamo has been in development before the official AWS libraries were stable. We use a different encoder and decoder than the [dynamodbattribute](https://godoc.org/github.com/aws/aws-sdk-go/service/dynamodb/dynamodbattribute) package. dynamo uses the `dynamo` struct tag instead of the `dynamodbav` struct tag, and we also prefer to automatically omit invalid values such as empty strings, whereas the dynamodbattribute package substitutes null values for them. Items that satisfy the [`dynamodbattribute.(Un)marshaler`](https://godoc.org/github.com/aws/aws-sdk-go/service/dynamodb/dynamodbattribute#Marshaler) interfaces are compatibile with both libraries.

In order to use dynamodbattribute's encoding facilities, you must wrap objects passed to dynamo with [`dynamo.AWSEncoding`](https://godoc.org/github.com/guregu/dynamo#AWSEncoding). Here is a quick example: 

```go
// Notice the use of the dynamodbav struct tag
type book struct {
	ID    int    `dynamodbav:"id"`
	Title string `dynamodbav:"title"`
}
// Putting an item
err := db.Table("Books").Put(dynamo.AWSEncoding(book{
	ID:    42,
	Title: "Principia Discordia",
})).Run()
// When getting an item you MUST pass a pointer to AWSEncoding!
var someBook book
err := db.Table("Books").Get("ID", 555).One(dynamo.AWSEncoding(&someBook))
```

### Integration tests

By default, tests are run in offline mode. Create a table called `TestDB`, with a Number Parition Key called `UserID` and a String Sort Key called `Time`. Change the table name with the environment variable `DYNAMO_TEST_TABLE`. You must specify `DYNAMO_TEST_REGION`, setting it to the AWS region where your test table is.

 ```bash
DYNAMO_TEST_REGION=us-west-2 go test github.com/guregu/dynamo/... -cover
 ``` 

### License

BSD
