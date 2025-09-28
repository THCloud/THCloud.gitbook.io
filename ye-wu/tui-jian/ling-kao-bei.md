# 零拷贝

正常：

device -> kernel -> application -> socket -> 网卡

dma：

device  dma到kernel-> application -> socket dma到网卡

sendfile函数：

* app调用sendfile函数
* device dma到kernel
* kernel cpu copy到socket
* socket dma 网卡
* socket返回sendfile函数

sg-dma：

* app调用sendfile
* device dma到kernel
* kernel sgdma到网卡
* socket返回sendfile函数
