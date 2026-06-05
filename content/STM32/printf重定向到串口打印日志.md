在 `usart.c`尾部`/* USER CODE BEGIN 1 */`添加如下代码
```C
/* USER CODE BEGIN 1 */
#include <stdio.h>
// printf("text") → _write() [syscalls.c] → __io_putchar() [usart.c] → HAL_UART_Transmit(&huart1)
// syscalls.c 中已有 _write() 遍历字符串逐字符调用 __io_putchar 的框架
// 该函数原本是 __weak 弱符号，我们提供的强符号会覆盖它。
int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
/* USER CODE END 1 */
```