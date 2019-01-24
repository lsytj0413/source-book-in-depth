# 使用goast为xorm生成建表sql语句 #

## 缘起 ##

最近在工作学习的过程中，接触到一些通过自动生成代码的方式来减少重复的工作量，以及自动生成文档或桩代码的方式，包括:

- 使用[Kubernetes CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)，可以通过[code-generator](https://github.com/kubernetes/code-generator)工具自动生成客户端代码以及一些其他工具函数
- 使用[goa](https://github.com/goadesign/goa)，可以通过定义的DSL自动生成服务端的框架代码，以及文档等

一般有以下几种方式来自动生成代码:

- 通过 goast 获取代码的抽象语法树，然后通过抽象语法树来生成对应的代码
- 通过自定义DSL的方式获取代码信息，然后生成代码
- 通过反射获取信息来生成代码

正好最近在学习使用golang的ast解析工具，遂通过实现一个简单的工具来加深理解。该工具将自动读取xorm的类型信息，并自动生成对应的建表sql语句。

## 目标 ##

首先明确该工具的适用目标及范围:

- 只支持xorm框架
- 只支持少量的xorm框架特性: 包括created, updated, pk, unique, notnull
- 只支持少量的sql类型: 包括BIGINT, INT, VARCHAR, DATETIME
- 支持设定表名: 通过 +genTable spec
- 只支持mysql

测试数据如下:

```
package main

import "time"

// User is orm for user
// +genTable: user
type User struct {
	Id      int64
	Name    string `xorm:"unique notnull"`
	Salt    string
	Age     int
	Passwd  string    `xorm:"varchar(200)"`
	Created time.Time `xorm:"created"`
	Update  time.Time `xorm:"updated"`
}

/*
+genTable:
*/
// User2 and User3
type (
	// User2
	// +genTable: user2
	User2 struct { // user2 line
		Uid int `xorm:"pk 'uid'"`
	}

	// comment group

	// User3
	// +genTable:
	User3 struct { // user3 line
	}
)

type I1 interface {
}
```

最终输出的建表语句如下:

```
CREATE TABLE `user` (
	`Id` BIGINT,
	`Name` VARCHAR(255) NOT NULL,
	`Salt` VARCHAR(255),
	`Age` INT,
	`Passwd` varchar(200),
	`Created` DATETIME DEFAULT CURRENT_TIMESTAMP,
	`Update` DATETIME ON UPDATE CURRENT_TIMESTAMP,
	PRIMARY KEY(`Id`),
	UNIQUE KEY `Name` (`Name`)
);
CREATE TABLE `user2` (
	`uid` INT,
	PRIMARY KEY(`uid`)
);
```

## 实现 ##

### 1. 生成 ast ###

ast 即抽象语法树, go/parser 包提供了工具来解析生成ast:

```
fs := token.NewFileSet()
f, err := parser.ParseFile(fs, file, src, parser.ParseComments)
```

其中:

- 通过token.NewFileSet来生成一个FileSet对象，这个对象会保存更详细的源码位置信息
- file是文件名, 当src为nil时会读取该文件的内容；当src是字节数组/字符串时会直接将src的内容作为文件内容解析
- 需要通过ParseComments参数告知parser解析注释，因为我们的 +genTable 是通过注释实现的

### 2. 获取所有的table struct ###

在得到 ast 之后，我们需要筛选出对应的表的struct，首先定义一个结构体保存具体的信息:

```
type tableStruct struct {
        node      *ast.StructType
        tableName string
}
```

其中表名可能来自以下内容:

- 在多个类型上的 +genTable spec，这时不能跟表名，表示使用类型名作为表名，当有多个类型都直接使用类型名作为表名时可使用
- 在单个类型上的 +genTable spec，这时可以指定表名

#### 2.1 解析 +genTable spec ####

首先提供一个函数来获取 +genTable spec:

```
func isGenTableDoc(doc *ast.CommentGroup) (string, bool) {
        if doc == nil {
                return "", false
        }

        for _, comment := range doc.List {
                if comment == nil {
                        continue
                }

                value := strings.TrimSpace(comment.Text)
                if strings.HasPrefix(value, "//") {    // 如果是单行注释
                        value = strings.TrimSpace(value[2:])
                        if strings.HasPrefix(value, "+genTable:") {
                                return strings.TrimSpace(value[10:]), true
                        }
                } else {
                        lines := strings.Split(value, "\n")     // 多行注释，拆分后处理每一行
                        for _, line := range lines {
                                value := strings.TrimSpace(line)
                                if strings.HasPrefix(value, "+genTable:") {
                                        return strings.TrimSpace(value[10:]), true
                                }
                        }

                }
        }

        return "", false
}
```

该函数在获取到一个有效的 +genTable spec 之后即返回，并能处理多行注释的情况。

#### 2.2 对 ast 迭代获取所有 table struct ####

首先定义一个函数处理 *ast.File 对象:

```
func filterFileTableStruct(file *ast.File) ([]*tableStruct, error) {
        genTables := []*tableStruct{}

        for _, decl := range file.Decls {
                var n ast.Node = decl
                if n, ok := n.(*ast.GenDecl); ok {
                        tables, err := filterGenDeclTableStruct(n)
                        if err != nil {
                                return nil, err
                        }
                        genTables = append(genTables, tables...)
                }
        }

        return genTables, nil
}
```

该函数会迭代所有的Decls，并且只有在当类型为\*ast.GenDecl时才进一步解析。因为所有的类型定义都被解析为\*ast.GenDecl对象。

例如:

```
type S struct {}

type (
    S1 struct{}
    S2 struct{}
)
```

上面的每一个type关键字都被视为一个GenDecl对象。

然后处理每一个GenDecl对象，并解析出table struct:

```
func filterGenDeclTableStruct(n *ast.GenDecl) ([]*tableStruct, error) {
        tableName, isGenTable := isGenTableDoc(n.Doc)
        tables := []*tableStruct{}

        for _, spec := range n.Specs {
                if n, ok := spec.(*ast.TypeSpec); ok {
                        typeTableName, typeGenTable := isGenTableDoc(n.Doc)
                        if !typeGenTable && !isGenTable {
                                fmt.Println(n.Name, " doesn't have genTable spec, skip it")
                                continue
                        }
                        if tableName != "" && typeTableName != "" {
                                return nil, fmt.Errorf("%s has multi spec tableName", n.Name)
                        }
                        if tableName != "" && len(tables) > 0 {
                                return nil, fmt.Errorf("%s spec multi struct", n.Name)
                        }

                        structType, ok := n.Type.(*ast.StructType)
                        if !ok {
                                if typeTableName != "" {
                                        return nil, fmt.Errorf("%s genTable spec on %v", n.Name, reflect.TypeOf(n.Type).Elem().Name())
                                }
                                fmt.Println(n.Name, " isn't struct, skip it")
                                continue
                        }

                        if typeTableName == "" {
                                typeTableName = tableName
                        }
                        if typeTableName == "" {
                                typeTableName = n.Name.Name
                        }

                        tables = append(tables, &tableStruct{
                                tableName: typeTableName,
                                node:      structType,
                        })
                }
        }

        if isGenTable && len(tables) == 0 {
                return nil, fmt.Errorf("has genTable spec but on table exists")
        }

        return tables, nil
}
```

首先解析GenDecl的Doc注释，只有紧接着定义上的注释被视为Doc，而且因为单个的type关键字也被视为GenDecl，所以该Doc不会出现在TypeSpec上。

然后针对每一个TypeSpec进行处理，当TypeSpec是StructType时就保存下来。此处需要对genTable的有效性进行校验，并且如果没有设置自定义的表名时使用类型名作为表名。

### 3 从table struct中解析出对应的列 ###

在获得了 table struct 之后，从struct tag中解析出如下的字段对象:

```
type Column struct {
        pk      string
        unique  string
        notnull string
        created string
        updated string
        name    string
        t       string
}
```

首先迭代所有的ast.Field对象，生成对应的Column。

```
func genColumn(field *ast.Field) (*Column, error) {
	col := &Column{}

	if len(field.Names) > 1 {
		return nil, fmt.Errorf("%v  should only be one", field.Names)
	}

	if len(field.Names) == 1 {
		col.name = field.Names[0].Name
	} else {
		ident, ok := field.Type.(*ast.Ident)
		if !ok {
			return nil, fmt.Errorf("field doesn't have fieldNames, and type isn't Ident, skip it")
		}
		col.name = ident.Name
		col.t = col.name
	}

	tag := ""
	if field.Tag != nil {
		var err error
		tag, err = strconv.Unquote(field.Tag.Value)
		if err != nil {
			fmt.Println("unquote tag failed, %v", err)
			os.Exit(1)
		}
		structTag, ok := reflect.StructTag(tag).Lookup("xorm")
		if ok {
			tag = structTag
		} else {
			tag = ""
		}
	}

	tags := strings.Split(tag, " ")
	for _, tagField := range tags {
		if tagField == "" {
			continue
		}

		switch tagField {
		case "notnull":
			col.notnull = "NOT NULL"
		case "unique":
			col.unique = "UNIQUE KEY"
		case "pk":
			col.pk = "PRIMARY KEY"
		case "created":
			col.created = "DEFAULT CURRENT_TIMESTAMP"
		case "updated":
			col.updated = "ON UPDATE CURRENT_TIMESTAMP"
		case "int":
			col.t = "INT"
		case "bigint":
			col.t = "BIGINT"
		default:
			if strings.HasPrefix(tagField, "'") {
				if len(tagField) <= 2 {
					return nil, fmt.Errorf("%v used as fieldname but doesn't have value", tagField)
				}
				if !strings.HasSuffix(tagField, "'") {
					return nil, fmt.Errorf("%v used as fieldname but doesn't close quote", tagField)
				}
				tagField = tagField[1 : len(tagField)-1]
				col.name = tagField
			} else {
				if strings.HasPrefix(tagField, "varchar") {
					col.t = tagField
				} else {
					return nil, fmt.Errorf("unknown tag: %v", tagField)
				}
			}
		}
	}

	if col.t == "" {
		switch ident := field.Type.(type) {
		case *ast.Ident:
			switch ident.Name {
			case "int64":
				col.t = "BIGINT"
			case "time.Time":
				col.t = "DATETIME"
			case "string":
				col.t = "VARCHAR(255)"
			case "int":
				col.t = "INT"
			default:
				return nil, fmt.Errorf("%v type to sqltype failed, unknown: %v", col.name, ident.Name)
			}
		case *ast.SelectorExpr:
			if ident.Sel.Name == "Time" {
				if ident, ok := ident.X.(*ast.Ident); ok {
					if ident.Name == "time" {
						col.t = "DATETIME"
					}
				}
			}
		default:
			return nil, fmt.Errorf("%v field type unknown: %v", col.name, reflect.TypeOf(field.Type).Elem().Name())
		}
	}

	if col.t == "" {
		return nil, fmt.Errorf("%v doesn't have type", col.name)
	}

	return col, nil
}
```

- 当指定了列名时使用指定的值，否则使用字段名，对于匿名字段则使用类型名
- 不支持单行定义多个字段
- 当指定了列类型时使用指定的值，否则使用字段类型对应的sql类型, 其中time.Time默认对应DATETIME类型

然后返回所有的列:

```
func genColumns(table *tableStruct) ([]*Column, error) {
        columns := []*Column{}

        for _, field := range table.node.Fields.List {
                col, err := genColumn(field)
                if err != nil {
                        return nil, err
                }
                columns = append(columns, col)
        }

        for _, col := range columns {
                if col.name == "Id" && col.t == "BIGINT" {
                        col.pk = "PRIMARY KEY"
                }
        }

        if 0 == len(columns) {
                return nil, fmt.Errorf("no column")
        }

        return columns, nil
}
```

这里有如下要求:

- 一个表至少要有一个列
- 如果有列名为Id且类型为BIGINT则默认作为主键

### 4 生成sql语句 ###

在获取了表名以及列属性列表之后，生成sql语句:

```
func genSql(table *tableStruct) (string, error) {
	columns, err := genColumns(table)
	if err != nil {
		return "", err
	}

	// 开始生成sql
	builder := strings.Builder{}
	builder.WriteString("CREATE TABLE `")
	builder.WriteString(table.tableName)
	builder.WriteString("` (")

	for i, col := range columns {
		builder.WriteString("\n\t`")
		builder.WriteString(col.name)
		builder.WriteString("` ")
		builder.WriteString(col.t)
		if col.notnull != "" {
			builder.WriteString(" NOT NULL")
		}
		if col.created != "" {
			builder.WriteString(" " + col.created)
		}
		if col.updated != "" {
			builder.WriteString(" " + col.updated)
		}

		if i != len(columns)-1 {
			builder.WriteString(",")
		}
	}

	for _, col := range columns {
		if col.pk != "" {
			builder.WriteString(",\n\t")
			builder.WriteString("PRIMARY KEY(`")
			builder.WriteString(col.name)
			builder.WriteString("`)")
		}
		if col.unique != "" {
			builder.WriteString(",\n\t")
			builder.WriteString("UNIQUE KEY `")
			builder.WriteString(col.name)
			builder.WriteString("` (`")
			builder.WriteString(col.name)
			builder.WriteString("`)")
		}
	}

	builder.WriteString("\n);")
	return builder.String(), nil
}
```

## 总结 ##

可以看到，通过goast我们可以提取非常详细的源码信息，但是相应的，这种操作具有相当的复杂度，并且不如反射能得到更多的运行时信息。

# 参考资料 #

- [The ultimate guide to writing a Go tool](https://arslan.io/2017/09/14/the-ultimate-guide-to-writing-a-go-tool/)