

# 编译原理研讨课实验PR003实验报告

## 任务说明

1. 对通过词法、语法和语义分析的源程序，得到中间代码。
2. 从中间代码生成RISC-V GC汇编代码，得到.s文件。
3. 在中间代码和汇编代码的基础上进行优化变换，提高性能。



## 成员组成

陈兢宇、夏瑞阳、余秋初



## 实验设计与实现

#### 中间代码生成

##### 中间代码的设计

在PR003中，我们首先需要实现从PR002中得到的注释语法树到中间代码的生成。为此，首先需要设计和中间代码相关的数据结构。综合目标机性质和减少`codegen`模块工作量两方面因素，我们可以设计这样的中间代码，它有三个操作数：目的操作数、源操作数1、源操作数2，以及相应的操作码。因此，我们对于一条IR语句的数据结构设计如下：

```c++
struct Quad
{
    Opcode op;
    Var* arg1;
    Var* arg2;
    Var* dest;
    QuadDelete quad_delete;
    
	//...constructors...
    //...
};
   
using IRList = std::list<Quad>;
```

这里的`Opcode`是操作码相关的成员；`Var`是和IR语句操作数相关的成员；`quad_delete`用来表示当前IR语句是否已经被优化（删除）。由于在中间代码的优化过程中，会涉及大量中间代码的插入和删除操作，我们选择使用链表结构串联起所有的IR语句以节省计算开销。

对于操作码，我们希望它能够和RISC-V指令集相贴合，为此，我们对操作码有如下设计：

```c++
enum Opcode
{
    OP_ASSIGN,                      //  赋值语句                             
    OP_ASSIGN_GETVAL,               //  t0 = *t1           
    OP_ASSIGN_SETVAL,               //  *t0 = t1           
    OP_ASSIGNV,                     //  array1 = array2    
    OP_VARDEC,                      //  变量声明            
    OP_FUNCDEC,                     //  函数声明             
    OP_FUNCEND,                     //  函数声明（结束）     
    OP_ADD, OP_ADDV,                //  加法操作            
    OP_SUB, OP_SUBV,                //  减法操作        
    OP_MUL, OP_MULV,                //  乘法操作      
    OP_DIV, OP_DIVV,                //  除法操作      
    OP_MOD, OP_MODV,                //  取模操作        
    OP_MINUS,                       //  取相反数           
    OP_LABEL,                       //  添加标签          
    OP_BEQ, OP_BNQ,                 //  等于/不等于的条件跳转     
    OP_BGT, OP_BGE,                 //  大于/大于等于的条件跳转OK 
    OP_GOTO,                        //  无条件跳转OK1
    OP_NOT,                         //  布尔表达式取非     
    OP_RETURN,                      //  返回语句         
    OP_ARG,                         //  函数实参
    OP_EXPLICIT_ARG,                //  显式ARG
    OP_CALL,                        //  函数调用
    OP_PARAM,                       //  函数形参          
};
```

这里需要注意的几个特殊的操作码是和**数组向量运算的相关操作码**、**数组访问的操作码**、以及**显式`ARG`**。就一般情况而言，我们不需要这些操作码也能够支持数组的向量运算，但我们选择将数组的向量运算和访问延后到`codegen`部分进行处理，所以设计了和数组向量运算和访问相关的操作码。此外，对于没有函数参数的`CALL`指令，在我们的翻译模式中是不会出现`ARG`语句的。这一点当我们没有做寄存器分配时并不会对我们最后的代码生成造成影响；但一旦做了寄存器分配，我们就需要在执行`CALL`语句时需要检查所有在`CALL`语句前保持有活跃变量的寄存器，并且将他们保存在栈上，在函数调用返回后进行恢复。因此，对于没有函数参数的`CALL`语句，我们选择在其前面添加一条`EXPLICIT_ARG`来要求代码生成器去完成保存寄存器的过程。

对于`Var`这一数据结构，我们的设计如下：

```C++
struct Var
{
    VarType type;                       //变量类型
    VarData data;                       //变量的数值类型
    std::string value = "";             //当变量类型为常量时，表示常量值；当变量类型为函数时，表示函数名
    int var_no = 0;                     //变量、临时变量、标签的序号
    int size = 0;                       //变量大小（变量声明时需要记录）

    bool is_global = false;     //是否为全局变量
    bool is_array  = false;     //是否为数组

    RegDiscriptor reg;          //该变量对应的寄存器描述符
    int bitmap = 0;				//对应的位向量比特

	//...constructors...
    //...
};
```

对于操作数，我们希望记录操作数的类型（变量、临时变量、标签、函数、常量），操作数的数据类型（由于代码生成部分的指令选择）、常量值（或者函数名、变量名）、变量号（用于区分不同的变量）、大小（主要用于数组类型的变量，记录在栈帧中开辟的空间大小）、该变量是否为全局变量、该变量是否为数组，以及其和寄存器分配相关的信息（寄存器描述符和对应的位向量比特）



##### 产生中间代码

为了生成中间代码，我们需要运用SDT，为每个主要的语法单元***X***设计相应的翻译函数***translateX***

，我们对PR002中语法树的遍历过程也就是这些函数之前的相互调用的过程，每种特定的语法结构都对应了固定模式的翻译模板。为了简洁，我们在报告里只给出***EXP***的翻译函数，其设计如下：

```C++
TranslateExp(Exp, temp) = case Exp of
    INT:							//整数常数
		value = get_value(INT)
        return [temp := #value]
            
    ID:								//变量名
		var = sym_table.look_up(ID)
        return [temp := var.name]
    Exp1 = Exp2:					//赋值表达式
    (Exp1 -> ID)
        var = sym_table.lookup(ID)
        t1 = new_temp()
        code1 = TranslateExp(Exp2, t1)
        code2 = [var.name := t1] + [temp := var.name]
        return code1 + code2	
    Exp1 + Exp2:					//四则运算表达式
        t1 = new_temp()
        t2 = new_temp()
        code1 = TranslateExp(Exp1, t1)
        code2 = TranslateExp(Exp2, t2)
        code3 = [temp := t1 + t2]
        return code1 + code2 + code3
    ID Func(Args):
            function = sym_table.lookup(ID)
    		arg_list = NULL
    		code1 = TranslateArgs(Args, arg_list)
    		for(i = 1;i < length(arg_list);i++) code2 += [ARG arg_list[i]
  	]
    		return code1 + code2 + [temp := CALL function.name]
    //...

		

```

​    我们以四则运算的翻译为例，给出具体的代码实现

```C++
void SemanticAnalysis::enterAddExpAddMul(CACTParser::AddExpAddMulContext * ctx)
{
    ctx->addExp()->temp = tac.new_temp();
    ctx->mulExp()->temp = tac.new_temp();
}
```

在进入加法表达式节点时，`tac`（`TacBuilder`类的一个实例）会调用`new_temp`方法，新建出两个临时变量，用于存储中间表达式的翻译结果，这一临时变量将随着注释语法树的遍历作为继承属性沿着`addExp`和`mulExp`的子节点传播，直至叶子节点，当遍历到叶子节点后，实际上就对应到了我们的翻译模板中最基本的case，其代码实现如下：

```c++
void SemanticAnalysis::exitNumberIntConst(CACTParser::NumberIntConstContext * ctx)
{
    ctx->expression_type = PARAM_INT;
    ctx->temp->data = IR::DATA_INT;
    //exp -> int
    tac.add(IR::OP_ASSIGN, ctx->temp, tac.new_const(ctx->getText(), IR::DATA_INT));
}
```

这一节点会产生一条赋值语句，其目的操作数实际上就是我们在进入加法表达式节点时新建的临时变量，源操作数1是我们通过调用`new_const`方法新建出来的常量。

除了***EXP***这一主要的语法单元外，CACT的语法里还有***STMT***和***COND***，我们同样需要为它们设计类似的翻译函数，具体的设计可以见本仓库`resources`目录下的`pre-5.27.pptx`，在实验报告中就不再详细介绍。

这里我们对设计里感到遗憾的地方是有关数组类型的设计：我们在计算偏移量时，希望在`var::data`里存储的是`int`类型，但我们又希望知道它具体的存储的数据类型。对于这一点，我们在生成`RISC-V`代码时做了一些妥协，对所有的case做了单独处理，回想起来当初在这里的设计还是有很大问题。



#### 机器无关优化

在机器无关优化的部分，我们主要是生成基本块，并以基本块为单位对块内的IR做局部优化。下面逐点进行介绍，并附上相应的代码。

##### 基本块的生成

这个部分的目标是将一个包含所有IR的`List`分成`pair <func_name, func_IR>`的对，每个`pair`对应一个函数的IR，并以函数名为索引。`func_IR`是一个`basic_block`的单向链表头，可以通过`next_seq`域来访问这个函数的所有基本块（与块内的IR）。`basic_block`定义如下：

```c++
struct basic_block{
    IR::IRList ircode;      // 基本块内的IR list
    int block_num;          // 基本块的序号

    basic_block* next_true = nullptr;
    basic_block* next_false = nullptr;
    basic_block* next_seq = nullptr;

    BeginType begin_type;   // 基本块起始IR的类型，优化用
    EndType end_type;       // 基本块末尾IR的类型，优化用
        
    int block_use;          // block 的自定义字段，可供不同阶段的优化函数使用，目前暂且设置为一个 int
    std::set<int> succs;    // Live Analysis中的后继集合
    std::set<IR::Var*> LiveOut; // Live Analysis中的LiveOut集合
};
```

其中值得注意的是，一个基本块有三个后继指针`next_seq`、`next_true`、`next_false`，其中next_seq代表的是基本块之间的文本顺序，但由于跳转语句会使得程序实际运行时的执行顺序和文本顺序并不一定相同，所以还有代表基本块之间逻辑顺序的`next_true`和`next_false`，分别指向基本块末尾的`branch`条件为真/假时下一个基本块，若基本块不以条件跳转指令结尾，那么`next_false`为`nullptr`。

在设计之初，我添加了成员`block_use`域，这是考虑到优化过程中不同的PASS可能需要一些不同的、各自额外的域来辅助执行，各个PASS内部自行利用、处理。现在看来，这大概率不是一个好的设计，在程序可读性和代码分工的角度来说差了很多。只能说庆幸优化部分只需我一个人完成，不然这样的设计在交接的时候会增添许多不必要的工作量。

生成基本块的过程中，主要思路是扫描一遍IRList，并通过`func_flag`、`branch_flag`、`return_flag`等标志进行基本块的划分，定下`next_seq`的顺序，记录下`Label`标号和`block_num`基本块序号之间的对应关系；再扫描一遍基本块，并按照对应关系确定各个基本块的`next_true`、`next_false`，从而完成基本块的生成。

由于IR的生成过程需要考虑所有可能的`CACT`语句，因此生成的IR必将有优化空间，下面介绍基于基本块的机器无关优化。



##### 基本块的逻辑关系优化

考虑下列`CACT`语句：

```
int main(){
    int a;
    if(a == 1){
        return 0;
    }
    else{
        return 1;
    }

    print_int(a);
}
```

由于if的两个分支内都会执行`return`语句，因此最后的`print_int(a);`并不会被执行。这一特点表现在基本块的逻辑关系上，就是从`main`的首基本块出发，沿`next_true`和`next_false`能到达的基本块，就是从函数头可能抵达的语句，否则则是“死语句”，无论什么情况下一定无法被执行，因此可以删去。



##### 基本块的合并

考虑下列三地址代码IR语句组成的基本块：

```
BLOCK 0:
v0 := v1 + v2
goto LABEL 0

BLOCK 1:
LABEL 0
v0 := v1 - v2
```

如果没有其他的基本块以`BLOCK 1`作为后继，那么我们便能将`BLOCK 0`和`BLOCK 1`成一个基本块，并省去其中的`goto` IR语句。

```
BLOCK 0:
v0 := v1 + v2
v0 := v1 - v2
```

值得注意的是，这个“没有其他的基本块以`BLOCK 1`作为后继”并不能直接等同成“除`BLOCK 0`外没有其他的`goto LABEL 0`语句”，因为可能出现下列情况：

```
BLOCK 0:
v0 := v1 + v2
goto LABEL 0

BLOCK 1:
if v0 < v1 goto LABEL 1

BLOCK 2:
LABEL 0
v0 := v1 * v2

BLOCK 3:
LABEL 1
```

此时`BLOCK 1`的`next_false`“隐式”地指向`BLOCK 2`。

此外，基本块的合并也有可能打乱原本的文本顺序。

```
BLOCK 0:
v0 := v1 + v2
goto LABEL 0

BLOCK 1:
v0 := v1 - v2

BLOCK 2:
LABEL 0
v0 := v1 * v2

BLOCK 3:
LABEL 1
v0 := v1 / v2
```

简单合并后如下：

```
BLOCK 0:
v0 := v1 + v2
v0 := v1 * v2
// goto LABEL 1

BLOCK 1:
v0 := v1 - v2

BLOCK 3:
LABEL 1
v0 := v1 / v2
```

程序的执行顺序错误，因此这种情况下需要在`BLOCK 0`末尾添加额外的`goto`语句。



##### 常量传播与折叠

虽然常量的传播与折叠并非一个概念，但我将其放在同一个函数中实现。这两者本质上是对一个基本块内的IR求不动点——IR的条数与操作数的数量是有限的，且非常量的操作数每一次一定会被替换成常量，故最终一定能收敛到不动点。这一过程并不难，用`while(changed)`不断求不动点，遍历该基本块中的每一条IR即可。

常量传播与折叠的过程同样也会影响`branch`指令的操作数，倘若`branch`的跳转结果为常量，那么`branch`指令将会被替换成`goto`或者被删去——这带来了合并基本块的可能，而合并后的大基本块有可能蕴藏着新的常量传播和折叠的机会，因此这又是一个不断求不动点的过程。

因此这个函数主要有两层`while(changed)`构成，外层是在条件跳转变为直接跳转或被删去时，找寻是否有合并基本块的机会；内层是在有形如`v0 := #1`或`v0 := #1 + #2`的IR语句时，找寻是否有常量传播、折叠的机会。下面是代码示例：

```c++
void const_spread(){
    for(auto pile : block_pile){
        for(auto block : pile->second){
            auto ir = block->ircode.begin();

            while(ir != block->ircode.end()){
                bool ir_changed = false;

                /* 考虑两个操作数为常数值的IR语句 */
                // v0 := #0 + #1

                /* 考虑两个操作数为常数值的Branch-IR语句 */
                // if #0 < #1 goto LABEL 0
                
                /* 考虑操作数为常数值的Minus-IR语句 */
                // v0 := - #0

                /* 考虑ASSIGN 常数值的IR语句 */
                // v0 := #0

                if(!ir_changed){
                    ir++;
                }
            }
        }
    }
}
```

```c++
    do{
        block_changed = false;
        const_spread();
        delete_unused_block();
        merge_block();
    }
    while(block_changed == true);
```



##### 表达式强度削减、代数恒等式

这一部分的逻辑比较简单，即遍历IR并找寻是否有形如`x + 0`、`x * 1`、`x % x`或是`x * 2`、`x / 8`的IR语句，并将其删去或是替换成等效的语句。



##### 基于基本块的RETURN check

在`PR002`的语义分析部分中，我们需要对`AST`进行分析，来确保程序每一个可到达的分支都以`return`结尾，在antlr-4的框架下，通过语义分析完成这一功能似乎有些复杂，但在生成了基本块后，这一需求变得十分简单——只需对每一个`next_ture`和`next_false`都为`nullptr`的基本块，检查其末尾是否是`RETURN`即可。



##### 基于活跃变量分析的未初始化变量检查

在项目规划初期，优化除了上面提及的内容外，还计划在活跃变量分析的基础上完成CFG的构建，并为之后的寄存器分配、公共子表达式替换、死代码消除等打下基础，但事实上活跃变量分析只用来检查了变量的初始化情况，说实话有些可惜。

活跃变量分析本质上也是求数据流方程的不动点，依照龙书和理论课上的算法，不断计算集合内的元素的增加情况，并最终得到各个基本块的`LiveOut`、`Succs`集合。由于书上讲的十分详细，因此我这里就不再赘述，下面是代码实现：

```c++
void LiveVariableAnalyser::solve(CFG::basic_block* block){
    bool change = true;
    while(change){
        change = false;
        auto block_pos = block;

        while(block_pos != nullptr){
            std::set<IR::Var*> temp;

            auto temp_live_set = block_pos->LiveOut;

            for(auto iter = block; iter != nullptr; iter = iter->next_seq){
                auto num = iter->block_num;
                if(block_pos->succs.find(num) != block_pos->succs.end()){   
                    // iter指向的块的序号在block_pos的succs内，即iter是block_pos的后继
                    auto next_UEVar = UEVar[num];
                    auto next_VarKill = VarKill[num];
                    auto next_VarKill_C = set_diff(universal_set, next_VarKill);    // universal_set - Varkill， 即Varkill的补集
                    temp = set_union(temp, set_union(next_UEVar, set_inter(block_pos->next_true->LiveOut, next_VarKill_C)));
                }
            }

            block_pos->LiveOut = temp;
            
            if(temp_live_set.size() != block_pos->LiveOut.size()){
                change = true;
            }
            block_pos = block_pos->next_seq;
        }
    }
}

```

#### RISC-V代码生成

对于RISC-V代码生成，我们实际上需要解决三个问题：指令选择、寄存器分配、内存管理。

##### 指令选择

对于指令选择，由于我们采用的IR已经和RISC结构十分相似，因此，对于每一条IR语句，我们基本上都能够找到相应的指令与之对应：对于运算指令，我们需要根据`data`记录的数值类型来进行具体的指令选择，例如，如果`var`的类型为`float`且对应的`opcode`为`OP_ADD`，则应当选择`fadd.s`指令；对于条件跳转和无条件跳转的`opcode`，对应于`bgt, bge, blt, ble, j`五条指令，函数调用的`opcode`对应于`call`指令。

需要关注的是和数组访问、数组向量运算和过程调用相关的`ARG, PARAM, RETURN`。对于`ASSIGN_GETVAL`语句，此时对应的源操作数1是一个地址，而目的操作数是一个具体的变量/临时变量，这时我们可以选择`lw/ld/flw/fld`指令；对于`ASSIGN_SETVAL`语句，相应地，选择`sw/sd/fsw/fsd`指令；对于`ARG`语句，如果目的操作数被溢出到了内存中，我们选择`lw/ld/flw/fld`指令，根据函数调用约定，将栈上的内容转移至相应寄存器；如果在寄存器中，则只需要一条`mv`指令；对于`PARAM`语句，与`ARG`同理，只不过需要选择`mv`或`store`类型的指令。

做完指令选择后我们下一步需要考虑的是，究竟是用文本形式的RISC-V代码还是像IR语句一样的in-memory形式的RISC-V代码。为了简化实现，我们优先选择了文本形式的RISC-V代码，其生成按照上面给出的指令选择策略逐一进行翻译即可。但其缺点也很明显，当我们做完机器无关优化和寄存器分配试图进行机器无关优化时，我们就无能为力了，而当我们试图重新设计时，时间上已经不允许了。



##### 寄存器分配

对于寄存器分配，我们采用的是`global`的寄存器分配策略，具体的参考实现是***Linear Scan Register Allocation***。为了实现这一寄存器分配策略，我们需要设计寄存器描述符相关的数据结构和用于寄存器分配的模块。

```C++
struct RegDiscriptor
{
    std::string name;		//寄存器名
    bool spill;				//是否溢出(或许放在var这一数据结构中更合适？)
    int frame_offset;		//溢出时相对于栈的偏移
    int reg_no;				//寄存器序号
    bool allocated;			//该寄存器是否已经被分配

	//...constructors...
    //...
};
```

首先是寄存器描述符的数据结构，我们记录了它是否被分配、寄存器名字、序号、对应的变量是否溢出、溢出时相对于栈的偏移，这里`frame_offset`和`spill`记录在`var`这一数据结构中应该更为合理。为了降低模块之前的耦合度，我们通过辅助的数据结构来记录不同函数寄存器内保存的变量信息。

```
std::unordered_map<std::string, std::unordered_map<int, std::vector<IR::Var>>> reg_contains;
std::unordered_map<std::string, std::unordered_map<int, std::vector<IR::Var>>> float_reg_contains; ;
```

寄存器分配也离不开活跃变量分析，因此我们设计了位向量的数据结构来加快求解数据流方程：

```C++
struct Bitvec
{
    std::string vec;								//位向量

    Bitvec() = default;
    Bitvec(int num) {vec.assign(num, '0');}
    Bitvec(std::string vec) :vec (vec) { }
    Bitvec(const Bitvec& rhs) {vec = rhs.vec; }

    Bitvec operator& (const Bitvec& rhs);			//向量按位与
    Bitvec operator| (const Bitvec& rhs);			//向量按位或
    Bitvec operator- (const Bitvec& rhs);			//向量求差（集合差）
    Bitvec operator= (const Bitvec& rhs);			
    bool operator!=(const Bitvec& rhs);
    char& operator[](int i) {return vec[i];}
};
```

有了上面的铺垫后，我们设计了下面的模块用于完成寄存器分配的工作：

```C++
struct RegAllocator
{

    int var_num;								//当前函数的变量数目
    int temp_num;								//当前函数的临时变量数目
    int ir_count = 1;							//IR序号，用于输出调试信息
    	
    std::string func_name;						//当前处理的函数名，用于输出调试信息
    IR::IRList local_ircode;					//传递给代码生成器的中间代码


    std::unordered_map<IR::Quad*, std::vector<IR::Quad*>> succs; //每条IR语句的后继信息
    std::vector<IR::Quad*> ordered_ircode;		//编号后的IR语句

    std::unordered_map<IR::Quad*, Bitvec*> in;	//每条IR语句对应的Livein集合
    std::unordered_map<IR::Quad*, Bitvec*> out;	//每条IR语句对应的Liveout集合
    std::unordered_map<IR::Quad*, Bitvec*> use;	//每条IR语句对应的use集合
    std::unordered_map<IR::Quad*, Bitvec*> def;	//每条IR语句对应的def集合

    void gen_ordered_ircode(IR::IRList& ircode);//生成编号的IR语句
    void gen_succs(IR::IRList& ircode);			//生成每条IR语句的后继信息

    void InitAllSets(IR::IRList& ircode, int var_num, int temp_num); //初始化每条IR语句的维向量
    void InitRegisterPool();					//初始化寄存器池
    void CalculateUseDef(IR::IRList& ircode, int var_num, int temp_num); //计算每条IR语句的use, def信息
    void SolveEquation(IR::IRList& ircode, int var_num, int temp_num); //解数据流方程

    std::vector<Interval> interval;				//所有整数变量/临时变量的live interval(defined by paper)
    std::vector<Interval> float_interval;		//所有浮点整数变量/临时变量的live interval(defined by paper)
    std::vector<Interval> active; 				//所有对应整数变量/临时变量的live interval的集合(defined by paper)
    std::vector<Interval> float_active; 		//所有对应浮点变量/临时变量的live interval的集合(defined by paper)
    std::vector<RegDiscriptor*> register_pool; 	//整数寄存器池
    std::vector<RegDiscriptor*> float_register_pool; //浮点寄存器池

    void LiveAnalysis(IR::IRList& ircode, std::vector<IR::Var*> var_list, std::vector<IR::Var*> temp_list); //活跃变量分析
    void Expire(Interval inter); 				//收回生命周期不再重叠的寄存器
    void SpillAtInterval(Interval inter); 		//溢出
    void LSRA(IR::IRList& ircode, std::vector<IR::Var*> var_list, std::vector<IR::Var*> temp_list); // Linear Scan算法实现
    void dump(IR::IRList ircode, std::vector<IR::Var*> var_list, std::vector<IR::Var*> temp_list, std::ofstream& ofs, std::string func_name); 							//输出调试信息
    void clear_data(); 							//清除数据，用于进行下一个函数的寄存器分配
    void alloc(CFG::basic_block_builder& BB_Builder);
    void config_bitmap(std::vector<IR::Var*> var_list, std::vector<IR::Var*> temp_list);
    //配置var结构体里的bitmap
    void getReg(IR::Var* var); 					//为var分配寄存器
    void gen_use_info(std::string func_name); 	//生成当前函数的寄存器使用信息
};
```

`RegAllocator`接受来自`BB_Builder`的中间代码，调用`alloc`方法对每个函数进行寄存器分配。

其首先遍历当前函数所有的IR语句，收集所有的变量和临时变量，并且配置好它们在位向量中的对应位置，接着运行***LSRA***算法，逐步生成下一阶段的中间代码，并且生成当前函数的寄存器使用信息，最后清除当前函数缓存的数据和已经分配的寄存器。这里的核心是***LSRA***算法，其具体实现如下：

```C++
void RegAllocator::LSRA(IR::IRList& ircode, std::vector<IR::Var*> var_list, std::vector<IR::Var*> temp_list)
{
    InitRegisterPool();
    LiveAnalysis(ircode, var_list, temp_list);
    for(auto iter = interval.begin(); iter != interval.end(); iter++)
    {
        if(iter->var->is_global == false && iter->var->is_array == false && iter->var->data != IR::DATA_DEFUALT)
        {
            Expire((*iter));
            //x0 - x4 is not pushed into the pool
            if(active.size() == REG_NUM  - 5)
                SpillAtInterval((*iter));
            else
            {
                getReg(iter->var);
                active.push_back((*iter));
                sort(active.begin(), active.end(), sort_active);
            }
        }
        else
        {
            iter->var->reg.spill = true;
        }

    }
    for(auto iter = float_interval.begin(); iter != float_interval.end(); iter++)
    {
        if(iter->var->is_global == false && iter->var->is_array == false && iter->var->data != IR::DATA_DEFUALT)
        {
            Expire((*iter));
            //f0
            if(float_active.size() == REG_NUM - 1)
                SpillAtInterval((*iter));
            else
            {
                getReg(iter->var);
                float_active.push_back((*iter));
                sort(float_active.begin(), float_active.end(), sort_active);
            }
        }
        else
        {
            iter->var->reg.spill = true;
        }
    }
}
```

它首先初始化寄存器池，将所有能够被分配的寄存器加入寄存器池中，接着进行活跃变量分析，生成每条IR语句的活跃信息。以整数寄存器分配为例，它接着会调用`Expire`函数，其具体实现如下：

```C++
void RegAllocator::Expire(Interval inter)
{
    if(inter.var->data == IR::DATA_INT || inter.var->data == IR::DATA_BOOL)
    {
        auto iter_int = active.begin();
        while(iter_int != active.end())
        {
            if(iter_int->end >= inter.start)
                return;
            register_pool.push_back(&reg[iter_int->var->reg.reg_no]);
            iter_int = active.erase(iter_int);
        }
    }
    else
    {
        auto iter_float = float_active.begin();
        while(iter_float != float_active.end())
        {
            if(iter_float->end >= inter.start)
                return;
            float_register_pool.push_back(&float_reg[iter_float->var->reg.reg_no]);
            iter_float = float_active.erase(iter_float);
        }
    }
}
```

其中，`active`按照较小结束点优先对各个变量/临时变量的`live interval`进行排序的链表，`Expire`函数将扫描这一链表，如果发现扫描到的`live interval`的结束点大于等于当前分析的`live interval`的开始点，说明两个`live interval`发生重叠，则这两个`interval`对应的变量/临时变量不能分配到同一寄存器；否则，回收扫描到的`live interval`对应变量分配到的寄存器，并且将这一`live interval`从`active`链表中去除。

这里是论文里最原本的解决冲突的做法，实际上，在我们的设计中，冲突的条件还可以进一步进行细化：

1. 在赋值操作`x := y`，即使`x`和`y`在这条代码之后都活跃，因为二者值是相等的，它们仍然可以共用寄存器
2. 在类似`x := y + z`这样的中间代码中，如果变量`x`在这条代码之后不再活跃，但变量`y`仍然活跃，那么此时虽然`x`和`y`不同时活跃，二者仍然要避免公用寄存器以防止之后对`x`的赋值会将活跃变量`y`在寄存器中的值覆盖掉

不过为了简化实现，在我们的设计中，我们没有把这样的条件添加到该函数中。

接着，如果`active`链表的长度达到了限定的数目，说明寄存器池内的寄存器已经全部被分配，无法再分配新的寄存器，则会调用`SpillAtInterval`函数，选择一个寄存器换出，并且将寄存器内存储的内容溢出到栈上。这里需要提的是，由于生成全局变量RISC-V部分的代码设计不是很合理，我们没有对全局变量和数组类型的变量/临时变量做寄存器分配，它们都被溢出到了栈上。

`SpillAtInterval`函数具体实现如下：

```C++
void RegAllocator::SpillAtInterval(Interval inter)
{
    if(inter.var->data == IR::DATA_INT || IR::DATA_BOOL)
    {
        auto spill = (*active.rbegin());
        if(spill.end > inter.end)
        {
            inter.var->reg = spill.var->reg;

            spill.var->reg.spill = true;

            active.pop_back();
            active.push_back(inter);
            sort(interval.begin(), interval.end(), sort_active);
        }
        else
        {
            inter.var->reg.spill = true;
        }
    }
    else
    {
        auto spill = (*float_active.rbegin());
        if(spill.end > inter.end)
        {
            inter.var->reg = spill.var->reg;

            spill.var->reg.spill = true;

            float_active.pop_back();
            float_active.push_back(inter);
            sort(float_interval.begin(), float_interval.end(), sort_active);
        }
        else
        {
            inter.var->reg.spill = true;
        }
    }
}
```

它检查`active`的链表尾元素，如果其生存周期长于当前周期，则溢出其对应的变量，并将当前考察的`live interval`对应的变量/临时变量分配到的寄存器设置为尾元素分配到的寄存器；否则溢出当前周期对应的变量/临时变量。

如果`active`链表的长度未达到限定的数目，则说明还有可分配的寄存器，则从寄存器池中取出寄存器分配给给当前周期对应的变量/临时变量。



##### 内存管理

我们采取栈式内存管理策略，并且基于此设计了两种不同的代码生成策略。

第一种是不做寄存器分配的代码生成，所有的变量/临时变量都将分配到栈上，这样生成的代码固然可靠，因为对于每个变量的修改总是伴随着`load`和`store`，只有个别寄存器被拿来当作缓存使用，大大增加了机器的访存开销。

第二种是带寄存器分配的代码生成，当中间代码经过寄存器分配后，寄存器分配信息已经存储在了各变量/临时变量中，但这样也会带来新的问题：如何去维护寄存器的生命周期？

对于函数体内不带过程调用的函数，寄存器分配算法确保了具有重叠生命周期的变量/临时变量分配到了不同的寄存器或者溢出到了栈上，可以确保结果的正确性。但对于函数体内带过程调用的函数，我们需要分调用者和被调用者两种情况来进行处理。

对于被调用者，按照RISC-V函数调用约定，需要在进入函数体时保存被调用者保存的寄存器，退出时恢复寄存器的值；问题的关键在于如何知道哪些被调用者寄存器在当前函数体内会被使用。这一点我们在`RegAllocator`模块里生成了`use_info`信息，只需要根据函数名进行查找即可；

对于调用者，按照RISC-V函数规定，如果在`call`指令后有些当前调用者保存寄存器中的值将被使用，我们也需要保存这些寄存器；问题的关键在于如何知道`call`语句后的寄存器活跃信息。这一点我们在`RegAllocator`记录了`in`和`out`信息，在翻译到`call`语句前进行查询`call`语句对应的`in`集合即可。

具体到代码实现上，如下所示：

```C++
void RARISCVBuilder::emit_arg(IR::IRList::iterator iter, std::ofstream& ofs, Frame* stack_frame, int& arg_num)
{
    if(arg_flag == false)
    {
        for(int i = 0; i < REG_NUM; i++)
        {
            auto temp_iter = iter;
            while(temp_iter->op != IR::OP_CALL)
                temp_iter++;
            if(use_info[func_name][i] == true && reg_is_caller_saved(reg[i]))
            {
                for(auto var = reg_contains[func_name][i].begin(); var != reg_contains[func_name][i].end(); var++)
                {
                    if(live_at((*var), &(*temp_iter)))		//查询call语句前的变量的活跃信息
                    {
                        int growth = 8;
                        spill_size += growth;
                        reg[i].frame_offset = spill_size + stack_frame->frame_size;
                        caller_saved.push_back(reg[i]);
                        break;
                    }
                }
            }
            if(float_use_info[func_name][i] == true && reg_is_caller_saved(float_reg[i]))
            {
                for(auto var = float_reg_contains[func_name][i].begin(); var != float_reg_contains[func_name][i].end(); var++)
                {
                    if(live_at((*var), &(*temp_iter)))		//查询call语句前的变量的活跃信息
                    {
                        int growth = 8;
                        spill_size += growth;
                        float_reg[i].frame_offset = spill_size + stack_frame->frame_size;
                        float_caller_saved.push_back(float_reg[i]);
                        continue;
                    }
                }
            }
        }
        if(spill_size != 0)
            ofs << "\taddi\t" << "sp, sp, -" << std::to_string(spill_size) << "\n";

        for(auto reg : caller_saved)
        {
            ofs << "\tsd  \t" << reg.name << ", -" << std::to_string(reg.frame_offset) << "(s0)\n";
        }
        for(auto reg : float_caller_saved)
        {
            ofs << "\tfsd \t" << reg.name << ", -" << std::to_string(reg.frame_offset) << "(s0)\n";
        }

        arg_flag = true;
        stack_frame->frame_size += spill_size;
        spill_size = 0;
    }
	//......
}
```

在翻译到`call`语句前的第一条`ARG`语句时，代码生成器会查询被调用者保存寄存器的分配信息及其保存变量在`call`语句前的活跃信息，我们将满足条件的需要保存的寄存器添加到`caller_saved`中，并且在栈上划分好空间用于保存

```C++
void RARISCVBuilder::emit_call(IR::IRList::iterator iter, std::ofstream& ofs, Frame* stack_frame, int& arg_num)
{
	//......

    for(auto reg : caller_saved)
    {
        ofs << "\tld  \t" << reg.name << ", -" << std::to_string(reg.frame_offset) << "(s0)\n";
    }
    for(auto reg : float_caller_saved)
    {
        ofs << "\tfld \t" << reg.name << ", -" << std::to_string(reg.frame_offset) << "(s0)\n";
    }

    caller_saved.clear();
    float_caller_saved.clear();
}
```

在翻译完`CALL`语句的主体部分后，我们将保存好的`caller_saved`寄存器内容重新从栈上加载出来。

对于被调用者而言，也是类似的，在翻译`FUNCDEC`语句时保存被调用者保存的寄存器，翻译`RETURN`语句时恢复寄存器，只是保存时在检查寄存器的时候，只需要检查相应的寄存器的分配情况，而不用进一步寄存器内检查变量的活跃情况。



##### 杂项

###### 全局变量/数组代码的RISC-V代码生成

根据RISC-V汇编的书写规则，我们将全局变量/数组放置在`.data`段，在生成具体汇编语句前，我们需要先收集`.data`段的信息，其实现如下：

```C++
void RARISCVBuilder::gen_data(IR::IRList global_ircode)
{
    for(auto iter = global_ircode.begin(); iter != global_ircode.end(); iter++)
    {
        if(iter->op == IR::OP_VARDEC && iter->dest->is_array == false)
        {
            int align = 0;
            int size = 0;
            std::string name = iter->dest->value;
            std::vector<std::string> words;
            switch(iter->dest->data)
            {
                case IR::DATA_INT:
                    align = 2;
                    size = 4;
                    words.push_back(next(iter)->arg1->value);
                    break;
                case IR::DATA_BOOL:
                    align = 2;
                    size = 1;
                    if(next(iter)->arg1->value == "true")
                        words.push_back("1");
                    else
                        words.push_back("0");
                    break;
                case IR::DATA_FLOAT:
                    align = 2;
                    size = 4;
                    words.push_back(float_to_word(std::stof(next(iter)->arg1->value)));
                    break;
                case IR::DATA_DOUBLE:
                    align = 3;
                    size = 8;
                    words.push_back(double_to_word(std::stod(next(iter)->arg1->value)));
                    break;
            }
            GlobalLabel* new_label = new GlobalLabel(align, size, iter->dest->data, name, words);
            data.insert({iter->dest, new_label});
        }

        if(iter->op == IR::OP_VARDEC && iter->dest->is_array == true)
        {
            int align = 3;
            int size = 0;
            std::string name = iter->dest->value;
            std::vector<std::string> words;

            auto temp_dest = iter->dest;

            switch(iter->dest->data)
            {
                case IR::DATA_INT:
                    align = 2;
                    while(next(iter)->arg1->type == IR::TYPE_CONST)
                    {
                        size += 4;
                        words.push_back(next(iter)->arg1->value);
                        iter++;
                    }
                    break;
                case IR::DATA_BOOL:
                    align = 2;
                    while(next(iter)->arg1->type == IR::TYPE_CONST)
                    {
                        size += 1;
                        if(next(iter)->arg1->value == "true")
                            words.push_back("1");
                        else
                            words.push_back("0");
                        iter++;
                    }
                    break;
                case IR::DATA_FLOAT:
                    align = 2;
                    while(next(iter)->arg1->type == IR::TYPE_CONST)
                    {
                        size += 4;
                        words.push_back(float_to_word(std::stof(next(iter)->arg1->value)));
                        iter++;
                    }
                    break;
                case IR::DATA_DOUBLE:
                    align = 3;
                    while(next(iter)->arg1->type == IR::TYPE_CONST)
                    {
                        size += 8;
                        words.push_back(double_to_word(std::stod(next(iter)->arg1->value)));
                        iter++;
                    }
                    break;
            }
            GlobalLabel* new_label = new GlobalLabel(align, size, iter->dest->data, name, words);
            data.insert({temp_dest, new_label});
        }
    }
}
```

简单来说，思路是为每一个全局变量生成对应的描述其信息的`Label`，在生成汇编代码时访问其`Label`信息即可

```C++
void RARISCVBuilder::emit_data(std::ofstream& ofs)
{
    int flag = 0;
    for(auto iter = data.begin(); iter != data.end(); iter++)
    {
        if(flag == 0)
        {
            ofs << "\t.section\t.data\n";
            flag = 1;
        }
        ofs << "\t.align\t" << std::to_string(iter->second->align) << "\n";
        ofs << "\t.type\t" << iter->second->name << ", @object\n";
        ofs << "\t.size\t" << iter->second->name << ", " << std::to_string(iter->second->size) << "\n";
        if(iter->first->is_array == false)
        {
            if(iter->first->data == IR::DATA_INT || iter->first->data == IR::DATA_FLOAT || iter->first->data == IR::DATA_BOOL)
            {
                ofs << iter->second->name << ":\n"; 
                ofs << "\t.word\t" << iter->second->words[0] << "\n";
            }
            if(iter->first->data == IR::DATA_DOUBLE)
            {
                ofs << iter->second->name << ":\n"; 
                std::string word0 = hex_to_decimal(iter->second->words[0].substr(8, 8));
                std::string word1 = hex_to_decimal(iter->second->words[0].substr(0, 8));
                ofs << "\t.word\t" << word0 << "\n";
                ofs << "\t.word\t" << word1 << "\n";
            }
        }
        if(iter->first->is_array == true)
        {
            if(iter->first->data == IR::DATA_INT || iter->first->data == IR::DATA_FLOAT || iter->first->data == IR::DATA_BOOL)
            {
                ofs << iter->second->name << ":\n"; 
                for(auto word : iter->second->words)
                    ofs << "\t.word\t" << word << "\n";
            }
            if(iter->first->data == IR::DATA_DOUBLE)
            {
                ofs << iter->second->name << ":\n"; 
                for(auto word : iter->second->words)
                {
                    std::string word0 = hex_to_decimal(word.substr(8, 8));
                    std::string word1 = hex_to_decimal(word.substr(0, 8));
                    ofs << "\t.word\t" << word0 << "\n";
                    ofs << "\t.word\t" << word1 << "\n";
                }
            }
        }
    }
}   
```



###### 局部变量/数组的RISC-V代码生成

这一部分，我们设计了五个主要函数`alloc_frame, load_arg1, load_arg2, load_dest, store_dest`，其中`load`相关的函数处理逻辑如下：

* 如果对应的var是变量/临时变量且是溢出的，从没有被分配的寄存器中取出一个用于存储它的值，并返回该寄存器对应的寄存器描述符
* 如果对应的var是常量，从预留的寄存器中取出一个用于存储它的值，并返回该寄存器对应的寄存器描述符
* 如果对应的var是变量/临时变量且没有溢出，则返回该var对应的寄存器描述符

这里以`load_dest`为例，给出它的代码实现：

```C++
RegDiscriptor RARISCVBuilder::load_dest(IR::Var* arg, Frame* stack_frame, std::ofstream& ofs)
{
    switch(arg->type)
    {
        case IR::TYPE_VAR:
            if(arg->is_global)
            {
                if(arg->is_array)
                {
                    ofs << "\tlui \t" << "t4" << ", " << "%hi(" << arg->value << ")\n";
                    ofs << "\taddi\t" << "t4, t4, " << "%lo(" << arg->value << ")\n";
                    return RegDiscriptor("t4");
                }
                else
                {
                    if(arg->data == IR::DATA_INT || arg->data == IR::DATA_BOOL)
                    {
                        ofs << "\tlui \t" << "t4" << ", " << "%hi(" << arg->value << ")\n";
                        ofs << "\tlw  \t" << "t4" << ", " << "%lo(" << arg->value << ")(t4)\n";
                        return RegDiscriptor("t4");
                    }
                    if(arg->data == IR::DATA_FLOAT)
                    {
                        ofs << "\tlui \t" << "t4" << ", " << "%hi(" << arg->value << ")\n";
                        ofs << "\tflw \t" << "f1" << ", " << "%lo(" << arg->value << ")(t4)\n";
                        return RegDiscriptor("f1");
                    }
                    if(arg->data == IR::DATA_DOUBLE)
                    {
                        ofs << "\tlui \t" << "t4" << ", " << "%hi(" << arg->value << ")\n";
                        ofs << "\tfld \t" << "f1" << ", " << "%lo(" << arg->value << ")(t4)\n";
                        return RegDiscriptor("f1");
                    }
                }
            }
            else
            {
                if(arg->is_array)
                {
                    ofs << "\taddi\t" << "t4, s0, -" << std::to_string(stack_frame->var_s0_offset[arg->var_no]) << "\n";  
                    return RegDiscriptor("t4");
                }
                else
                {
                    if(arg->reg.spill == false)
                    {
                        return arg->reg;
                    }
                    else
                    {
                        if(arg->data == IR::DATA_INT || arg->data == IR::DATA_BOOL)
                        {
                            ofs << "\tlw  \t" << "t4, -" << std::to_string(stack_frame->var_s0_offset[arg->var_no]) << "(s0)\n"; 
                            return RegDiscriptor("t4");
                        }
                        if(arg->data == IR::DATA_FLOAT)
                        {
                            ofs << "\tflw  \t" << "f1, -" << std::to_string(stack_frame->var_s0_offset[arg->var_no]) << "(s0)\n"; 
                            return RegDiscriptor("f1");
                        }
                        if(arg->data == IR::DATA_DOUBLE)
                        {
                            ofs << "\tfld  \t" << "f1, -" << std::to_string(stack_frame->var_s0_offset[arg->var_no]) << "(s0)\n"; 
                            return RegDiscriptor("f1");
                        }
                    }
                }
            }
            break;
        
        case IR::TYPE_TEMP_VAR:
            if(arg->is_array)
            {
                ofs << "\taddi\t" << "t4, s0, -" << std::to_string(stack_frame->temp_s0_offset[arg->var_no]) << "\n";  
                return RegDiscriptor("t4");
            }
            else
            {
                if(arg->reg.spill == false)
                {
                    return arg->reg;
                }
                else
                {
                    if(arg->data == IR::DATA_INT || arg->data == IR::DATA_BOOL)
                    {
                        ofs << "\tlw  \t" << "t4, -" << std::to_string(stack_frame->temp_s0_offset[arg->var_no]) << "(s0)\n"; 
                        return RegDiscriptor("t4");
                    }
                    if(arg->data == IR::DATA_FLOAT)
                    {
                        ofs << "\tflw  \t" << "f1, -" << std::to_string(stack_frame->temp_s0_offset[arg->var_no]) << "(s0)\n"; 
                        return RegDiscriptor("f1");
                    }
                    if(arg->data == IR::DATA_DOUBLE)
                    {
                        ofs << "\tfld  \t" << "f1, -" << std::to_string(stack_frame->temp_s0_offset[arg->var_no]) << "(s0)\n"; 
                        return RegDiscriptor("f1");
                    }
                    else
                    {
                        return RegDiscriptor();
                    }
                }
            }
            break;
        case IR::TYPE_CONST:
            if(arg->data == IR::DATA_INT)
            {
                ofs << "\tli  \t" << "t4" << ", " << arg->value << "\n";
                return RegDiscriptor("t4");
            }
            if(arg->data == IR::DATA_BOOL)
            {
                if(arg->value == "true")
                    ofs << "\tli  \t" << "t4" << ", " << "1" << "\n";
                if(arg->value == "false")
                    ofs << "\tli  \t" << "t4" << ", " << "0" << "\n";
                return RegDiscriptor("t4");
            }
            if(arg->data == IR::DATA_FLOAT)
            {
                ofs << "\tlui \t" << "t4" << ", " << "%hi(.LC" << std::to_string(rodata[arg]->var_no) << ")\n";
                ofs << "\tflw \t" << "f1" << ", " << "%lo(.LC" << std::to_string(rodata[arg]->var_no) << ")(t4)\n";
                return RegDiscriptor("f1");
            }
            if(arg->data == IR::DATA_DOUBLE)
            {
                ofs << "\tlui \t" << "t4" << ", " << "%hi(.LC" << std::to_string(rodata[arg]->var_no) << ")\n";
                ofs << "\tfld \t" << "f1" << ", " << "%lo(.LC" << std::to_string(rodata[arg]->var_no) << ")(t4)\n";
                return RegDiscriptor("f1");
            }
            break;
        default:
            break;
    }
}
```

`store_dest`的逻辑较为简单，检查目的操作数对应的var是否溢出，如果溢出则将至存入对应的栈中的位置。

基于此，以加法运算为例，我们有如下的翻译策略：

```C
void RARISCVBuilder::emit_add(IR::IRList::iterator iter, std::ofstream& ofs, Frame* stack_frame)
{
    switch(iter->dest->data)
    {
        case IR::DATA_INT:
        {
            auto dest_reg = load_dest(iter->dest, stack_frame, ofs);
            auto arg1_reg = load_arg1(iter->arg1, stack_frame, ofs);
            auto arg2_reg = load_arg2(iter->arg2, stack_frame, ofs);
            ofs << "\tadd \t" << dest_reg.name << ", " << arg1_reg.name << ", " << arg2_reg.name << "\n";
            store_dest(iter->dest, stack_frame, ofs);
        }
            break;
        case IR::DATA_BOOL:
        {
            auto dest_reg = load_dest(iter->dest, stack_frame, ofs);
            auto arg1_reg = load_arg1(iter->arg1, stack_frame, ofs);
            auto arg2_reg = load_arg2(iter->arg2, stack_frame, ofs);
            ofs << "\tadd \t" << dest_reg.name << ", " << arg1_reg.name << ", " << arg2_reg.name << "\n";
            store_dest(iter->dest, stack_frame, ofs);
        }
            break;
        case IR::DATA_FLOAT:
        {
            auto dest_reg = load_dest(iter->dest, stack_frame, ofs);
            auto arg1_reg = load_arg1(iter->arg1, stack_frame, ofs);
            auto arg2_reg = load_arg2(iter->arg2, stack_frame, ofs);
            ofs << "\tfadd.s  " << dest_reg.name << ", " << arg1_reg.name << ", " << arg2_reg.name << "\n";
            store_dest(iter->dest, stack_frame, ofs);
        }
            break;
        case IR::DATA_DOUBLE:
        {
            auto dest_reg = load_dest(iter->dest, stack_frame, ofs);
            auto arg1_reg = load_arg1(iter->arg1, stack_frame, ofs);
            auto arg2_reg = load_arg2(iter->arg2, stack_frame, ofs);
            ofs << "\tfadd.d  " << dest_reg.name << ", " << arg1_reg.name << ", " << arg2_reg.name << "\n";
            store_dest(iter->dest, stack_frame, ofs);
        }
            break;
    }
}
```

值得一提的是，对于ASSIGN语句，如果目的操作数在寄存器中且源操作数1为常量，我们的翻译策略会多出一条`mv`指令，这一点可以通过机器相关优化解决，但是我们没有时间去做这一点。

`alloc_frame`负责初始时栈帧的分配，其检查IR语句中所有被溢出的变量，并为它们设置好偏移量，设置栈帧大小。

```C++
Frame* RARISCVBuilder::alloc_frame(IR::IRList::iterator iter)
{   
    int frame_size = 16;  //s0, ra
    std::unordered_map<int, int> var_s0_offset;
    std::unordered_map<int, int> temp_s0_offset;

    if(iter->dest->value != "main")
    {
        for(int i = 0; i < REG_NUM; i++)
        {
            if(use_info[func_name][i] == true && reg_is_callee_saved(reg[i]))
            {
                int growth = 8;
                frame_size += growth;
                reg[i].frame_offset = frame_size;
                callee_saved.push_back(reg[i]);
                continue;
            }
            if(float_use_info[func_name][i] == true && reg_is_callee_saved(float_reg[i]))
            {
                int growth = 8;
                frame_size += growth;
                float_reg[i].frame_offset = frame_size;
                float_callee_saved.push_back(float_reg[i]);
                continue;
            }
        }
    }

    iter++;
    while(iter->op != IR::OP_FUNCEND)
    {
        if(iter->op == IR::OP_EXPLICIT_ARG)
            iter++;
        if(iter->op == IR::OP_VARDEC || iter->op == IR::OP_PARAM)
        {
            if(iter->dest->reg.spill == true)
            {
                int growth = (iter->dest->size == 1) ? 4 : (iter->dest->size);
                frame_size += growth;
                var_s0_offset.insert(std::pair<int, int>(iter->dest->var_no, frame_size));
            }
        }
        else
        {
            if(iter->dest->type == IR::TYPE_TEMP_VAR && iter->dest->data != IR::DATA_DEFUALT)
            {
                if(iter->dest->reg.spill == true)
                {
                    if(temp_s0_offset.count(iter->dest->var_no) > 0)
                    {
                        iter++;
                        continue;
                    }
                    else
                    {
                        if(iter->dest->is_array == false)
                        {
                            int size = getDataSize(iter->dest->data);
                            int growth = (size == 1) ? 4 : size;
                            frame_size += growth;
                            temp_s0_offset.insert(std::pair<int, int>(iter->dest->var_no, frame_size));
                        }
                        else
                        {
                            int size = iter->dest->size;
                            int growth = (size == 1) ? 4 : size;
                            frame_size += growth;
                            temp_s0_offset.insert(std::pair<int, int>(iter->dest->var_no, frame_size));
                        }
                    }
                }
            }
        }
        iter++;
    }

    int align = frame_size % 16;
    frame_size = (align == 0) ? frame_size : ((frame_size / 16) * 16 + 16); 

    return new Frame(frame_size, var_s0_offset, temp_s0_offset);
}
```



###### （局部）浮点数的RISC-V代码生成

浮点数的生成逻辑和全局变量基本一致，只不过我们首先遍历一次IR去搜集局部浮点常量的信息，并且设置好相应的`FloatLabel`，在翻译的时候利用`FloatLabel`信息就可以了，它们被放在`.rodata`段

```C++
void RARISCVBuilder::gen_rodata(IR::IRList ircode)
{
    int var_no = 0;
    for(auto iter = ircode.begin(); iter != ircode.end(); iter++)
    {
        if(iter->op == IR::OP_ASSIGN || iter->op == IR::OP_ASSIGN_SETVAL || iter->op == IR::OP_MINUS)
        {
            if(iter->arg1->type == IR::TYPE_CONST)
            {
                if(iter->arg1->data == IR::DATA_FLOAT)
                {
                    float value_float = std::stof(iter->arg1->value);
                    FloatLabel* new_label = new_float_label(2, float_to_word(value_float), var_no++);
                    rodata.insert({iter->arg1, new_label});
                }
                if(iter->arg1->data == IR::DATA_DOUBLE)
                {
                    double value_double = std::stod(iter->arg1->value);
                    FloatLabel* new_label = new_float_label(3, double_to_word(value_double), var_no++);
                    rodata.insert({iter->arg1, new_label});
                }
            }
        }
        if(iter->op == IR::OP_ADD || iter->op == IR::OP_SUB || iter->op == IR::OP_DIV || iter->op == IR::OP_MOD || iter->op == IR::OP_MUL ||\
                iter->op == IR::OP_BEQ || iter->op == IR::OP_BNQ || iter->op == IR::OP_BGT || iter->op == IR::OP_BGE)
        {
            if(iter->arg1->type == IR::TYPE_CONST)
            {
                if(iter->arg1->data == IR::DATA_FLOAT)
                {
                    float value_float = std::stof(iter->arg1->value);
                    FloatLabel* new_label = new_float_label(2, float_to_word(value_float), var_no++);
                    rodata.insert({iter->arg1, new_label});
                }
                if(iter->arg1->data == IR::DATA_DOUBLE)
                {
                    double value_double = std::stod(iter->arg1->value);
                    FloatLabel* new_label = new_float_label(3, double_to_word(value_double), var_no++);
                    rodata.insert({iter->arg1, new_label});
                }
            }
            if(iter->arg2->type == IR::TYPE_CONST)
            {
                if(iter->arg2->data == IR::DATA_FLOAT)
                {
                    float value_float = std::stof(iter->arg2->value);
                    FloatLabel* new_label = new_float_label(2, float_to_word(value_float), var_no++);
                    rodata.insert({iter->arg2, new_label});
                }
                if(iter->arg2->data == IR::DATA_DOUBLE)
                {
                    double value_double = std::stod(iter->arg2->value);
                    FloatLabel* new_label = new_float_label(3, double_to_word(value_double), var_no++);
                    rodata.insert({iter->arg2, new_label});
                }
            }
        }
    }
}
```



###### 向量运算指令的RISC-V代码生成

之前我们提到，我们将向量运算的任务交给了代码生成部分来处理，我们目前没有添加V扩展的指令，对于向量运算，本质上还是通过循环，不断计算偏移量、访存来实现的翻译，这里以加法的向量运算为例，给出它的具体实现：

```C++
void RARISCVBuilder::emit_addv(IR::IRList::iterator iter, std::ofstream& ofs, Frame* stack_frame)
{
    int loops = iter->dest->size / getDataSize(iter->dest->data);
    int i = 0;
    auto arg1_reg = load_arg1(iter->arg1, stack_frame, ofs);
    auto arg2_reg = load_arg2(iter->arg2, stack_frame, ofs);
    auto dest_reg = load_dest(iter->dest, stack_frame, ofs);

    if(iter->dest->data == IR::DATA_INT || iter->dest->data == IR::DATA_BOOL)
    {
        ofs << "\taddi\t" << "sp, sp, -" << std::to_string(16) << "\n";
        stack_frame->frame_size += 8;
        arg1_reg.frame_offset = stack_frame->frame_size;
        ofs << "\tsd  \t" << arg1_reg.name << ", -" << std::to_string(stack_frame->frame_size) << "(s0)\n";
        stack_frame->frame_size += 8;
        arg2_reg.frame_offset = stack_frame->frame_size;
        ofs << "\tsd  \t" << arg2_reg.name << ", -" << std::to_string(stack_frame->frame_size) << "(s0)\n";
    }
    
    while(i < loops)
    {
        switch(iter->dest->data)
        {
            case IR::DATA_INT:
                ofs << "\tlw  \t" << arg1_reg.name << ", " << std::to_string(i * getDataSize(iter->dest->data)) << "(" << arg1_reg.name << ")\n";
                ofs << "\tlw  \t" << arg2_reg.name << ", " << std::to_string(i * getDataSize(iter->dest->data)) << "(" << arg2_reg.name << ")\n";
                ofs << "\tadd \t" << arg1_reg.name << ", " << arg1_reg.name << ", " << arg2_reg.name << "\n";
                ofs << "\tsw  \t" << arg1_reg.name << ", " << std::to_string(i * getDataSize(iter->dest->data)) << "(" << dest_reg.name << ")\n";
                break;
            case IR::DATA_BOOL:
                ofs << "\tlw  \t" << arg1_reg.name << ", " << std::to_string(i * getDataSize(iter->dest->data)) << "(" << arg1_reg.name << ")\n";
                ofs << "\tlw  \t" << arg2_reg.name << ", " << std::to_string(i * getDataSize(iter->dest->data)) << "(" << arg2_reg.name << ")\n";
                ofs << "\tadd \t" << arg1_reg.name << ", " << arg1_reg.name << ", " << arg2_reg.name << "\n";
                ofs << "\tsw  \t" << arg1_reg.name << ", " << std::to_string(i * getDataSize(iter->dest->data)) << "(" << dest_reg.name << ")\n";
                break;
            case IR::DATA_FLOAT:
                ofs << "\tflw  \t" << "f1" << ", " << std::to_string(i * getDataSize(iter->dest->data)) << "(" << arg1_reg.name << ")\n";
                ofs << "\tflw  \t" << "f2" << ", " << std::to_string(i * getDataSize(iter->dest->data)) << "(" << arg2_reg.name << ")\n";
                ofs << "\tfadd.s  "<< "f1, f1, f2\n";
                ofs << "\tfsw  \t" << "f1" << ", " << std::to_string(i * getDataSize(iter->dest->data)) << "(" << dest_reg.name << ")\n";
                break;
            case IR::DATA_DOUBLE:
                ofs << "\tfld  \t" << "f1" << ", " << std::to_string(i * getDataSize(iter->dest->data)) << "(" << arg1_reg.name << ")\n";
                ofs << "\tfld  \t" << "f2" << ", " << std::to_string(i * getDataSize(iter->dest->data)) << "(" << arg2_reg.name << ")\n";
                ofs << "\tfadd.d  "<< "f1, f1, f2\n";
                ofs << "\tfsd  \t" << "f1" << ", " << std::to_string(i * getDataSize(iter->dest->data)) << "(" << dest_reg.name << ")\n";
                break;
        }
        i++;
        if(iter->dest->data == IR::DATA_INT || iter->dest->data == IR::DATA_BOOL)
        {
            ofs << "\tld  \t" << arg1_reg.name << ", -" << std::to_string(arg1_reg.frame_offset) << "(s0)\n";
            ofs << "\tld  \t" << arg2_reg.name << ", -" << std::to_string(arg2_reg.frame_offset) << "(s0)\n";
        }
    }
}
```



## 总结

### 实验结果总结

最终，我们完成了四种优化选择。如果需要添加优化选择，在`./compiler` 和`[path of test case]`之间分别添加`-O0`,`-O1`,`-O2`三个可选选项；其中，`-O0`为删除不可达基本块，基本块合并，表达式强度削减优化。`-O1`为在`-O0`基础上添加常量传播。`-O2`为在`-O1`基础上添加寄存器分配优化。如果不加可选选项表示默认不优化代码。

为了测试优化效果，我们选择测试样例中的1~20在开发板上进行实验对比，使用time命令得到程序运行时间。但是由于部分测试样例过于简单，如打印数字的程序，没有过多的优化空间，因此我们选择具有代表性的16~20样例进行对比试验，经多次测量取平均值得到如下测试数据：

| test_case | None（s） | -O0（s） | -O1（s） | -O2 （s） |
| --------- | --------- | -------- | -------- | --------- |
| 16        | 0.052     | 0.052    | 0.052    | 0.022     |
| 17        | 0.123     | 0.123    | 0.123    | 0.005     |
| 18        | 0.123     | 0.122    | 0.123    | 0.004     |
| 19        | 0.123     | 0.122    | 0.122    | 0.006     |
| 20        | 0.123     | 0.122    | 0.122    | 0.005     |

绘制折线图如下所示：

![QQ图片20220708000836](https://raw.githubusercontent.com/0x1z2c3v/image/master/QQ%E5%9B%BE%E7%89%8720220708000836.jpg)

从测试数据以及折线图可以看到`-O0`和`-O1`的优化效果并不明显，然而在添加了寄存器分配的`-O2`优化之后程序的运行时间相较于之前有了大幅度的缩减。 下面是第16个测试样例未经优化的部分汇编代码结果：

```C
L5:
        lw      t2, -24(s0)
        mv      t1, t2
        sw      t1, -36(s0)
        lw      a0, -36(s0)
        call    print_int
        li      t2, 0
        mv      t1, t2
        sw      t1, -40(s0)
        ld      t2, -40(s0)
        mv      a0, t2
        ld       ra, -8(s0)
        ld       s0, -16(s0)
        addi    sp, sp, 48
        ret
```

经过优化后的对应部分的汇编代码：

```
L5:
        mv      a1, a1
        mv      a0, a1
        call    print_int
        li      t5, 0
        mv      a1, t5
        mv      a0, a1
        ld      ra, -8(s0)
        ld      s0, -16(s0)
        addi    sp, sp, 16
        ret
```

经过对比可以发现，合理使用寄存器可以明显地减少数据的访存操作（7次->2次），从而达到减少运行时间的效果。



### 分成员总结

#### 余秋初的总结

我在PR003中主要负责了中间代码的设计与生成、寄存器分配模块的设计与实现、RISC-V代码的生成。我个人感觉编译原理及其研讨课是国科大课程体系中为数不多的和PL/SE相关的课程，实验做起来也能明显感觉到和其他研讨课的区别，越做越能感觉到自己平时在编写代码时的不严谨性。在进行PR003之前以及PR003过程中，我们之前写的模块没有任何单元测试，也没有任何严格封装的抽象，很多模块之前甚至具有很高的耦合度，这都给我在编写最后的寄存器分配模块和代码生成器模块时带来了调试上的难题。一会这里double free了，另一边又Segmentation fault了......这些都是后期调试过程中常有的事情。不过经历了这次实验（可能是代码量最大的研讨课实验？），我也更好地明白了Software为什么能成为一项工程，对上学期所选的面向对象程序设计课程有了更加深刻的理解，慢慢发现了一点读完paper然后复现出来的乐趣，这些经历都让我受益匪浅。



#### 夏瑞阳的总结

我在PR003中主要负责了机器无关优化部分的设计与实现。在我看来，编译原理研讨课最大的一个特点是，其代码量比较大，所以需要多人合作完成，这与以往的操作系统或是计算机网络的单人研讨课完全不同。实验中我们在编写代码之外，还花了许多时间与小组内成员讨论各自部分的设计细节与模块间接口的规范。这还仅仅是一个三人规模的实验性编译器，如果到了几十人甚至上百人规模的商业软件开发时，又需要遵循怎样的流程与规范以确保代码的可用性、可读性和可移植性呢？除了编译原理课程本身的知识外，也让我对软件开发的methodology有了更深的理解，也希望之后的研讨课中，老师们在实验开始前的课程里，能基于多人项目的特点给同学们一些建议。除此之外，，因为我们三人是同一寝室的室友，我们在实验过程中会采用每周组会的方式汇报本周的进度与下周的计划，否则面对时长一个多月的PR003，很有可能大量工作会被放在最后几周甚至一周内完成，因此我感觉到定期汇报进度真是挺好的制度。在这里我也感谢我的两位队友和室友，能容忍我的摸鱼和懒惰，也感谢他们的努力工作与辛劳付出。



#### 陈兢宇的总结

我在PR003实验中主要完成了中间代码向RISCV汇编代码翻译和最终性能测试的工作。通过编译原理的实验使我对编译器的设计有了更加深入的理解。编译器从原理，到实现，再到后来的改进和强化是一个复杂的工程。编译器的出现让高级语言到汇编语言成为现实，从而更好地方便我们使用计算机。通过cact的实验，虽然最终完成的是一个简单的编译器，但是通过实验将理论课上学到的知识落实到实践中去，提高代码能力的同时也进一步强化了理论知识。只有将理论落实到实践才能够真正领悟到编译器设计中的种种精妙绝伦之处。除此之外，通过实验也学习到了许多多人合作开发的技巧。得益于队友之前代码写的完整和规范，使得后来的翻译过程进行的比较顺利，感谢队友和老师在实验过程中给予的帮助。

