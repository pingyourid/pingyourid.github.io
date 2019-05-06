---
layout: post
title: ios自动布局以及对Masonry的扩展
catalog:    true
---

苹果的Autolayout是做适配的好东西，但是api比较难以使用，幸好还有一些优秀的第三方库帮我们封装了一层。这篇文章主要扩展了下Masonry:[masonry的扩展](https://github.com/pingyourid/MasonryHelper)

我用过的是[Purelayout](https://github.com/smileyborg/PureLayout.git)和[Masonry](https://github.com/SnapKit/Masonry.git)。

##### 先用的是PureLayout

- 优点：

对autolayout封装的比较浅，代码看起来跟oc风格比较像；
 
有在某一方向均匀分布view的功能，可以按照固定间隙或固定view长度2种方式可以选。

- 缺点：

api太多，即使有代码自动提示，使用起来还是很不方便。

##### 后用的是Masonry

- 优点：

api比较少，基本上就3个，很容易记住；

有刷新或者重绘约束的功能，就不用记住哪些约束是第一次进来要加上的，哪些是经常要刷新的；

链式语法语意更明显。
 
- 缺点：

没有均匀分布的功能！需要调用的地方写for循环；

链式语法刚开始看的时候会觉得和oc语法差异太大。

##### 改进

针对Masonry没有均匀分布这个问题，我写了一个封装，代码见[demo](https://github.com/pingyourid/MasonryHelper)。

- 固定view宽度

<img src="/img/in-post/post-masonry-helper/QQ20150730-1@2x.png" alt="" title="" width="200" /> <img src="/img/in-post/post-masonry-helper/QQ20150730-2@2x.png" alt="" title="" width="200" />

- 固定间隙

<img src="/img/in-post/post-masonry-helper/QQ20150730-3@2x.png" alt="" title="" width="200" /> <img src="/img/in-post/post-masonry-helper/QQ20150730-4@2x.png" alt="" title="" width="200" />

- 针对固定宽度空格均匀分布

```javascript
MAS_VIEW *prev;
        for (int i = 0; i < self.count; i++) {
            MAS_VIEW *v = [self objectAtIndex:i];
            [v mas_makeConstraints:^(MASConstraintMaker *make) {
                if (prev) {
                    make.width.equalTo(prev);
                    make.left.equalTo(prev.mas_right).offset(paddingSpace);
                    if (i == (CGFloat)self.count - 1) {//last one
                        make.right.equalTo(tempSuperView).offset(-tailSpacing);
                    }
                }
                else {//first one
                    make.left.equalTo(tempSuperView).offset(leadSpacing);
                }
                
            }];
            prev = v;
        }

```

比如水平方向，只要控制去头去尾，view宽度相等，view之间间距是传入的固定间距，就完成了。

- 针对固定view宽度均匀分布。

```javascript
MAS_VIEW *prev;
        for (int i = 0; i < self.count; i++) {
            MAS_VIEW *v = [self objectAtIndex:i];
            [v mas_makeConstraints:^(MASConstraintMaker *make) {
                if (prev) {
                    CGFloat offset = (1-(i/((CGFloat)self.count-1)))*(itemLength+leadSpacing)-i*tailSpacing/(((CGFloat)self.count-1));
                    make.width.equalTo(@(itemLength));
                    if (i == (CGFloat)self.count - 1) {//last one
                        make.right.equalTo(tempSuperView).offset(-tailSpacing);
                    }
                    else {
                        make.right.equalTo(tempSuperView).multipliedBy(i/((CGFloat)self.count-1)).with.offset(offset);
                    }
                }
                else {//first one
                    make.left.equalTo(tempSuperView).offset(leadSpacing);
                    make.width.equalTo(@(itemLength));
                }
            }];
            prev = v;
        }
```

理论上也应该是去头去尾，间隙相等，view宽度固定。问题是，autolayout中并没有任何api可以表示间隙在没有定值的情况下相等。但是有基础api中可以让一个view的属性＝另一个view的属性*倍数+offset。

这里就考虑计算小view的right属性＝父view的right属性*倍数+offset。于是建模，列方程组，推导如图：

<img src="/img/in-post/post-masonry-helper/QQ20150901-1@2x.png" alt="" title="" width="200" /> <img src="/img/in-post/post-masonry-helper/QQ20150901-2@2x.png" alt="" title="" width="200" />

---

##### 结尾

没改进之前我甚至考虑过2个一起用，但是考虑到Masonry有些重建约束之类的api没法覆盖到通过Purelayout创建的约束，所以扩展了masonry，现在我可以开开心心地用Masonry了～～