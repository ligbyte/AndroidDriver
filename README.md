# 内核驱动程序执行 a + b 操作及通过 JNI 在 Android 中调用

本文档描述了如何编写一个简单的内核驱动程序来执行 `a + b` 操作，并展示了如何在Android应用中通过JNI调用这个驱动。

## 内核驱动程序

首先，创建一个名为 `add_module.c` 的文件，包含以下代码实现加法操作。

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

#define DEVICE_NAME "add_dev"

static int major;

static ssize_t dev_read(struct file *filep, char *buffer, size_t len, loff_t *offset) {
    return 0;
}

static ssize_t dev_write(struct file *filep, const char *buffer, size_t len, loff_t *offset) {
    int num;
    int a, b;
    copy_from_user(&num, buffer, sizeof(int)*2);
    a = num & 0xFFFFFFFF;
    b = (num >> 32) & 0xFFFFFFFF;
    num = a + b;
    copy_to_user(buffer, &num, sizeof(int));
    return len;
}

static struct file_operations fops = {
   .read = dev_read,
   .write = dev_write,
};

int init_module(void) {
    major = register_chrdev(0, DEVICE_NAME, &fops);
    if (major < 0) {
        printk(KERN_ALERT "Registering char device failed with %d\n", major);
        return major;
    }
    printk(KERN_INFO "I was assigned major number %d. To talk to\n", major);
    return 0;
}

void cleanup_module(void) {
    unregister_chrdev(major, DEVICE_NAME);
}

```

## 编译步骤

将上述代码保存为 add_module.c。
创建一个 Makefile 文件以编译该模块：

```makefile
obj-m += add_module.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

## 安装步骤

1. 使用 insmod add_module.ko 加载模块。
2. 确认设备 /dev/add_dev 已创建。如果没有，请手动创建：mknod /dev/add_dev c <major> 0（其中 <major> 是你从日志或注册过程中获得的主要编号）。


## Android JNI 调用

在Android项目中，你需要使用JNI来调用这个驱动。

1. 在你的Android项目中创建一个本地方法：

```java
public class AddDriverJNI {
    static {
        System.loadLibrary("add_jni");
    }

    public native int add(int a, int b);
}
```

2. 创建相应的C代码 (add_jni.c) 来实现这个方法：

```c
#include <jni.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>

JNIEXPORT jint JNICALL Java_com_your_package_AddDriverJNI_add(JNIEnv *env, jobject obj, jint a, jint b) {
    int fd, result;
    long inputs = ((long)a << 32) + b;
    fd = open("/dev/add_dev", O_RDWR);
    write(fd, &inputs, sizeof(long));
    read(fd, &result, sizeof(int));
    close(fd);
    return result;
}
```

3.编译并将其链接到你的Android项目中，确保正确设置NDK路径和配置。
请注意，这只是一个基本的指南，实际实现可能需要根据具体情况调整。此外，在Android设备上加载自定义内核模块通常需要root权限，并且可能违反设备保修条款。确保了解所有风险并在合法范围内进行实验。
