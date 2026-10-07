# C2C 硬件






hopper 中有了 c2c ，cpu 和 gpu 有了内存一致性的内存通路，可以做很多事情，

[No Buffer, No Bottleneck: Efficient Zero-Copy KV Cache Offloading for Long-Context LLMs | USENIX](https://www.usenix.org/conference/osdi26/presentation/luo)

这篇论文就是讲kv cache 放到的 host 侧内存上，调整 fa 的遍历顺序，得到一个不错的效果。

除此之外，他增强了  uvm 的能力，  uva 让 cpu 和 gpu 可以有一个相同的地址空间，uvm 则可以让  gpu  访问cpu 的内存，但是之前由于没有这个通路，主要是通过pcie 和中断来做的。

之前的控制流

1. gpu 要访问 host 的内存，通过 ld 这种内存语义命令
2. gpu 访问这个 host 侧的 内存地址，由于没有映射页表，硬件会报错
3. gpu 会想host发起中断，里面包含 这次的访问的地址 以及 当前 cuda 上下文
4. cpu 侧接收到中断，根据 当前 cuda 上下文 以及地址，知道对应的物理页面
5. 把 host 侧页表的下了，应该这块的页表 host 不能在访问了，刷新 tlb
6. 通过驱动拷贝到gpu侧，并且在 gpu 侧建立页表
7. gpu 再次的读取这个数据，这个时候就有这个数据了

以上应该是个大致的控制流，

‍

有了 c2c 之后，gpu 就可以 使用内存语义直接访问  host 侧内存了，

我们需要解决一些问题

1. gpu 访问的时候，页表怎么建立

   1. gpu 自身也有页表
   2. 但是 gpu 也维护一份 host 页表 的话，也会相当麻烦
   3. 随着linux 的发展， pcie 有了的   ats 这个功能，简单说，就是 设备 可以直接去问  一块地址 的实际物理地址

      1. 虽然最开始的  也有这个功能， 但是相当于 直接把 物理地址给设备，没有任何的防护，相当于 device 运行在 ring 0，还可以随意的读写内存，太危险了
      2. 之后有了 iommu之后，给他整了 iova ，就是每个设备 也有了 va，需要转到的 pa，但是需要做校验
      3. 后来又有了 ats，通过 pagid 来校验权限

‍

因为有了  硬件一致性，英伟达会把 自己 device 内存注册成一个 numa node，可以参与 内存的分配，并且一些 页面的迁移也可以复用 linux 中的 numa 迁移，可以将 整个host 和 device 统一起来

‍

```cpp
void* p = mmap(nullptr, bytes, PROT_READ | PROT_WRITE,
               MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
// 此处暂不读写 p；已有 VMA，数据页可能尚未分配。
gpu_write<<<...>>>(p);
```

我们假设有这么一个代码

1. mmap 在linux 中，申请一个匿名页，在linux 内核中就是一个 vma，并不会分配页面
2. 然后我们把这个指针的传给  gpu
3. gpu 访问这个数据

   1. 可能 gpu 的mmu 以及 ats 缓存查找是同时做的
   2. 发起一次的ats 请求

      1. ats 请求是硬件完成的，硬件 跟 pasid 和 地址的，让mmu 去走pagewalk，如果有结果的话，就缓存，没有的话就报错
4. host 收到中断报错了，ats 请求失败了
5. 内核线根据 pasid 得到mm，然后再根据虚拟地址得到 vma，发现这个地址连页面也没有分配
6. 然后就分配页面，建立页面 pte，

   1. 建立页面这一块， 我们在 allocpage 的时候是可以指定 node id ，这里默认情况下可以直接 走  device  node 的  伙伴系统去分配页面
   2. 在 host 侧建立页表吗？还是说在gpu 上建立页表呢？
7. 把物理地址返回给gpu
8. gpu 似乎不会建立页表的，也是放到 tlb 中，当缓存
9. gpu 再次访问，tlb 中找到，通过c2c 直接访问

   1. 这里的话，我认为mmu 还是有一些能力的，他可以判断这个是 本机的hbm ，还是别人的
   2. 就比如 nvlink 的访问，他根据地址知道这个是别的卡，就会构建一个 nvlink 的请求
   3. 同理，这里，应该会构建一个c2c 的请求

‍

‍

这个大致的流程就是这样，

