hello，大家好，我们直接切入正题，今天来讨论一下如何使用red语言制作一款简单的推箱子游戏。

首先，我们来看下推箱子游戏。

![1.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/1.gif)

# 1
看完之后想必大家以及对这个游戏有个大致的了解了。那么好的，我们开始进入准备环节，俗话说得好，巧妇难为无米之炊。我们接下来的准备工作便是选好关卡地图的设置文件以及人物的动作图片，墙，地，目标位置，箱子的图片。

```red
load-bin: func [file][reduce load decompress read/binary file]
maps: load-bin %data1.txt.gz'
```
## 1.1
ok,我们先来看下如上的代码实现了什么功能。load-bin方法的主要步骤：

1. 使用read/binary 方法将文件读取到red中来。

2. 使用decompress解压缩方法将该文件（这里提供的是.gz压缩文件）解压缩。

3. 使用load方法将decompress出来的二进制码转换成block：

```red
[
    make object! [
        start: 8x7 
        boxes: [7x6 9x6 7x7 8x8] 
        map: [
            "^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@" "^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@" "^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@" "^@^@^@^@^@^@^A^A^A^@^@^@^@^@^@^@" "^@^@^@^@^@^@^A^C^A^@^@^@^@^@^@^@" "^@^@^@^@^@^@^A^B^A^A^A^A^@^@^@^@" "^@^@^@^@^A^A^A^B^B^B^C^A^@^@^@^@" "^@^@^@^@^A^C^B^B^B^A^A^A^@^@^@^@" "^@^@^@^@^A^A^A^A^B^A^@^@^@^@^@^@" "^@^@^@^@^@^@^@^A^C^A^@^@^@^@^@^@" "^@^@^@^@^@^@^@^A^A^A^@^@^@^@^@^@" "^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@" "^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@" "^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@"]
    ]
    ...
]
```
4. 以上是一个load之后的block片段，但是这个block只存在于形而不存在于神，需要使用reduce方法来执行这个block中的代码，使其成为真正的键对值形式的block。（其实此时你可以从上面的代码隐隐约约看到地图的轮廓）

## 1.2
```
all-image: load %all-image.png
extrat: function [offset [integer!] size [pair!]][
		copy/part skip all-image offset size 
	]
	l1: extrat 0 30x30 
    ...
```
以上通过对整张png图片进行裁剪以得到相应的图片。这里l1得到了小人向左方向时候的第一个动作图片。


# 2 
好，接下来我们就开始制作游戏了，在做游戏的过程中我们应该在大脑中有个具体的思路（第一步做什么，接下来做什么，之后做什么），很明显，我们首先应该做的就是绘制地图。
```
draw-map: has [tile lx ly][
		map-img/rgb: black
		level-data: maps/:level
		level-txt/data: :level
		best-move-txt/data: pick moves-file :level
		for-pair pos 0x0 15x13 [
			tile: 0
			unless zero? tile: tile-type? pos [
				if 3 = tile [
					append targets pos * 30 + 0x20 
				]
				tile: decode-tile tile
				change-image tile map-img pos * 30  
			]
		]
	]
```
以上draw-map方法，主要思路是通过读取从第一步骤中得到的地图信息中具体的关卡信息存在level-data中，再初始化一块背景图片（这里是黑色的），使用for-pair方法
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
之后使用change-image方法将我们选择到的小图片（或是墙或是地板或是目标）覆盖掉大图片（draw-map中的map-img）中具体坐标的地方（实现思路现阶段是将小图片(30x30)的像素点通过循环“刻画”到大图片上）。
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

![2.png](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/2.png)


# 3 
ok，通过上面的步骤我们已经可以看到地图已经出现了，那么我们此时可以加入小人了，从图片中抽取出小人的编码后将其显示在地图上。(因为小人的初始位置和地图相关，故而我们可以将以下位置添加到上面的2中的draw-map方法中)。	
```
man-pos: undo-man: mad-man/offset: level-data/start * 30 + 0x20
```
而这只是一个坐标，我们要想将小人显示在地图上还需要创建一个类型为window名为box-world的对象，并将map-img和mad-man作为box-world的儿子放入其中。这样我们就能在地图上看到小人了：

![3.png](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/3.png)

# 4 
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
现在我们已经实现了判断键盘中上下左右键，此时应该加入对应的turn方法：
```
turn: function [value [word!] /local box c-pos b-pos][
		c-pos: mad-man/offset + dir-to-pos value
		if can-move? value [
			;--some function
		]
	]
dir-to-pos: func [value [word!]][
		select [up 0x-30 down 0x30 left -30x0 right 30x0] value
	]
```
以上turn方法中预留了小人移动逻辑的代码（如5中所提到的）。
现在我们可以看到小人在地图上自由的穿梭了：

![4.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/4.gif)
# 5
可以看到小人以及能够在图上自由的移动了，那么接下来我们应该思考小人在地图上的移动是不是有限制的呢？答案是肯定的，小人在遇到墙体的时候是不能移动的。
```
can-move?: func [value [word!] pos [pair!]/local new1 new nx ny][
		new1: pos - 0x16 + dir-to-pos value 
		nx: new1/x / 30
		ny: new1/y / 30
		new1: as-pair nx ny
		new: tile-type? new1 
		find [2 3] new 
	]
```
以上代码是首先得到小人下一步的位置（即操作键盘后理论上小人的下一个坐标），之后将这个坐标进行类型判断，看看是否为地板或者目标，若是则can-move?方法返回true则表示小人可以向该方向进行移动。
```
if can-move? value mad-man/offset [
				mad-man/offset: mad-man/offset + dir-to-pos value
			]
```

# 6
好了，给小人加上限制之后，小人就被关在了地图里了。对，我们这个游戏叫做推箱子游戏。所以当然要加上箱子了。
```
draw-boxes: has [bx pos pb][
		foreach pos level-data/boxes [
			pb: p1-to-p2 pos
			append box-world/pane bx: make face![type: 'base size: 30x30 offset: pb image: box]
			append boxes pb
		]
	]
```
从level-data中我们可以得到每一张地图中的boxes的坐标，之后我们将坐标转换一下，因为文件中存储方式为30x30大小的图片为一个坐标点的形式，而我们的图片中的逻辑判断是采用每一张图片左上角像素点的坐标来进行的（并且伴随有偏移）。而每一次循环的坐标我们将其存入一个block中以供之后使用。
可使用如下方法进行坐标转换：
```
p1-to-p2: function [pos [pair!] /local yb xb pb][
		xb: pos/x * 30 
		yb: pos/y * 30 + 16 
		pb: as-pair xb yb
		pb 
	]
```

# 7
加入箱子后，我们此时应该想一下箱子的逻辑，即小人推箱子时，若下一个点是墙体则推不动，若下一个地点是空地，则可以推动。
```
b-pos: find boxes c-pos
bp: index? b-pos 
pb: bp + 2
next-box: c-pos + dir-to-pos value 
if all [can-move? value c-pos  next-is-box? next-box][
	box-world/pane/:pb/offset: next-box
	poke boxes bp box-world/pane/:pb/offset
	mad-man/offset: c-pos
] 

next-is-box?: func[pos [pair!]][
	either find boxes pos [return false][return true]
]
```
从上面代码我们可以看出，首先拿到逻辑上操作键盘推动箱子后下一步的坐标位置，若这个位置是墙体，则不能进行移动；若是地板或者目标，则将箱子和小人的坐标按照键盘操作的方向进行移动，移动成功之后我们还不要忘了记得改变存储箱子的boxes中的箱子的位置坐标。好了，现在箱子可以移动起来了：

![5.png](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/5.gif)
# 8
箱子可以推起来了！那么思考一下游戏胜利的方式便是将所有箱子移动到指定的目标点。所以此时加入箱子坐标与目标点重合时胜利的逻辑以及之后弹出关卡胜利窗口的功能。
```
check-win?: has [win? box a][
		win?: yes 
		foreach box boxes [
			a: either find targets box [true][false]
			win?: win? and a ]
		win? 
	]
```
逻辑很简单，只需要将目标block中的坐标和boxes中的坐标进行对比，若全部重合check-win?方法将返回true。
好了，判断关卡胜利的逻辑已经写好了，此时我们应该写一下弹出窗口了：
```
alert-win: layout [
		text center "you have done a good job" return 
		pad 30x0 button "ok" [
			unview 
		]
	]
```
以上两步都准备好之后想一下应该何时使用check-win?方法才比较好呢？应该是每一次移动箱子都判断一次。则我们将以下方法加入到7中的移动箱子的操作中:
```
if check-win? [
					view/flags alert-win 'modal
				]
```
以上view的refinement为flags设置为modal，其具体的功能是将该窗口设置成只有在关闭之后才能对其他的窗体进行操作，详细的可以查看red文档https://doc.red-lang.org/en/view.html#_window


# 9
好，第一关已经制作完毕，那么接下来我们应该加入更多的图片，想一想我们之前准备的关卡地图文件已经被读入了一个block里了，所以此时我们只需要将索引指向下一关卡（即level+1）使得level-data接收到下一关卡的数据即可重新绘制关卡地图。
```
level: level + 1
init-world
```
我们可以将以上代码加入到8中的alert-win中的button按钮中，即点击OK按钮后将跳转到下一个关卡。仔细想一下，我们在跳转到下一个关卡的时候有哪些参数需要初始化？boxes
,targets,以及box-world的子孙（这里box-world的前两个子孙(每个face的子孙存储在一个名为pane的block中)分别为map-img和mad-man,而之后便是每个地图中的box）都需要重新清除掉（这里使用clear方法），然后使用draw-map和draw-boxes方法重新绘制地图以及地图上的箱子。具体步骤如下:
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
有心的朋友可能会发现我们这里使用了system/view/auto-sync?，而这是什么呢，在这里其主要的作用是将view的自动同步显示关掉（若不关掉的话，我们在绘制地图的时候就会将地图绘制的过程都显示在窗口中，效率非常的低，有兴趣的朋友可以尝试一下），在绘制好地图以及箱子之后再将其开启（这个速度比不使用system/view/auto-sync?的速度提升了N倍，肉眼很难看清）。
具体关于system/view/auto-sync?可查看文档。https://doc.red-lang.org/en/view.html#_realtime_vs_deferred_updating_a_id_realtime_vs_deferred_updating_a

![6.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/6.gif)

# 10
此时已经完成了大部分的工作了，你可以操作小人推动箱子，并且在所有箱子与目标点重合的时候弹出关卡胜利窗口，当你点击确定的时候则自动跳到下一关卡。那么接下来的工作便是精装修啦！首先看到图片文件中小人的图片不止一个，也可以看到例子gif中小人的动作会随着时间的改变，方向的改变而切换。
看了一下图片你会发现每一个方向都会有两张图片，所以我们可以使用一个block在每一次方向变换时在其中加入该指定方向的图片。只需要对方向选择的时候进行一些改造：
```
switch event/key [
		up [poke man-img 1 u1 poke man-img 2 u2 turn 'up ]
		down [poke man-img 1 d1 poke man-img 2 d2 turn 'down]
		left [poke man-img 1 l1 poke man-img 2 l2 turn 'left]
		right [poke man-img 1 r1 poke man-img 2 r2 turn 'right]
        ]
```
其中```poke man-img 1 u1```可以直接使用```man-img/1: u1```替代。
那如何让小人能够动起来呢？答案如下：
```
mad-man: base 30x30 rate 10  on-time [
			judge: not judge
			mad-man/image: pick man-img judge
		]  
```
我们可以将box-world中的mad-man改变为以上形式。(详细)
好了，现在我们的小人可以动起来了。噢，此时你可能发现我们的地图背景颜色变成黑色了，没事，这只是对
map-img进行了一个小小的设置:
```
map-img/rgb: black
```
![7.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/7.gif)

# 11 
小人活了！可能你在玩的时候会想，前面的关卡都太简单了，要来一些有难度的。所以是不是应该加入直选关卡的功能呢？好！现在加入直选关卡的功能:
```
level-choose: layout [
		text "please enter the level that you what" return
		pad 30x0 fld: field 40x20 return 
		button "ok" [
			level: to-integer fld/text
			init-world
			unview]
	] 
```
当level-choose窗口弹出的之后，你需要在按下“ok”按钮时完成level的切换以及地图的初始化工作。
当然，我们还需要在主窗口中加入goto按钮不是嘛.
```
box-world: layout/tight [
	at 0x0 button "goto" bold 30x16 [view level-choose ]
]
```
# 12 
可能你会在玩的时候由于操作太快刹不了车了，多走出了一步怎么办？没关系我们可以加上undo即撤销功能。看下功能如何实现，即每一步键盘操作小人的时候都将上一步的小人和箱子的坐标记录下来。我们这里采用了分开记录的方法。
将
```		
undo-box: 0x0
undo-man: mad-man/offset
```
加入到turn方法中，只要一执行turn方法，则首先对以上两个变量进行初始化。
```undo-box```只有在盒子有移动的时候才会赋值给它。
加入undo和goto按钮及功能之后：
![9.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/9.gif)

# 13
上面的撤销功能只能撤销一步，那么若是多走了n步怎么办呢？那么我们只能重新开始该关卡了。重新初始化该关卡游戏即可。在```box-world```中添加：
```
at 60x0 button "retry" bold 30x16 [init-world]
```
搞定。
单纯是完成游戏太没有挑战性了！于是乎我们注意到了例子gif中的最下面的那一行状态栏，加入当前步数（这里指的是箱子移动的步数）。加入最佳步数（即移动的最少的步数,也是指箱子移动的步数）。加入当前关卡的等级数。
我们在```box-world```中加入：
```
at 0x405  text 70x30 black font-size 10 font-color white bold "your move:"
at 70x405 move-txt: text 15x30 black font-size 10 font-color white bold "0"
at 85x405 text 70x30 black font-size 10 font-color white bold "best move:"
at 150x405 best-move-txt: text 15x30 black font-size 10 font-color white bold "0"
at 165x405 text 70x30 black font-size 10 font-color white bold "your level:"
at 230x405 level-txt: text 15x30 black font-size 10 font-color white bold "1"
```
(以上统一的text属性代码我们会在后期优化的时候采用style替代)
加入以上状态栏之后，我们还需要在move-txt，best-move-txt,level-txt需要改变时对其进行值变换。
1. move-txt 只需要在箱子移动的时候加入就可以了。
2. best-move-txt 我们则采用了一个文件%moves.ini来记录其最好成绩的记录，在每一次开始游戏的时候将%moves.ini文件读入，并在每一次关卡结束时候使用如下方法进行对比，若是比之前的成绩要好则将本次成绩替换掉原来的最佳成绩并写入到%moves.ini文件中。
```
is-best?: func [/local bt mt][
	mt: move-txt/data 
	bt: best-move-txt/data
	either bt = 0 [
		poke moves-file level mt
	][
		if bt > mt [
			poke moves-file level mt 
		]
	]
	write %moves.ini mold moves-file
	]
```
3. level-txt 这个非常简单，只需要在```init-world```或者```draw-map```的时候在其中加入```level-txt/data: :level```即可。
![10.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/10.gif)
看看我们上面gif中最下面的状态栏即是实现后的结果。
# 14 
好了，完成了上面的步骤后我们便成功的使用red语言制作了一款简单的推箱子游戏了，游戏效果如下。

![11.gif](https://raw.githubusercontent.com/hyzwhu/redboxblog/master/image/11.gif)

可以看到当我们的箱子与目标点重合的时候会变成红色，那它是怎么实现的呢？有兴趣的朋友可以看一下源码(https://github.com/hyzwhu/redbox/tree/new1)或者自己实现（在%all-img.png文件中我们提供了红色箱子的图片）。
