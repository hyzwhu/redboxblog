Hello，大家好，我们直接切入正题，今天来讨论一下如何使用red语言制作一款简单的推箱子游戏。

首先，我们来看下推箱子游戏。

![1.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/1.gif)

# 1 准备原材料及绘制地图

看完之后想必大家已经对这个游戏有个大致的了解了。

开始进入准备环节，俗话说得好，巧妇难为无米之炊。接下来的准备工作便是选好关卡地图的设置文件以及人物的动作图片，墙，地，目标位置，箱子的图片。

## 1.1 提取地图信息

![p1.png](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/p1.png)

如上图所示,提供的地图信息(包括关卡地图的信息，小人初始位置信息，关卡箱子初始位置信息)就在%map.gz(https://github.com/hyzwhu/redbox/blob/new1/map.gz)中。

那么如何将其提取出来呢？可以通过以下三个步骤：

1. 使用read/binary 方法将文件读取进来。

2. 使用decompress解压缩方法将该文件（因为这里提供的是.gz压缩文件）解压缩。

3. 使用load方法将decompress出来的二进制码转换成block类型，得到如下：

```
[
    make object! [						;--第一关地图信息
        start: 8x7 
        boxes: [7x6 9x6 7x7 8x8] 
        map: [
			#{00000000000000000000000000000000}
			#{00000000000000000000000000000000}
			#{00000000000000000000000000000000}
			#{00000000000001010100000000000000}
			#{00000000000001030100000000000000}
			#{00000000000001020101010100000000}
			#{00000000010101020202030100000000}
			#{00000000010302020201010100000000}
			#{00000000010101010201000000000000}
			#{00000000000000010301000000000000}
			#{00000000000000010101000000000000}
			#{00000000000000000000000000000000}
			#{00000000000000000000000000000000}
			#{00000000000000000000000000000000}
		]
    ]

	make object! [						;--第二关地图信息
		start: 5x3
		boxes: [6x4 7x4 6x5]
		map: [
			#{00000000000000000000000000000000} 
			#{00000000000000000000000000000000} 
			#{00000000010101010100000000000000} 
			#{00000000010202020100000000000000} 
			#{00000000010202020100010101000000} 
			#{00000000010202020100010301000000} 
			#{00000000010101020101010301000000} 
			#{00000000000101020202020301000000} 
			#{00000000000102020201020201000000} 
			#{00000000000102020201010101000000} 
			#{00000000000101010101000000000000} 
			#{00000000000000000000000000000000} 
			#{00000000000000000000000000000000} 
			#{00000000000000000000000000000000}
		]
]
    ...
]
```

以上是一个load之后的block片段，但是这个block只存在于形而不存在于神，需要使用reduce方法来执行这个block中的代码，使其成为真正的键对值形式的block。其实此时可以从上面的编码中隐隐约约看到地图的轮廓。

接下来分析一下这个对象中存储的属性，start即是小人的初始位置，boxes则是关卡中的箱子的初始位置，map则是这个关卡的地图设置，其中00是空，01，02，03分别代表了墙，地，目标，其实知道了这些之后读者也可以设计自己的地图，然后存储形式像如上block中的对象就可以了。

## 1.2 提取图片材料

![p2.png](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/p2.png)

以上all-image.png文件存储了所有的图片。

![p3.png](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/p3.png)

可以看到%all-image.png(https://github.com/hyzwhu/redbox/blob/new1/all-image.png)中存储了这个游戏中需要的所有的图片。

那么问题来了，为什么要这样做而不是将其分成一个一个的图片呢？这样在读取图片的时候岂不是方便很多？而这里将其压成了一张图片的原因，主要是为了减小空间。

将文件%all-image.png load进到all-image变量中。

那么此时问题来了，要如何从这个大图片中得到我们要用的一个一个的小图片呢？这里可以告诉读者的是图片的大小为30x30的。读者可以使用red.exe(https://www.red-lang.org/p/download.html)，敲入```help copy```，可以得到：

![p4.png](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/p4.png)

可以看到refinement中的part正是所需要的，尝试一下```copy/part all-image 30x30```，可以得到第一张图片。那么如何得到第二张图片？第三张甚至更多呢？只需要skip一个offset就行了。最后可以将这个抽取图片的功能集成成为一个方法：

```
extract: function [offset [integer!] size [pair!]][
		copy/part skip all-image offset size 
	]
```

每一次只要给定想要的图片在all-image中的偏移值和图片大小就可以得到想要的图片了。

## 1.3 绘制地图

好，接下来就开始制作游戏了，在做游戏的过程中应该在大脑中有个具体的思路（第一步做什么，接下来做什么，之后做什么），很明显，首先应该做的就是绘制地图。

可以观察到1.1中3的对象中所包含的类型为block的map，以每两个数字为一组，可以数出这是一个16x14大小的地图。那其中的01，02，03各代表什么呢？这里规定好的01为wall,02为floor,03为target。

此时写代码的思路就是，只要循环遍历map中的每一个坐标，然后得到其值进行判断得到相应的图片，并将图片写到大的背景图片中就可以了。

遍历整个map，使用for-pair方法(很容易读懂)：

```
for-pair: function [
		'word 	[word!]
		start 	[pair!] 
		end 	[pair!] 
		body 	[block!] 
		/local
		do-body
		val 
	][
		do-body: func reduce [word] body
		val: start 
		while [val/y <= end/y][
			val/x: start/x
			while [val/x <= end/x][
				do-body val 
				val/x: val/x + 1 
			]
			val/y: val/y + 1
		]
	]

```

循环地将小的图片（通过1.2中extrat得到的图片），这里指的主要是构成地图的墙，地板，目标图片。替换掉map-img图片相应坐标的图片，具体的思路是使用tile-type？方法对每次循环对应的坐标类型进行判断。

```
tile-type?: function [pos [pair!]][
		pos: pos + 1x1
		to-integer pick pick level-data/map pos/y pos/x
	]
```

之后使用change-image方法将选择到的小图片（或是墙或是地板或是目标）覆盖掉大图片（draw-map中的map-img）中相应坐标的地方。

实现思路现阶段是将小图片(30x30)的像素点通过循环“刻画”到大图片上的指定位置。

```
change-image: function [src [image!] dst [image!] pos [pair!]][
		sx: src/size/x
		dx: dst/size/x 
		sy: src/size/y
		px: pos/x
		py: pos/y	
		repeat y sy [
			xs: y - 1 * sx  + 1 
			xd: y + py - 1 * dx  + 1 + px 
			repeat l sx [
				dst/:xd: src/:xs
				xd: xd + 1
				xs: xs + 1
			] 
		]
	]
```

下图是绘制的第一百关的关卡地图：

![2.png](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/2.png)


# 2 小人

## 2.1 绘制小人

ok，通过上面的步骤已经可以看到地图已经出现了，那么此时可以加入小人了，从图片中抽取出小人的图片后将其显示在地图上。

因为小人的初始位置和地图相关，故而需要将小人的位置在每一次画地图操作时读取到出来以供之后移动小人操作的时候使用。

而这只是一个坐标，若是要想将小人显示在地图上还需要创建一个类型为window名为box-world的对象，并将map-img和mad-man作为box-world的儿子放入其中。

```
box-world: layout [
		at 0x16 bg: image map-img
		at 0x16 mad-man: base 30x30 l1
	]
```

之后
```view box-world```

即可。

这样就能在地图上看到小人了：

![3.png](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/3.png)

## 2.2 控制小人移动

小人成功上图，此时应该是建立小人和键盘之间的联系，即使用上下左右键控制小人的移动。此时应该设置好box-world中actors属性，如下代码所示：

```
box-world/actors: make object! [
    on-key-down: func [face [object!] event [event!]][
        switch event/key [
		up [turn 'up]
		down [turn 'down]
		left [turn 'left]
		right [turn 'right]
        	]
    	]
	]
```

现在已经实现了判断键盘中上下左右键，此时应该加入对应的turn方法，读者可以想象一下，小人在移动的时候是不是每一次都会偏移30（因为小人图片的大小就是30x30），当其往上移动的时候只要给小人的偏移量的纵坐标减去30就可以了（因为起始坐标为0x0，若是向上则纵向偏移量就少了30），以此类推可以得到小人向左，向右，向下的方法。

现在可以看到小人在地图上自由的穿梭了：

![4.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/4.gif)

## 2.3 限制小人移动

可以看到小人以及能够在图上自由的移动了，那么接下来应该思考小人在地图上的移动是不是有限制的呢？答案是肯定的，小人在遇到墙体的时候是不能移动的。

那如何来设置小人移动的逻辑呢，首先得到小人下一步的位置（即操作键盘后理论上小人的下一个坐标），之后将这个坐标进行类型判断，看看是否为地板或者目标，如果这个位置是地板或者目标，则小人可以向这个方向移动，如果这个位置是墙体，则小人不能向这个方向移动。

# 3	箱子

## 3.1 绘制箱子

好了，给小人加上限制之后，小人就被关在了地图里了。对的，这个游戏叫做推箱子游戏。所以当然要加上箱子了。

可以看到1.1的3中的对象中有着一个类型为block的boxes属性，里面存了几个坐标，这几个坐标就是箱子在关卡中的初始坐标。

那如何将箱子显示在地图上呢？可以看到之前写了一个box-world的window对象，并且将小人加入进box-world的pane中去，同样的，也可以通过这个方法将箱子加入到box-world的pane中，但是每一关的箱子个数都是动态的，所以这里可以使用循环的方法将箱子一个一个append到box-world的pane中。

## 3.2 箱子移动逻辑

加入箱子后，此时应该考虑一下箱子的逻辑，即小人推箱子时，若下一个点是墙体则推不动，若下一个地点是空地，则可以推动。

读者可以想象一下如何实现这个逻辑，首先可以拿到逻辑上操作键盘推动箱子后下一步的坐标位置，若这个位置是墙体，则不能进行移动；若是地板或者目标，则将箱子和小人的坐标按照键盘操作的方向进行移动，移动成功之后还不要忘了记得改变存储箱子的boxes中的箱子的位置坐标。

好了，现在箱子可以移动起来了：

![5.png](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/5.gif)

# 4 关卡胜利效果

## 4.1 关卡胜利逻辑判断

箱子可以推起来了！那么思考一下游戏胜利的方式便是将所有箱子移动到指定的目标点。所以此时加入箱子坐标与目标点重合时胜利的逻辑以及之后弹出关卡胜利窗口的功能。

逻辑很简单，只需要遍历boxes中的坐标和目标block中的坐标进行对比，若全部重合则可判断成功通关。

## 4.2 绘制关卡胜利提示窗口

好了，判断关卡胜利的逻辑已经写好了，想一下成功通关之后是不是需要有窗口提示，可以写一下窗口提示的代码，只要不是通关第100关都可以像下面这么写：

```
alert-win: layout [
		text center "you have done a good job" return 
		pad 30x0 button "ok" [
			unview 
		]
	]
```

在成功通关后调用以下来显示通关成功的窗口提示：

```view/flags alert-win 'modal```

以上view的refinement为flags设置为modal，其具体的功能是将该窗口设置成只有在关闭之后才能对其他的窗体进行操作，详细的可以查看red文档https://doc.red-lang.org/en/view.html#_window


## 4.3 关卡跳转效果

第一关已经制作完毕，想想当完成了一个关卡之后画面的跳转应该是什么样子的呢？应该是弹出4.2中一样的通关窗口的提示，并且在关闭这个窗口后能够自动跳转到下一个关卡对吧？想一想之前准备的关卡地图文件已经被读入了一个block里了，所以此时我们只需要将索引指向下一关卡（即level+1）使得level-data接收到下一关卡的数据即可重新绘制关卡地图。

```
level: level + 1
init-world
```

可以将以上代码加入到4.2中的alert-win中的button按钮中，即点击OK按钮后将跳转到下一个关卡。仔细想一下，在跳转到下一个关卡的时候有哪些参数需要初始化？

其中名为boxes的block,名为targets的block,以及box-world的子孙（这里box-world的前两个子孙(每个face的子孙存储在一个名为pane的block中)分别为map-img和mad-man,而之后便是每个地图中的box）都需要清除掉，即初始化为空的block（这里使用clear方法）。

然后使用像前面提到的步骤一样来重新绘制地图以及地图上的箱子即可，我们将其集成在了一个名init-world的方法中：

```
init-world: func[][
	system/view/auto-sync?: no
	clear boxes 
	clear targets 
	clear skip box-world/pane 2
	draw-map
	draw-boxes
	system/view/auto-sync?: yes 
	]
```

有心的朋友可能会发现这里使用了system/view/auto-sync?，而这是什么呢，在这里其主要的作用是将view的自动同步显示关掉（若不关掉的话，在绘制地图的时候就会将地图绘制的过程都显示在窗口中，效率非常的低，有兴趣的读者可以尝试一下），在绘制好地图以及箱子之后再将其开启（这个速度比不使用system/view/auto-sync?的速度提升了N倍，肉眼很难看清）。

具体关于system/view/auto-sync?可查看文档。https://doc.red-lang.org/en/view.html#_realtime_vs_deferred_updating_a_id_realtime_vs_deferred_updating_a

![6.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/6.gif)

# 5 附加功能

## 5.1 动态小人

此时已经完成了主体部分的工作了，也就是已经完成了游戏的基本功能：可以操作小人推动箱子，并且在所有箱子与目标点重合的时候弹出关卡胜利窗口，当点击"OK"按钮的时候则自动跳到下一关卡。

那么接下来的工作便是精装修啦！首先看到图片文件中小人的图片不止一个，也可以看到例子gif中小人的动作会随着时间的改变，方向的改变而切换。

看了一下all-image图片读者会发现每一个方向都会有两张图片，这两张图片的主要作用是随着时间切换，在视觉呈现上给玩家一种小人原地摆手的错觉。那么如何达到这样的效果呢？

可以使用一个block在每一次方向变换时在其中加入该指定方向的两张图片。完成这个操作只需要对方向选择的时候进行一些改造。

那么如何让小人能够原地动起来呢？答案如下：

```
mad-man: base 30x30 rate 10  on-time [
			judge: not judge
			mad-man/image: pick man-img judge
		]  
```

可以将box-world中的mad-man改变为以上形式。rate用于控制小人两个动作之间交换的频率，这里是10（当设置的越高，小人原地“动”的越快）。

完成之后，小人可以原地动起来了.

![7.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/7.gif)

## 5.2 关卡选择功能

小人活了！可能玩家在玩的时候会想，前面的关卡都太简单了，要来一些有难度的。所以是不是应该加入直选关卡的功能呢？

那么现在加入直选关卡的功能，首先应该在主窗口中加入一个名为goto的按钮，但是此时这个按钮还是没有任何的功能的。此时再加入level-choose的窗体，其中可以加入"please enter the level that you want"的提示，然后加入一个field来接收玩家输入的关卡数值，最后在名为"ok"的button出发的时候将关卡改为从field中接受的关卡数，再调用9中的init-world方法之后关闭这个窗口即可。

## 5.3 撤销功能

可能玩家会在玩的时候由于操作太快刹不了车了，多走出了一步怎么办？

不用着急，此时可以加上undo即撤销功能。看下功能如何实现，即每一步键盘操作小人的时候都将上一步的小人和箱子的坐标记录下来就可以了。像这样：

```		
undo-box: 0x0
undo-man: mad-man/offset
```

这里的思路应该是运用以上两个变量来分别存储小人上一次移动的位置和箱子上一次移动的位置。每一次小人移动的时候都运用undo-man来存储小人移动之前的位置。

只有在盒子有移动的时候才会赋值给```undo-box```。

加入undo和goto按钮及功能之后:

![9.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/9.gif)

## 5.4 关卡重玩功能

上面的撤销功能只能撤销一步，那么若是多走了n步怎么办呢？那么只能重新开始该关卡了。重新初始化该关卡游戏即可。在```box-world```中添加```retry```按钮，并且当按下这个按钮的时候调用9中的init-world方法就可以了。

## 5.5 游戏状态栏显示 

玩家在玩游戏时可能会想单纯是完成游戏太没有挑战性了！

注意到例子gif中的最下面的那一行状态栏：

1. 当前步数（这里指的是箱子移动的步数）。

1. 最佳步数（即移动的最少的步数,也是指箱子移动的步数）。

1. 当前关卡的等级数。

在```box-world```的pane中加入以下的text即可：

```
	style txt: text 85x20 black font-size 10 font-color white bold
	style num: text 15x20 black font-size 10 font-color white bold 
	at 0x420  txt "your move: "
	at 85x420 move-txt: num "0"
	at 100x420 txt "   best move: "
	at 185x420 best-move-txt: num "0"
	at 200x420 txt "   your level: "
	at 285x420 level-txt: num "1"
```

加入以上状态栏之后，还需要在move-txt，best-move-txt,level-txt需要改变时对其值进行变换。

1. move-txt 只需要在箱子移动的时候进行值的改变（这里指+1）就可以了。

2. best-move-txt 则采用了一个文件%moves.ini来记录其最好成绩的记录，在每一次开始游戏的时候将%moves.ini文件读入，并在每一次关卡结束时候使用如下方法进行对比，若是比之前的成绩要好则将本次成绩替换掉原来的最佳成绩并写入到%moves.ini文件中。

![10.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/10.gif)

看看上面gif中最下面的状态栏即是实现后的结果。

# 6 游戏制作完成

好了，完成了上面的步骤后便成功的使用red语言制作了一款简单的推箱子游戏了，游戏效果如下。

![11.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/11.gif)

观察到上面的gif中，箱子与目标点重合的时候会变成红色，那它是怎么实现的呢？有兴趣的朋友可以看一下源码https://github.com/hyzwhu/redbox/tree/new1或者自己实现（在%all-img.png文件中我们提供了红色箱子的图片）。