# 知识随笔

## Linux相关

## C++相关
1.回调与回调函数
  - 回调是一种C/C++语言特性，可以将自己写的代码嵌入某些官方库函数中。
  - 回调函数，就是普通函数，只不过会作为参数传递给某些官方函数调用。
  - 为什么要使用回调函数？
    - 首先，如果我们不需要在使用官方库函数时进行特异性操作，那么完全用不到回调特性；
    - 其次，如果我们想要在官方库函数的指定位置实现特异性功能，官方可不知道我们需要进行怎样的操作；
    - 因此，我们需要写好自己的函数，然后传递进去，让官方库函数去调用，这样就实现了既有官方通用功能（官方库函数），又有我们自己特质功能（我们自己写的回调函数）的函数啦。
