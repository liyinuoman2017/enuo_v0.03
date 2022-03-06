# enuo_v0.03
从零开始构建嵌入式实时操作系统3——任务状态切换

![在这里插入图片描述](https://img-blog.csdnimg.cn/be49d66fc36a4b9f9d0208e662c447c9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

## 1.前言

> 一个行者问老道长：“您得道前，做什么？”老道长：“砍柴担水做饭。”行者问：“那得道后呢？”老道长：“砍柴担水做饭。”行者又问：“那何谓得道？”老道长：“得道前，砍柴时惦记着挑水，挑水时惦记着做饭；得道后，砍柴即砍柴，担水即担水，做饭即做饭。”

  是不是茅塞顿开？生活中许多至高至深的道理往往都是含蕴在一些极其简单的思想中，正所谓大道至简。完美的常常是最简单的，简单就是聪明，简单就是高级形式的复杂，简到极致，便是大智。厉害的人往往是把复杂的问题简单化，世上再大再难的事情，只要“一分为二”就可以分解成许多简单的事。
 
**在软件设计五大原则中的单一原则，与大道至简的思想不谋而合。单一原则的核心思想是让软件模式结构简单，每个模块只实现一个功能。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/083ce8b986f048a8892da98096678cc8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)


## 2.设计背景

在enuo v0.02版本中增加了task.c 和 interface.c两个文件：
1、task.c 中包含了与任务操作相关的函数，如任务创建函数task_create和系统启动调度函数enuo_schedule。
2、interface.c中包含了与处理器硬件相关的操作，增强了系统的移植性。
    在enuo v0.02版本中增加了任务抽象对象，任务相关的数据全部封装到任务抽象对象中。与此同时，在enuo v0.02版本中增加了链表数据结构，任务链表的作用是将多个任务串联起来，能高效地实现任务检索和操作。
    

## 3.设计目标

目前enuo系统可以创建任务，创建的任务在就绪表中，系统定时器可以轮询调度就绪表中的任务。显然任务创建后除了处于运行状态，任务还需要支持停止。因此这个版本为enuo增加一个停止表。

**任务有两种状态就绪状态和停止状态，处理器会轮流执行就绪状态下的任务，停止状态下的任务不会得到运行。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/67564df1d7494a22a908122a0750ab16.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

在enuo 系统中就绪表和停止表都需要进行链表操作，因此完善链表操作并将链表独立出一个C文件，遵守单一原则。

## 4.设计环境

硬件环境是使用STM32F401RE为核心的自制开发板，软件环境是使用的KEIL V5.2 开发工具。

![在这里插入图片描述](https://img-blog.csdnimg.cn/3a264918fcda4f1da0098bbf18c10b30.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

## 5.设计过程

## 5.1链表操作

链表需要实现两个基本操作：
**1、按照顺序将节点插入链表
2、从链表中移除一个节点**

链表需要实现排序操作，因此在链表结构中增加一个排序值sort_value，新链表新数据结构如下：

```c
struct list_node_def
{
	struct list_node_def *  next;		/* 指向下一个列表节点 */
	struct list_node_def *  previous;	/* 指向上一个列表节点 */
	void 	*owner;						/* 指向链表节点拥有者 */
	uint32_t sort_value;				/* 链表排序数值*/
};
```

链表的初始化，插入和删除函数如下：

```c
/*********************************************************************************************************
* @名称	: list_initialise
* @描述	: 列表初始化
**********************************************************************************************************/ 
void list_initialise( list_t * const list )
{
	/* 滑动指针指向表头 */
	list->sliding_pointer = &list->head ;			

	/* 表头的前后指针设置为NULL */
	list->head.next = &list->head;	
	list->head.previous = &list->head;

	/* 表头的排序值为0 */
	list->head.sort_value = MIN_VALUE;
}
/*********************************************************************************************************
* @名称	: list_sort_insert
* @描述	: 按照sort_value值升序插入列表
**********************************************************************************************************/ 
uint8_t list_sort_insert( list_t * const list  , list_node_t * const new_node)
{
	list_node_t *seek;
	const uint32_t current_value = new_node->sort_value;
	uint16_t i = 0;


	/*   按照 sort_value值  找到对应位置  */
	for( seek =  & list->head ; seek->next->sort_value <= current_value; seek = seek->next )  
	{
		/*   限制查找次数  */
		if( (i++) > 255 )
			return 0;
	}
	/*   新节点插入列表  */
	new_node->next = seek->next;
	new_node->next->previous = new_node;
	new_node->previous = seek;
	seek->next = new_node;

	return 1;	
}
/*********************************************************************************************************
* @名称	: list_remove
* @描述	: 列表删除一个节点
**********************************************************************************************************/ 
void list_remove( list_node_t  * const remove_node )
{
	/*删除节点 */
	remove_node->next->previous = remove_node->previous;
	remove_node->previous->next = remove_node->next;	
}
```
**list_sort_insert插入是参考sort_value数值进行升序排列。**

列表中有一个**滑动指针sliding_pointer，滑动指针指向当前节点**，我们构造一个操作：插入链表时，将数据插入滑动指针的末尾。这种操作可以简化轮询任务时将新任务加入到链表尾部，代码如下：

```c
/*********************************************************************************************************
* @名称	: list_insert_sliding_pointer_end
* @描述	: 节点插入滑动指针指向的后
**********************************************************************************************************/ 
void list_insert_sliding_pointer_end( list_t * const list , list_node_t  * const new_node )
{
	/* 保存滑动指针位置 */
	list_node_t * const seek = list->sliding_pointer;

	/* 节点插入滑动指针指向的后 */
	new_node->next = seek;
	new_node->previous = seek->previous;

	seek->previous->next = new_node;
	seek->previous = new_node;

}
```

使用list_insert_sliding_pointer_end函数插入一个节点后的链表关系图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/bf08b111eba0463b8208e96ec34748e7.png)

使用list_insert_sliding_pointer_end函数再插入节点后的链表关系图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/450f728f41374c5ebafd128944245461.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

使用list_remove函数移除一个节点后的链表关系图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/4ec88a8022d64b14be2fc283383563f5.png)

## 5.2任务操作

任务有以下三个个基本操作：
**1、创建一个任务
2、停止一个指定任务
3、恢复一个指定任务**

为了实现停止任务，需要创建一个延时等待列表。需要停止的任务放在延时等待表中。到目前为止enuo将拥有两个列表：

![在这里插入图片描述](https://img-blog.csdnimg.cn/de9564c6a87d4c459658b775d5d9948e.png)

就绪表存放就绪状态的任务，延时等待表存放需要停止的任务，处理器会轮流执行就绪表中的任务，延时等待表中的任务不会得到运行。**任务的操作函数如下：**

```c
/*********************************************************************************************************
* @名称	: task_create
* @描述	: 创建任务
**********************************************************************************************************/ 
void task_create( task_tcb_t *task , task_function_t function , uint32_t *stack_space , uint32_t stack_number )
{
	/* 保存滑动指针位置 */
	list_node_t * const new_node = &task->link;
	/* 链表使用者指向任务 */	
	task->link.owner = task;
	/* 插入滑动指针末尾 */	
	list_insert_sliding_pointer_end( &ready_list , new_node);
	/* 初始化任务栈  */	
	task_stack_init( (uint32_t *)task, function , stack_space , stack_number );	
}
/*********************************************************************************************************
* @名称	: task_stop
* @描述	: 暂停任务
**********************************************************************************************************/ 
void task_stop( task_tcb_t *task  )
{
	/* 保存滑动指针位置 */
	list_node_t * const new_node = &task->link;
	/* 移除原有链表关系 */
	list_remove(new_node);
	/* 插入滑动指针末尾 */	
	list_insert_sliding_pointer_end( &delay_list , new_node);
}
/*********************************************************************************************************
* @名称	: task_resume
* @描述	: 恢复任务
**********************************************************************************************************/ 
void task_resume( task_tcb_t *task  )
{
	/* 保存滑动指针位置 */
	list_node_t * const new_node = &task->link;
	/* 移除原有链表关系 */
	list_remove(new_node);
	/* 插入滑动指针末尾 */	
	list_insert_sliding_pointer_end( &ready_list , new_node);
}
```
使用task_create函数创建task0任务后的就绪表和延时表的关系图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/a5705d015d6f40a28865f71235f410b7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

使用task_create函数创建task1任务后的就绪表和延时表的关系图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/0dd48696ac6b43edab5aa59a6dff6641.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

使用task_stop函数停止task0任务，任务停止后的就绪表和延时表的关系图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/f3e35053f7f843a98918c4a0324a855b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

使用task_resume函数恢复task0任务，任务恢复后的就绪表和延时表的关系图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/93b50ddf50d24753ae5cb9a420385883.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)


## 5.3任务调度

任务调度使用轮询的方法，再链表中读取下一个任务，并使用滑动指针sliding_pointer指向下一个节点。任务调度函数如下：

```c
/*********************************************************************************************************
* @名称	: SysTick_Handler
* @描述	: 系统中断服务程序
**********************************************************************************************************/
void SysTick_Handler(void)
{	
	list_t * const const_list = &ready_list ;													
	/* sliding_pointer 实现循环滑动指向列表中的下一个节点 */										
	( const_list )->sliding_pointer = ( const_list )->sliding_pointer->next;
	
	if( const_list ->sliding_pointer == & const_list ->head  )	
	{																						
		 const_list->sliding_pointer = const_list->sliding_pointer->next;						
			/* 若列表为ListEnd 则指向下一个  就是第一个  实现循环*/							
	}
	next_task = const_list->sliding_pointer->owner;
	
	/* 请求切换任务 */
	TSAK_SWITCH_REQUEST;
	return;
}
```
任务切换使用TSAK_SWITCH_REQUEST宏，宏定义使用linux宏常用的do-while形式，TSAK_SWITCH_REQUEST宏如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/8c8e261257e643239cdd54cf2c20781b.png)

## 5.4功能测试

enuo系统增加了任务停止和任务恢复功能，在task0中设计一个任务控制逻辑，周期性停止和恢复task1和task2两个任务。task0任务设计如下：

```c
void task0(void)
{
	static uint16_t clk = 0;
	static uint16_t state = 0;
	while(1)
	{
		if( ( ( clk++ )%9999 ) == 0 ) 
		{
			task_debug_num0++;	/* 测试跟踪 */
			test_function();
			/* 控制任务1和任务2的运行状态*/
			switch( state )
			{
				case 0:/* 暂停任务1 任务2 */
					task_stop( &my_task1 );
					task_stop( &my_task2 );
				break;
				case 50:/* 恢复任务1 */
					task_resume( &my_task1 );
				break;
				case 75:/* 恢复任务2 */
					task_resume( &my_task2 );
				break;				
				default:
				break;					
			}
			/* 周期计时 */
			if( state++ > 100 )
				state = 0;		
		}
	}
}
```
使用state设计一个状态机：
当state为0时停止task1和task2；
当state为50时恢复task1；
当state为75时恢复task2 。
**如果功能正常运行结果为task0，task1和task2运行次数监控task_debug_num的数值之比约为4：2：1** 。

## 6.运行结果

代码仿真运行后的结果如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/43f57094c7144f0c8d3940bd036dec96.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

**task0，task1和task2运行task_debug_num的数值之比约为4：2：1
证明enuo系统的停止任务和恢复任务的功能正常。**

<font color=red>**希望获取源码的朋友们在评论区里留言。**

> <font color=red>**未完待续…
  
**实时操作系统系列将持续更新**
  
**创作不易希望朋友们点赞，转发，评论，关注。**
  
**您的点赞，转发，评论，关注将是我持续更新的动力**
  
**作者：李巍**
  
**Github：liyinuoman2017**
  
**CSDN：liyinuo2017**
  
**今日头条：程序猿李巍**
