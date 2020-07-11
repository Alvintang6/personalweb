---
layout: tech_post
title:  "Linux -- Driver framework"
date:   2020-07-04 11:51:36 -0300
catalogue: Mcu&Mpu
tags: Linux kernel Device-tree
description: kernel Device tree build 
#chinese_link: /chinese/linux_driver_framework.html
---

* TOC
{:toc}


## General device driver framework


A typical character device driver may include following components.

1.initial & exit function (initial and delete driver module)  
2.register & unregister function (inside initial, exit function and used for configurating operations of drivers)  
3.file_operation structure (include basic operation accept by drives like: open,read,write and etc..)



{% highlight C linenos %}


#include <linux/init.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/uaccess.h>
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/errno.h>
#include <asm/mach/map.h>
#include <asm/io.h>
#include <linux/device.h>

#define DEV_NAME "Embedchrdev"
#define DEV_CNT (1)


static int chr_dev_open(struct inode *inode, struct file *filp){
	/* Todo */
    printk("chrdevtest_open \n");
    return 0;
}

static int chr_dev_release(struct inode *inode, struct file *filp){
	/* Todo */
    printk("chrdevtest_release \n");
    return 0;
}

static ssize_t chr_dev_read(struct file *filp, char __user * buf, size_t count, loff_t *ppos)
{	/* Todo */
    printk("chrdevtest_read \n");
    return 0;
 }

static ssize_t chr_dev_write(struct file *filp, const char __user * buf, size_t count, loff_t *ppos)
{	/* Todo */
    printk("chrdevtest_write \n");
    return 0;
 }

// pack the chrdev property inside a structure
struct chrdev_dev{
	dev_t devid;/* deviceid */
  	struct cdev cdev; 
  	struct class *class; /* class */
  	struct device *device; /* device */
  	int major; /* major devid */
  	int minor;
  	struct device_node *rgb_led_node; /* rgb_led root node */
};



static struct file_operations chr_dev_fops={
	.owner = THIS_MODULE,
	// some operations are optional depends on usage
	.open = chr_dev_open,
	.read = chr_dev_read,
	.write = chr_dev_write,
	.release = chr_dev_release,
};

struct chrdev_dev chrdev;

static int __init chrdev_init(void)
{
 int ret = 0;
 printk("chrdev init\n");
 // step (1). register device and/or get device id
 
 

 if (chrdev.major) { /* using defineded deivce number */
        chrdev.devid = MKDEV(chrdev.major, 0);
        register_chrdev_region(chrdev.devid, DEV_CNT, DEV_NAME);
    } else {    /* not defined the device number */
		// device name is 'EmbedCharDev'， can be checked with cat /proc/devices
        ret = alloc_chrdev_region(&chrdev.devid, 0, DEV_CNT, DEV_NAME); /* apply for device number */ 
		if (ret < 0) {
			printk("fail to alloc devno\n");
 			goto alloc_err;
 		}
        chrdev.major = MAJOR(chrdev.devid); /* get major device number */ 
        chrdev.minor = MINOR(chrdev.devid); /* get minor device number */
    } 

 
 //Step (2). match the char device with file operations structure and functions.
 cdev_init(&chrdev.cdev, &chr_dev_fops);

 //Step (3)
 //add chrdev to cdev_map list
 ret = cdev_add(&chrdev.cdev, chrdev.devid, DEV_CNT);
 if (ret < 0) {
 printk("fail to add cdev\n");
 goto add_err;
 }
 return 0;


 add_err:
 //if add char device failed， unregister the chrdev
 unregister_chrdev_region(chrdev.devid, DEV_CNT);
 alloc_err:
 return ret;
 }

 static void __exit chrdev_exit(void){
	 unregister_chrdev_region(chrdev.devid, DEV_CNT);
	 cdev_del(&chrdev.cdev);
 }


 module_init(chrdev_init);
 module_exit(chrdev_exit);
 MODULE_LICENSE("GPL");
{% endhighlight %}


## 1. Driver with device tree and bus(i2c bus for example)

The main framework of bus based driver is similar with general framework, but added device_driver match table which is used to match the driver with device(device tree or device resource file). 


### 1.1 Framework of i2c device driver based on i2c bus 

The `i2c_driver` is used to characterize the property of a i2c device.  

```c
static const struct i2c_device_id gtp_device_id[] ={
    {"jie,i2c_mpu6050", 0},
    {}
};

static const struct of_device_id mpu6050_match_table[]={
    {.compatible = "jie,i2c_mpu6050"},
    {/* Sentinel */}
};


struct i2c_driver mpu6050_driver = {
    .probe = mpu6050_probe,
    .remove = mpu6050_remove,
    .id_table = gtp_device_id,
    .driver ={
            .name = "mpu6050",
            .owner = THIS_MODULE,
            .of_match_table = mpu6050_match_table,
    },
};

```

The .probe indicate the initial functional which will be loaded when mpu6050_driver match with a device. Most of time in .probe we only register and add the device.

The .of_match_table and .id_table will compare and match the driver with a device tree or board-level resource files. Following is a i2c device tree node and the property of compatible will be matched with mpu6050_match_table and gtp_device_id. 

THe main purpose of `.of_match_table` and `.i2c_device_id` is same. The only difference between them is priority. System will try match with `.of_match_table` first if there is no rules, it will try to match with `.i2c_device_id` .


```c
&i2c1 {
	clock-frequency = <100000>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_i2c1>;
 	status = "okay";


	i2c_mpu6050@68 {
		compatible = "jie,i2c_mpu6050";
 		reg = <0x68>;
		position = <0>;

	};
};

```


The following part is the framework of a i2c driver. The start flow can be simplified as:

  1. register the i2c driver inside initial function
  2. if driver match with the device, it goes into `.probe` function
  3. inside `.probe` register your i2c device (register,bind,add,initial).
  4. running the file operation function(open,release,write, read, etc.)


{% highlight c linenos %}

#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of_gpio.h>
#include <linux/semaphore.h>
#include <linux/timer.h>
#include <linux/i2c.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>


struct mpu6050_dev { 
    dev_t devid;
    struct cdev cdev; /* cdev */
    struct class *class; /* class */
    struct device *device; /* device */
    struct device_node * dev_node; /* device_node */
    int major; /* major deviceid */
    void *private_data; /* data */

    short mpu6050_result[6]; /* 6 aixs gyro and accel data */
};



static int mpu6050_open(struct inode *inode, struct file *file)
{

    /* TODO*/
    return 0;
}

static ssize_t mpu6050_read(struct file *filp, char __user *buf, size_t cnt, loff_t *off)
{
    return 0;
}

static int mpu6050_release(struct inode *inode, struct file *filp)
{
    return 0;
} 

static struct file_operations mpu6050_chr_dev_fops =
    {
            .owner = THIS_MODULE,
            .open = mpu6050_open,
            .read = mpu6050_read,
            .release = mpu6050_release,
};

static int mpu6050_probe(struct i2c_client *client, const struct i2c_device_id *id)
{
    
    return 0;
}

static int mpu6050_remove(struct i2c_client *client)
{
    
    return 0;
}


static const struct i2c_device_id gtp_device_id[] ={
    {"jie,i2c_mpu6050", 0},
    {}
};

static const struct of_device_id mpu6050_match_table[]={
    {.compatible = "jie,i2c_mpu6050"},
    {/* Sentinel */}
};



struct i2c_driver mpu6050_driver = {
    .probe = mpu6050_probe,
    .remove = mpu6050_remove,
    .id_table = gtp_device_id,
    .driver ={
            .name = "mpu6050",
            .owner = THIS_MODULE,
            .of_match_table = mpu6050_match_table,
    },
};


static int __init mpu6050_driver_init(void)
{
    int ret;
    pr_info("mpu6050_driver_init");
    ret = i2c_add_driver(&mpu6050_driver);

    return 0;
}


static void __exit mpu6050_driver_exit(void){
    
    pr_info("mpu6050_driver_exit");
    i2c_del_driver(&mpu6050_driver);
        

}


module_init(mpu6050_driver_init);
module_exit(mpu6050_driver_exit);
MODULE_LICENSE("GPL");


{% endhighlight %}




## 2. Driver with device tree and platform 
Platform bus can be seen as a kind of virtual bus, and the usage of platform bus is very similar with normal <b>BUS</b>. 

### 2.1


