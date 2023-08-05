# 金士顿 DTSE9G2 128G U盘量产踩坑记


<meta name="referrer" content="no-referrer" />

## 前言

买过金士顿U盘的应该都知道一般金士顿是不支持bitlocker的，但是这样又很不方便，就想捣鼓一下。

从naivekun师傅那里知道了一个词叫量产，通过给U盘刷固件，就可以让U盘被识别为一个CD或者硬盘，从而支持bitlocker。结果折腾了两天。。。踩了各种坑。一开始刷炸了之后128g缩水成32g，然后又捣鼓捣鼓救了回来，反而扩容到了132g？神秘。

## 前期准备

型号：金士顿 DTSE9G2 128G

工具：ChipGenius

​			ST-TOOL_9000_v3.7F.92

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614689541756-5362b169-4462-417b-b4f2-d44ccc490832.png)

## 参数设置

下载工具解压后打开STTOOL_F1_90_v200_00_SZ.exe

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614688151229-50a092b2-09db-427f-aebd-6d2102274ada.png)

点击更新识别U盘，然后进入设定

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614689662363-feb2c4a5-ce80-4699-8039-a346515d433b.png)

固件档案中选择的是2309_hv3_ED3_T_1P.BIN而不是上文教程中的2309_hv3_ED3_M_1P.BIN。因为猜测M是mlc的意思，T是tlc的意思。ChipGenius中显示U盘是tlc，所以换成了2309_hv3_ED3_T_1P.BIN。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614683892494-b1015522-d629-48ee-ae81-7008806bf949.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614684012623-d50d69a0-003b-4d66-886d-62e347fffb4f.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614684027055-eb205961-d12a-4e78-8c8f-a52684d7c99b.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614684032917-f90006df-a3cd-4ad8-b11e-1020cd1d27d5.png)

这里选择容量优先

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614684040144-39e6a18c-637d-492f-b635-1412d646a5f9.png)

自己是已经低格一遍了，所以选的高格扫描，分类方式选择容量有限。

低格一次4-5个小时，高格一次3-5分钟左右。

因为我的CE是4个就选的4，Capacity是U盘容量大小，我选择的是自动，也可以设置指定大小。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614684053391-17c96e20-72d3-4fdf-a456-fa9f600cf61d.png)

搞完后点击保存，返回上个界面。

## 坑

### 0x01

量产工具要在本机运行，不要在虚拟机里面运行，否则会提示奇奇怪怪的错误。被坑了好久

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614668632227-037b759a-8621-4177-957c-184e11772896.png)

### 0x02

开始naivekun师傅是按照这个教程来的[[教程\] 群联PS2251-09(PS2309)U盘量产](http://bbs.wuyou.net/forum.php?mod=viewthread&tid=417696&extra=&page=1)，刷完之后发现128g缩水到了32g。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614690324324-8c7b51f5-adbb-493d-a0a8-f4c432f1d4b5.png)

帖子下面也有人出现了同样的问题

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614687752396-f7b44df1-42a1-4e24-86db-16584babd2a3.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614687777323-22850737-7ba0-447b-acbf-b17a29c0310b.png)

64g的没事，128g的就缩水。刚开始以为是CE太多，固件不兼容啥的，后来发现其实是因为工具默认使用的是速度优先策略，会把低速数据块抛弃，才会导致量产后容量变小但是用起来非常顺畅。

### 0x03

格完之后不要急着拔U盘，在U盘里新建一个文件再拔，否则再次插入会不识别U盘。神秘

## 量产过程

第一次是选择了低格+高格，贼鸡儿慢。。。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614668605274-d497a6d0-19ff-4916-b47b-e65b912824ce.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614676197893-ba7e74f2-071f-4d76-ad1a-039e1f0fada3.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614679572568-ecfaa77a-0d4d-4d8f-ad6a-540b7999e948.png)

完事之后去看设备管理器发现已经量产成功，但是拔出U盘再插入就会无法识别。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614679583008-244d50f7-11c3-43a3-a69a-33ec0a3afd61.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614679638921-0155c412-640c-4385-b6b8-f70d3aacdf32.png)

后来用高格又刷了一遍，刚刷完之后没先拔出来，在U盘里新建了一个txt，然后拔出U盘，再次读取，成功识别！

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614688796939-70295c46-4cb4-4065-a845-4ffbbca57955.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614684188015-a9c246de-2e40-4a9a-83ec-fdb333888456.png)

但是怎么变成132g了。。。还扩容了呢

测试一下读写

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614680458215-70378558-68c5-4766-9c7a-722b99bc2df4.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614684204711-1e265145-f084-45f6-816d-0438c28511f2.png)

360U盘鉴定一下容量

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614683314738-5a322691-ffea-4bf6-a9ed-3673ebf78190.png)

还行吧，预期范围之内



选中U盘右键，终于出现了bitlocker的选项。。。

加密驱动器，成功！

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1614684982853-e045b1c0-32e3-47e6-ab9b-6df341695b18.png)

## 最后

没事还是不要搞量产orz。
