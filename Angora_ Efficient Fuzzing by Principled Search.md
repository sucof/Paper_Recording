# Angora: Efficient Fuzzing by Principled Search

标签（空格分隔）： Paper

---

## 论文来源
S&P - 2018

## 工作简介
Angora 的主要目标是通过解决路径约束而无需符号执行来增加分支覆盖范围。为了有效地解决路径约束，我们介绍了几个关键技术：弹性的字节级污点跟踪，上下文敏感分支计数，基于梯度下降的搜索和输入样本长度探索

## 工作背景
一些 Fuzzer 使用符号执行来解决路径约束，但符号执行速度较慢，无法有效地解决许多类型的约束

AFL 在程序执行上的开销很低，但是它创建的大多数输入都是无效的（即它们未能探索新的程序状态），因为它在不利用程序中数据流的情况下盲目地改变输入样本

 - 上下文敏感的分支覆盖。 AFL 使用上下文不敏感的分支覆盖来近似程序状态。我们的经验表明，为分支覆盖添加上下文允许 Angora 更普遍地探索程序状态
 - 弹性字节级污点跟踪。大多数路径约束仅取决于输入中的几个字节。通过跟踪哪些输入字节流入哪个路径约束，Angora 只会变换这些字节而不是整个输入样本，因此大大减少了勘探空间
 - 基于梯度下降搜索。在畸变输入样本以满足路径约束时，Angora 避免了使用符号执行这种代价高昂的技术，并且它不能解决许多类型的约束。相反，Angora  使用机器学习中流行的梯度下降算法来解决路径约束
 - 类型与 shape 推断。输入中的许多字节在程序中统一用作单个值，例如，输入中的一组四个字节用作程序中的 32 位有符号整数。为了让梯度下降有效地进行搜索，Angora 定位上述 shape 并推断其类型
 - 输入样本长度探索。只有当输入的长度超过某个阈值时，程序才会探索某些状态，但符号执行和梯度下降都不能告诉模糊器何时增加输入长度。 Angora 检测输入的长度是否会影响路径约束，然后充分增加输入长度

### AFL
AFL 采用轻量级编译时插桩与遗传算法自动发现可能在目标程序中触发新内部状态的测试用例。作为基于覆盖范围的模糊器，AFL 生成输入样本来遍历程序中的不同路径以触发错误
#### 分支覆盖度
在执行期间 AFL 会计算每个分支执行的次数，它将分支表示为元组 (lprev, lcur)，lprev 和 lcur 分别是条件语句之前和之后的基本块 ID。AFL 通过使用轻量级插桩获取分支覆盖信息，在每个分支点注入插桩。每次运行，AFL 都分配 path trace table 来为分支执行计数，该表的索引是分支的散列值 h(lprev, lcur)

AFL 还保留了全局的 branch coverage table，每个条目都包含一个 8 位向量，该向量记录分支执行的次数。向量中的每一位都代表一个范围，b0,````,b7 分别代表 [1], [2], [3], [4, 7], [8, 15], [16, 31], [32, 127], [128, ∞)。例如，如果设置了 b3，则表示存在一个该分支执行 4 到 7 次的执行

AFL 比对这两张表，以启发式确定输入样本是否触发程序的新内部状态。如果发生以下任一情况，输入样本将触发新的内部状态

 - 程序执行了新分支。即 path trace table 中存在该分支的入口，但是 branch coverage table 没有该分支的入口
 - 当前运行中执行的该分支的次数与之前任何一次都不同。AFL 通过检查表示范围 n 的位是否在 branch coverage table 的相应位向量中被设置来确定

#### 畸变策略
AFL 随机采用以下畸变：

 - 位、字节翻转
 - 尝试 interesting bytes\word\dwords
 - 小整型对 bytes\word\dwords 的加减
 - 全随机地单字节集合
 - 通过覆写、插入、memset 来进行块删除、块复制
 - 在随机位置拼接两个不同的样本文件

## 工作设计
算法一展示了 Angora 的两个阶段：插桩与模糊测试循环。在模糊测试循环的每次迭代中，Angora 选择一个未探索的分支并搜索用以探索该分支的输入样本。我们引入以下技术来有效地查找输入样本：

 - 对于大多数条件语句，其谓词仅受输入样本中的几个字节的影响。在探索分支时，Angora 会确定哪些输入字节流入相应的谓词，并着重于仅畸变这些字节
 - 确定那些输入字节需要畸变后，Angora 需要决定如何畸变。使用基于随机/启发式的畸变不能有效地找到令人满意的值，相反的，我们将分支上的路径约束看作输入样本上黑盒函数的约束条件，使用梯度下降算法来解决这些约束
 - 在梯度下降期间，根据参数评估黑盒函数，其中一些参数由多个字节组成。例如，当输入样本中的四个连续字节总是作为一个整数一起送入条件语句时，应该视这四个字节为该函数的单个参数，而不是四个独立的参数。为了实现这一目标，需要推断输入样本中哪些字节被统一用作单个值以及该值的类型
 - 仅仅改变输入样本中的字节是不够的，只有在输入样本超过阈值后才会触发的一些错误也需要被发现。输入太短，可能不会触发某些错误，如果输入太长，程序运行速度可能会太慢。Angora 检测程序在更长的输入下可能探索新分支的时间，并确定所需的最小长度

![image_1ccn7l7v61vr81bf51oonl1749s9.png-96.8kB][1]

## 工作细节

![image_1ccn7pcmmhjf1lle1ph31tk65qhm.png-25.7kB][2]

上图为条件分支模糊测试的流程，下面是核心技术示例程序：

![image_1ccn7q0sqa2f1s8m1r7n1p721b5q13.png-33.9kB][3]

 - 字节级污点跟踪。对第二行条件语句进行模糊测试时，使用字节级污点跟踪，Angora 确定 1024-1031 字节送入此表达式
 - 基于梯度下降的搜索算法。Angora 需要找到运行第二行条件语句的两个分支的输入。Angora 将条件语句中的表达式视为输入样本 x 上的函数 f(x)，并使用梯度下降找到两个输入使得 f(x)>0 与 f(x0)<0
 - shape 和类型推断：f(x)是向量 x 的函数。在梯度下降的过程中，Angora 分别计算 f 的每个分量的偏导数，所以它必须确定每个分量以及类型。第二行中，Angora 确定 x 由两个分量组成，每个分量由输入样本中的四个字节组成，并且类型为 32 位有符号整数
 - 输入样本长度探索。除非输入样本超过 1032 个字节，否则 main 不会调用 foo 函数，我们试图确定更长的输入是否会探索新的状态。第十二行，fread 的返回值和 1024 相比较，Angora 才会知道至少 1024 字节才会探测到错误分支，同样的，第十六行和第十九行也让 Angora 知道至少 1032 字节才能执行 foo 函数

### 上下文敏感分支计数
AFL 的分支覆盖有几个优点：

 - 节省空间。分支的数量与程序的大小呈线性关系
 - 使用范围来对分支执行计数为确定不同的执行计数启发式地表明程序是否进入新状态。当执行技术很小时，技术的每次变动都是显著的，但是当执行计数很大时，变换必须足够大才能被认为是重要的改变

其缺点在于，AFL 无法区分不同上下文中同一分支的执行情况。如下图所示：

![image_1ccnd18n0o3l1ceu10ir5g84dr1g.png-25.9kB][4]

第一次运行时，程序输入“10”，调用第十九行的 f 函数时执行第四行的真分支，稍后调用第二十一行的 f 函数时会执行第十行的假分支。由于 AFL 对分支的定义是上下文不敏感的，会认为两个分支都已经被执行了。随后送入新样本“01”时，AFL 会认为这个输入样本不会触发新的内部状态，因为第四行和第十行的分支都在前一次运行中执行过。但实际上，新输入会触发一个新的内部状态，因为当 `input[2]`==1 时会触发第六行的崩溃

我们将分支定义为元组 (lprev, lcur, context)，其中 lprev 和 lcur 分别是条件语句之前和之后的基本块 ID，context 是 h(stack)，h 为散列函数，stack 包含调用堆栈的状态。例如，让图3中的程序首先在输入 10 上运行。从第 19 行进入 f() 后，它将执行分支（l3，l4，[l19]）。然后，在它从第 21 行进入 f() 之后，它将执行分支（l3，l10，[l21]）。相反，当程序在输入 01 上执行时，它将执行分支（l3，l10，[l19]），然后执行（l3，l4，[l21]）。通过将调用上下文合并到分支的定义中，Angora 可以检测到第二次运行会触发一个新的内部状态，这会在畸变 `input[2]` 时触发第 6 行的崩溃

向分支添加上下文会增加独特分支的数量，当深度递归发生时这可能会非常明显。我们当前的实现通过选择特定函数 h 来计算调用堆栈的散列值，其中 h 计算堆栈上所有调用位置的 ID 的异或值，从而减轻了这个问题。当 Angora 对程序进行测试时，它会为每个调用位置分配一个随机ID。因此，当函数 f 递归调用自身时，无论 Angora 多少次将相同调用位置的 ID 压入调用堆栈，h(stack) 都会输出至多两个唯一值，这最多会使函数 f 独特分支的数量增加一倍。我们对真实世界程序的评估表明，在并入上下文之后，独特分支的数量增加了7.21倍，以换取代码覆盖率的改善

### 字节级污点跟踪
在大多数程序运行中，污点跟踪是不必要的。一旦我们对输入样本执行了污点跟踪，就可以记录哪些字节偏移送入哪个条件语句。然后对这些字节进行畸变时，可以只运行程序而不使用污点跟踪。这虽然使得对一个输入样本的污染跟踪成本高过了畸变，但仍然使 Angora 具有与 AFL 类似的样本执行吞吐量

Angora 将程序中的每个变量 x 与一个污点标签 tx 相关联，这个标签表示输入中可能送入 x 的字节偏移。污点标签的数据结构对它的内存占用有很大的影响。一个简单的实现是将每个污点标签表示为一个位向量，其中每个位 i 表示输入样本中的第 i 个字节。然而，由于这个位向量的大小在输入样本的大小上呈线性增长，所以这个数据结构对于大的输入样本来说是不可行的

为了减少污点标签的大小，我们可以将位向量存储在一个表格中，并将表格的索引用作污点标签，只要表中条目数的对数远小于最长位向量的长度，就可以大大减少污点标签的大小

但对这种新型的数据结构，污点标签必须支持以下操作

 - INSERT(b)，插入位向量 b 并返回其标签
 - FIND(t)，返回污点标签 t 的位向量
 - UNION(tx, ty)，返回表示污点标签 tx 和 ty 的位向量的并集的污点标签

FIND 开销不大，但 UNION 代价很高。首先它找到两个标签的位向量并计算它们的 union u。这一步很容易。接下来，它搜索表格以确定你是否已经存在。如果没有，它会向其中添加。但如何有效地搜索？线性搜索会很昂贵。或者，我们可以构建一个位向量的散列集，但是如果它们中有很多并且每个位向量很长，则计算散列码和存储散列集的空间需要很长时间。由于在跟踪算术表达式中的受污染数据时 UNION 是一种常见操作，因此它必须高效

我们提出了新的数据结构来存储，对于每个位向量，数据结构使用无符号整数为其分配唯一标签。当程序插入一个新的位向量时，数据结构会为其分配下一个可用的无符号整数

数据结构包含两个组件：

 - 二叉树将位向量映射到其标签。每个位向量 b 由级别为 |b| 的唯一树节点 vb 表示，其中 |b| 是 b 的长度。vb 存储 b 的标签。要从根到达 vb，要顺序检查 b0，b1，...。如果 bi 为 0，则转到左边的节点，否则转到右边的节点。每个节点都包含一个指向其父节点的返回指针，以允许我们检索从 vb 开始的位向量
 - 查找表将标签映射到它们的位向量。标签是该表的索引，并且相应的条目指向表示该标签的位向量的树节点

所有叶子节点都代表向量，内部节点都不代表向量。还使用了正则表达式在插入向量时修剪向量：

 - 删除向量尾部的 0
 - 按照向量第一位到最后一位的顺序遍历树
    - 如果某位为 0，请检查左边的节点
    - 否则检查右边的结点
    - 如果节点不存在则创建节点
 - 将向量的标签存储在我们访问的最后一个节点中

算法二阐述了插入操作、算法三阐述了查找操作、算法四阐述了并集操作。请注意，当我们创建节点时，最初它不包含标签。后来，如果这个节点是我们插入一个位向量时访问的最后一个节点，我们将这个位向量的标签存储在这个节点中。通过此优化，此树具有以下属性：

 - 每个叶子节点都包含一个标签
 - 内部节点可能包含标签。我们可能会在没有标签的内部节点中存储标签，但我们绝不会替换任何内部节点中的标签

这样从 O(nl) 的空间需求降低到了 O(l logl)，大大减少了内存占用量

### 基于梯度下降的搜索算法
粗糙的启发式算法不能快速找到合适的输入值，相反，我们认为这是一个搜索问题，可以利用机器学习中的搜索算法

我们视分支的谓词为黑箱函数 f(x) 的约束，其中 x 是送入谓词的输入值的向量，f() 捕获从程序开始到此谓词的路径。f(x) 有三种约束：

 - f(x) < 0
 - f(x) <= 0
 - f(x) == 0

所有形式的比较都能转化为以上三种约束条件。如果条件语句的谓词包含逻辑运算符 && 或 ||，则 Angroa 将该语句拆分为多个条件语句。例如，它将（a && b）{s} else {t} 分解为 if（a）{if（b）{s} else {t}} else {t}

![image_1ccnhnpgc2qo6m1i0d12ni1sud1t.png-7.8kB][5]

下图显示了搜索算法，从最初的 x0 开始，找到 x 使得 f(x) 满足约束条件。请注意，为了满足每种类型的约束，我们需要最小化 f(x)，并使用梯度下降

![image_1ccnholkp1u4o1nsq1ent2l81dt12a.png-48.9kB][6]

梯度下降找到函数 f(x) 的最小值，该方法是迭代的。每次迭代从 x 开始，计算 ∇xf(x)（在 x 处 f(x) 的梯度），并将 x 更新为 x − π∇xf(x) ，其中 π 是学习率

在训练神经网络时，研究人员使用梯度下降法找到一组权重，以尽量减少训练误差。但是，梯度下降具有的问题是，它有时可能会停留在不是全局最小值的局部最小值。幸运的是，这在模糊测试中通常不是问题，因为我们只需要找到一个足够好的输入 x 而不是全局最优的 x

但是，在应用梯度下降到模糊时，我们面临着独特的挑战。梯度下降需要计算梯度 ∇x f(x) 。在神经网络中，我们可以用 analytic form 写出∇x f(x) 。然而，在模糊测试中，我们没有 f(x) 的 analytic form。其次，在神经网络中，f(x) 是连续函数，因为 x 包含网络的权重，但在模糊 f(x) 中通常是离散函数。这是因为典型程序中的大多数变量都是离散的，所以 x 中的大多数元素都是离散的。而我们使用数值近似来解决这些问题

理论上梯度下降可以解决任何约束。在实践中，梯度下降能多快地解决一个约束取决于数学函数的复杂性

 - 如果 f(x) 是单调或凸的，则即使 f(x) 具有复杂的分析形式，梯度下降也可以快速找到解。例如，考虑约束 f(x) <0，其中 f(x) 使用一些多项式系列逼近log（x）。由于复杂的分析形式，这个约束对于符号执行来说很难解决。然而，由于 f(x) 是单调的，所以梯度下降很容易求解
 - 如果梯度下降发现的局部最小值满足约束，找到解决方案也很快
 - 如果局部最小值不满足约束，Angora 必须随机走到另一个值 x0，并开始执行梯度下降，希望找到另一个满足约束条件的局部最小值

请注意，Angora 不会生成 f(x) 的 analytic form，而是运行程序来计算 f(x)

### shape 与类型推断
一般来说，我们将 x 中的每个元素作为送入谓词的输入样本中的一个字节即可。但是由于类型不匹配，会导致梯度下降存在问题。程序将这些字节组合为单个值，并且只使用表达式中的组合值，因此，当我们向除最低有效字节以外的任何字节添加小 δ 时，我们将显着改变该组合值，这会导致计算出的偏导数成为真实价值的可靠近似值

为了避免这个问题，我们必须确定（1）输入中的哪些字节总是作为程序中的单个值一起使用，即 shape 推断；（2）该值的类型是什么，即类型推断

对于 shape 推断，最初输入中的所有字节都是独立的。在污点分析过程中，当一条指令将一串输入字节读入一个变量，该变量的大小与原始类型的大小相匹配时（例如，1,2,4,8字节），就将这些字节标记为属于相同的价值。当发生冲突时，会使用最小的尺寸

对于类型推断，Angora 依赖于对该值进行操作的指令的语义。例如，如果一条指令以一个有符号整数运算，则推断相应的操作数是一个有符号整数。当使用相同的值作为有符号和无符号类型时，Angora 将它视为无符号类型。请注意，即使未能推断出值的精确大小和类型时，也不防止梯度下降可以找到解决方案 - 只是搜索需要更长的时间

### 样本长度探索
与其他 Fuzzer 一致，Angora 也从尽可能小的样本开始启动。在污点跟踪期间，Angora 将类似 read 函数的目标内存与输入样本中相应的字节偏移相关联。它还标记来自具有特殊标签的 read 调用的返回值。如果在条件语句中使用返回值并且不满足约束条件，那么 Angora 就会增加输入样本的长度，以便 read 调用可以获取其请求的所有字节

## 工作评估
对于每个需要测试的程序，Angora 使用 LLVM Pass 插桩程序生成相应的可执行文件。插桩会

 - 收集条件语句的基本信息，通过污点分析将条件语句链接到其相应的输入样本字节偏移量。在每个输入样本中，Angora 只运行一次
 - 记录执行时 trace 来识别新输入样本
 - 在运行时支持上下文
 - 收集路径谓词中表达式的值

扩展 DataFlowSanitizer（DFSan）实现了 Angora 的污点跟踪，并为操作 FIND 和 UNION 提供了缓存，显著加速了污点跟踪

Angora 依赖于 LLVM 4.0.0，它的 LLVM Pass 包含 820 行 C++ 代码，运行时有 1950 行 C++ 代码用于存储污点标签的数据结构以及用于污染样本、跟踪条件语句的 Hook

除了两个分支的 if 语句外，LLVM IR 还支持 switch 语句，Angora 将每个 switch 语句转换为一系列 if 语句

Angora 识别在条件语句中出现的用于比较字符串和数组的 libc 函数。如，将 “strcmp(x, y)” 转换为 “x strcmp y”，其中 strcmp 是 Angora 能识别的比较运算符

我们在 4488 行 Rust 代码中实现了 Angora，使用 fork server 和 CPU binding 技术优化了 Angora

使用 gcov 测量 Angora 对八个真实程序的覆盖率，记录了程序执行的所有行和分支。使用 Angora 生成的每个输入提供给 gcov 编译的程序以获得累计代码覆盖率，afl-cov3 已经可以自动执行此操作

![image_1ccnn1il31a14ntlm0dat1ei12n.png-19.7kB][7]
梯度下降可以解决更多的约束

Angora 插桩后的程序与 AFL 插桩后的程序的执行速度大致相同，AFL 略高

## 工作思考

首先，LAVA 使用“魔术字节”来保护包含错误的分支，但是一些魔法字节不是直接从输入中复制，而是从输入中计算出来的。由于VUzzer和Steelix的“魔术字节”策略只能将魔术字节直接复制到输入中，因此该策略无法创建探索这些分支的输入。相比之下，Angora 追踪输入到谓词中的输入字节偏移量，然后通过梯度下降对这些偏移量进行变异，而不是假设“魔术字节”或输入与谓词之间的任何其他特殊关系，因此 Angora 可以找到可以探索这些偏移量的输入分支
其次，VUzzer 盲目地尝试“魔术字节”策略，而 Steelix 专注于“魔术字节”策略。相比之下，Angora 使用其所有计算能力来解决尚未探索的分支的路径约束，因此它可以覆盖更多分支，因此可以快速找到 LAVA-M 中大部分注入的错误

AFL 的作者观察到，独特分支的数量（没有上下文）通常在 2K 到 10K 之间，具有 2<sup>16</sup> 个分区的散列表巨足够了，而合并了上下文的独特分支数量会增加至多 8 倍，故而为保持相同的碰撞率，使用 2<sup>20</sup> 的哈希表。因为 Angora 对于每个输入样本都只遍历一次哈希表寻找新路径，其不太受散列表大小的影响

VUzzer 使用压缩的位集数据结构来表示污点标签，其中每个位对应于输入中的唯一字节偏移量。因此，对于具有复杂输入字节偏移模式的值，污点标签的大小很大，因为它们无法被有效压缩。相比之下，Angora 将字节偏移量存储在树中，并将索引作为污点标签使用，因此无论标签中有多少输入字节偏移量，污点标签的大小都是恒定的。例如，当多个值的污点标签具有相同的字节偏移量时，VUzzer 将这些字节偏移量重复存储在每个污点标签中，但是 Angora 将这些字节偏移量仅存储在树中一次，从而大大减少了内存消耗


  [1]: http://static.zybuluo.com/Titan/lvu1b2zq95ubwis6h8jyf9nk/image_1ccn7l7v61vr81bf51oonl1749s9.png
  [2]: http://static.zybuluo.com/Titan/6b1qlwc70n51xqrysz5wlmsn/image_1ccn7pcmmhjf1lle1ph31tk65qhm.png
  [3]: http://static.zybuluo.com/Titan/ebkman39pxt1bx4zzdro4z0h/image_1ccn7q0sqa2f1s8m1r7n1p721b5q13.png
  [4]: http://static.zybuluo.com/Titan/rvn14kq8ghvknidqxj6n2not/image_1ccnd18n0o3l1ceu10ir5g84dr1g.png
  [5]: http://static.zybuluo.com/Titan/y70sb19md6plx5itmbp2xesb/image_1ccnhnpgc2qo6m1i0d12ni1sud1t.png
  [6]: http://static.zybuluo.com/Titan/812703vkn0fywvgea5oeww03/image_1ccnholkp1u4o1nsq1ent2l81dt12a.png
  [7]: http://static.zybuluo.com/Titan/xglztk5fbshn23m3m55i6gll/image_1ccnn1il31a14ntlm0dat1ei12n.png