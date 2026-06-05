
只需要在 cmake/stm32cubemx/CMakeLists.txt里面加一句
```
target_link_options(${CMAKE_PROJECT_NAME} PRIVATE "-u _printf_float")
```
**解释：**

- `target_link_options`：**给链接器添加参数**，区别于 `target_compile_options`（编译器参数）。
- `PRIVATE`：链接选项只作用于当前可执行 / 库，对外头文件、依赖项不会继承该链接参数。
- `-u SYMBOL`：强制链接器**主动引入该符号对应的代码段**，哪怕代码没显式调用这个函数。
- `_printf_float` 是**newlib C 库**内部符号：
	newlib 默认精简版 `printf` **不支持浮点输出 % f/% lf**（为了减小固件体积，单片机常用裁剪版 newlib）；
	`_printf_float` 符号绑定了浮点数打印的实现代码；
	`-u_printf_float` → **强制链接器把 printf 浮点处理代码链接进程序**，启用`printf("%.2f", val)`浮点打印。

> 注意：Flash 增加 ~10KB

```c
void print_f(float f, uint8_t prec)
{
    if(f < 0){putchar('-');f=-f;}
    uint32_t mul=1;for(int i=0;i<prec;i++)mul*=10;
    uint32_t all = f*mul+0.5f;
    printf("%d.%.*u",all/mul,prec,all%mul);
}
```