使用STM32读写SD卡在低功耗存储中的应用是比较常见的，但是网上大多数资料都是基于标准库或者基于寄存器的开发。随着嵌入式设备越来越复杂，使用HAL库能够大大降低开发者的学习成本，从而提高开发效率。近年来，ST官方主推以STM32CubeMx为核心代码初始化工具，给开发者节省了配置硬件要花费的精力。

尽管上一篇博客填了ST留下来的坑，但是依然使用的是按扇区写入数据，通用性不是很好。因此本文探索使用FATFS这个模块来实现文件系统。

# 硬件准备
1、STM32F407VET6开发板，带SD卡槽
2、1G逻辑分析仪
# 软件准备
[STM32CubeMX](https://www.st.com.cn/zh/development-tools/stm32cubemx.html) (本项目使用6.12.1版本)

[IAR 9.50.2](https://pan.baidu.com/s/1OK0k6JNU_-GRZYnANJ5B-g?pwd=wata)（本项目主要使用IAR，相比于Keil编译速度更快，生成的文件体积更小，若需要Keil版本的代码，可通过STM32CubeMX生成对应版本）
# 操作步骤
## 使用STM32CubeMx生成代码
1、配置RCC
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/60f409cebb3043a2ae6b8c6edf29a171.png)
2、配置调试器
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/403f762cb4df4dbca921acb03616584f.png)
3、配置SDIO，注意这里要配置DMA和SDIO全局中断，其它默认
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e3a1bdb729da4d4c9fe6192218c607fa.png)

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/62a4b7a6bab74f3694237511ea3b0dbe.png)
4、添加一个串口用于调试
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/cd6450e6677f4889bfe64e1aa6ad1fd3.png)
5、配置时钟树
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c06c3dc7ad5047e8a29e450c04b5d51b.png)
6、这里配置FATFS
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/65b7dfb0ce8c4bccaf1c4152da937f23.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/0d0cb2f729fd490e9f808f2e10881425.png)
7、最后生成代码

## 修改代码
1、注意这里生成的代码是有问题的，需要给成1B，在前面一篇文章讲过
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/9cf15d4ffe354d2a904ddeeb48ccdbb6.png)

2、重定向printf函数输出到串口，用于调试

```c
#include <stdio.h>
int fputc(int ch, FILE *f) {
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```
3、主函数如下
```c
int main(void)
{

  /* USER CODE BEGIN 1 */
    
  /* USER CODE END 1 */

  /* MCU Configuration--------------------------------------------------------*/

  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();

  /* USER CODE BEGIN Init */
    
  /* USER CODE END Init */

  /* Configure the system clock */
  SystemClock_Config();

  /* USER CODE BEGIN SysInit */
    
  /* USER CODE END SysInit */

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_DMA_Init();
  MX_SDIO_SD_Init();
  MX_USART1_UART_Init();
  MX_FATFS_Init();
  /* USER CODE BEGIN 2 */
  setvbuf(stdout, NULL, _IONBF, 0);

    /* 挂载文件系统 */
    FRESULT res;
    const TCHAR* SDPath = "0:";  // SD 卡路径
    res = f_mount(&SDFatFS, SDPath, 1);
    if (res != FR_OK)
    {
        printf("f_mount error: %d\n", res);
        if (res == FR_NO_FILESYSTEM)
        {
            printf("No filesystem, trying to format...\n");
            // 格式化 SD 卡
            res = f_mkfs(SDPath, FM_EXFAT, 0, work, sizeof(work));
            if (res == FR_OK)
            {
                printf("Format success, trying to mount again...\n");
                res = f_mount(&SDFatFS, SDPath, 1);
                if (res == FR_OK)
                {
                    printf("Mount success after format\n");
                }
                else
                {
                    printf("Mount failed after format, error: %d\n", res);
                    while(1);
                }
            }
            else
            {
                printf("Format failed, error: %d\n", res);
                while(1);
            }
        }
        else
        {
            while(1);
        }
    }
    else
    {
        printf("Mount success\n");
    }
    
    /* 打开文件进行写入 */
    res = f_open(&MyFile, "0:/data.txt", FA_READ | FA_WRITE | FA_CREATE_ALWAYS);
    if (res == FR_OK)
    {
        UINT byteswritten;
        
        /* 将 data 数组重复写入文件 100 次 */
        for (int i = 0; i < 1000; i++)
        {
            /* 写入数据 */
            res = f_write(&MyFile, data, strlen(data), &byteswritten);
            if ((res == FR_OK) && (byteswritten == strlen(data)))
            {
                printf("Data written successfully (%d/100)\n", i + 1);
            }
            else
            {
                printf("Failed to write data at iteration %d, error: %d\n", i + 1, res);
                break;  // 发生错误，退出循环
            }
        }
        
        /* 关闭文件 */
        f_close(&MyFile);
    }
    else
    {
        printf("Failed to open file for writing, error: %d\n", res);
    }
    
    /* 读取文件内容 */
    res = f_open(&MyFile, "0:/data.txt", FA_READ);
    if (res == FR_OK)
    {
        char buffer[64];
        UINT bytesread;
        
        printf("File content:\n");
        do
        {
            res = f_read(&MyFile, buffer, sizeof(buffer)-1, &bytesread);
            if (res == FR_OK)
            {
                buffer[bytesread] = '\0';
                printf("%s", buffer);
            }
            else
            {
                printf("Failed to read file, error: %d\n", res);
                break;
            }
        } while (bytesread > 0);
        
        /* 关闭文件 */
        f_close(&MyFile);
    }
    else
    {
        printf("Failed to open file for reading, error: %d\n", res);
    }
    
    
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
    while (1)
    {
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
    }
  /* USER CODE END 3 */
}
```

4、结果如下
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7edefca0a5584b7592ad3388d1026b7e.png)
# 总结
可见，使用FATFS可以轻松实现数据的写入、读取，方便管理，大大降低了开发难度。后续结合以太网还可以实现FTP直接读取文件，或者直接将SD卡拔出来使用读卡器在PC上读取数据。
以上这些优点都是直接读取扇区无法做到的。
# 代码
> https://github.com/dwgan/STM32F407_SDIO
在sdio_fatfs这个分支
