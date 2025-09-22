# 存储虚拟化1

前面几节，我们讲了 CPU 和内存的虚拟化。我们知道，完全虚拟化是很慢的，而通过内核的 KVM 技术和 EPT 技术，加速虚拟机对于物理 CPU 和内存的使用，我们称为硬件辅助虚拟化。

\


对于一台虚拟机而言，除了要虚拟化 CPU 和内存，存储和网络也需要虚拟化，存储和网络都属于外部设备，这些外部设备应该如何虚拟化呢？

\


当然一种方式还是完全虚拟化。比如，有什么样的硬盘设备或者网卡设备，我们就用 qemu 模拟一个一模一样的软件的硬盘和网卡设备，这样在虚拟机里面的操作系统看来，使用这些设备和使用物理设备是一样的。当然缺点就是，qemu 模拟的设备又是一个翻译官的角色。虽然这个时候虚拟机里面的操作系统，意识不到自己是运行在虚拟机里面的，但是这种每个指令都翻译的方式，实在是太慢了。

\


另外一种方式就是，虚拟机里面的操作系统不是一个通用的操作系统，它知道自己是运行在虚拟机里面的，使用的硬盘设备和网络设备都是虚拟的，应该加载特殊的驱动才能运行。这些特殊的驱动往往要通过虚拟机里面和外面配合工作的模式，来加速对于物理存储和网络设备的使用。

### virtio 的基本原理

在虚拟化技术的早期，不同的虚拟化技术会针对不同硬盘设备和网络设备实现不同的驱动，虚拟机里面的操作系统也要根据不同的虚拟化技术和物理存储和网络设备，选择加载不同的驱动。但是，由于硬盘设备和网络设备太多了，驱动纷繁复杂。

\


后来慢慢就形成了一定的标准，这就是 **virtio**，就是**虚拟化 I/O 设备**的意思。virtio 负责对于虚拟机提供统一的接口。也就是说，在虚拟机里面的操作系统加载的驱动，以后都统一加载 virtio 就可以了。

\
![](<../.gitbook/assets/image (20).png>)

\


在虚拟机外，我们可以实现不同的 virtio 的后端，来适配不同的物理硬件设备。那 virtio 到底长什么样子呢？我们一起来看一看。

\


virtio 的架构可以分为四层。

\


* 首先，在虚拟机里面的 virtio 前端，针对不同类型的设备有不同的**驱动程序**，但是接口都是统一的。例如，硬盘就是 virtio\_blk，网络就是 virtio\_net。
* 其次，在宿主机的 qemu 里面，实现 virtio 后端的逻辑，主要就是**操作硬件的设备**。例如通过写一个物理机硬盘上的文件来完成虚拟机写入硬盘的操作。再如向内核协议栈发送一个网络包完成虚拟机对于网络的操作。
* 在 virtio 的前端和后端之间，有一个通信层，里面包含 **virtio 层**和 **virtio-ring 层**。virtio 这一层实现的是虚拟队列接口，算是前后端通信的桥梁。而 virtio-ring 则是该桥梁的具体实现。

\
![](<../.gitbook/assets/image (21).png>)

\


virtio 使用 virtqueue 进行前端和后端的高速通信。不同类型的设备队列数目不同。virtio-net 使用两个队列，一个用于接收，另一个用于发送；而 virtio-blk 仅使用一个队列。

\


如果客户机要向宿主机发送数据，客户机会将数据的 buffer 添加到 virtqueue 中，然后通过写入寄存器通知宿主机。这样宿主机就可以从 virtqueue 中收到的 buffer 里面的数据。

\


了解了 virtio 的基本原理，接下来，我们以硬盘写入为例，具体看一下存储虚拟化的过程。

### 初始化阶段的存储虚拟化

和咱们在学习 CPU 的时候看到的一样，Virtio Block Device 也是一种类。它的继承关系如下：

\


```c
static const TypeInfo device_type_info = {
    .name = TYPE_DEVICE,
    .parent = TYPE_OBJECT,
    .instance_size = sizeof(DeviceState),
    .instance_init = device_initfn,
    .instance_post_init = device_post_init,
    .instance_finalize = device_finalize,
    .class_base_init = device_class_base_init,
    .class_init = device_class_init,
    .abstract = true,
    .class_size = sizeof(DeviceClass),
};

static const TypeInfo virtio_device_info = {
    .name = TYPE_VIRTIO_DEVICE,
    .parent = TYPE_DEVICE,
    .instance_size = sizeof(VirtIODevice),
    .class_init = virtio_device_class_init,
    .instance_finalize = virtio_device_instance_finalize,
    .abstract = true,
    .class_size = sizeof(VirtioDeviceClass),
};

static const TypeInfo virtio_blk_info = {
    .name = TYPE_VIRTIO_BLK,
    .parent = TYPE_VIRTIO_DEVICE,
    .instance_size = sizeof(VirtIOBlock),
    .instance_init = virtio_blk_instance_init,
    .class_init = virtio_blk_class_init,
};

static void virtio_register_types(void)
{
    type_register_static(&virtio_blk_info);
}

type_init(virtio_register_types)

```

\


Virtio Block Device 这种类的定义是有多层继承关系的。TYPE\_VIRTIO\_BLK 的父类是 TYPE\_VIRTIO\_DEVICE，TYPE\_VIRTIO\_DEVICE 的父类是 TYPE\_DEVICE，TYPE\_DEVICE 的父类是 TYPE\_OBJECT。到头了。

\


type\_init 用于注册这种类。这里面每一层都有 class\_init，用于从 TypeImpl 生产 xxxClass。还有 instance\_init，可以将 xxxClass 初始化为实例。

\


在 TYPE\_VIRTIO\_BLK 层的 class\_init 函数 virtio\_blk\_class\_init 中，定义了 DeviceClass 的 realize 函数为 virtio\_blk\_device\_realize，这一点在[CPU](https://time.geekbang.org/column/article/109335)那一节也有类似的结构。

\


```c
static void virtio_blk_device_realize(DeviceState *dev, Error **errp)
{
    VirtIODevice *vdev = VIRTIO_DEVICE(dev);
    VirtIOBlock *s = VIRTIO_BLK(dev);
    VirtIOBlkConf *conf = &s->conf;
......
    blkconf_blocksizes(&conf->conf);
    virtio_blk_set_config_size(s, s->host_features);
    virtio_init(vdev, "virtio-blk", VIRTIO_ID_BLOCK, s->config_size);
    s->blk = conf->conf.blk;
    s->rq = NULL;
    s->sector_mask = (s->conf.conf.logical_block_size / BDRV_SECTOR_SIZE) - 1;
    for (i = 0; i < conf->num_queues; i++) {
        virtio_add_queue(vdev, conf->queue_size, virtio_blk_handle_output);
    }
    virtio_blk_data_plane_create(vdev, conf, &s->dataplane, &err);
    s->change = qemu_add_vm_change_state_handler(virtio_blk_dma_restart_cb, s);
    blk_set_dev_ops(s->blk, &virtio_block_ops, s);
    blk_set_guest_block_size(s->blk, s->conf.conf.logical_block_size);
    blk_iostatus_enable(s->blk);
}

```

\


在 virtio\_blk\_device\_realize 函数中，我们先是通过 virtio\_init 初始化 VirtIODevice 结构。

\


```c
void virtio_init(VirtIODevice *vdev, const char *name,
                 uint16_t device_id, size_t config_size)
{
    BusState *qbus = qdev_get_parent_bus(DEVICE(vdev));
    VirtioBusClass *k = VIRTIO_BUS_GET_CLASS(qbus);
    int i;
    int nvectors = k->query_nvectors ? k->query_nvectors(qbus->parent) : 0;

    if (nvectors) {
        vdev->vector_queues =
            g_malloc0(sizeof(*vdev->vector_queues) * nvectors);
    }
    vdev->device_id = device_id;
    vdev->status = 0;
    atomic_set(&vdev->isr, 0);
    vdev->queue_sel = 0;
    vdev->config_vector = VIRTIO_NO_VECTOR;
    vdev->vq = g_malloc0(sizeof(VirtQueue) * VIRTIO_QUEUE_MAX);
    vdev->vm_running = runstate_is_running();
    vdev->broken = false;
    for (i = 0; i < VIRTIO_QUEUE_MAX; i++) {
        vdev->vq[i].vector = VIRTIO_NO_VECTOR;
        vdev->vq[i].vdev = vdev;
        vdev->vq[i].queue_index = i;
    }
    vdev->name = name;
    vdev->config_len = config_size;
    if (vdev->config_len) {
        vdev->config = g_malloc0(config_size);
    } else {
        vdev->config = NULL;
    }
    vdev->vmstate = qemu_add_vm_change_state_handler(virtio_vmstate_change,
                                                     vdev);
    vdev->device_endian = virtio_default_endian();
    vdev->use_guest_notifier_mask = true;
}

```

\


从 virtio\_init 中可以看出，VirtIODevice 结构里面有一个 VirtQueue 数组，这就是 virtio 前端和后端互相传数据的队列，最多 VIRTIO\_QUEUE\_MAX 个。

\


我们回到 virtio\_blk\_device\_realize 函数。接下来，根据配置的队列数目 num\_queues，对于每个队列都调用 virtio\_add\_queue 来初始化队列。

\


```c
VirtQueue *virtio_add_queue(VirtIODevice *vdev, int queue_size,
                            VirtIOHandleOutput handle_output)
{
    int i;
    vdev->vq[i].vring.num = queue_size;
    vdev->vq[i].vring.num_default = queue_size;
    vdev->vq[i].vring.align = VIRTIO_PCI_VRING_ALIGN;
    vdev->vq[i].handle_output = handle_output;
    vdev->vq[i].handle_aio_output = NULL;

    return &vdev->vq[i];
}

```

\


在每个 VirtQueue 中，都有一个 vring，用来维护这个队列里面的数据；另外还有一个函数 virtio\_blk\_handle\_output，用于处理数据写入，这个函数我们后面会用到。

\


至此，VirtIODevice，VirtQueue，vring 之间的关系如下图所示。这是在 qemu 里面的对应关系，请你记好，后面我们还能看到类似的结构。

![](<../.gitbook/assets/image (22).png>)\


### qemu 启动过程中的存储虚拟化

初始化过程解析完毕以后，我们接下来从 qemu 的启动过程看起。

\


对于硬盘的虚拟化，qemu 的启动参数里面有关的是下面两行：

\


```
-drive file=/var/lib/nova/instances/1f8e6f7e-5a70-4780-89c1-464dc0e7f308/disk,if=none,id=drive-virtio-disk0,format=qcow2,cache=none-device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1
```

\


其中，第一行指定了宿主机硬盘上的一个文件，文件的格式是 qcow2，这个格式我们这里不准备解析它，你只要明白，对于宿主机上的一个文件，可以被 qemu 模拟称为客户机上的一块硬盘就可以了。

\


而第二行说明了，使用的驱动是 virtio-blk 驱动。

\


```
configure_blockdev(&bdo_queue, machine_class, snapshot);
```

\


在 qemu 启动的 main 函数里面，初始化块设备，是通过 configure\_blockdev 调用开始的。

\


```c
static void configure_blockdev(BlockdevOptionsQueue *bdo_queue, MachineClass *machine_class, int snapshot)
{
......
    if (qemu_opts_foreach(qemu_find_opts("drive"), drive_init_func,
                          &machine_class->block_default_type, &error_fatal)) {
.....
    }
}

static int drive_init_func(void *opaque, QemuOpts *opts, Error **errp)
{
    BlockInterfaceType *block_default_type = opaque;
    return drive_new(opts, *block_default_type, errp) == NULL;
}

```

\


在 configure\_blockdev 中，我们能看到对于 drive 这个参数的解析，并且初始化这个设备要调用 drive\_init\_func 函数，这里面会调用 drive\_new 创建一个设备。

\


```c
DriveInfo *drive_new(QemuOpts *all_opts, BlockInterfaceType block_default_type, Error **errp)
{
    const char *value;
    BlockBackend *blk;
    DriveInfo *dinfo = NULL;
    QDict *bs_opts;
    QemuOpts *legacy_opts;
    DriveMediaType media = MEDIA_DISK;
    BlockInterfaceType type;
    int max_devs, bus_id, unit_id, index;
    const char *werror, *rerror;
    bool read_only = false;
    bool copy_on_read;
    const char *filename;
    Error *local_err = NULL;
    int i;
......
    legacy_opts = qemu_opts_create(&qemu_legacy_drive_opts, NULL, 0,
                                   &error_abort);
......
    /* Add virtio block device */
    if (type == IF_VIRTIO) {
        QemuOpts *devopts;
        devopts = qemu_opts_create(qemu_find_opts("device"), NULL, 0,
                                   &error_abort);
        qemu_opt_set(devopts, "driver", "virtio-blk-pci", &error_abort);
        qemu_opt_set(devopts, "drive", qdict_get_str(bs_opts, "id"),
                     &error_abort);
    }

    filename = qemu_opt_get(legacy_opts, "file");
......
    /* Actual block device init: Functionality shared with blockdev-add */
    blk = blockdev_init(filename, bs_opts, &local_err);
......
    /* Create legacy DriveInfo */
    dinfo = g_malloc0(sizeof(*dinfo));
    dinfo->opts = all_opts;

    dinfo->type = type;
    dinfo->bus = bus_id;
    dinfo->unit = unit_id;

    blk_set_legacy_dinfo(blk, dinfo);

    switch(type) {
    case IF_IDE:
    case IF_SCSI:
    case IF_XEN:
    case IF_NONE:
        dinfo->media_cd = media == MEDIA_CDROM;
        break;
    default:
        break;
    }
......
}

```

\


在 drive\_new 里面，会解析 qemu 的启动参数。对于 virtio 来讲，会解析 device 参数，把 driver 设置为 virtio-blk-pci；还会解析 file 参数，就是指向那个宿主机上的文件。

\


接下来，drive\_new 会调用 blockdev\_init，根据参数进行初始化，最后会创建一个 DriveInfo 来管理这个设备。

\


我们重点来看 blockdev\_init。在这里面，我们发现，如果 file 不为空，则应该调用 blk\_new\_open 打开宿主机上的硬盘文件，返回的结果是 BlockBackend，对应我们上面讲原理的时候的 virtio 的后端。

\


```c
BlockBackend *blk_new_open(const char *filename, const char *reference,
                           QDict *options, int flags, Error **errp)
{
    BlockBackend *blk;
    BlockDriverState *bs;
    uint64_t perm = 0;
......
    blk = blk_new(perm, BLK_PERM_ALL);
    bs = bdrv_open(filename, reference, options, flags, errp);
    blk->root = bdrv_root_attach_child(bs, "root", &child_root,
                                       perm, BLK_PERM_ALL, blk, errp);
    return blk;
}

```

\


接下来的调用链为：bdrv\_open->bdrv\_open\_inherit->bdrv\_open\_common.

\


```c
static int bdrv_open_common(BlockDriverState *bs, BlockBackend *file,
                            QDict *options, Error **errp)
{
    int ret, open_flags;
    const char *filename;
    const char *driver_name = NULL;
    const char *node_name = NULL;
    const char *discard;
    QemuOpts *opts;
    BlockDriver *drv;
    Error *local_err = NULL;
......
    drv = bdrv_find_format(driver_name);
......
    ret = bdrv_open_driver(bs, drv, node_name, options, open_flags, errp);
......
}

static int bdrv_open_driver(BlockDriverState *bs, BlockDriver *drv,
                            const char *node_name, QDict *options,
                            int open_flags, Error **errp)
{
......
    bs->drv = drv;
    bs->read_only = !(bs->open_flags & BDRV_O_RDWR);
    bs->opaque = g_malloc0(drv->instance_size);

    if (drv->bdrv_open) {
        ret = drv->bdrv_open(bs, options, open_flags, &local_err);
    } 
......
}

```

\


在 bdrv\_open\_common 中，根据硬盘文件的格式，得到 BlockDriver。因为虚拟机的硬盘文件格式有很多种，qcow2 是一种，raw 是一种，vmdk 是一种，各有优缺点，启动虚拟机的时候，可以自由选择。

\


对于不同的格式，打开的方式不一样，我们拿 qcow2 来解析。它的 BlockDriver 定义如下：

\


```c
BlockDriver bdrv_qcow2 = {
    .format_name        = "qcow2",
    .instance_size      = sizeof(BDRVQcow2State),
    .bdrv_probe         = qcow2_probe,
    .bdrv_open          = qcow2_open,
    .bdrv_close         = qcow2_close,
......
    .bdrv_snapshot_create   = qcow2_snapshot_create,
    .bdrv_snapshot_goto     = qcow2_snapshot_goto,
    .bdrv_snapshot_delete   = qcow2_snapshot_delete,
    .bdrv_snapshot_list     = qcow2_snapshot_list,
    .bdrv_snapshot_load_tmp = qcow2_snapshot_load_tmp,
    .bdrv_measure           = qcow2_measure,
    .bdrv_get_info          = qcow2_get_info,
    .bdrv_get_specific_info = qcow2_get_specific_info,

    .bdrv_save_vmstate    = qcow2_save_vmstate,
    .bdrv_load_vmstate    = qcow2_load_vmstate,

    .supports_backing           = true,
    .bdrv_change_backing_file   = qcow2_change_backing_file,

    .bdrv_refresh_limits        = qcow2_refresh_limits,
......
};

```

\


根据上面的定义，对于 qcow2 来讲，bdrv\_open 调用的是 qcow2\_open。

\


```c
static int qcow2_open(BlockDriverState *bs, QDict *options, int flags,
                      Error **errp)
{
    BDRVQcow2State *s = bs->opaque;
    QCow2OpenCo qoc = {
        .bs = bs,
        .options = options,
        .flags = flags,
        .errp = errp,
        .ret = -EINPROGRESS
    };

    bs->file = bdrv_open_child(NULL, options, "file", bs, &child_file,
                               false, errp);
    qemu_coroutine_enter(qemu_coroutine_create(qcow2_open_entry, &qoc));
......
}

```

\


在 qcow2\_open 中，我们会通过 qemu\_coroutine\_enter 进入一个协程 coroutine。什么叫协程呢？我们可以简单地将它理解为用户态自己实现的线程。

\


前面咱们讲线程的时候说过，如果一个程序想实现并发，可以创建多个线程，但是线程是一个内核的概念，创建的每一个线程内核都能看到，内核的调度也是以线程为单位的。这对于普通的进程没有什么问题，但是对于 qemu 这种虚拟机，如果在用户态和内核态切换来切换去，由于还涉及虚拟机的状态，代价比较大。

\


但是，qemu 的设备也是需要多线程能力的，怎么办呢？我们就在用户态实现一个类似线程的东西，也就是协程，用于实现并发，并且不被内核看到，调度全部在用户态完成。

\


从后面的读写过程可以看出，协程在后端经常使用。这里打开一个 qcow2 文件就是使用一个协程，创建一个协程和创建一个线程很像，也需要指定一个函数来执行，qcow2\_open\_entry 就是协程的函数。

\


```c
static void coroutine_fn qcow2_open_entry(void *opaque)
{
    QCow2OpenCo *qoc = opaque;
    BDRVQcow2State *s = qoc->bs->opaque;

    qemu_co_mutex_lock(&s->lock);
    qoc->ret = qcow2_do_open(qoc->bs, qoc->options, qoc->flags, qoc->errp);
    qemu_co_mutex_unlock(&s->lock);
}

```

\


我们可以看到，qcow2\_open\_entry 函数前面有一个 coroutine\_fn，说明它是一个协程函数。在 qcow2\_do\_open 中，qcow2\_do\_open 根据 qcow2 的格式打开硬盘文件。这个格式[官网](https://github.com/qemu/qemu/blob/master/docs/interop/qcow2.txt)就有，我们这里就不花篇幅解析了。

### 总结时刻

我们这里来总结一下，存储虚拟化的过程分为前端、后端和中间的队列。

\


* 前端有前端的块设备驱动 Front-end driver，在客户机的内核里面，它符合普通设备驱动的格式，对外通过 VFS 暴露文件系统接口给客户机里面的应用。这一部分这一节我们没有讲，放在下一节解析。
* 后端有后端的设备驱动 Back-end driver，在宿主机的 qemu 进程中，当收到客户机的写入请求的时候，调用文件系统的 write 函数，写入宿主机的 VFS 文件系统，最终写到物理硬盘设备上的 qcow2 文件。
* 中间的队列用于前端和后端之间传输数据，在前端的设备驱动和后端的设备驱动，都有类似的数据结构 virt-queue 来管理这些队列，这一部分这一节我们也没有讲，也放到下一节解析。

<figure><img src="../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>
