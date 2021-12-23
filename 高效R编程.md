这是《高效R语言编程》的学习笔记，前面的笔记在这里：
[https://blog.csdn.net/zd200572/article/details/115349366](https://blog.csdn.net/zd200572/article/details/115349366)
[https://www.jianshu.com/p/71392ef45d01](https://www.jianshu.com/p/71392ef45d01)
很多R语言用户并不认为自己是程序员，我也是:)，精通专业知识，理解R语言的标准数据结构，但是缺乏正规编程训练，你是这样的吗？
# 高效编程的5个技巧
- 1、小心，尽量不要增大向量的大小
- 2、尽可能向量化代码
- 3、适当时机下使用因子
- 4、通过缓存变量避免不必要的计算
- 5、字节编译包可使性能轻而易举大幅提升
# 一般性建议
底层语言如C，需要你自己进行内存管理，而R语言这些不用你负责，优点是可交互，缺点是运行速度慢，特别是糟糕的代码，推荐书《The R Inferno》。**尽可能地访问底层的C函数**，函数调用越少越好。
# 内存分配
n=1000000时`seq_len(n)`瞬时完成，而`vec=numeric(n)#然后赋值要2s`，但是如果一个空向量`Vec=c()`要共一个半小时。
# 向量化代码
for循环代码慢不是因为循环，而是因为函数调用太多。
# 与用户交互
## 致使错误stop()
stop()抛出致命错误，执行终止，不再执行任何操作，下面的处理代替stop()更好些。
```
bad <- try(1+"1"),silent=T)
if(class(bad)=='try-error'
print("error")
```
try()和tryCatch()捕获错误，推荐《Advanced R》一书中介绍了更详细的错误处理方法。
#警告Warning()
解决警告，而不是忽略它。
`suppressWarnings()#隐藏警告`
# 信息输出
message()可以给出预计运行时间。cat()是另一个输出函数，仅用于print()/show()方法。
```
message()
suppressMessages() #禁止提示信息
cat()
```
# 不可见返回
比如绘图不可见，获得参数
 `invisible()`
# 因子
饱受争议，有用武之地，储存类别变量，看起来像字符，其实是int型的。总用或永远不用都是不明智的，通常，变量有固有顺序，或你有固定不变的类别集合，考虑使用因子。
##1） 内在排序 
因子可用于图形排序，通常read.csv()中自动转换为因子，我们一般options(stringsAsFactors = F)，但是作者出于可移植性考虑不建议将这个放到.Rprofile文件。
##2）固定类别
比如月份排序，因子可以实现，这指的英语的Dec这种。因子还比字符串稍微节约点空间。
# Apply函数家族
可以看作是循环的替代，第一次听说eapply()独立环境，这个我们应该用不到。将一个函数应用到每行或每列。参数可以放在后面传递给函数。
- apply()可以用于处理高维数组。
- lapply() 输入是向量/列表，返回列表。
- sapply()和vapply()与lapply()类似，返回值不一定是列表。
## 类型一致
函数的返回值以同样的形式是个好习惯，但是不是所有函数都这样，比如：
sapply() ，这会导致意想不到的问题。lapply()与vapply()一致，dplyr::select()与dplyr::filter()也是.purr中是map_dbl()代替Map()，flatten_df()代替unlist()。
# 缓存变量
也就是把一个计算过程存为变量，而不是每次计算，如果是100*1000的矩阵，速度会相差100倍。缓存更高级的形式是`memoise` 包，将已知结果存入可检索的缓存，加快运行速度。保存函数的运行结果，牺牲缓存换速度，最多能100倍的速度提升，在内存充足的今天应该还好，只要不上大数据，16G内存已经普遍了。典型应用是shiny app，可以回事用户得到结果，减少等待时间。
```
library(microbenchmark)
test <- function(x){
  a <- x*100000+x*10000000
  a <- a^3
  a
}
test_result <- memoise::memoise(test)
microbenchmark(times = 10, unit="ms", test_result(1000), test(1000))
# 结果，速度确实差了挺多的
Unit: milliseconds
              expr     min      lq mean median     uq  max neval
 test_result(1000) 0.09795 0.09942 0.18  0.110 0.1406 0.78    10
        test(1000) 0.00053 0.00062 0.24  0.001 0.0014 2.40    10
```
## 函数闭包
函数闭包可以提供更高级别的缓存，R中 函数闭包是包含函数及函数所依赖的环境对象（包围环境）。应该数嵌套函数直接调用？
```
stop_watch <- function(){
  start_time <- NULL
  start <- function() start_time <<- Sys.time()
  stop <- function() {
    stop_time <- Sys.time()
    difftime(start_time,stop_time)
  }
  list(start=start,stop=stop)
}
watch <- stop_watch()
watch$start()
watch$stop()
```
# 字节编译
 `compiler`包自2.13.0成为R的一部分，可以将函数编译成字节代码，从而使运行更快，清除了大量解释器必须执行的耗时操作，如变量查询的时间。可以通过基本函数mean()验证：
```
getFunction("mean")
# 结果，第三行显示是字节码bytecode
function (x, ...) 
UseMethod("mean")
<bytecode: 0x55ba277a8080>
<environment: namespace:base>
# 字节编译
cmpfun()
```
## 编译代码
安装时在DESCRIPTION文件中添加下面代码，就可以实现自动编译`ByteCompile: true`。对不同包的效果不一样，特别是某包已经有大量邓编译代码时。windows需要使用Rtools:
或者修改R.environ文件中的`R_COMPILE_PKGS`设为正整数并指定从source安装
```
install.packages("ggplot2", type="source")
```
如果不想改变这个文件，可以如下：
```
# 最后一个选项是实时编译JIT
install.packages("ggplot2", type="source", INSTALL_opts="--byte-compile")
```
