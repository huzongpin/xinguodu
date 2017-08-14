对于电平触发中断和沿触发中断，在Linux中分别用了handle_level_irq和handle_edge_irq进行处理。中断发生后，系统的中断开关会自动处于disable状态，这由CPU的硬件保证（至少arm中是这样），所以两个函数都在中断禁止的环境中执行。
handle_level_irq

void handle_level_irq (unsigned int irq, struct irq_desc *desc)

       spin_lock(&desc->lock);

       mask_ack_irq(desc, irq);              // 调用了mask，unmask之前不应该再进入这个中断

       // 如果是嵌套调用，直接退出

       if (unlikely(desc->status & IRQ_INPROGRESS))

              goto out_unlock;

       desc->status &= ~(IRQ_REPLAY | IRQ_WAITING);

       …

       action = desc->action;

       if (unlikely(!action || (desc->status & IRQ_DISABLED)))

              goto out_unlock;

 

       desc->status |= IRQ_INPROGRESS;   // 加中断处理，防止嵌套调用

       spin_unlock(&desc->lock);   // 解锁，为可能的嵌套做准备

       /* handle_IRQ_event中可能会开中断，如果unmask也被调用，可能会产生嵌套调用 */

       action_ret = handle_IRQ_event(irq, action);      

       if (!noirqdebug)

              note_interrupt(irq, desc, action_ret);

 

       spin_lock(&desc->lock);      // 重新加锁

       desc->status &= ~IRQ_INPROGRESS;             // 去除中断处理标记

       if (!(desc->status & IRQ_DISABLED) && desc->chip->unmask)

              // 不管前面handle_IRQ_event中有没有unmask，这里再unmask一次

              desc->chip->unmask(irq);    

out_unlock:

       spin_unlock(&desc->lock);

 

handle_edge_irq

void handle_edge_irq (unsigned int irq, struct irq_desc *desc)

       spin_lock(&desc->lock);

       desc->status &= ~(IRQ_REPLAY | IRQ_WAITING);

 

       if (unlikely((desc->status & (IRQ_INPROGRESS | IRQ_DISABLED)) ||

                  !desc->action)) {

              desc->status |= (IRQ_PENDING | IRQ_MASKED);   // 设置中断嵌套标记

              mask_ack_irq(desc, irq);

              goto out_unlock;

       }

       …

       /* ack，这个中断的状态标记被清除 */

       desc->chip->ack(irq);

       /* Mark the IRQ currently in progress.*/

       desc->status |= IRQ_INPROGRESS;                 // 设置中断处理标记位

 

       do {

              struct irqaction *action = desc->action;

              irqreturn_t action_ret;

              …

              if (unlikely((desc->status &

                            (IRQ_PENDING | IRQ_MASKED | IRQ_DISABLED)) ==

                           (IRQ_PENDING | IRQ_MASKED)))

             {      // 如果有中断嵌套发生，在这里调用unmask，重新允许嵌套

                     desc->chip->unmask(irq);

                     desc->status &= ~IRQ_MASKED;

              }

 

              desc->status &= ~IRQ_PENDING;

              spin_unlock(&desc->lock);

              /* handle_IRQ_event中可能会重新开中断，导致中断嵌套 */

              action_ret = handle_IRQ_event(irq, action);

              if (!noirqdebug)

                     note_interrupt(irq, desc, action_ret);

              spin_lock(&desc->lock);

       } while ((desc->status & (IRQ_PENDING | IRQ_DISABLED)) == IRQ_PENDING);

 

       desc->status &= ~IRQ_INPROGRESS;

out_unlock:

       spin_unlock(&desc->lock);

根据代码可知，这两个函数的处理大同小异，www.linuxidc.com都是调用handle_IRQ_event进行实质性的中断处理工作；它们的区别在于中断嵌套的处理。

在电平触发中断处理中，在handle_level_irq的一开始就调用了mask_ack_irq，屏蔽此中断，所以理论上不会产生同一个中断的嵌套调用。但是，在handle_IRQ_event中，会调用驱动程序登记的中断回调函数，而这些函数内核无法控制，无法保证其不调用unmask过程。所以，在调用handle_IRQ_event时，可能仍会出现同一个中断嵌套的情况。解决这个问题的方法很简单，在刚进入handle_level_irq时，使用desc->status & IRQ_INPROGRESS判断当前是否处于嵌套中，如果是嵌套，则直接返回。

在沿触发中断处理中，允许同一个中断的嵌套处理，此时使用IRQ_PENDING来标记嵌套，然后在handle_edge_irq中使用do-while来循环处理嵌套中断。沿触发中断在判断当前处于嵌套时，会调用mask禁止再一次出现嵌套，防止中断处理过程被频繁打断。

在实际写驱动过程中，经常会出现错误地使用handle_level_irq处理沿触发中断，或者错误地使用handle_edge_irq处理电平触发中断的情况，下面分析一下出现这两种错误时的现象：

使用handle_level_irq处理沿触发中断

由于handle_level_irq中调用了mask，直到中断处理完成之后才调用unmask，在这期间不会响应新的中断，所以沿触发的中断在这个过程中会丢失。

使用handle_edge_irq处理电平触发中断

handle_edge_irq不会调用mask，所以在调用handle_IRQ_event时，如果重新打开中断，就会进入嵌套。嵌套过程中会调用mask，禁止第二次嵌套出现，同时设置IRQ_PENDING标记。等到handle_IRQ_event返回时，由于那个IRQ_PENDING的存在，会再一次调用handle_IRQ_event，导致效率降低。
