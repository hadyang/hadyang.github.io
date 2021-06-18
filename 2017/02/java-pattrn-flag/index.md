
正则表达式是很强大的，今天先来总结一下Java正则表达式的几种匹配模式。

<!--more-->

### Java Pattern的模式

  - `UNIX_LINES`：Unix格式，在这种模式下，`\n`能够通过`.`,`^`,`$`的匹配

  - `CASE_INSENSITIVE`：大小写敏感，`CASE_INSENSITIVE`只能使`US-ASCII`码里的字符大小写敏感

  - `COMMENTS`：注释模式，这种模式下会忽略空白，以及以`#`开头的注释

  - `MULTILINE`：开启多行模式，在这种模式下`^`,`$`会匹配每一行的开始和结束。在默认情况下`^`,`$`只匹配输入的开始和结束

  - `LITERAL`：字面值模式，在这个模式下正则表达式的 *转义字符* 和 *元字符* 都不会有特殊的意义，只表示字面值。当此标记与`CASE_INSENSITIVE` 和 `UNICODE_CASE`之外的标记一起使用时不会有效果。

  - `DOTALL`：在此模式下`.`可以匹配任意字符，包括换行符。在默认模式下`.`不能匹配换行符

  - `UNICODE_CASE`：Unicode字符大小写敏感，开启此标记可能会降低性能。

  - `CANON_EQ`：Unicode等价性，在此模式下只要两个字符在Unicode的等价规则下相等就会匹配，比如`a\u030A`和 `\u00E5`是等价的都表示字符`å`。关于Unicode等价规则参见[百度百科](http://baike.baidu.com/link?url=SQe6OOJCjKy4uqcb4ZCkYtJi6WoLU-WpXiwGU6wIRCzhP7U61-NPgCygRWflmJr-hBTAeMJPCLwZliSpdIxcs60LKiUbqdNAE8flP1BbP8bQxXDQBd7sE3ddp4lUJv5Yah6otQrynm7vf5RvnT6rqK)
