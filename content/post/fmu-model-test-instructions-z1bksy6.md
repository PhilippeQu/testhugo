---
title: FMU模型测试说明
slug: fmu-model-test-instructions-z1bksy6
url: /post/fmu-model-test-instructions-z1bksy6.html
date: '2025-06-10 13:32:20+08:00'
lastmod: '2025-06-11 05:55:44+08:00'
toc: true
isCJKLanguage: true
---



# FMU模型测试说明

# 文档描述

本文档服务于FMU除tank以外的传感器模型校验。

# 测试目的

在确保tank模型的液位高度计算相对误差小于千分之一的基础上，进行其余六个传感器模型的校验。

校验分为两步：

1. 验证传感器模型测试表现和源码算法一致性
2. 验证源码算法和项目要求一致性

# FMI非标实现问题

理论上，我们应该保证两个一致性：

1. FMU模型源码变量和`modelDescription.xml`​一致。
2. 联合仿真过程中，master严格按照`modelDescription.xml`​调度slave模型。

但实际上，FMU大多由仿真软件自动化生成，手动编写FMU模型源码存在诸多困难，可参考案例较少，即便是官方示例也没有完全遵守FMI标准。因此项目中存在一些需要非标准化处理的地方。这里陈列一些和官方FMI标准可能有出入的地方，作为注意事项。

1. ​`modelFunctions.c`​一致性问题。目前的6个传感器模型算法各自都用一个c函数实现，为了便于封装，最初都写在同一个`modelFunctions.c`​中，在不同的FMU项目中复制粘贴以简化操作。但由于后续缺乏维护，不同FMU模型的`modelFunctions.c`​文件只能保证当前模型计算函数正确，不具备一致性。
2. ​`modelDescription.xml`​变量冗余。同样为了简化操作，不同传感器模型中的`modelDescription.xml`​关于变量声明的部分直接复制粘贴，导致`modelDescription.xml`​中的变量冗余，不能完全反应源码真实情况，二者缺乏一致性。
3. 默认值硬编码问题。部分模型将 input 变量的默认值硬编码在模型结构体中，且没有在`modelDescription.xml`​中注明。
4. tank模型将`rotation_x`​等input旋转值，和加速度合并为总旋转值后重新赋值给原变量，可能不符合`modelDescription.xml`​。
5. master-slave联合仿真时，master没有完全按照`modelDescription.xml`​调度变量。使用master调度7个模型时，会获取tank模型的input并输入至其他模型。这在一定程度上简化了计算，但如果要调整旋转顺序，可能需要调换变量真实值的顺序，导致歧义。

# 处理方式

由于我们不需要将FMU模型导入至仿真软件使用，因此暂不考虑重构代码使其符合FMI标准，只考虑以下三者之间的一致性：

1. 单个fmu核心算法模块正确
2. 联合仿真调度无异常
3. java api正常工作

# 传感器模型文档

具体模型文档参考FMU传感器模型文档[^1]。

# 已有测试结果总结

‍

[^1]: # FMU传感器模型文档

    # 0: tank 油箱模型

    ## 要求

    根据油箱三维模型以及姿态、加速度、油量，静态计算液面高度。

    ## 模型变量

    ```c
    //无用变量
        int counter;
        double calculate_step_size;
    //通过java api输入
        const char *STL_path;
        double delta_z;
        double rotation_x;
        double rotation_y;
        double rotation_z;
        double acceleration_x;
        double acceleration_y;
        double acceleration_z;
        double volume;
    //输出
        double min_z;
        double height;
        double rotation_x;//合并加速度的旋转角
        double rotation_y;//合并加速度的旋转角
        double rotation_z;//合并加速度的旋转角
    ```
    ## 模型算法

    王洪轩_飞机燃油油箱油液体积的快速网格折叠体积算法

    # 1: consumption sensor 耗量传感器

    ## 要求

    ![image](/images/image-20250610183343-jo6b8u5.png)

    ## 模型变量

    ```c
    //无用输入
        int counter;
        double calculate_step_size;
    //通过java api输入
        double x;
        double y;
        double z;
        double K;//算法输入
        double delta_v;//算法输入
    //tank输出再输入
        double rotation_x;//合并加速度的旋转角
        double rotation_y;//合并加速度的旋转角
        double rotation_z;//合并加速度的旋转角
        double min_z;
        double height;
    //输出
        double n;
    ```
    ## 模型算法

    ```c
    double model_consumption_sensor(double K, double delta_v)
    {
        double n = K * delta_v;
        return n;
    }
    ```
    ## 是否满足要求

    已满足

    # 2: density sensor 密度传感器

    ## 要求

    ![image](/images/image-20250610183500-5g6utig.png)

    ## 模型变量

    ```c
    //无用输入
        int counter;
        double calculate_step_size;
    //通过java api输入
        double x;
        double y;
        double z;
        double K0;//算法输入
        //double K1;
        double K2;//算法输入
        double T;//算法输入
    //tank输出再输入
        double rotation_x;//合并加速度的旋转角
        double rotation_y;//合并加速度的旋转角
        double rotation_z;//合并加速度的旋转角
        double min_z;
        double height;
    //输出
        double density;
    ```
    ## 模型算法

    ```c
    double model_density_sensor(double T, double K0, double K2)
    {
        double density= K0 + K2 * pow(T, 2);
        return density;
    }
    ```
    ## 是否满足要求

    尚未满足，谐振频率需要透传

    # 3: dielectric constant sensor 介电常数传感器

    ## 要求

    ![image](/images/image-20250610183836-icjjopt.png)

    ## 模型变量

    ```c
    //无用输入
        int counter;
        double calculate_step_size;
    //通过java api输入
        double x;//算法输入
        double y;//算法输入
        double z;//算法输入
        double epsilon_oil;//算法输入
        double sensor_height;//算法输入
        double K00;//算法输入
        double K10;//算法输入
        double K01;//算法输入
        double K20;//算法输入
        double K11;//算法输入
        double K02;//算法输入
    //tank输出再输入
        double rotation_x;//合并加速度的旋转角 //算法输入
        double rotation_y;//合并加速度的旋转角 //算法输入
        double rotation_z;//合并加速度的旋转角 //算法输入
        double min_z;//算法输入
        double height;
    //输出
        double C;
    ```
    ## 模型算法

    ```c
    double H;//旋转变换后的传感器安装点z坐标 - min_z
    double h = sensor_height;
    ...

    double model_dielectric_constant_sensor(double h, double epsilon_oil, double H, double K00, double K10, double K01, double K20, double K11, double K02)
    {
        // double deta_C = 2 * PI * delta_epsilon_oil / log(r1 / r2);
        // return deta_C;
        double C = K00 + K10 * epsilon_oil + K01 * h + K20 * epsilon_oil * epsilon_oil + K11 * epsilon_oil * h + K02 * h * h;
        return C;
    }
    ```
    未使用的H输入，算法中并未利用，可能是忘删了。

    ## 是否满足要求

    已满足

    # 4: fuel level indicator 油液信号器

    ## 要求

    ![image](/images/image-20250610184032-mrvwfzv.png)

    ## 模型变量

    ```c
    //无用输入
        int counter;
        double calculate_step_size;
    //通过java api输入
        double x;//算法输入
        double y;//算法输入
        double z;//算法输入
        int type;//算法输入
        double tension;//算法输入
    //tank输出再输入
        double rotation_x;//合并加速度的旋转角 //算法输入
        double rotation_y;//合并加速度的旋转角 //算法输入
        double rotation_z;//合并加速度的旋转角 //算法输入
        double min_z; //算法输入
        double height; //算法输入
    //输出
        double S;
    ```
    ## 模型算法

    ```c
    //  type 0 : full-fuel indicator
    //  type 1 : empty-fuel indicator
    double H;//旋转变换后的传感器安装点z坐标 - min_z
    double Hf = H;
    double He = H;
    double h = height;
    int model_fuel_level_indicator(int type, double Hf, double He, double h, double tension)
    {
        if (type == 0)
        {
            if (h < Hf)
                return 0;
            else
                return tension;
        }
        else if (type == 1)
        {
            if (h < He)
                return tension;
            else
                return 0;
        }
        else
            printf("Wrong type! Type must be 0(full-fuel indicator) or 1(empty-fuel indicator)!\n");
        return 0;
    }
    ```
    ## 是否满足要求

    存疑，这里将信号器压缩为一个点，油液高度大于安装高度即可输出满油电压，理论上没问题。

    # 5: fuel level sensor 油液传感器

    ## 要求

    ![image](/images/image-20250610184723-nldyzmf.png)

    ## 模型变量

    ```c
    //无用输入
        int counter;
        double calculate_step_size;
    //通过java api输入
        double x_up;//算法输入
        double y_up;//算法输入
        double z_up;//算法输入
        double x_down;//算法输入
        double y_down;//算法输入
        double z_down;//算法输入
        double epsilon_oil;//算法输入
        double K00;//算法输入
        double K10;//算法输入
        double K01;//算法输入
        double K20;//算法输入
        double K11;//算法输入
        double K02;//算法输入
    //tank输出再输入
        double rotation_x;//合并加速度的旋转角 //算法输入
        double rotation_y;//合并加速度的旋转角 //算法输入
        double rotation_z;//合并加速度的旋转角 //算法输入
        double min_z; //算法输入
        double height; //算法输入
    //输出
        double height_in_oil;
        double C;
    ```
    ## 模型算法

    ```c
    double h = height_in_oil

    double model_fuel_level_sensor(double h, double epsilon_oil, double H, double K00, double K10, double K01, double K20, double K11, double K02)
    {
        // double C = (h * 2 * PI * (epsilon_oil - epsilon_air) + 2 * PI * epsilon_air * H) / log(r2 / r1);
        double C = K00 + K10 * epsilon_oil + K01 * h + K20 * epsilon_oil * epsilon_oil + K11 * epsilon_oil * h + K02 * h * h;
        return C;
    }
    ```
    ## 是否满足要求

    根据油液高度和安装位置计算浸油高度，再根据浸油高度拟合电容值，理论上正确。

    # 6: temperature sensor 温度传感器

    ## 要求

    ![image](/images/image-20250610184931-wfykn2x.png)

    ## 模型变量

    ```c
    //无用输入
        int counter;
        double calculate_step_size;
    //通过java api输入
        double x;
        double y;
        double z;
        double A;//算法输入
        double B;//算法输入
        double C;//算法输入
        double R0;//算法输入
        double T;//算法输入
    //tank输出再输入
        double rotation_x;
        double rotation_y;
        double rotation_z;
        double min_z;
        double height;

    //输出
        double RT;
    ```
    ## 模型算法

    ```c

    double model_temperature_sensor(double R0, double A, double B, double C, double T)
    {
        double R_T;
        if (T < 0)
            R_T = R0 * (1 + A * T + B * pow(T, 2) + C * (T - 100) * pow(T, 3));
        else
            R_T = R0 * (1 + A * T + B * pow(T, 2));
        return R_T;
    }
    ```
    ## 是否满足要求

    已满足
