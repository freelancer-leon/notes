# Wrapper Headers

有时调整一个系统提供的头文件的内容但不直接编辑它是有必要的。例如，GCC的`fixincludes`操作就是做这个的。一种方法是，以相同的名字创建一个新的头文件并将它插入到搜索路径里的原始头文件之前。只要你愿意整个替换就的头文件，这么做是可以的。但如果你只是想在新的头文件里引用旧的头文件呢？

你不能简单地用`#include`包含旧的头文件。那会回到开头并再次找到你的新的头文件。如果你的头文件没有被多重包含保护（参看[Once-Only Headers](https://gcc.gnu.org/onlinedocs/cpp/Once-Only-Headers.html#Once-Only-Headers)），将会引起无限递归并导致一个致命错误。

你可以用绝对路径来包含旧的头文件：
```c
#include "/usr/include/old-header.h"
```
这也行，但不干净；如果系统头文件位置变了，你不得不编辑新头文件去匹配它。

标准C无法解决此类问题，但你可以用GNU扩展`#include_next`。它的含义是，“包含下一个用这个名字的文件”。这个指示符像`#include`一样，除了搜索指定的文件：它在头文件目录列表中从当前文件被找到的目录之后开始搜索。

假设你指定` -I /usr/local/include`，并且搜索的目录列表也包含`/usr/include`；假设两个目录都包含`signal.h`。通常`#include <signal.h>`会在`/usr/local/include`下找到文件。如果那个文件包含`#include_next <signal.h>`，它会在那个目录之后开始搜索并在`/usr/include`里找到文件。

`#include_next`不区分`<file>`和`"file"`包含，也不检查你指定的文件与当前文件是否同名。它只是简单地在搜索路径中从当前文件被找到的目录之后开始查找指名的文件。

`#include_next`的用法会导致极大的困惑。我们推荐只在没有其他替代方法的时候使用它。特别是，它不应该被用与属于一个特定程序的头文件中；它应该只被用在作出全局的更正，沿用`fixincludes`的方式。


# 参考资料
- [Wrapper Headers - The C Preprocessor](https://gcc.gnu.org/onlinedocs/cpp/Wrapper-Headers.html)
- [gcc fixincludes简介](http://www.wanglianghome.org/org/programming/fixincludes.html)
