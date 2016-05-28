---
title: StateMachine
date: 2016-01-05 14:18:22
tags:
- wifi
- android
- statemachine
---
# Hierarchy StateMachine Simple Guide

>状态机是一种设计模式，层级状态机是由多个有层次结构的状态构成，不同的状态处理各自能够处理的消息并实现相应功能。

一个State需要实现processMessage以及三个可选enter/exit/getName  
enter/exit类似构造和析构函数，用于State初始化和退出清除  
getName获得该State对象的名字，一种State可以有多个实现方案  
processMessage用来处理该当前状态接收到的消息  

***

当StateMachine对象创建后，需要用addState方法构造树状的层级结构，并且设置初始化状态，即默认状态  
创建完成后调用start方法从而进入初始化状态，启动会调用初始化状态所有父节点的enter方法直到初始化状态的enter  
>需要注意的是，这种调用是通过message-handler的方式进行的，并不是直接调用,所有消息的处理都是在一个HandlerThread中进行

enter过程完成后当前初始化状态即可接收消息并处理之  

    SampleStateMachine
        mP1
      /     \
    mS2     mS1 ------> initialed state
当该StateMachine创建并添加mS1和mS2且设置mS1为初始状态，enter的顺序为mP1->mS1  
启动后，可通过sendMessage发送消息命令给该StateMachine，消息可通过obtainMessage得到  
StateMachine接收到消息mS1状态会调用其processMessage方法处理，可通过transitionTo切换到其他State  
如果当前State处理不了该条消息，则processMessage返回false或NOT_HANDLED交由父状态处理  

***

切换状态时，exit/enter的执行顺序是从当前状态调用exit开始，直到与另一个状态的公共父节点，不包括公共父节点  
然后开始调用公共父节下字节点到目标状态的所有状态的enter  
如果没有公共父节点则退出所有状态然后重新进入  

***

另外两个传递消息的方法是sendMessageAtFrontOfQueue和deferMessage
sendMessageAtFrontOfQueue是将消息放到消息队列头部  
deferMessage是将消息延迟到下一个状态并放到消息队列头部  
```java
Override public boolean processMessage(Message message) {
    boolean retVal;
    log("mP1.processMessage what=" + message.what);
    switch(message.what) {
    case CMD_2:
        // CMD_2 will arrive in mS2 before CMD_3
        sendMessage(obtainMessage(CMD_3));
        deferMessage(message);
        transitionTo(mS2);
        retVal = HANDLED;
        break;
    default:
        // Any message we don't understand in this state invokes unhandledMessage
        retVal = NOT_HANDLED;
        break;
    }
    return retVal;
}
```
CMD_2和CMD_3都会在mS2中被处理，且CMD_2在CMD_3前被处理
