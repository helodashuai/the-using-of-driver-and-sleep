
#include <linux/init.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/miscdevice.h>
#include <linux/sched.h> //TASK_*
#include <linux/irq.h>
#include <linux/interrupt.h>
#include <linux/gpio.h>
#include <mach/platform.h>
#include <linux/uaccess.h>

//声明按键的硬件信息数据结构
struct btn_resource {
    int gpio;
    char *name;
};

//定义初始化四个按键硬件信息对象
static struct btn_resource btn_info[] = {
    {
        .gpio = PAD_GPIO_A+28,
        .name = "KEY_UP"
    },
    {
        .gpio = PAD_GPIO_B+9,
        .name = "KEY_DOWN"
    }
};

//定义等待队列头对象(构造鸡妈妈)
static wait_queue_head_t rwq; 

static int kstate; //记录按键的状态

static ssize_t btn_read(struct file *file,
                        char __user *buf,
                        size_t count,
                        loff_t *ppos)
{
    //1.定义装载休眠进程的容器(构造小鸡)
    //一个小鸡对应一个读进程
    wait_queue_t wait;

    //2.将当前进程添加到容器中
    init_waitqueue_entry(&wait, current);

    //3.将当前进程添加到等待队列中
    //注意：此时进程还没有休眠
    add_wait_queue(&rwq, &wait);

    //4.设置进程的休眠类型
    //注意：此时进程还没有休眠
    set_current_state(TASK_INTERRUPTIBLE);
    
    //5.让当前进程正式进入休眠状态
    //此时代码不止不前,进程释放CPU资源
    //等待被唤醒,一旦被唤醒立马返回接续往下执行
    schedule();
    
    //6.一旦被唤醒,进程继续执行,设置为运行状态
    set_current_state(TASK_RUNNING);

    //7.将当前进程从等待队列中移除
    remove_wait_queue(&rwq, &wait);

    //8.对于可中断类型的进程,判断进程被唤醒的原因
    if(signal_pending(current)) {
        printk("读进程[%s][%d]由于接收到了信号引起唤醒\n", current->comm, current->pid);
        return -ERESTARTSYS;
    } else {
        //拷贝按键状态给用户
        //此时的kstate已经被中断处理函数赋值了
        copy_to_user(buf, &kstate, 4);
    }
    return count;
}

//中断处理函数
static irqreturn_t button_isr(int irq, void *dev)
{
    //1.先获取按键信息和状态
    struct btn_resource *pdata = dev;

    kstate = gpio_get_value(pdata->gpio);

    //2.唤醒读进程
    wake_up(&rwq);
    
    return IRQ_HANDLED; //成功
}

static struct file_operations btn_fops = {
    .read = btn_read,
};

static struct miscdevice btn_misc = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = "mybtn",
    .fops = &btn_fops
};

static int btn_init(void)
{
    int i;
    misc_register(&btn_misc);
    //初始化等待队列头(武装鸡妈妈)
    init_waitqueue_head(&rwq);
    //申请GPIO资源
    //申请中断资源并且注册中断处理函数
    for(i = 0; i < ARRAY_SIZE(btn_info); i++) {
        int irq = gpio_to_irq(btn_info[i].gpio);
        gpio_request(btn_info[i].gpio,
                        btn_info[i].name);
        request_irq(irq, button_isr,
         IRQF_TRIGGER_FALLING|IRQF_TRIGGER_RISING,
         _info[i].namebtn, &btn_info[i]);
    }
    return 0;
}

static void btn_exit(void)
{
    int i;

    misc_deregister(&btn_misc);
   
    //释放资源
    for(i = 0; i < ARRAY_SIZE(btn_info); i++) {
        int irq = gpio_to_irq(btn_info[i].gpio);
        gpio_free(btn_info[i].gpio);
        free_irq(irq, &btn_info[i]);
    }
}
module_init(btn_init);
module_exit(btn_exit);
MODULE_LICENSE("GPL");


