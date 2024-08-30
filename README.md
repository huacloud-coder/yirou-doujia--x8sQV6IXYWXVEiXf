
[![](https://img2024.cnblogs.com/blog/1060878/202408/1060878-20240829161126058-1990466990.png)](https://img2024.cnblogs.com/blog/1060878/202408/1060878-20240829161126058-1990466990.png)


程序员会遇到一种情况，一个bug排查到最后是由一个很小的问题导致的。在昨天的日常搬砖中遇到一个问题，耽搁了我大半天的时间，最后查明原因让我很无语。


首先介绍一下背景，我是做算法模型训练，目前手上的工作是迭代一个算法，添加最新的数据集训练出一个精度更好的模型。


拿到标注好的xml的数据集，我首先做了一个格式的转换和数据集的拆分。使用的代码如下：



```
import os
import shutil
import xml.etree.ElementTree as ET

def list_dir(path: str) -> str:
    """列出目录下所有的文件"""
    for item in os.listdir(path):
        yield item


def parse_xml():

    base_path = "/home/lijinkui/Desktop/head_shoudler_20240824/头肩检测_V1.13_20240823083458_V1"
    label_dir = f"{base_path}/gt"
    count = 0

    test_obj = open(f"{base_path}/test/test_ssd.txt", "a")
    # train_obj = open(f"{base_path}/train/train.txt", "a")


    for item in list_dir(label_dir):
        xml_path = f"{label_dir}/{item}"
        # print(xml_path)
        # 从文件中解析XML，获取根元素

        root = ET.parse(xml_path).getroot()
        filename = root.find('filename').text.strip()
        width = int(root.find('size').find('width').text)
        height = int(root.find('size').find('height').text)
        # print(width, height)
        count += 1
        print(count)
        box_list = []
        for index, label in enumerate(root.iter('object')):
            category = label.find('name').text

            bbox = label.find('bndbox')
            x1 = bbox.find('xmin').text
            y1 = bbox.find('ymin').text
            x2 = bbox.find('xmax').text
            y2 = bbox.find('ymax').text

            box_list.extend([x1, y1, x2, y2, "1"])
        txt_string = " ".join(box_list) + "\n"
        if count <= 1200:
            # test_obj.write(txt_string)
            # shutil.move(f"{base_path}/images/{filename}", f"{base_path}/test/images/{filename}")
            test_obj.write(f"# 20240824/{filename} \n")
            test_obj.write(txt_string)
        # else:
        #     train_obj.write(txt_string)
        #     # shutil.copy(f"{base_path}/images/{filename}", f"{base_path}/train/images/{filename}")
    test_obj.close()
    # train_obj.close()


if __name__ == '__main__':
    parse_xml()
```

将前1200张保存为测试集



```
if count <= 1200:
    # test_obj.write(txt_string)
    # shutil.move(f"{base_path}/images/{filename}", f"{base_path}/test/images/{filename}")
    test_obj.write(f"# 20240824/{filename} \n")
    test_obj.write(txt_string)
```

保存好的格式是：



```
# meili/00320.jpg                                                                                                       
1796 550 1861 618 1
# meili/00330.jpg
1674 515 1749 585 1
# meili/00340.jpg
1527 473 1609 545 1
# meili/00350.jpg
1373 457 1455 531 1
```

然后我就配置好参数，愉快的启动了训练。结果还没跑多久，就报错了。经过排查，报错的代码如下：



```
def converter(args):
    im_file, image_name, labels = args
    try:
        # 这种方式直接获取图片的信息，加载速度快，但是会漏掉一些图片崩溃的情况
        # im = Image.open(im_file)
        # im.verify()  # PIL verify
        # img_w, img_h = exif_size(im)  # (width, height)

        # 采用opencv读取能够发现数据集中崩溃的图片，不至于影响训练
        im = cv2.imread(im_file)

        img_h, img_w = im.shape[:2]

        tmp = []
        for l, x1, y1, x2, y2 in labels:
            x, y, w, h = (x1 + x2) // 2, (y1 + y2) // 2, x2 - x1, y2 - y1
            x, y, w, h = x / img_w, y / img_h, w / img_w, h / img_h
            tmp.append([l, x, y, w, h])
        return image_name, tmp
    except Exception as e:
        print("-------------------------")
        print(im_file)
        print(f"{im_file} has broken..: {e}")
        return None, None
```

报错的信息是：20240824/0f4963ca91ff7c26d66c69b028415243b8a8a405\.jpg has broken ... : NoneType has no shape。


我第一反应就是这个图片可能是损坏了，最简单的验证方法就是用opencv的库show一下。按照这个思路我打开了一个python终端，在终端中show图片。



```
>>> import cv2 as cv
>>> 
>>> image = cv.imread("/h3c_data/data/recognize_new_data/project_dataset/HeadShoulder/Test/Image/20240824/002bf647508fac15babe697c625c3004589b1607.jpg")
>>> 
>>> image.shape
(1080, 1920, 3)
>>>
```

这么检查下来，发现也没有问题啊，图片明明没有损坏。
然后我猜测难道是图片的权限或用户组问题导致不可读吗？于是检查了文件的权限
[![](https://img2024.cnblogs.com/blog/1060878/202408/1060878-20240829161140929-1845956015.png)](https://img2024.cnblogs.com/blog/1060878/202408/1060878-20240829161140929-1845956015.png)


读写权限和用户组都没问题。奇怪了，到底是啥问题呢？


这时我就准备祭出断点大招，在图片读取之前打一个断点，断点一步步向下走，看看是哪里的问题。结果断点也打不上。因为这个函数是放在线程池中执行的，每次8个线程并发执行，遇到断点就直接退出了。



```
with Pool(NUM_THREADS) as pool:
    pbar = tqdm(pool.imap_unordered(converter, zip(image_pathes, image_names, labels)),
                desc=desc, total=len(image_pathes))
    for image_name, tmp in pbar:
        if image_name:
            dst_ret.write("%s" % image_name)
            for l, *info, in tmp:
                line = (l, *info)
                dst_ret.write((" %d" + " %g"*len(info)) % line)
            dst_ret.write("\n")
```

断点也不行，那我就没招了。其实我打端点就是想看看文件的路径是不是有问题，既然不能看每一个，那我干脆就看所有的。所以我打印了所有图片路径列表的变量，于是发现了问题。
原来每一个图像路径的后面都多了一个空格，我恍然大悟，怪不得用python终端show的时候没有发现，终端里面不打印空格标识。我从终端里复制图像路径，根本不知道路径的后面还有一个空格。
[![](https://img2024.cnblogs.com/blog/1060878/202408/1060878-20240829161201921-1355800287.png)](https://img2024.cnblogs.com/blog/1060878/202408/1060878-20240829161201921-1355800287.png)


而且回过头来再看空格是哪来的，就是在数据集处理的时候随手多打了一个空格。
[![](https://img2024.cnblogs.com/blog/1060878/202408/1060878-20240829161157841-1216345124.png)](https://img2024.cnblogs.com/blog/1060878/202408/1060878-20240829161157841-1216345124.png)


这个问题本来不难发现，但是由于第一空格在终端里面不展示导致没发现这个空格，第二想要断点调试时由于程序在线程池中断点也不生效，导致也不能断点调试变量，不能发现这个空格。
在一些巧合之前，让我折腾了大半天的时间，最后才解决这个问题。程序员的日常就是和各种bug斗智斗勇。
[![](https://img2024.cnblogs.com/blog/1060878/202408/1060878-20240829161153097-459487408.png)](https://img2024.cnblogs.com/blog/1060878/202408/1060878-20240829161153097-459487408.png):[MeoMiao 萌喵加速](https://biqumo.org)


\_\_EOF\_\_

![](https://github.com/goldsunshine/p/18386910.html)goldsunshine 本文链接：[https://github.com/goldsunshine/p/18386910\.html](https://github.com)关于博主：评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。版权声明：本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！声援博主：如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。您的鼓励是博主的最大动力！
