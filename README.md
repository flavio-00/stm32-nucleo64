# Inter-Thread Communication
The CMSIS-RTOS APIv1 provides various ways for exchanging messages between threads, improving inter-thread communication and resource sharing. The following methods are some of function available to the user:

**Resource Sharing**
* [Binary Semaphore](#binary-semaphore)

**Inter-Thread Communication**
* [Message Queue](#message-queue)
* [Mail Queue](#mail-queue)

## How to include printf library
Since the project will include the use of `printf` library, in order to print some text in the serial console, the following code should be included into main.c file to redirect printf output stream to UART2.

```c
/* USER CODE BEGIN Includes */
    #include "stdio.h"
/* USER CODE END Includes */

/* USER CODE BEGIN 0 */
    #ifdef __GNUC__
        #define PUTCHAR_PROTOTYPE int __io_putchar(int ch)
    #else
        #define PUTCHAR_PROTOTYPE int fputc(int ch, FILE *f)
    #endif
    PUTCHAR_PROTOTYPE{
        HAL_UART_Transmit(&huart2, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
        return ch;
    }
/* USER CODE END 0 */
```

## FreeRTOS Configuration
By default, the HAL driver will use the Systick as its primary time base, but this timer should be left to the AzureRTOS only. To prevent this conflict, we can simply select a different time base for the HAL by clicking in the System Core/SYS and selecting the time base Source as TIM6.

<img src="./Inter-Thread Communication/Binary Semaphore/Images/SYS.png" alt="SYS" width="400" />

To configure the High Speed Clock (HSE), in section **System Core** click RCC -> High Speed Clock (HSE) -> Crystal/Ceramic Resonator.

<img src="./Inter-Thread Communication/Binary%20Semaphore/Images/RCC.png" alt="RCC" width="400" />

Finally, to enable FreeRTOS, in section **Middleware and Software Packs** click FreeRTOS and select the Interface CMSIS_V1.

<img src="./Inter-Thread Communication/Binary%20Semaphore/Images/FreeRTOS.png" alt="FreeRTOS" width="400" />


## Projects 
### [Binary Semaphore](./Binary%20Semaphore/)

Semaphores are used to synchronize tasks with other events in the system. Waiting for semaphore is equal to wait() procedure, task is in a blocked state not taking CPU time. In FreeRTOS implementation semaphores are based on queue mechanisms. There are some types of semaphores in FreeRTOS, like binary, counting or mutex.

<img src="./Inter-Thread Communication/Binary%20Semaphore/Images/semaphore.png" alt="Semaphore" width="300" />

**FreeRTOS** Task and Queues setup:

<img src="./Inter-Thread Communication/Binary%20Semaphore/Images/tasksAndQueues.png" alt="Task And Queues" width="500" />

**FreeRTOS** Timers And Semaphore setup:

<img src="./Inter-Thread Communication/Binary%20Semaphore/Images/timersAndSemaphores.png" alt="Timers And Semaphore" width="500" />

In main.c file, 
```c
void TaskNormalFunction(void const * argument)
{
  /* USER CODE BEGIN 5 */
  /* Infinite loop */
  for(;;)
  {
	  printf("Entered Normal Thread and waiting for a Semaphore\r\n");
	  osSemaphoreWait(binarySemaphoreHandle, osWaitForever);
	  printf("Semaphore of Normal Thread acquired\r\n");
	  while(HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13));
	  printf("Leaving Normal Thread and releasing Normal Semaphore\r\n\n");
	  osSemaphoreRelease(binarySemaphoreHandle);
	  osDelay(2000);
  }
  /* USER CODE END 5 */
}
```
this **TaskNormalFunction** shows how to use a Binary Semaphore to stop the execution of other threads until the current thread finish his work, in this algorithm waiting for the user Button1 is pressed.

### [Message Queue](./Message%20Queue/)
A Message Queue is the basic communication model between threads. The data is passed from one thread to another in a FIFO-like operation. Using Message Queue functions, you can send, receive, or wait for messages. The queue is made by integers or pointers 32-bit long. The length of queue is declared during creation phase and is defined as a number of items which will be send via queue.

<img src="./Inter-Thread Communication/Message%20Queue/Images/messageQueue.png" alt="Message Queue" width="300" />

**FreeRTOS** Task and Queues setup:

<img src="./Inter-Thread Communication/Message%20Queue/Images/tasksAndQueues.png" alt="Task And Queues" width="500" />

**FreeRTOS** Timers And Semaphore setup:

<img src="./Inter-Thread Communication/Message%20Queue/Images/timersAndSemaphores.png" alt="Timers And Semaphore" width="500" />

In main.c file, 
```c
/* USER CODE BEGIN 0 */
typedef struct{
	uint16_t Value;
	uint8_t Source;
} MessageStruct;
/* USER CODE END 0 */
```
we define the MessageStruct made by two variable: one for the message value (Value) and another one used for the number of the sender thread (Source).

```c
void Sender1TaskFunction(void const * argument)
{
  /* USER CODE BEGIN 5 */
  /* Infinite loop */
  for(;;)
  {
	  osSemaphoreWait(binarySemaphoreHandle, osWaitForever);
	  osDelay(1000);
	  printf("Sender1\r\n");
	  MessageStruct DataToSend1={1000,1};
	  osMessagePut(myQueueHandle,(uint32_t) &DataToSend1,200);
	  osSemaphoreRelease(binarySemaphoreHandle);
  }
  /* USER CODE END 5 */
}
```
this **Sender1TaskFunction** combine the use of semaphore with Message Queue inter-thread communication. With `osMessagePut` we put DataToSend1 variable in the queue with `Value=1000` and `Source=1`. Then, wait for `osDelay(1000)` to release semaphore to the other **Sender2TaskFunction**. This second task will do the same, with `Value=2000` and `Source=2` and `osDelay(5000)`. 

```c
void ReceiverTaskFunction(void const * argument)
{
  /* USER CODE BEGIN ReceiverTaskFunction */
	osEvent retvalue;
  /* Infinite loop */
  for(;;)
  {
	  osSemaphoreWait(binarySemaphoreHandle, osWaitForever);
	  retvalue=osMessageGet(myQueueHandle,4000);
	  if(((MessageStruct*)retvalue.value.p)->Source!=0 || ((MessageStruct*)retvalue.value.p)->Source!=1){
		  printf("Data from %d: %d \r\n", ((MessageStruct*)retvalue.value.p) -> Source, ((MessageStruct*)retvalue.value.p) -> Value);
	  }
	  osSemaphoreRelease(binarySemaphoreHandle);
	  osDelay(1);
  }
  /* USER CODE END ReceiverTaskFunction */
}
```
this **ReceiverTaskFunction** will receive the streamed message. With `retvalue=osMessageGet(myQueueHandle,4000)` we get data from queue and with `((MessageStruct*)retvalue.value.p) -> Source` we decode data from osEvent structure.

### [Mail Queue](./Mail%20Queue/)
A Mail Queue operates similarly to a Message Queue, but instead of transferring a 32-bit value, the data that is being transferred consists of memory blocks that need to be allocated, before putting data in, and free-up, after taking data out. The Mail Queue uses a memory pool to create formatted and fixed-size memory blocks and passes pointers to these blocks in a Message Queue. This allows the data to stay in an allocated memory block while only a pointer is moved between the separate threads, and it’s an advantage to Message Queues because there are no big data transfers.

<img src="./Inter-Thread Communication/Mail%20Queue/Images/mailQueue.png" alt="Mail Queue" width="300" />

**FreeRTOS** Task and Queues setup:

<img src="./Inter-Thread Communication/Mail%20Queue/Images/tasksAndQueues.png" alt="Task And Queues" width="500" />

In main.c file,
```c
/* USER CODE BEGIN 0 */
#define COUNTER_MAX 256
typedef struct
{
    uint8_t buffer[COUNTER_MAX];
    uint32_t counter;
} MailStruct;

osMailQId mailHandle; // Mail ID identifies the mail queue (pointer to a mail queue control block)
/* USER CODE END 0 */
```
in USER CODE 0 section we define the Mail data structure used for inter-thread communication, with a buffer
array of 256 elements. The `osMailQDef` function is used to define the attributes of a Mail Queue, like the
name, the maximum number of messages and the data type of a single message element.

```c
void sender1Task(void const * argument)
{
  /* USER CODE BEGIN 5 */
	MailStruct mailData;
	for (int i = 0; i < COUNTER_MAX; i++){
		mailData.buffer[i] = i;
	}
  /* Infinite loop */
  for(;;)
  {
	  MailStruct *p = osMailAlloc(mailHandle, osWaitForever);
	  p = &mailData;
	  osMailPut(mailHandle, p);
	  osDelay(100);
  }
  /* USER CODE END 5 */
}
```
In those code lines of the **sender1Task**, `osMailAlloc` is used to allocate a memory block from the buffer filled with the mail information we want to transfer, in this case we load the `mailData.buffer[]` with 256 numbers from 0 to 255. While, with `osMailPut` put the memory block specified into a Mail Queue.

```c
void receiverTask(void const * argument)
{
  /* USER CODE BEGIN receiverTask */
	static int i = 0;
  /* Infinite loop */
  for(;;)
  {
	  osEvent event = osMailGet(mailHandle, osWaitForever);
	  if (event.status == osEventMail){
		  MailStruct *p = (MailStruct*)event.value.p;
		  printf("%d \r\n", p->buffer[i]);
		  if ((p->buffer[i] & 0x01) == 0){ // even
			  HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
		  } else { // odd
			  HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);
		  }
		  osMailFree(mailHandle, p);
		  if (++i >= COUNTER_MAX){
			  printf("END BUFFER\r\n");
			  i = 0;
		  }
	  }
	  osDelay(1);
  }
  /* USER CODE END receiverTask */
}
```
In the **Receiver Thread**, `osMailGet` is used to suspend the execution of the current RUNNING thread until a mail arrives, then returning the mail information. While, `osMailFree` free the memory block read by the receiving function. After having printed the value of the message received in the console with `printf()`, we control the LED2 making it light up if the number is odd, or making it turn off if it is even.
