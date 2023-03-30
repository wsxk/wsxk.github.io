---
layout: post
tags: [iot]
title: "intel SGX"
date: 2023-3-29
author: wsxk
comments: true
---

- [introduction](#introduction)
- [大致逻辑图](#大致逻辑图)
  - [oncall\_table](#oncall_table)
- [sgx\_create\_enclave](#sgx_create_enclave)
- [sgx\_ecall](#sgx_ecall)


## introduction<br>
intel sgx是intel的一种硬件保护措施，其实是trustzone那一套<br>

## 大致逻辑图<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230330123437.png)
其实看图可以发现，intel sgx的可信区域和不可信区域是可以互通的（有2组接口分别处理）<br>
详情可以看[https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-biondo.pdf](https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-biondo.pdf)<br>
总的来说，从untrusted code 调用 trusted code 需要经过`EENTER`,这里有一个`ecall table`，`ecall table`是受信任的函数的表格。<br>
同时也有一个`ocall table`，`ocall table`是trusted code处理函数信息时可能会用到的untrusted code的函数列表<br>

### oncall_table<br>
ocall_table参数在sgx_ecall中用于处理从Enclave内部发出的不可信代码回调。在Intel SGX中，Enclave（可信代码）与主机应用程序（不可信代码）之间的通信是双向的。Enclave可以通过sgx_ecall从主机应用程序中调用可信函数，同时Enclave内的代码也可以通过sgx_ocall调用主机应用程序中的不可信函数。<br>
ocall_table是一个指向sgx_ocall函数表的指针，它包含了Enclave允许调用的不可信函数的列表。当Enclave中的代码需要调用不可信函数时，它将使用ocall_table中的函数指针来进行调用。<br>
ocall_table的作用主要有两个方面：<br>
安全性：通过限制Enclave可以调用的不可信函数，可以减少潜在的安全风险。Enclave内的代码只能调用已在ocall_table中定义的函数，这有助于避免不必要的代码执行或数据泄露。<br>
模块化：ocall_table允许开发者在Enclave内部实现与主机应用程序之间的接口，使Enclave内的代码能够与外部系统交互。这有助于将受信任的代码与不可信的代码分离，实现代码的模块化。


## sgx_create_enclave<br>
`sgx_create_enclave`是一个用于创建和初始化 Intel SGX Enclave 的函数。Intel SGX（Software Guard Extensions）是一种安全技术，用于保护应用程序的敏感数据和代码免受未经授权访问。Enclave 是 SGX 中的一个受保护内存区域，用于安全地执行敏感操作。<br>
`sgx_create_enclave` 函数是 Intel SGX 开发者参考的一部分，它在创建一个新的 Enclave 时起着关键作用。通过这个函数，应用程序可以将一个已签名的 Enclave 映像加载到受保护的内存中，并初始化 Enclave。初始化完成后，应用程序可以调用 Enclave 内的功能，执行安全操作。<br>
`sgx_create_enclave` 函数的原型如下<br>
```c
sgx_status_t sgx_create_enclave(
    const char *file_name,  // 已签名的 Enclave 映像文件的路径，即你要放入enclave（即可信区域的共享库的）名称
    int debug,//调试标志。设置为 1，表示允许调试；设置为 0，表示禁止调试
    sgx_launch_token_t *launch_token,//指向 sgx_launch_token_t 类型的指针，用于提供用于加载 Enclave 的启动令牌。首次加载 Enclave 时，传递一个空指针；后续调用时，使用先前的令牌
    int *launch_token_updated,//指示启动令牌是否已更新。如果此值为 1，表示应用程序需要保存更新后的令牌
    sgx_enclave_id_t *enclave_id,//返回创建的 Enclave 的标识符
    sgx_misc_attribute_t *misc_attr//一个可选参数，用于返回 Enclave 的其他属性，如处理器相关的信息
);
```
成功调用 `sgx_create_enclave` 后，可以使用返回的 `enclave_id` 在应用程序中调用 Enclave 的功能。在不再需要 Enclave 时，使用 `sgx_destroy_enclave` 函数销毁 Enclave。<br>



## sgx_ecall<br>
`sgx_ecall` 是一个函数，用于从非Enclave（untrusted）代码调用Enclave（trusted）内部的函数。在Intel SGX（Software Guard Extensions）中，Enclave是一个受保护的内存区域，用于安全地执行敏感操作。为了确保数据和代码安全，只有受信任的Enclave内部函数能直接访问这些受保护的资源。因此，在从外部应用程序调用Enclave内部函数时，需要使用sgx_ecall来实现<br>
sgx_ecall函数是Intel SGX SDK中的一个核心功能，用于在untrusted（不可信）应用程序代码中调用Enclave中的trusted（可信）函数。然而，sgx_ecall的实现是依赖于特定的Intel SGX SDK和底层硬件的，因此它的源代码通常不在开源SDK中提供。sgx_ecall的具体实现是在底层硬件上执行一个特殊的ECALL指令，将控制权从不可信代码转移到Enclave内的可信代码。<br>
尽管我们不能提供sgx_ecall的源代码，但我们可以解释其工作原理。当调用sgx_ecall时，需要提供以下参数：<br>

> 1. eid：Enclave的ID，用于标识要与之通信的Enclave。
> 2. func_id：Enclave中的受信任函数的ID，用于确定要调用的具体函数。
> 3. ocall_table：一个指向sgx_ocall函数表的指针，用于处理Enclave内部的对不可信代码的回调。
> 4. ms：一个指向sgx_ms_t结构的指针，该结构包含受信任函数的输入参数和输出结果。

当调用sgx_ecall时，底层硬件将执行以下操作：<br>

> 1. 保存当前untrusted（不可信）应用程序的执行环境。
> 2. 验证Enclave的内存边界和权限，确保只有受信任代码可以访问Enclave内的数据。
> 3. 将控制权转移到Enclave内的受信任代码，并传递func_id、ocall_table和ms参数。
> 4. 在Enclave内执行受信任函数。
> 5. 当受信任函数执行完成后，将控制权返回给untrusted（不可信）应用程序，并恢复之前保存的执行环境。

