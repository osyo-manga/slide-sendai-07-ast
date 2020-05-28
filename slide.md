### RubyVM::AbstractSyntaxTree を
### 試してみた話

---

#### 自己紹介
- - -

* 名前：osyo
* Twitter : [@pink_bangbi](https://twitter.com/pink_bangbi)
* github  : [osyo-manga](https://github.com/osyo-manga)
* ブログ  : [Secret Garden(Instrumental)](http://secret-garden.hatenablog.com)
* Rails 歴 約 2年
* 趣味で Ruby にパッチを投げたりしてます
  * Ruby 2.7 に [Time#floor](https://bugs.ruby-lang.org/issues/15653) / [Time#ceil](https://bugs.ruby-lang.org/issues/15772) を追加したり

---

## 今日話すこと
### RubyVM::AbstractSyntaxTree を使って Ruby の解析をしてみる

---

#### RubyVM::AbstractSyntaxTree とは
- - -

* Ruby のソースコードから AbstractSyntaxTree を生成するライブラリ
* AbstractSyntaxTree とは抽象構文木と呼ばれるもの
* 実際見てもらったほうが早いので早速使ってみる

---

#### メソッド呼び出し
- - -

```ruby
# Ruby のコードをパースする
ast = RubyVM::AbstractSyntaxTree.parse("func()")

# おまじない
node = ast.children.last

# node の中身
pp node

# node のタイプ
pp node.type

# node の子、タイプによって形態が異なる
pp node.children
# => [:func, nil]

# メソッド呼び出しなら以下みたいな構造
name, args = node.children
pp name    # => :func
pp args    # => nil
```
---


#### 引数を追加したメソッド呼び出し
- - -

```ruby
# Ruby のコードをパースする
ast = RubyVM::AbstractSyntaxTree.parse("func(1)")

# おまじない
node = ast.children.last

pp node
pp node.type
# => :VCALL
pp node.children
# => [:func, nil]

# メソッド呼び出しなら以下みたいな構造
name, args = node.children
pp name    # => :func
pp args    # => nil

# 引数の中身
pp args.type
# => :LIST
pp args.children
# => [(LIT@1:5-1:6 1), nil]
```
---


#### 演算子を含むコードの場合
- - -

```ruby
# Ruby のコードをパースする
ast = RubyVM::AbstractSyntaxTree.parse("1 + 2")
node = ast.children.last

pp node
# => (OPCALL@1:0-1:5 (LIT@1:0-1:1 1) :+ (LIST@1:4-1:5 (LIT@1:4-1:5 2) nil))

# node のタイプ
pp node.type
# => :OPCALL

# node の子、タイプによって形態が異なる
pp node.children
# => [(LIT@1:0-1:1 1), :+, (LIST@1:4-1:5 (LIT@1:4-1:5 2) nil)]

# こんな感じで各要素を分割して受け取れる
(left, op, right) = node.children
pp left    # => (LIT@1:0-1:1 1)
pp op      # => :+
pp right   # => (LIST@1:4-1:5 (LIT@1:4-1:5 2) nil)
```

---

#### ブロックを対象としてパースする
- - -

```ruby
# Ruby のコードをパースする
# ブロックを渡す場合は .of を使う
ast = RubyVM::AbstractSyntaxTree.of(-> { 1 + 2 })

node = ast.children.last
pp node
# => (OPCALL@1:0-1:5 (LIT@1:0-1:1 1) :+ (LIST@1:4-1:5 (LIT@1:4-1:5 2) nil))
pp node.type
# => :OPCALL
pp node.children
# => [(LIT@1:0-1:1 1), :+, (LIST@1:4-1:5 (LIT@1:4-1:5 2) nil)]

(left, op, right) = node.children
pp left    # => (LIT@1:0-1:1 1)
pp op      # => :+
pp right   # => (LIST@1:4-1:5 (LIT@1:4-1:5 2) nil)
```
---

#### AST から Ruby のコードを生成してみる
- - -

```ruby
def unparse(ast)
  case ast&.type
  when :FCALL
    name, *args = ast.children
    "#{name}(#{args.map(&method(:unparse)).join(", ")})"
  when :OPCALL
    left, op, right = ast.children
    "#{unparse(left)} #{op} #{unparse(right)}"
  when :LIT
    value = ast.children.first
    "#{value}"
  when :LIST
    list = ast.children
    list.map(&method(:unparse)).compact.join(", ")
  else
    nil
  end
end

ast = RubyVM::AbstractSyntaxTree.parse("func 1 + 2")
ast = ast.children.last

pp unparse(ast)
# => func(1 + 2)
```
---

#### まとめ
- - -

* Ruby の気持ちがわかってきた
  * どういう命令でどう変換しているのか
* ブロックの中身を文字列に変換できるの便利
