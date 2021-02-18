# image源码分析

## 主要接口
```go
// package: image/color

type Color interface {
    // 返回rgba值
    RGBA() (r, g, b, a uint32)
    // 将颜色转换成对应的颜色格式
    Convert(c Color) Color
}

type Model interface {
    // 将颜色转换成对应的颜色类型
	Convert(c Color) Color
}

// package: image

type Image interface {
    // 返回image使用的model类型
    ColorModel() color.Model
    // 返回图像边界
    Bounds() Rectangle
    // 返回图像上某一点的颜色
    At(x, y int) color.Color
}

type PalettedImage interface {
    image.Image
    // 返回图像上某一点的颜色在调色板中的索引值
    ColorIndexAt(x, y int) uint8
}

// package: image/draw

type Image interface {
	image.Image
    // 设置图像中某一点的颜色
	Set(x, y int, c color.Color)
}

type Quantizer interface {
	// Quantize appends up to cap(p) - len(p) colors to p and returns the
	// updated palette suitable for converting m to a paletted image.
    // (TODO: 没看懂。。。而且源码中也没有哪个struct实现它)
	Quantize(p color.Palette, m image.Image) color.Palette
}

type Drawer interface {
    // 不带蒙板的alpha合成(Porter-Duff composition)
	Draw(dst Image, r image.Rectangle, src image.Image, sp image.Point)
}

// color.Image和draw.Image的区别在于，后者多了绘图功能
```

## 几个重要概念
- 色彩模式: RGBA/YCbCr/CMYK(打印色)
- 颜色类型(Color): 包括 RGBA/NRGBA/RGBA64/NRGBA64/Alpha/Alpha16/Gray/Gray16, 非N开头的是预乘
- 模型(Model): 所有Color都对应一个Model，主要用于颜色类型转换
- 调色板(Palette): 包括 plan9/网页安全色

## 几个重要type
- Palette: 调色板，[]Color类型，实现了Model接口，但在pallete包中生成的两个调色板仍然使用[]Color，不知道为啥这么写
- Uniform: 无限大的均色图像，实现了color.Color, color.Model, Image三个接口，其实还实现了opaquer接口
- Config: 包含图片的像素model和尺寸

## 两处generate
image/color/palette和image/internal/imageutil两个包中，主要代码使用go:generate生成的，gen.go中使用**// +build ignore**来告诉编译器忽略编译

## image/image.go
每种Color的具体实现，实现了color.Image和draw.Image接口

每种Color的一些通用成员变量：
```go
// 图像的通用存储方法，将图像中的每个像素的每个分量按顺序存储到数组中，
//
// 图像的像素每个分量根据顺序存储在数组中
Pix []uint8
// 跨度，两像素垂直距离
Stride int
// 图片边界
Rect Rectangle
```
另外还有一些通用的方法：
```go
// {$Color}代表颜色类型，如RGBA/NRGBA等，还包括CMYK和Paletted
// 对应坐标的颜色
{$Color}At(x, y int) color.{$Color}
// 对应坐标在Pix列表中的索引值
PixOffset(x, y int) int
// 和Set方法一样的效果，只是输入的是具体颜色类型，不需要转换
Set{$Color}(x, y int, c color.{$Color})
// 子图像，从原图像中抠出来的一个相交区域
SubImage(r Rectangle) Image
// 图像是否透明
Opaque() bool
```

## image/format.go
定义了解码的基本框架

## image/geom.go
几何定义，包括点和矩形
```go
// Point
// 
// 返回 (x,y) 这种字符串
String() string
// 两个点向量加法
Add(q Point) Point
// 两个点向量减法
Sub(q Point) Point
// 该点的数乘
Mul(k int) Point
// 向量除法
Div(k int) Point
// 该点是否在r这个区域中
In(r Rectangle) bool
// 该点与r这个区域的模运算，主要是为了让该点落到r区域内
Mod(r Rectangle) Point
// 两个点是否是同一个点
Eq(q Point) bool

var ZP Point
// Zero-Point，已废弃，现用
Pt(X, Y int) Point
// 替代

// Rectangle: 实现了image接口
//
// 返回 (x1,y1)-(x2,y2) 这种字符串
String() string
// 矩形区域的宽
Dx() int
// 矩形区域的高
Dy() int
// 矩形区域的尺寸，点的xy代表宽高
Size() Point
// 返回矩形区域使用p平移后的结果
Add(p Point) Rectangle
// 返回矩形区域使用-p平移后的结果
Sub(p Point) Rectangle
// (TODO: 没看懂，只知道是内嵌资源)
Inset(n int) Rectangle
// 返回两个矩形的相交部分
Intersect(s Rectangle) Rectangle
// 返回可以将两个矩形框住的最小矩形
Union(s Rectangle) Rectangle
// 矩形面积是否为非正数
Empty() bool
// 两个矩形是否是同一个
Eq(s Rectangle) bool
// 两个矩形是否在重叠，相切不算
Overlaps(s Rectangle) bool
// s是否在r内部，可以内相切
In(s Rectangle) bool
// 返回标准矩形，即xy的min<max
Canon() Rectangle

var ZR Rectangle
// Zero-Rectangle，已废弃，现用
Rect(x0, y0, x1, y1 int) Rectangle
// 替代
```

## package: image/draw
主要实现了Floyd-Steinberg扩散抖动算法，用于gif

## package: image/jpeg
实现了两个算法：
- 离散余弦变换
- 反式离散余弦变换

主要部分：
- 解码器(reader.go)
- 编码器(writer.go)

## package: image/png
实现了一个算法：
- Paeth推断算法

主要部分：
- 解码器(reader.go)
- 编码器(writer.go)

## package: image/gif
使用了image/draw中的Floyd-Steinberg扩散抖动算法

主要部分：
- 解码器(reader.go)
- 编码器(writer.go)