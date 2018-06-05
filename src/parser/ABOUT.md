---
title: about
date: 2018.6.5
---
# about Parser.jj
这个文件主要分为两个部分，一个是**词法分析部分**，另外一个是**语法树生成**部分。

虽然说是两个部分，但是不能简单的将其看成两个独立的部分，词法分析部分的每一次的结果都用于构建语法树。

而且这一份文件中也引用了其他的Java类，说明这里的语法树节点是在其他的文件夹里面定义的，本项目中
使用AST文件夹来包含所有的语法树节点，这里只有一点要注意，即本文件中使用的语法树节点的构造函数
要出现了对应的类中。

这份文件相当于是一份JavaCC和Java的混合文件。

# 作者的说法
本文分析的程序表现出来的是**定义**的集合。

而向perl和ruby可以直接在命令行中写出来的程序表现为**语句**的集合。

# 语法的单位
1. 定义(definition):
2. 声明(declaration):
3. 语句(statement):
4. 表达式(expression):是比语句小，具有值的语法单位。
5. 项(item):项时表达式中构成二元运算的一方，也就是仅由一元运算符构成的语法。

> 定义包含语句，语句包含表达式，表达式包含项。

# 容易产生疑惑的语法结构
## Slot结构
```
// #@@range/defstruct{
StructNode defstruct():
{
    Token t;
    String n;
    List<Slot> membs;
}
{
    t=<STRUCT> n=name() membs=member_list() ";"
        {
            return new StructNode(location(t), new StructTypeRef(n), n, membs);
        }
}
// #@@}

// #@@range/member_list{
List<Slot> member_list():
{
    List<Slot> membs = new ArrayList<Slot>();
    Slot s;
}
{
    "{" (s=slot() ";" { membs.add(s); })* "}"
        {
            return membs;
        }
}
// #@@}

// #@@range/slot{
Slot slot():
{
    TypeNode t;
    String n;
}
{
    t=type() n=name() { return new Slot(t, n); }
}
// #@@}
```
可以看出来，Slot表示类型`(type)`和名字`(name)`的集合体，严格来说这里并不算是声明。

## type和typeref结构
```
// #@@range/type{
TypeNode type():
{ TypeRef ref; }
{
    ref=typeref() { return new TypeNode(ref); }
}
// #@@}

// #@@range/typeref{
TypeRef typeref():
{
    TypeRef ref;
    Token t;
    ParamTypeRefs params;
}
{
    ref=typeref_base()
    ( LOOKAHEAD(2)
      "[" "]"
        {
            ref = new ArrayTypeRef(ref);
        }
    | "[" t=<INTEGER> "]"
        {
            ref = new ArrayTypeRef(ref, integerValue(t.image));
        }
    | "*"
        {
            ref = new PointerTypeRef(ref);
        }
    | "(" params=param_typerefs() ")"
        {
            ref = new FunctionTypeRef(ref, params);
        }
    )*
        {
            return ref;
        }
}
// #@@}

// #@@range/typeref_base{
TypeRef typeref_base():
{
    Token t, name;
}
{
      t=<VOID>          { return new VoidTypeRef(location(t)); }
    | t=<CHAR>          { return IntegerTypeRef.charRef(location(t)); }
    | t=<SHORT>         { return IntegerTypeRef.shortRef(location(t)); }
    | t=<INT>           { return IntegerTypeRef.intRef(location(t)); }
    | t=<LONG>          { return IntegerTypeRef.longRef(location(t)); }
    | LOOKAHEAD(2) t=<UNSIGNED> <CHAR>
        { return IntegerTypeRef.ucharRef(location(t)); }
    | LOOKAHEAD(2) t=<UNSIGNED> <SHORT>
        { return IntegerTypeRef.ushortRef(location(t)); }
    | LOOKAHEAD(2) t=<UNSIGNED> <INT>
        { return IntegerTypeRef.uintRef(location(t)); }
    | t=<UNSIGNED> <LONG>
        { return IntegerTypeRef.ulongRef(location(t)); }
    | t=<STRUCT> name=<IDENTIFIER>
        { return new StructTypeRef(location(t), name.image); }
    | t=<UNION> name=<IDENTIFIER>
        { return new UnionTypeRef(location(t), name.image); }
    | LOOKAHEAD({isType(getToken(1).image)}) name=<IDENTIFIER>
        { return new UserTypeRef(location(name), name.image); }
}
// #@@}
```
从这里可以看出`type`和`typeref`之间的对应关系，同时这里的`param_typerefs()`时将函数的变量名去掉以后的类型，比如`fun(int, int, char)`类型。

## 项(item)的分析
```
// #@@range/unary{
ExprNode unary():
{
    ExprNode n;
    TypeNode t;
}
{
      "++" n=unary()    { return new PrefixOpNode("++", n); }
    | "--" n=unary()    { return new PrefixOpNode("--", n); }
    | "+" n=term()      { return new UnaryOpNode("+", n); }
    | "-" n=term()      { return new UnaryOpNode("-", n); }
    | "!" n=term()      { return new UnaryOpNode("!", n); }
    | "~" n=term()      { return new UnaryOpNode("~", n); }
    | "*" n=term()      { return new DereferenceNode(n); }
    | "&" n=term()      { return new AddressNode(n); }
    | LOOKAHEAD(3) <SIZEOF> "(" t=type() ")"
        {
            return new SizeofTypeNode(t, size_t());
        }
    | <SIZEOF> n=unary()
        {
            return new SizeofExprNode(n, size_t());
        }
    | n=postfix()       { return n; }
}
// #@@}
```
从这一段代码里面也可以看见，并不是只有一个`-`号才是一个一元运算，取反，解引用都是一元运算。其中，`postfix`语法结构如下：
```
// #@@range/postfix{
ExprNode postfix():
{
    ExprNode expr, idx;
    String memb;
    List<ExprNode> args;
}
{
    expr=primary()
    ( "++"                  { expr = new SuffixOpNode("++", expr); }
    | "--"                  { expr = new SuffixOpNode("--", expr); }
    | "[" idx=expr() "]"    { expr = new ArefNode(expr, idx); }
    | "." memb=name()       { expr = new MemberNode(expr, memb); }
    | "->" memb=name()      { expr = new PtrMemberNode(expr, memb); }
    | "(" args=args() ")"   { expr = new FuncallNode(expr, args); }
    )*
        {
            return expr;
        }
}
// #@@}
```
