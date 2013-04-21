---
author: sunsj1231
comments: true
date: 2010-12-30 05:52:20
layout: post
slug: little-tip
title: Little Tip
wordpress_id: 25001
categories:
- coding
---



    
    struct __XXX{
        void (*p)(void);
        .......
        char a[0];
    };
    




a[0] is a flexible array member ,can be used in C99 protocol.As a special case,the last element of the structure can be an incomplete member(called flexible array member). The flexible array member does not take any memory.if you want allocate some dynamic memory,the flexible array member is worth to use.




for example,you could call function malloc() as below: malloc(sizeof(struct __XXX) + Dynamic_len); the Dynamic_len is the length of the dynamic memory, and you can access the memory use the a[0] ,just as same as a pointer. But if you use a pointer ,i would take 4 bytes(32bit OS) space.The flexible array member will save this space for you~ sounds great. But this tip can not use in C++ ,since the space layout of the structure is unknown -___-.




by the way,in the PC environment, who cases 4 bytes memory.....but in embedded system , this tip should be helpful~_~



