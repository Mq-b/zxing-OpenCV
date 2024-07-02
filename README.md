# 编译构建

需要使用到 CMake、Git、GCC 或 MSVC。

github 链接：https://github.com/zxing-cpp/zxing-cpp

编译之前请确保：

1. 确保安装了 CMake 版本 3.15 或更高版本。

2. 确保安装了与 C++17 兼容的编译器(最低VS 2019 16.8 / gcc 7 / clang 5)。

编译构建很简单，如同官网：

```txt
git clone https://github.com/zxing-cpp/zxing-cpp.git --recursive --single-branch --depth 1
cmake -S zxing-cpp -B zxing-cpp.release -DCMAKE_BUILD_TYPE=Release
cmake --build zxing-cpp.release -j8 --config Release
```

如果出错那就只能自行解决，不过不用担心，我们已经提供了编译完成的静态库。

此库不支持动态库，我们提供了 debug 和 release 两种 `.lib`。

之后就是正常的引入静态库的环境配置了，不再介绍。

## OpenCV

ZXing 理论上不依赖 OpenCV 等其它库，不过涉及图形操作，我们的第一选择基本也就是 OpenCV 了。实测 OpenCV3 与 OpenCV4 都可使用。

## OpenCV + Zxing 根据字符串生成 PDF417 条形码

```cpp
#include <opencv2/opencv.hpp>
#include <BarcodeFormat.h>
#include <BitMatrix.h>
#include <TextUtfEncoding.h>
#include <MultiFormatWriter.h>

using namespace cv;
using namespace ZXing;

int main() {
    std::string input = "Hello, ZXing! This is a PDF417 barcode.";

    MultiFormatWriter writer(BarcodeFormat::PDF417);
    auto bitMatrix = writer.encode(input, 200, 100);

    int width = bitMatrix.width();
    int height = bitMatrix.height();
    Mat barcodeImage(height, width, CV_8UC1);

    for (int y = 0; y < height; ++y) {
        for (int x = 0; x < width; ++x) {
            barcodeImage.at<uint8_t>(y, x) = bitMatrix.get(x, y) ? 0 : 255;
        }
    }

    namedWindow("PDF417 Barcode", WINDOW_NORMAL);
    imshow("PDF417 Barcode", barcodeImage);
    waitKey(0);

    imwrite("pdf417_barcode.png", barcodeImage);
}
```

![1](image/运行示例.png)

实测扫码枪可以识别。实测不管 windows 是否开启全局 utf8，都可正常识别数据。（未测试中文）

我们稍微详细的来解释一下上面的代码。

1. **包含头文件**：

- 包含 OpenCV 和 ZXing 库的头文件。以及引入对应的命名空间

```cpp
#include <opencv2/opencv.hpp>
#include <BarcodeFormat.h>
#include <BitMatrix.h>
#include <TextUtfEncoding.h>
#include <MultiFormatWriter.h>

using namespace cv;
using namespace ZXing;
```

2. **输入字符串并生成 PDF417 条形码**

   - 定义一个要编码为 PDF417 条形码的字符串：

     ```cpp
     std::string input = "Hello, ZXing! This is a PDF417 barcode.";
     ```

   - 生成 PDF417 条形码：

     创建一个 `MultiFormatWriter` 对象，指定条形码格式为PDF417。

     使用 `encode` 方法将输入字符串编码为一个位矩阵（bit matrix），并指定图像的宽度和高度（200 x 100）。

     ```cpp
     MultiFormatWriter writer(BarcodeFormat::PDF417);
     auto bitMatrix = writer.encode(input, 200, 100); // 宽度和高度可以根据需要调整
     ```

3. **将条形码数据转换为 OpenCV Mat 对象：**

   - 获取位矩阵的宽度和高度。

   - 创建一个 OpenCV 的灰度图像矩阵。

   - 遍历位矩阵的每个像素点，如果为真则设为黑色（0），否则设为白色（255）

   ```cpp
   int width = bitMatrix.width();
   int height = bitMatrix.height();
   Mat barcodeImage(height, width, CV_8UC1);
   
   for (int y = 0; y < height; ++y) {
       for (int x = 0; x < width; ++x) {
           barcodeImage.at<uint8_t>(y, x) = bitMatrix.get(x, y) ? 0 : 255;
       }
   }
   ```

4. **显示条形码图像**：

   创建窗口并显示，等待用户按下任意键关闭窗口：

   ```cpp
   namedWindow("PDF417 Barcode", WINDOW_NORMAL);
   imshow("PDF417 Barcode", barcodeImage);
   waitKey(0);
   ```

5. **保存条形码为本地图像**：

   ```cpp
   imwrite("pdf417_barcode.png", barcodeImage);
   ```

## 优化

在将条形码数据转换为 OpenCV 的 Mat 对象时，原始代码使用了一个双重循环进行逐个像素的遍历。虽然对于小数据集来说这没什么问题，但在面对庞大的数据时，这种方法的效率可能会受到影响。因此，我们可以通过以下方式进行优化，减少不必要的遍历。

首先，初始化一个白色背景的Mat对象：

```cpp
Mat barcodeImage(height, width, CV_8UC1, Scalar(255)); // 初始化为白色
```

然后，创建一个**指向 Mat 对象数据的指针**，以便直接访问和修改像素值：

```cpp
uint8_t* ptr = barcodeImage.ptr<uint8_t>(0);
```

接下来，使用双重循环遍历位矩阵。在每次迭代中，检查当前位是否设置为真（即需要绘制为黑色）。如果是，则直接通过指针修改相应位置的像素值为 0（黑色）。这种方法减少了逐个像素访问的开销，提高了效率：

```cpp
for (int y = 0; y < height; ++y) {
    for (int x = 0; x < width; ++x) {
        if (bitMatrix.get(x, y)) {
            ptr[y * width + x] = 0; // 设置为黑色
        }
    }
}
```

---

当然了，最直观有效的方式应该是利用硬件性能，也就是：创建线程......

## 总结

最大的问题不在于代码的理解，而是在于环境的构建。

代码本身是简单的，根据这个小例子足以修改、增加，完成许多的需求。
