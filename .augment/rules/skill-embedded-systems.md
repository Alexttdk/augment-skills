---
type: agent_requested
description: Embedded/firmware development for STM32/ARM MCUs: FreeRTOS task management, I2C/SPI/UART/CAN protocols, HAL patterns, memory constraints, and power management.
---

# Embedded Systems

## When to Use
Activate when working with:
- Microcontrollers (STM32, ESP32, Arduino, NXP, Nordic nRF)
- RTOS-based firmware (FreeRTOS, Zephyr, ThreadX)
- Hardware peripherals (GPIO, ADC, I2C, SPI, UART, CAN, USB)
- Bare-metal or low-level C/C++ firmware development
- Real-time constraints, power optimization, or memory-limited environments

## Architecture: Bare-Metal vs RTOS

| Aspect | Bare-Metal | RTOS (FreeRTOS) |
|--------|-----------|-----------------|
| Complexity | Low | Medium |
| Real-time | Poll/interrupt-driven | Priority-based preemptive |
| RAM overhead | ~2–10 KB | ~10–40 KB |
| Multitasking | Super-loop + ISRs | True concurrent tasks |
| Timing jitter | High | Low (deterministic) |
| Best for | Simple sensors, fixed pipeline | Multiple concurrent subsystems |

**Choose RTOS when:** multiple independent tasks, inter-task communication, complex state machines, or strict timing requirements across subsystems.

## RTOS Task Management (FreeRTOS)

```c
// Task creation
xTaskCreate(vSensorTask, "Sensor", 256, NULL, tskIDLE_PRIORITY + 2, &xSensorHandle);

// Periodic task — use vTaskDelayUntil to prevent drift
void vSensorTask(void *pvParams) {
    TickType_t xLastWake = xTaskGetTickCount();
    for (;;) {
        ReadSensor();
        xQueueSend(xDataQueue, &sensor_val, pdMS_TO_TICKS(10));
        vTaskDelayUntil(&xLastWake, pdMS_TO_TICKS(100));  // 10 Hz, no drift
    }
}

// Signal from ISR (use FromISR variants — never block in ISR)
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc) {
    BaseType_t xWoken = pdFALSE;
    xSemaphoreGiveFromISR(xDataReadySem, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

// Mutex: shared resource (I2C bus)
if (xSemaphoreTake(xI2CMutex, pdMS_TO_TICKS(100)) == pdTRUE) {
    HAL_I2C_Master_Transmit(&hi2c1, addr, data, len, 100);
    xSemaphoreGive(xI2CMutex);
}
```

**Primitives cheat-sheet:**
- `xQueueCreate / Send / Receive` — typed data passing between tasks
- `xSemaphoreCreateBinary` — ISR → task signaling
- `xSemaphoreCreateMutex` — shared resource protection (priority inheritance)
- `xTimerCreate(pdTRUE)` — auto-reload software timer
- `xEventGroupSetBits / WaitBits` — multiple event synchronization
- `xTaskNotifyFromISR / xTaskNotifyWait` — fastest ISR→task signal (no semaphore overhead)

## Memory Constraints

```c
// FreeRTOSConfig.h
#define configTOTAL_HEAP_SIZE           (20 * 1024)  // 20 KB
#define configMINIMAL_STACK_SIZE        128           // words (× 4 = bytes on ARM)
#define configUSE_MALLOC_FAILED_HOOK    1
#define configCHECK_FOR_STACK_OVERFLOW  2

// Monitor at runtime
size_t freeHeap  = xPortGetFreeHeapSize();
size_t minFree   = xPortGetMinimumEverFreeHeapSize();
UBaseType_t hwm  = uxTaskGetStackHighWaterMark(NULL);  // tune stack sizes

void vApplicationMallocFailedHook(void)                        { Error_Handler(); }
void vApplicationStackOverflowHook(TaskHandle_t t, char *name) { Error_Handler(); }
```

**Rules:** Avoid dynamic allocation after initialization. Size stacks conservatively, then tune with `uxTaskGetStackHighWaterMark()`. Use heap_4 for fragmentation resistance.

## Communication Protocols

| Protocol | Speed | Wires | Addressing | Nodes | Best For |
|----------|-------|-------|-----------|-------|----------|
| I2C | 100k–1M bps | 2 (SDA+SCL) | 7-bit address | 127 | Sensors, slow peripherals |
| SPI | 1–80M bps | 4 (MOSI/MISO/SCK/CS) | CS line | Unlimited | Display, flash, high-speed ADC |
| UART | 9600–4M bps | 2 (TX+RX) | Point-to-point | 2 | Debug, GPS, BLE modules |
| CAN | up to 1M bps | 2 (differential) | 11/29-bit ID | 128 | Automotive, industrial noise |
| USB | 12M–5G bps | 2 + power | Endpoint | 127 | Host/device bulk data |

**Protocol patterns:**
- I2C: Use `HAL_I2C_Mem_Read/Write` for register-based sensors; add pull-ups (4.7 kΩ @ 100 kHz)
- SPI: Use DMA (`HAL_SPI_TransmitReceive_DMA`) for transfers > 64 bytes to free CPU
- UART: Use circular buffer + RXNE interrupt; never poll in main loop; handle overrun errors
- CAN: Set acceptance filters to minimize CPU interrupts; use FIFO0/FIFO1 for priority separation

## Power Management

```c
// Graduated sleep modes (STM32 HAL)
HAL_PWR_EnterSLEEPMode(PWR_MAINREGULATOR_ON, PWR_SLEEPENTRY_WFI);   // ~10 mA, wake on any IRQ
HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI); // ~100 µA, wake on EXTI/RTC
HAL_PWR_EnterSTANDBYMode();                                          // ~2 µA, wake on WKUP/RTC

// CRITICAL: reconfigure clocks after STOP/STANDBY wakeup
SystemClock_Config();

// Disable unused peripherals to save current
__HAL_RCC_GPIOC_CLK_DISABLE();
__HAL_RCC_TIM3_CLK_DISABLE();
```

**Power tiers:** Active (100 mA+) → Sleep (1–10 mA) → Stop (100 µA) → Standby (1–5 µA)

**Tips:** Measure first with ammeter before optimizing. Configure unused GPIO as analog input. Use RTC alarm for timed wakeup. Clock-gate peripherals when not in use.

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/embedded-systems
- FreeRTOS docs: https://www.freertos.org/Documentation/RTOS_book.html
- STM32 HAL: https://www.st.com/en/embedded-software/stm32cube-mcu-packages.html


