---
layout: tech_post
title:  "Linux DTB -- device tree"
date:   2020-05-25 11:51:36 -0300
catalogue: Mcu&Mpu
tags: Linux kernel Device-tree
description: kernel Device tree build 
chinese_link: /chinese/dts.html
---

* TOC
{:toc}



## 1. Device Build 

At `/linux-x.x.xx` file path using `make xxx.dtb` to compile device tree binary only.  
To build customized dts adding the corresponding name in `/linux-x.x.xx/arch/arm/boot/dts/Makefile`.


## 2. Device tree grammar

- dts beginning from `/`.
- dts may include header `dtsi` which means some common device feature shared with certain SOC.
- Using `$` symbol to append device in configured dts file.
- node defination format: 
label: node-name@reg-address 
	we can use `&label ` to quick acess the node to append some devices.
	Most of the time the reg-address is the start address of register. It might also be device nums or some thing also, etc: like i2c device number:

```c

&i2c1 {
	clock-frequency = <100000>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_i2c1>;
	status = "okay";

	mag3110@e {
		compatible = "fsl,mag3110";
		reg = <0xe>;
		position = <2>;
	};

	fxls8471@1e {
		compatible = "fsl,fxls8471";
		reg = <0x1e>;
		position = <0>;
		interrupt-parent = <&gpio5>;
		interrupts = <0 8>;
	};
};

``` 

## 3. Some specical node
- aliases

- chosen
   Uboot bootargs


## 4. Property

- compatible: The compatible board module can use
  `compatible` will compare the property with 'arch/xxx/mash-xxx'
  and see if any label match with DT_MACHINE_START().

```c

static const char * const imx53_dt_board_compat[] __initconst = {
	"fsl,imx53",
	NULL
};

DT_MACHINE_START(IMX53_DT, "Freescale i.MX53 (Device Tree Support)")
	.init_early	= imx53_init_early,
	.init_machine	= imx53_dt_init,
	.init_late	= imx53_init_late,
	.dt_compat	= imx53_dt_board_compat,
MACHINE_END

```


- model: exactly name of the module
- #address-cells  & #size-cells 
  This two properties will influence the format of <reg> in subnodes:
For example:

```c

spi4{
	#address-cells = <1>; 
	#size-cells = <1>;
	
	xxx{
		reg = <0x10000000 0x1000>;
	}
}

i2c4{
	#address-cells = <1>; 
	#size-cells = <0>;
	
	mag3110@e{
		reg = <0x1e>; 
	} 
}

```
- ranges: address mapping
  blank when there is no address mapping from child-bus-address to parent-bus-address
For example:

```c

soc{
	...
	ranges = <0x0 0xe0000000 0x00100000>;


	serial{
		...
		reg = <0x4600 0x100>;
		...
	}
}

```
With this mapping serial device node would be remapped at address 0xe0004600.

- name: 
- device_type: mostly in "cpu","mem"


- Tips: Reference document: 
Document/devicetree/bindings/xxx



## 5. "of" function

In linux system lib, a series "of" function are provided to obtain the node information from the device tree.


- struct device_node *of_find_node_by_path(const char *path);
- struct device_node *of_find_node_by_name(struct device_node *from,const char *name);
- struct device_node *of_find_node_by_type(struct device_node *from,const char *type);
  ......

The return structure `device_node` is defined in the following:

```c
struct device_node {
    const char *name;  // node name
    const char *type;  // device type
    phandle phandle;
    const char *full_name; // full name
    struct fwnode_handle fwnode;

    struct  property *properties; // property
    struct  property *deadprops;    /* removed properties */
    struct  device_node *parent; // parent node 
    struct  device_node *child;  // child node
    struct  device_node *sibling;
#if defined(CONFIG_OF_KOBJ)
    struct  kobject kobj;
#endif
    unsigned long _flags;
    void    *data;
#if defined(CONFIG_SPARC)
    const char *path_component_name;
    unsigned int unique_id;
    struct of_irq_controller *irq_trans;
#endif
};

```


- struct property *of_find_property(const struct device_node *np,const char *name,int *lenp)

Properties node is listed :

```c

struct property {
    char    *name;  // property name
    int     length;     // property length
    void    *value; // property value
    struct property *next; 
#if defined(CONFIG_OF_DYNAMIC) || defined(CONFIG_SPARC)
    unsigned long _flags;
#endif
#if defined(CONFIG_OF_PROMTREE)
    unsigned int unique_id;
#endif
#if defined(CONFIG_OF_KOBJ)
    struct bin_attribute attr;
#endif
};

```

For reading the properties linux provide the function:

- if the property is a array:
- int of_property_read_uXX_array(const struct device_node *np, const char *propname, u32 *out_values, size_t sz) (XX = 8 16 32 64)


- if the property is int:
- int of_property_read_uXX (const struct device_node *np, const char *propname,u8 *out_values)(XX = 8 16 32 64)

- if the string: 
- int of_property_read_string(const struct device_node *np,const char *propname, const char **out_string)


## 6. Match between device and drive

### 6.1 Device tree device match with driver
 device tree matching driver will use of_match_table. The of_match_table should be define inside other structure.

```c
static const struct of_device_id rgb_led[] = {
	{.compatible = "fire,rgb_led"},
	{/* sentinel */}};

struct platform_driver led_platform_driver = {
.driver = {
		.name = "rgb-leds-platform",
		.owner = THIS_MODULE,
		.of_match_table = rgb_led,
	}
};
```

## 7. Pinctrl & GIOP subsystem

### 7.1. Pinctrl:

Pinctrl system is just other device tree nodes but under iomuxc node.

For example create a rgb led device tree using GPIO1_IO04, GPIO4_IO20, GPIO4_IO19:

```c

&iomuxc {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_hog_1>;

    pinctrl_rgb_led:rgb_led{
                    fsl,pins = <
                            MX6UL_PAD_GPIO1_IO04__GPIO1_IO04        0x000010B1
                            MX6UL_PAD_CSI_HSYNC__GPIO4_IO20         0x000010B1
                            MX6UL_PAD_CSI_VSYNC__GPIO4_IO19         0x000010B1
                    >;
            };

```
Here the `MX6UL_PAD_GPIO1_IO04__GPIO1_IO04` refers the register(MUX,PAD,INPUT) address related with GPIO1_IO04, and 0x00010B1 is the value written into PAD_CTL register.


- Macro definition of Pin property: 

Some marco definition of IO-Pin property can be find in `/arch/arm/boot/dts/xxx-pinfun.h` 
 For example:

 ```cpp

/*
 * The pin function ID is a tuple of
 * <mux_reg conf_reg input_reg mux_mode input_val>
 */

 #define MX6UL_PAD_GPIO1_IO04__ENET1_REF_CLK1		0x006c 0x02f8 0x0574 0 1
 #define MX6UL_PAD_GPIO1_IO04__PWM3_OUT			    0x006c 0x02f8 0x0000 1 0
 #define MX6UL_PAD_GPIO1_IO04__USB_OTG1_PWR		    0x006c 0x02f8 0x0000 2 0
 #define MX6UL_PAD_GPIO1_IO04__REF_CLK_24M		    0x006c 0x02f8 0x0000 3 0
 #define MX6UL_PAD_GPIO1_IO04__USDHC1_RESET_B		0x006c 0x02f8 0x0000 4 0
 #define MX6UL_PAD_GPIO1_IO04__GPIO1_IO04		    0x006c 0x02f8 0x0000 5 0
 #define MX6UL_PAD_GPIO1_IO04__ENET2_1588_EVENT0_IN	0x006c 0x02f8 0x0000 6 0
 #define MX6UL_PAD_GPIO1_IO04__UART5_DCE_TX		    0x006c 0x02f8 0x0000 8 0
 #define MX6UL_PAD_GPIO1_IO04__UART5_DTE_RX		    0x006c 0x02f8 0x0644 8 2

 ```

 Combining the iomuxc root node inside the `xxxx.dtsi` in the following:

 ```cpp

	iomuxc: iomuxc@20e0000 {
				compatible = "fsl,imx6ul-iomuxc";
				reg = <0x20e0000 0x4000>;
			};

 ```
 The explanation of following field:   
 \<mux_reg conf_reg input_reg mux_mode input_val\>
- mux_reg:
  This field indicated the offset of SW_MUX_CTL Register (multiplexer)  
  Address: 20E_0000h base + 6Ch offset = 20E_006Ch  


- conf_reg:
  This field indicated the offset of SW_PAD_CTL Register (pads)  
  Address: 20E_0000h base + 2F8h offset = 20E_02F8h

- input_reg:
  This field indicated the offset of DATA_SELECT_INPUT DAISY Register.  
  Take UART5_DTE_RX for example:
  Address: 20E_0000h base + 644h offset = 20E_0644h

- mux_mode:
  the value configured in MUX_CTL register 
  0101 ALT5 — Select mux mode: ALT5 mux port: GPIO1_IO04 of instance: gpio  

- input_val:
  the value will be written into input_reg.


### 7.2 pinctrl driver:

  When pinctrl node compatible property match with the compatible member of platform device structure `  const struct of_device_id ` the .probe function which inside platform_device will be executed. 

  ```cpp

	static const struct of_device_id imx6ul_pinctrl_of_match[] = {
		{ .compatible = "fsl,imx6ul-iomuxc", .data = &imx6ul_pinctrl_info, },
		{ .compatible = "fsl,imx6ull-iomuxc-snvs", .data = &imx6ull_snvs_pinctrl_info, },
		{ /* sentinel */ }
	};


   static struct platform_driver imx6ul_pinctrl_driver = {
	 .driver = {
		.name = "imx6ul-pinctrl",
		.of_match_table = of_match_ptr(imx6ul_pinctrl_of_match),
	 },
	 .probe = imx6ul_pinctrl_probe,
   };

  ```
  inside `imx_pinctrl_probe` a pinctrl_register function will register a pinctrl_desc structure.
 The pinctrl_desc contains all operations to configure the properties of 'cpu' or 'mpu' chips.
 like : 
  - pinconf_set : set the Pads of pins.
  - pmx_set : set the multiplex of pins.


### 7.3. GPIO subsystem

After we configured the pins as GPIO. The next step is setting some properties of our GPIO pins. Here the GPIO subsystem can be used to do this work.

```cpp




```

<span style="color:red;">Tips:</span>


