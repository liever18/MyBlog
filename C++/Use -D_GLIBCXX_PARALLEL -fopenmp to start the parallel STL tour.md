# 使用 -D_GLIBCXX_PARALLEL -fopenmp 开启并行STL之旅

最近在看现代C++白皮书，看到C++17引入了并行STL。

> [8.5 并行 STL](https://github.com/Cpp-Club/Cxx_HOPL4_zh/blob/main/08.md#85-%E5%B9%B6%E8%A1%8C-stl)
>
> 从长远来看，并行算法的使用将是非常重要的，因为从用户角度看，没有什么比只说“请执行这个算法”更简单的了。从实现者的角度来看，算法中有一套特定接口而没有对算法的串行约束将是一个机会。C++17 只迈出了一小步，但这远比没有开始好得多，因为它指明了方向。不出意外，委员会中有一些反对的声音，大多数来自于希望为专家级用户提供复杂接口的人。有些人对这样简单的一个方案是否可行表示严重怀疑，并主张推迟这一方案。
>
> 基本的想法是，为每个标准库算法提供一个额外参数，允许用户请求向量化和/或多线程。例如：
>
> ```cpp
> sort(par_unseq, begin(v), end(v));  // 考虑并行和向量化
> ```
>
> 但这还只适用于 STL 算法，所以重要的 `find_any` 和 `find_all` 算法被忽略了。将来我们会看到专门为并行使用而设计的算法。这正在 C++20 中变为现实。
>
> 另一个弱点是，仍然没有取消一个线程的标准方法。例如，在搜索中找到一个对象后，一个线程不能停止其他正在并行执行的搜索。这是 POSIX 干预的结果，它反对所有形式的取消操作（[§4.1.2](https://github.com/Cpp-Club/Cxx_HOPL4_zh/blob/main/04.md#412-线程和锁)）。C++ 20 提供了协作式取消（[§9.4](https://github.com/Cpp-Club/Cxx_HOPL4_zh/blob/main/09.md#94-并发)）。
>
> C++17 的并行算法也支持向量化。这很重要，因为对 SIMD 的优化支持是硬件在单线程性能方面仍然（2017 年后）有巨大增长的少数领域之一。
>
> 在 C++20 中，我们（总算）能用范围库（[§6.3](https://github.com/Cpp-Club/Cxx_HOPL4_zh/blob/main/06.md#63-concepts-ts)）来避免显式使用容器的元素序列，只要这么写：
>
> ```cpp
> sort(v);
> ```
>
> 不幸的是，并行版本的范围在 C++20 中没有及时完成，因此我们只能等到 C++23 才能这么写：
>
> ```
> sort(par_unseq, v);  // 使用并行和向量化来对 v 进行排序
> ```
>
> 不想等 23 的话，我们可以自己实现适配器：
>
> ```cpp
> template<typename T>
> concept execution_policy = std::is_execution_policy<T>::value;
> 
> void sort(execution_policy auto&& ex, std::random_access_range auto& r)
> {
>  sort(ex, begin(r), end(r));  // 使用执行策略 ex 来排序
> }
> ```
>
> 毕竟标准库是可扩展的。



**先说自己探索的结论，目前的gcc/g++只需要在编译是添加参数 -D_GLIBCXX_PARALLEL -fopenmp 即可达到并行化的效果，不需要对源码进行修改。sort并行化效果很不错，遍历的函数大部分都能有一定的提升，二分查找的性能反而会有明显的下降。**

> [libstdc++/manual/parallel_mode_using](http://gcc.gnu.org/onlinedocs/libstdc++/manual/parallel_mode_using.html)



**网上已有的方法**，需要添加头文件`#include <execution> `，还需要形如`  sort(execution::par, begin(d), end(d)); `一样设置并行化算法。

```cpp
#include <algorithm>
#include <execution>
#include <iostream>
#include <random>
#include <chrono>   

using namespace std;
using namespace chrono;

int main() {
    vector<long long> d1(30000000);
    vector<long long> d2(30000000);

    mt19937 gen;
    uniform_int_distribution<long long> dis(0, 100000000);
    auto rand_num([=]() mutable { return dis(gen); });

    generate(execution::par, begin(d1), end(d1), rand_num);
    d2 = d1;
    
    auto start_t = high_resolution_clock::now();
    sort(begin(d1), end(d1));
    auto end_t = high_resolution_clock::now();
    auto duration = duration_cast<nanoseconds>(end_t - start_t);
    cout << "The run time is: " << double(duration.count()) * nanoseconds::period::num / nanoseconds::period::den << "s" << endl;

    start_t = high_resolution_clock::now();
    sort(execution::par, begin(d2), end(d2));
    end_t = high_resolution_clock::now();
    duration = duration_cast<nanoseconds>(end_t - start_t);
    cout << "The run time is: " << double(duration.count()) * nanoseconds::period::num / nanoseconds::period::den << "s" << endl;
  
    return 0;
}
```

我在windows的环境中找不到execution头文件，因此我先在linux(gcc 10.2.0)上运行了下该代码

```shell
┌──(kali㉿kali)-[~/testCpp]
└─$ g++ par.cpp -std=c++17                                                                                                                                                                             
┌──(kali㉿kali)-[~/testCpp]
└─$ ./a.out               
The run time is: 10.6875s
The run time is: 11.6801s
```

在这个代码的基础上，我想加上`-D_GLIBCXX_PARALLEL -fopenmp`看看有啥效果。

```shell
┌──(kali㉿kali)-[~/testCpp]
└─$ g++ par.cpp -std=c++17 -D_GLIBCXX_PARALLEL -fopenmp
                                                                                                                          
┌──(kali㉿kali)-[~/testCpp]
└─$ ./a.out                                            
The run time is: 2.51153s
The run time is: 2.63921s                     
```

结果两个排序的效果都得到了明显提高。



不过，可以看到两种sort的速度没啥区别，然后我把使用到execution的部分给注释掉了，只对`d1`进行排序，然后发现仅添加`-D_GLIBCXX_PARALLEL -fopenmp`参数，linux和windows(gcc 8.1.0)上都能跑，并且都能达到加速的效果。



为了验证并行化STL算法的正确性，我使用了算法库中的多种函数来进行测试，测试代码如下：

```cpp
#include <algorithm>
//#include <execution>
#include <iostream>
#include <random>
#include <chrono>   
#include <vector>   
#include <numeric>     // iota
using namespace std;
using namespace chrono;

int main() {
    vector<long long> d1(30000000);
    iota(d1.rbegin(), d1.rend(), 0);

    auto start_t = high_resolution_clock::now();
    sort(begin(d1), end(d1));
    auto end_t = high_resolution_clock::now();
    auto duration = duration_cast<nanoseconds>(end_t - start_t);
    cout << "Sort costs: " << double(duration.count()) / nanoseconds::period::den << "s" << endl;

    start_t = high_resolution_clock::now();
    auto res1 = binary_search(begin(d1), end(d1), 0x123456);
    end_t = high_resolution_clock::now();
    duration = duration_cast<nanoseconds>(end_t - start_t);
    cout << "Binary_search costs: " << double(duration.count()) << "ns, " << "result is " << res1 << endl;

    start_t = high_resolution_clock::now();
    auto res2 = lower_bound(begin(d1), end(d1), 0x123456);
    end_t = high_resolution_clock::now();
    duration = duration_cast<nanoseconds>(end_t - start_t);
    cout << "Lower_bound costs: " << double(duration.count()) << "ns, " << "result is " << *res2 << endl;

    start_t = high_resolution_clock::now();
    auto res3 = find(begin(d1), end(d1), 0xffffff);
    end_t = high_resolution_clock::now();
    duration = duration_cast<nanoseconds>(end_t - start_t);
    cout << "Find costs: " << double(duration.count()) / nanoseconds::period::den << "s, " << "result is " << *res3 << endl;

    start_t = high_resolution_clock::now();
    auto res4 = count(begin(d1), end(d1), 0x123456);
    end_t = high_resolution_clock::now();
    duration = duration_cast<nanoseconds>(end_t - start_t);
    cout << "Count costs: " << double(duration.count()) / nanoseconds::period::den << "s, " << "result is " << res4 << endl;

    start_t = high_resolution_clock::now();
    nth_element(begin(d1), d1.begin() + 0xff, end(d1));
    end_t = high_resolution_clock::now();
    duration = duration_cast<nanoseconds>(end_t - start_t);
    cout << "Nth_element costs: " << double(duration.count()) / nanoseconds::period::den << "s, " << "nth_element is " << d1[0xff - 1] << endl;
    
    start_t = high_resolution_clock::now();
    auto res5 = max_element(begin(d1), end(d1));
    end_t = high_resolution_clock::now();
    duration = duration_cast<nanoseconds>(end_t - start_t);
    cout << "Max_element costs: " << double(duration.count()) / nanoseconds::period::den << "s, " << "result is " << *res5 << endl;
    return 0;
}
```

上述代码在linux和windows环境都能正常运行，命令行参数也相同。

下面是在linux平台（gcc 10.2.0, 4cores）进行测试的结果。

不加参数`-D_GLIBCXX_PARALLEL -fopenmp`输出如下：

```shell
┌──(kali㉿kali)-[~/testCpp]
└─$ g++ par.cpp                                                                                                                                        
┌──(kali㉿kali)-[~/testCpp]
└─$ ./a.out
Sort costs: 7.59176s
Binary_search costs: 4348ns, result is 1
Lower_bound costs: 501ns, result is 1193046
Find costs: 0.0747495s, result is 16777215
Count costs: 0.294351s, result is 1
Nth_element costs: 0.483194s, nth_element is 254
Max_element costs: 0.33075s, result is 29999999
```

加上之后输出如下：

```shell
┌──(kali㉿kali)-[~/testCpp]
└─$ g++ par.cpp -D_GLIBCXX_PARALLEL -fopenmp                                                                                                                   
┌──(kali㉿kali)-[~/testCpp]
└─$ ./a.out                                 
Sort costs: 1.85084s
Binary_search costs: 7585ns, result is 1
Lower_bound costs: 1042ns, result is 1193046
Find costs: 0.0965801s, result is 16777215
Count costs: 0.218396s, result is 1
Nth_element costs: 0.296512s, nth_element is 254
Max_element costs: 0.212634s, result is 29999999
```

可以发现以上的算法，排序算法效率都有较大的提高，接近4核的并行化理论上限，遍历操作并行化操作大部分有一定的提高，只有find有少量的下降。二分查找的效率反而下降，并且下降的挺多的。并行算法输出结果与普通算法相同。



**总结**

目前execution头文件好像用处不大，也不用指定`-std=c++17`，仅添加参数`-D_GLIBCXX_PARALLEL -fopenmp`，即可达到并行化STL加速的效果，加速效果还挺不错的，但是对于二分查找的算法，并行化反而会有很大的性能上的恶化。由于现在无法指定哪些STL的算法使用并行化，哪些不能，所以无脑加上这两个参数如果想高效并行化使用STL的话，并不一定能使得程序的性能得到稳定的提升，还是要等标准库提供支持，这样的话就能在源码层面自己指定哪些函数的调用是使用原始的STL算法，哪些使用并行化的STL算法。

