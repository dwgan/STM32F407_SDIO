使用STM32读写SD卡在低功耗存储中的应用是比较常见的，但是网上大多数资料都是基于标准库或者基于寄存器的开发。随着嵌入式设备越来越复杂，使用HAL库能够大大降低开发者的学习成本，从而提高开发效率。近年来，ST官方主推以STM32CubeMx为核心代码初始化工具，给开发者节省了配置硬件要花费的精力。

然而，由于HAL是一个硬件抽象层的库，它将不同系列的芯片硬件封装成了统一的接口，但是无法保证能够涵盖所有开发情况。在使用STM32F4开发SD卡读写功能的时候，我发现ST官方提供的HAL存在一些严重Bug，无法直接使用。本文就来填一填ST官方留下的坑。
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
![在这里插入图片描述](https://dwgan.top/PicGo/img/202410210102121.png)

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/62a4b7a6bab74f3694237511ea3b0dbe.png)
4、添加一个串口用于调试
![在这里插入图片描述](https://dwgan.top/PicGo/img/202410210102106.png)
5、配置时钟树
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c06c3dc7ad5047e8a29e450c04b5d51b.png)
6、生成代码
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/ffd4062cdfb04364a04466fdb6e58a69.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f2c41de7bdef4cd79894938262b2a523.png)
## 修改代码
1、重定向printf函数输出到串口，用于调试

```c
#include <stdio.h>
int fputc(int ch, FILE *f) {
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```
2、主函数如下
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
  /* USER CODE BEGIN 2 */
    setvbuf(stdout, NULL, _IONBF, 0);
    printf("初始化完毕\n");
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
3、编译下载发现无法输出预期，于是开始了Debug，发现跳转到了Error_Handler
![在这里插入图片描述](https://dwgan.top/PicGo/img/202410210102921.png)
4、打开Call Stack，发现错误在MX_SDIO_SD_Init();这个函数
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/ddde6e2ed59a41caa8af2f1c445b37f7.png)
5、于是继续跟踪，发现这里出错了，查找资料后发现这里生成的代码是有问题的（ST官方代码的第一个大坑）
![在这里插入图片描述](https://dwgan.top/PicGo/img/202410210102665.png)
6、将代码改为如下后重新运行
```c
void MX_SDIO_SD_Init(void)
{

  /* USER CODE BEGIN SDIO_Init 0 */

  /* USER CODE END SDIO_Init 0 */

  /* USER CODE BEGIN SDIO_Init 1 */

  /* USER CODE END SDIO_Init 1 */
  hsd.Instance = SDIO;
  hsd.Init.ClockEdge = SDIO_CLOCK_EDGE_RISING;
  hsd.Init.ClockBypass = SDIO_CLOCK_BYPASS_DISABLE;
  hsd.Init.ClockPowerSave = SDIO_CLOCK_POWER_SAVE_DISABLE;
  hsd.Init.BusWide = SDIO_BUS_WIDE_1B; // 这里只能是使用SDIO的1Bit总线模式进行初始化
  hsd.Init.HardwareFlowControl = SDIO_HARDWARE_FLOW_CONTROL_DISABLE;
  hsd.Init.ClockDiv = 0;
  if (HAL_SD_Init(&hsd) != HAL_OK)
  {
    Error_Handler();
  }
  if (HAL_SD_ConfigWideBusOperation(&hsd, SDIO_BUS_WIDE_4B) != HAL_OK)
  {
    Error_Handler();
  }
  /* USER CODE BEGIN SDIO_Init 2 */

  /* USER CODE END SDIO_Init 2 */

}
```
可以看到输出，说明初始化通过
![在这里插入图片描述](https://dwgan.top/PicGo/img/202410210102379.png)

## 测试SD卡读写

```c
/* USER CODE BEGIN Header */
/**
  ******************************************************************************
  * @file           : main.c
  * @brief          : Main program body
  ******************************************************************************
  * @attention
  *
  * Copyright (c) 2024 STMicroelectronics.
  * All rights reserved.
  *
  * This software is licensed under terms that can be found in the LICENSE file
  * in the root directory of this software component.
  * If no LICENSE file comes with this software, it is provided AS-IS.
  *
  ******************************************************************************
  */
/* USER CODE END Header */
/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "dma.h"
#include "sdio.h"
#include "usart.h"
#include "gpio.h"

/* Private includes ----------------------------------------------------------*/
/* USER CODE BEGIN Includes */

#include <stdio.h>
#include <string.h>
/* USER CODE END Includes */

/* Private typedef -----------------------------------------------------------*/
/* USER CODE BEGIN PTD */

/* USER CODE END PTD */

/* Private define ------------------------------------------------------------*/
/* USER CODE BEGIN PD */
#define BLOCK_SIZE         512 // 一个块的字节数节
#define DMA_NUM_BLOCKS_TO_WRITE 64 // 每一次DMA写入块的数量
#define DMA_NUM_BLOCKS_TO_READ  64 // 每一次DMA读出块的数量
#define BUFFER_SIZE_W         DMA_NUM_BLOCKS_TO_WRITE*BLOCK_SIZE         // 写缓冲区大小
#define BUFFER_SIZE_R         DMA_NUM_BLOCKS_TO_READ*BLOCK_SIZE         // 读缓冲区大小

/* USER CODE END PD */

/* Private macro -------------------------------------------------------------*/
/* USER CODE BEGIN PM */

/* USER CODE END PM */

/* Private variables ---------------------------------------------------------*/

/* USER CODE BEGIN PV */

uint8_t buff_w[BUFFER_SIZE_W];
uint8_t buff_r[BUFFER_SIZE_R];
uint8_t sdio_write_done=0;
uint8_t sdio_read_done=0;
/* USER CODE END PV */

/* Private function prototypes -----------------------------------------------*/
void SystemClock_Config(void);
/* USER CODE BEGIN PFP */

/* USER CODE END PFP */

/* Private user code ---------------------------------------------------------*/
/* USER CODE BEGIN 0 */
// 重定向printf函数输出到串口
int fputc(int ch, FILE *f) {
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
void HAL_SD_TxCpltCallback(SD_HandleTypeDef *hsd)
{    
    sdio_write_done = 1;
}

void HAL_SD_RxCpltCallback(SD_HandleTypeDef *hsd)
{
    sdio_read_done = 1;
}
/* USER CODE END 0 */

/**
  * @brief  The application entry point.
  * @retval int
  */
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
    /* USER CODE BEGIN 2 */
    setvbuf(stdout, NULL, _IONBF, 0);
    printf("初始化完毕\n");
    
    // 生成测试数据
    printf("正在生成测试数据\n");
    for (uint32_t i=0; i<sizeof(buff_w); i++)
    {
        buff_w[i]=i;
    }
    // 写入SD卡
    printf("正在写入数据\n");
    // 将数据通过DMA写入
    HAL_SD_WriteBlocks_DMA(&hsd, buff_w, 0, DMA_NUM_BLOCKS_TO_WRITE);
    while (sdio_write_done == 0);
    printf("数据写入完成\n");
    
    // 读取SD卡数据并且通过串口输出
    printf("正在读取数据\n");
    // 将数据读出到buff_r中
    HAL_SD_ReadBlocks_DMA(&hsd, buff_r, 0, DMA_NUM_BLOCKS_TO_READ);
    while (sdio_read_done == 0);
    if (0 == memcmp(buff_w, buff_r, sizeof(buff_r)))
    {
        printf("数据是一致的\n");
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

/**
  * @brief System Clock Configuration
  * @retval None
  */
void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  /** Configure the main internal regulator output voltage
  */
  __HAL_RCC_PWR_CLK_ENABLE();
  __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

  /** Initializes the RCC Oscillators according to the specified parameters
  * in the RCC_OscInitTypeDef structure.
  */
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSI;
  RCC_OscInitStruct.PLL.PLLM = 8;
  RCC_OscInitStruct.PLL.PLLN = 168;
  RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV2;
  RCC_OscInitStruct.PLL.PLLQ = 7;
  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  /** Initializes the CPU, AHB and APB buses clocks
  */
  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_5) != HAL_OK)
  {
    Error_Handler();
  }
}

/* USER CODE BEGIN 4 */

/* USER CODE END 4 */

/**
  * @brief  This function is executed in case of error occurrence.
  * @retval None
  */
void Error_Handler(void)
{
  /* USER CODE BEGIN Error_Handler_Debug */
  /* User can add his own implementation to report the HAL error return state */
  __disable_irq();
  while (1)
  {
  }
  /* USER CODE END Error_Handler_Debug */
}

#ifdef  USE_FULL_ASSERT
/**
  * @brief  Reports the name of the source file and the source line number
  *         where the assert_param error has occurred.
  * @param  file: pointer to the source file name
  * @param  line: assert_param error line source number
  * @retval None
  */
void assert_failed(uint8_t *file, uint32_t line)
{
  /* USER CODE BEGIN 6 */
  /* User can add his own implementation to report the file name and line number,
     ex: printf("Wrong parameters value: file %s on line %d\r\n", file, line) */
  /* USER CODE END 6 */
}
#endif /* USE_FULL_ASSERT */

```
结果如下
```bash
初始化完毕
正在生成测试数据
正在写入数据
数据写入完成
正在读取数据
数据是一致的
写入的数据如下
读出的数据如下
```

# 总结
ST官方的代码有两大坑：
1、SDIO初始化的坑，必须要使用1bit总线的SDIO来初始化SD卡，否则会导致初始化失败
2、采用轮询方式或者中断方式读写SDIO有问题，这里建议采用DMA进行读写
