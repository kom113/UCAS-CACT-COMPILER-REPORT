[TOC]

# 编译原理研讨课实验PR002实验报告

## 任务说明

本次实验主要有以下两个任务：

* **任务一：根据给出的CACT文法和语义规范，进一步完善`.g4`文件，通过`Antlr`生成的访问语法树的接口，对输入的CACT语言源代码（`.cact`文件）进行语义分析**
* **任务二：遇到语义错误时，进行相应的处理**

本次实验需要在 PR001 的基础上实现对 CACT 语言进行语义分析和类型检查，使得对于语义错误能进行相应的处理，即对于符合语义规范的` .cact `文件，编译器返回 0，对于有语义错误的` .cact `文件，编译器返回非 0 值。

## 成员组成

夏瑞阳、陈兢宇、余秋初

## 实验设计与实现

### 任务一的设计与实现

为了进行语义分析，我们需要遍历AST，这里需要用到的数据结构是符号表，因此我们需要首先完成符号表设计。

#### 符号表

设计符号表首先需要设计符号表项，而符号项目本质上记录的是名字`(string)`和内容`entry_content`的binding。基于此，我们认为符号表项主要有两大类：`NameEntry`和`FuncEntry`，它们都继承自`BaseSymbolTableEntry`的基类。

`BaseSymbolTableEntry`类的设计如下：

```C++
class BaseSymbolTableEntry                                                      //符号表项的基类          
{
    private:
        entry_type etype;                                                       	//符号表项的类型，用于输出调试信息
    
    public:
        BaseSymbolTableEntry() { }
        BaseSymbolTableEntry(entry_type e) : etype(e) { }
        virtual ~BaseSymbolTableEntry() { }

        /* get & set 方法，子类中不再赘述 */
        entry_type      getEntryType()              const       {return etype;}     //获取符号表项类型
        void            setEntryType(const entry_type& e)       {etype = e;}        //设置符号表项类型
};
```

在这一基类里，我们定义了一个私有成员`etype`，表示当前符号表项的类型。这里的虚析构函数是必要的，它使得我们能够在一个容器中能够同时保存基类指针和子类指针。

`NameEntry`类的设计如下：

```C++
class NameEntry : public BaseSymbolTableEntry                                   //由于.g4文法中关于变量声明和常量声明的语法重合度很高，用一个子类来记录AST中变量/常量信息
{
    private:
        param_type ptype;                                                       //参数类型
        bool isconst;                                                           //变量/常量
        int array_len;                                                          //数组长度

    public:
        NameEntry(entry_type e, param_type b, bool ifconst, int array_length = 0) : BaseSymbolTableEntry(e), ptype(b), isconst(ifconst), array_len(array_length) { }  
        void                setParamType(const param_type& type)    {ptype = type;}
        param_type          getParamType()                 const    {return ptype;}
        bool                getIsConst()                   const    {return isconst;}
        void                setIsConst(bool ifconst)                {isconst = ifconst;}
        int                 getArrayLen()                  const    {return array_len;}
        void                SetArrayLen(int len)                    {array_len = len;}
        int                 getIsArray()                   const    {return paramIsArray(ptype);}
};
```

`NameEntry`类记录了除函数名字之外的符号表项（变量/常量/数组），变量和常量信息记录在`isconst`成员里，`ptype`成员记录了该名字的参数类型，数组长度由`array_len`给出。在PR002中，我们没有必要在符号表里记录变量/常量/数组的右值。

`FuncEntry`类的设计如下：

```C++
class FuncEntry : public BaseSymbolTableEntry                                   //函数项子类，用于记录AST中函数信息
{
    private:
        return_type                                          rtype;                                          		//函数返回值类型
        std::vector<std::pair<std::string, param_type>>           param_list;                                     	//函数的参数列表

    public:
        FuncEntry(entry_type e, return_type r, std::vector<std::pair<std::string, param_type>> p) : BaseSymbolTableEntry(e), rtype(r), param_list(p) { }
        void                                setReturnType(const return_type& type)         {rtype = type;}
        void                                setParamList (const std::vector<std::pair<std::string, param_type>>& param)   {param_list = param;}
        return_type                         getReturnType()                         const  {return rtype;}
        std::vector<std::pair<std::string, param_type>>             getParamList()  const  {return param_list;}
};
```

对于函数，我们需要记录函数的参数列表、返回值类型，因此我们需要将函数名和函数名之外的名字区分开来单独作为另一个子类。

基于以上的符号表项的设计，我们对符号表类的设计如下：

```C++
class SymbolTable
{
    private:
        std::shared_ptr<SymbolTable> parent;                                                    //嵌套的上一层作用域
        std::map<std::string , std::shared_ptr<BaseSymbolTableEntry>> var_table[HASH_8BITS];    //变量/常量 符号表
        std::map<std::string , std::shared_ptr<BaseSymbolTableEntry>> func_table[HASH_8BITS];   //函数符号表
        bool iswhile;                                                                           //是否处于循环体作用域
    public:
        SymbolTable(std::shared_ptr<SymbolTable> upper_scope) : parent(upper_scope) { }

        void                                setUpperScope(const std::shared_ptr<SymbolTable>& scope) {parent = scope;}
        std::shared_ptr<SymbolTable>             getUpperScope()                               const {return parent;}
        void                                     setIsWhile(bool value) {iswhile = value;}
        bool                                     getIsWhile()                               const {return iswhile;}
        std::shared_ptr<BaseSymbolTableEntry>    lookup(const std::string& identifier, entry_type etype);                                           //符号表的查询操作
        std::shared_ptr<BaseSymbolTableEntry>    insert(const std::string& name, std::shared_ptr<BaseSymbolTableEntry> entry);  //符号表的插入操作，失败返回nullptr
        void print();
};
```

对于符号表，我们首先需要解决的是符号表项的存放问题。由于我们的符号表项的子类都继承自`BaseSymbolTableEntry`的基类，如果只是在容器中存放基类对象，那么我们在做插入操作时会发生`Object Slicing`，导致子类存储的额外信息丢失。为了解决这一问题，我们把符号表设计成存放基类指针的`HashMap`。

在语义分析中，我们还需要处理嵌套作用域的问题。我们设计思路是：为每一个嵌套的作用域都配置一张符号表，并且在符号表里设置指向上层符号表（作用域）的指针。在进行符号表初始化时，我们会初始化当前作用域和全局作用域，当前作用域被设置为全局作用域。在进入一个新的`block`时，我们建立一张新的符号表，将它的上层作用域设置为当前作用域，最后将当前作用域指针指向这张新的符号表。这样，在遍历完语法树后，我们的符号表形成了树形的嵌套结构，和`cact`的嵌套作用域结构保持一致。

#### 语义分析

语义分析通过访问各个语法符号的结点实现。以stmt语法符号为例，有如下语法规则：
```ANTLR
stmt
    locals[
        bool iswhile
    ]
    : block                                     #stmtBlock
    ;
```
我们可以通过以下两个函数，在语法分析进入和离开stmt时，来调用并执行enterStmtBlock()或exitStmtBlock()函数。函数的参数指针ctx指向的是内存中构造出的stmt结点。stmt的locals属性（例如iswhile）是CACTParser::StmtBlockContext类型的一个成员，能通过ctx->iswhile的方式访问。stmt的下属节点（例如block）是CACTParser::StmtBlockContext类型的一个成员函数，能通过ctx->block()的方式访问，从而我们能在进入、离开stmt时进行需要的语义分析和检查。那么我们能将iswhile作为stmt的一个继承属性传递给block。下面是具体的实现：

```C++
void SemanticAnalysis::enterStmtBlock(CACTParser::StmtBlockContext * ctx){
    ctx->block()->iswhile = ctx->iswhile;
}
```
类似的，我们可以按照同样的方式完成对例如exp结点的expression_type或array_len等属性的求值。


### 任务二的设计与实现

任务二要求在遇到语义错误时，能够检查并返回非0值。事实上，这部分的工作往往是和语义分析同时完成的，即在结点的enter、exit函数中同时检查语义是否合法。下面将按照各个语义要求进行介绍。

#### 常量声明

对一个常量的声明，可能有下列非法情况：重复的常量名、常量的初值表达式类型与声明的类型不符。我们在进入constDef节点时会在当前作用域中查找Ident，若有重复项则说明常量重名（由于antlr的g4文件的优先级，关键字不会被文法分析认作Ident）。我们在离开constInitVal节点时，会检查常量表达式的类型和声明的类型是否相符。这里其实是检查以下两种可能：①基本类型的常量类型是否符合；②数组类型的常量表达式中元素类型是否相等且符合声明。值得指出的是，常量表达式的常量性可以由antlr的语法分析完成，无需语义分析中再次检查。

```C++
void SemanticAnalysis::enterConstDefNumber(CACTParser::ConstDefNumberContext* ctx)
{
    if (current_scope->lookup(ctx->Ident()->getText(), ENTRY_VAR)){
        printf("const name repeats.\n");
        exit(1);
    }
    ctx->basic_or_array_and_type = false;
}

void SemanticAnalysis::exitConstInitValNumber(CACTParser::ConstInitValNumberContext * ctx)
{
    ctx->ptype = ctx->constExp()->expression_type;
    
    if(ptype != ctx->ptype){
        printf("const expression type does not match decleration.\n");
        exit(1);
    }
}

void SemanticAnalysis::exitConstInitValArrayExist(CACTParser::ConstInitValArrayExistContext * ctx)
{
    ctx->ptype = ctx->constArrayAll()->ptype;

    if(ptype != ctx->ptype){
        printf("const array type does not match decleration.\n");
        exit(1);
    }
}
```

#### 变量声明

对一个变量的声明，可能有下列非法情况：重复的变量名、变量显式定义初值且类型与声明不符。我们在进入varDef节点时会在当前作用域中查找Ident，若有重复项则说明变量重名。在变量声明时也会用到constInitVal，因此这里复用其逻辑。

```C++
void SemanticAnalysis::enterVarDefIdentNumberDecl(CACTParser::VarDefIdentNumberDeclContext * ctx)
{
    if(current_scope->lookup(ctx->Ident()->getText(), ENTRY_VAR)){
        printf("var name repeats.\n");
        exit(1);
    }
    ctx->basic_or_array_and_type = false;
}
```

#### 函数定义

对一个函数的定义，可能有以下非法情况：重复的函数名、函数形参列表中有重复项。我们在进入funcDef节点时，类似常量变量中的操作，通过在符号表中查找func_name，来检查是否有重名的函数（因为CACT中不支持函数重载，因此只需检查func_name即可，不用考虑符号表）。同时在进入FuncFParams节点时，会检查每一个形参是否重名。值得一提的是，由于函数体内可能递归调用其本身，因此我们需要在语义分析进入funcDef对应的block之前，就将得到的函数项插入符号表中。

```C++
void SemanticAnalysis::enterFuncDefWithoutParam(CACTParser::FuncDefWithoutParamContext * ctx)
{
    param_list.clear();
    with_param = false;
    func_name = ctx->Ident()->getText();
    if(global_scope->lookup(ctx->Ident()->getText(), ENTRY_FUNC)){
        printf("func name repeats.\n");
        exit(1);
    }
    ctx->block()->iswhile = false;
}

void SemanticAnalysis::exitFuncFParamNumber(CACTParser::FuncFParamNumberContext* ctx)
{
    ctx->ptype = ptype;
    ctx->param_name = ctx->Ident()->getText();
    // 检查当前参数是否在param_list中有重复
    for(auto iter = param_list.begin(); iter != param_list.end(); iter++)
    {
        if(iter->first == ctx->param_name){
            printf("function param name repeats.\n");
            exit(1);
        }
    }
    param_list.push_back(pair<string, param_type>(ctx->param_name, ctx->ptype));
}
```

#### 常量、变量、函数调用

常量、变量和函数调用会作为表达式的一部分出现，可能有以下的非法情况：表达式中出现的常量或变量或函数未声明、函数调用的实参列表与符号表中记录的形参列表不相符（参数个数相同，且类型需一一对应）。我们可以通过表达式的ptype和array_len属性来进行判断。

```C++
void SemanticAnalysis::exitUnaryExpFuncWithParam(CACTParser::UnaryExpFuncWithParamContext * ctx)
{
    auto temp_entry = global_scope->lookup(ctx->Ident()->getText(), ENTRY_FUNC);
    auto temp_func = dynamic_pointer_cast<FuncEntry>(temp_entry);

    if(temp_entry == nullptr){
        printf("function has not been declared.\n");
        exit(1);
    }
    else{
        auto temp_param_list = temp_func->getParamList();
        if(temp_func->getParamList().empty()){
            printf("function param list is empty.\n");
            exit(1);
        }
        if(temp_param_list.size() != ctx->funcRParams()->exp().size())
        {
            printf("function param list length does not match.\n");
            exit(1);
        }
        for(int i = 0; i < temp_param_list.size(); i++)
        {
            if(ctx->funcRParams()->exp(i)->expression_type != temp_param_list[i].second)
            {
                printf("function param list type does not match.\n");
                exit(1);
            }
            i++;
        }
    }

    ctx->expression_type = return2Param(temp_func->getReturnType());
    ctx->isconst = true; // TODO: 实际上isconst和can_be_left_value有区别，例如func(a)不是一个const，但不能为左值，此处先模糊处理
}
```

或是数组的下标不是整数类型。

```C++
    // 省略前面的语句
    if(ctx->exp()->expression_type != PARAM_INT){
        printf("array index is not an integer.\n");
        exit(1);
    }
    ctx->ptype = array2Number(temp_entry->getParamType());
    ctx->isconst = temp_entry->getIsConst();
```

#### 由运算符和Ident组成的表达式

CACT中出现的算术运算符（+、-、*、/、%）、相等性运算符（==、!=）、关系运算符（>、<、>=、<=）和逻辑运算符（!、&&、||），其两端类型必须相同。按照运算符的优先级顺序从高到低如下表：

|运算符 |                       操作数类型                                       |
|-------|-----------------------------------------------------------------------|
|  =    | int int_array float flaot_array double double_array bool bool_array   |
|  \|\| | bool                                                                  |
|  &&   | bool                                                                  |
|==、!= | int float double bool                                                 |
|<、>、<=、>=| int float double                                                 |
| +、-  | int int_array float float_array double double_array                   |
| *、/  | int int_array float float_array double double_array                   |
| %     | int int_array                                                         |
| +、- (单目) | int float double                                                  |
| ! (单目) | bool                                                                 |

那么有可能有以下的非法情况：运算符两端类型不相同、运算符操作数为数组且其长度不相同、运算符不支持其操作数的类型（例如true + false）、赋值运算符（=）的左侧不是一个左值（例如不能对常量赋值），我们以加法/减法为例：

```C++
void SemanticAnalysis::exitAddExpAddMul(CACTParser::AddExpAddMulContext * ctx)
{
    if(ctx->addExp()->expression_type == PARAM_BOOL || ctx->addExp()->expression_type == PARAM_BOOLARRAY){
        printf("+ or -'s left side type is boolean.\n");
        exit(1);
    }
    if(ctx->mulExp()->expression_type == PARAM_BOOL || ctx->mulExp()->expression_type == PARAM_BOOLARRAY){
        printf("+ or -'s right side type is boolean.\n");
        exit(1);
    }
    if(ctx->addExp()->expression_type != ctx->mulExp()->expression_type){    
        printf("+ or -'s two sides types are different.\n");
        exit(1);
    }
    if(paramIsArray(ctx->addExp()->expression_type) && ctx->addExp()->array_len != ctx->mulExp()->array_len){
        printf("+ or -'s two sides array length are different.\n");
        exit(1);
    }
    ctx->expression_type = ctx->addExp()->expression_type;
    ctx->isconst = ctx->addExp()->isconst && ctx->mulExp()->isconst;
    if(paramIsArray(ctx->addExp()->expression_type)){
        ctx->array_len = ctx->addExp()->array_len;
    }
}
```

#### if、while语句

对于if、while语句，需要检查其cond是否是BOOL类型的表达式。

```C++
void SemanticAnalysis::exitStmtIf(CACTParser::StmtIfContext * ctx)
{
    if(ctx->cond()->expression_type != PARAM_BOOL){
        printf("statement expression is not boolean type.\n");
        exit(1);
    }    
}
```

#### break、continue语句

对于break、continue语句，需要检查其是否在一while语句内，我们通过stmt的iswhile属性来记录。

```C++
void SemanticAnalysis::exitStmtBreak(CACTParser::StmtBreakContext * ctx)
{
    if(!ctx->iswhile){
        printf("break is not in a while block.\n");
        exit(1);
    }
}
```

#### return语句

对于return语句，我们需要检查其返回的表达式类型与函数的返回值类型是否相符。

```C++
void SemanticAnalysis::exitStmtReturn(CACTParser::StmtReturnContext * ctx)
{
    if(rtype == RETURN_VOID){
        if(ctx->exp() != nullptr){
            printf("return exp occurs within the no-return-value function.\n");
            exit(1);            
        }
    }
    else{
        if(ctx->exp() == nullptr){
            printf("return exp should occur within the return-value function.\n");
            exit(1);
        }
        else if(ctx->exp()->expression_type != return2Param(rtype)){
            printf("return type is different from the function's return type.\n");
            exit(1);
        }
    }
}
```

#### main函数

作为语义分析的最后一步，我们需要检查程序中是否有且仅有一个main函数，并且main函数没有参数，返回值为int。

```C++
void SemanticAnalysis::exitCompUnit(CACTParser::CompUnitContext * ctx)
{
    auto temp = global_scope->lookup("main", ENTRY_FUNC);
    if (!temp) {
        printf("no main function.\n");
        exit(1);
    }
    else {
        auto temp_func = dynamic_pointer_cast<FuncEntry>(temp);
        auto temp_params = temp_func->getParamList();
        if (!temp_params.empty()){
            printf("main function's param list is not empty.\n");
            exit(1);
        }
        else if (temp_func->getReturnType() != RETURN_INT){
            printf("main function's return type is not integer.\n");
            exit(1);
        }
    }
}
```

## 总结

### 实验结果总结

实验代码能够对源程序进行语义分析，并通过48个测试点。

```C++
compiler27: 48 / 48
```





### 分成员总结

#### 余秋初的总结

在本次实验中，我负责了符号表以及符号表项的设计，参与了部分语义分析代码的编写与调试。这个实验给我的感觉就是思维量不大，理解清楚`Antlr`的运行机制后，结合理论课知识，对`cact`进行语义分析只需要根据语义规范逐一检查即可，但这个过程需要做到十分细致。整个实验带给我一点关于编译器实践启发是，我们没有必要把语义分析中所有要用到的属性都添加到注释语法树中，一些属性的传递完全可以通过使用临时变量的方法进行替代，这样可以减少AST所占据的内存空间。

#### 陈兢宇的总结


在本次实验中，我参与了符号表设计和语义分析的讨论。感觉本次实验比较琐碎，需要考虑的细节很多。通过本次实验，能够同之前的词法分析和语法分析结合起来，形成一个整体，使得对编译器的设计有了进一步的理解，进一步地学习了编译器内部的运行机制。除此之外，通过和同学的交流，对语法树的分析过程及方法有了新的理解，这对将语义检查落实到代码上有很大帮助。


#### 夏瑞阳的总结

在本次实验中，我参与了符号表的设计讨论，主要完成了语义分析代码的编写与调试。实验完成后，我理解清楚了`Antlr`的运行机制，并将其和理论课中的算法对应起来。此外，这也是我第一次参与多人完成代码的项目，深刻体会到了模块化设计、解耦等思想在多人完成代码时的必要性。





