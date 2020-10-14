### Go语言

------

#### 1、omitempty的使用

omitempty作用是在json数据结构转换时，当该字段的值为该字段类型的零值时，忽略该字段。例如为false，0，零指针，nil接口值以及任何空数组，切片，映射或字符串，则该字段应从编码中省略。举个例子：

```go
// 场景一 类型不加omitempty
type stu struct {
	Name string `json:"name,omitempty"`
	Age  int    `json:"age,omitempty"`
}
// 场景二 int、string 类型不加omitempty
type stu struct {
	Name string `json:"name"`
	Age  int    `json:"age"`
}

func main() {
	data := stu{
		Name: "",
	}
	fmt.Printf("start,%v\n", data)
	// 有omitempty { 0}
	// 没有omitempty { 0}
	res, err := json.Marshal(data)
	if err != nil {
		fmt.Printf("%v\n", err)
	}
	fmt.Printf("marshall,%s\n", res)
	// BAD!!有omitempty {} 优点：““以及0的字段不显示, 缺陷：如果填了有意义的"",则不会显示
	// BAD!!没有omitempty {"name":"","age":0} ““以及0的字段显示，如果没填会显示默认值
	var req = new(stu)
	err = json.Unmarshal(res, req)
	if err != nil {
		fmt.Printf("%v\n", err)
	}
	fmt.Printf("unmarshall,%v\n", req)
	// 有omitempty { 0}
	// 没有omitempty { 0}
}
```

但是这个时候，如果我们想要Name为""时就是有意义，即要显示出来。如果我不填这个字段，这个字段不显示。也就是要区分空字符串到底是没传这个字段还是就是填了一个"",那这个时要怎么做呢?

答案就是把字段定义成指针类型

```go
type stu struct {
	Name *string `json:"name,omitempty"`
	Age  *int    `json:"age,omitempty"`
}
func main() {
	a := ""
	data := stu{
		Name: &a,
	}
	fmt.Printf("start,%v\n", data)
	// 有omitempty {0xc00003a260 <nil>}
	// 没有omitempty {0xc00003a260 <nil>}
	res, err := json.Marshal(data)
	if err != nil {
		fmt.Printf("%v\n", err)
	}
	fmt.Printf("marshall,%s\n", res)
	// GOOD!!有omitempty {"name":""} 我们填的空依旧显示，但是没填的age没有显示，说明name是填了“” 
	// BAD!!没有omitempty {"name":"","age":null} 优点：name填了空显示，没填的会显示默认值 缺陷：无法区分name的“”是没填这个值还是填了一个“”
	var req = new(stu)
	err = json.Unmarshal(res, req)
	if err != nil {
		fmt.Printf("%v\n", err)
	}
	fmt.Printf("unmarshall,%v\n", req)
}
```

定义成指针我们就可以识别空串，0等空值