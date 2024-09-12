+++
author = "Laurenfrost"
date = "2016-02-20T13:56:01-08:00"
meta = true
math = true
title = "Hugo-Tufte Features"
subtitle = "A Fancy Subtitle"
toc = true
categories = ["katex", "latex", "tufte-css"]
+++

This is a quick demonstration post.  It serves as an example of the features
of this theme. 同时还添加了大量中文/日语文本，既是作为测试，亦可看作参考。 One of them is $ \LaTeX $ via [Katex](https://katex.org/). 

## A Bit About Mathematics 来一点数学

{{< epigraph pre="Shawn O'Hare, " cite="Math is Fun" >}}
This is an example of an epigraph with some math
$ \mathbb N \subseteq \mathbb R $
to start the beginning of a section.
{{< /epigraph >}}

<!--more-->

### Inline 行内公式
Some inline math:
{{< marginnote "mn-example" >}}This is a margin note.{{< /marginnote >}}
$e^{i \pi} = -1$
 and $\sqrt{-1} = i $
and $ a_2 = 3 $.  
来点嵌入到文本内部的数学符号：
{{< marginnote "mn-example" >}}这是一个 margin 注释。{{< /marginnote >}}
$e^{i \pi} = -1$
 and $\sqrt{-1} = i $
and $ a_2 = 3 $.

### Display 展示公式
And display math using this symbol `$$`:
{{< sidenote "sn-example" >}}This is a sidenote!{{< /sidenote >}}  
还可以另起一行展示公式 `$$`：
{{< sidenote "sn-example" >}}这是个侧边注释 sidenote！{{< /sidenote >}}
$$
  -- \cdot_H -- \colon B(G,H) \times B(H, K) \to B(G, K), \quad ([X], [Y]) \mapsto [X \times_H Y].
$$

### Environments 环境

Currently, certain $\LaTeX$ environments need to be escaped so that
the markdown processor does not override Katex.  Currently, display
environments should be enclosed in `<p>` tags and blank lines.
For instance:

<p>
$$
\begin{aligned}  
  \mu(A) &= \iint_{I^2} \chi_A (x,y) \ d(x,y) 
  = \int_I \left( \int_I  \chi_A (x,y) \ dx\right) dy 
  = \int_I 0 \ dy= 0 \quad \text{and} \\  
  \mu(A) &=\iint_{I^2}  \chi_A (x,y) \ d(x,y) 
  = \int_I \left(  \int_I \chi_A (x,y) \ dy \right) dx 
  =\int_I dx = 1,
\end{aligned} 
$$
</p>
<!-- See https://github.com/jgm/pandoc/issues/3953#issuecomment-334670625 -->

is produced from

```txt
<p>
$$
\begin{aligned}  
  \mu(A) &= \iint_{I^2} \chi_A (x,y) \ d(x,y) 
  = \int_I \left( \int_I  \chi_A (x,y) \ dx\right) dy 
  = \int_I 0 \ dy= 0 \quad \text{and} \\  
  \mu(A) &=\iint_{I^2}  \chi_A (x,y) \ d(x,y) 
  = \int_I \left(  \int_I \chi_A (x,y) \ dy \right) dx 
  =\int_I dx = 1,
\end{aligned} 
$$
</p>
```

### Matrix 矩阵

<p>
$$
\begin{matrix}
\alpha& \beta^{*}\\
\gamma^{*}& \delta
\end{matrix}
$$
</p>

<p>
$$
\begin{bmatrix}
\alpha& \beta^{*}\\
\gamma^{*}& \delta
\end{bmatrix}
$$
</p>



<p>
$$
\begin{Vmatrix}
\alpha& \beta^{*}\\
\gamma^{*}& \delta
\end{Vmatrix}
$$
</p>

### Limits 极限

$$
\lim_{n \to \infty}
    \sum_{k=1}^n \frac{1}{k^2}
    = \frac{\pi^2}{6}
$$


$$
\lim_{n \to \infty}
     \frac{1}{x^n}
    = 0
$$

### Various symbols 各种不同的数学符号

  $$\lambda,\xi,\pi,\theta,
    \mu,\Phi,\Omega,\Delta$$

## Blockquotes 引用
Some blockquotes.  But first, we try to manually cite via
<cite>This is between cite tags and has math: $e^x $</cite>

{{< blockquote cite="www.shawnohare.com" footer="Shawn O'Hare" >}}
This is a blockquote with two paragraphs, that employs the
theme's `blockquote` shortcode. Lorem ipsum dolor sit amet,
consectetuer adipiscing elit. Aliquam hendrerit mi posuere lectus.
Vestibulum enim wisi, viverra nec, fringilla in, laoreet vitae, risus.

Donec sit amet nisl. Aliquam semper ipsum sit amet velit. Suspendisse
id sem consectetuer libero luctus adipiscing.
Vestibulum enim wisi, viverra nec, fringilla in, laoreet vitae, risus.
{{< /blockquote >}}

## New thoughts 新想法

<span class="newthought">Sometimes a new thought</span> distinguishes a section,
as here.  
There are currently two ways to create one.  One way is with raw HTML such as: 
`<span class="newthought">...</span>"`. The theme also provides the `newthought` 
shortcode.

<span class="newthought">有时候会有一些新想法</span>，这些文字需要与普通文本区别开来。  
这里有两种方式标记。一种是用纯 HTML 格式：`<span class="newthought">...</span>"`；另一
种则是使用 Hugo 的快捷代码 `newthought`。

## Code 代码块
As an example of some inline code: `go test -v -short`.
And this is some block-code:
```go {linenos=table,hl_lines=["2-5"],linenostart=199}
package main

import "log"

func add(x int, y int) int {
  log.Println("We are going to take the sum of two numbers, and leave a long comment.")
  return x + y
}

func main() {
  y := add(1, 2)
  log.Println(y)
}
```

Here's an example without line numbers. 
```go {hl_lines=["2-5"],linenostart=199}
package main

import "log"

func add(x int, y int) int {
  log.Println("We are going to take the sum of two numbers, and leave a very very very long comment.")
  return x + y
}

func main() {
  y := add(1, 2)
  log.Println(y)
}
```

## Figure 图表
Below we have an example of a regular width figure.
{{< figure
  src="https://github.com/edwardtufte/tufte-css/raw/gh-pages/img/exports-imports.png"
  class="class param"
  title="The image title."
  caption="This is the image caption."
  label="mn-export-import"
  attr="Image attribution"
  attrlink="attribute link"
  alt="alt"
  link="link"
 >}}
{{< section "end" >}}

{{< figure
  src="https://edwardtufte.github.io/tufte-css/img/rhino.png"
  class="class param"
  type="margin"
  label="mn-rhino"
  title="The image title."
  label="mn-export-import"
  caption="This is the image caption."
  attr="Image attribution"
  attrlink="https://edwardtufte.github.io/tufte-css"
  alt="alt"
  link="link"
 >}}
 But tight integration of graphics with text is central to Tufte’s work even when those graphics are ancillary to the main body of a text. In many of those cases, a margin figure may be most appropriate.
{{< section "end" >}}

Below is a full-width figure.
{{< figure
  src="https://edwardtufte.github.io/tufte-css/img/napoleons-march.png"
  type="full"
  label="mn-napoleonic-wars"
  title="Napoleonic wars"
  caption="This the image caption."
  attr="Image attribution"
  attrlink="attribute link"
  alt="Napoleonic wars"
  link="link"
 >}}
{{< section "end" >}}

## Table 表格

| Tables   |      Are      |  Cool |
|:----------|:-------------:|------:|
| col 1 is |  left-aligned | $1600 |
| col 2 is |    centered   |   $12 |
| col 3 is | right-aligned |    $1 |

## {{<ruby mark="</ruby>">}}Ruby{{</ruby>}} Annotation 文字注音

Another very important feature is a `</ruby>` mark. It supports not only latin but also CJK character sets.

```html {linenos=table, linenostart=1}
<!-- {{<ruby mark="Pronunciation">}}The Word{{</ruby>}} -->
<ruby mark="Pronunciation">The Word</ruby>
```

{{<ruby mark="zhōng wén">}}中文{{</ruby>}}可以注音，日本語の文字に{{<ruby mark="ふりがな">}}振り仮名{{</ruby>}}をつけることも可能である。

{{< div class="myclass" >}}

## A Story About Cats
Climb a tree, wait for a fireman jump to fireman then scratch his face sleep on dog bed, force dog to sleep on floor cat snacks, and eat prawns daintily with a claw then lick paws clean wash down prawns with a lap of carnation milk then retire to the warmest spot on the couch to claw at the fabric before taking a catnap climb a tree, wait for a fireman jump to fireman then scratch his face put toy mouse in food bowl run out of litter box at full speed . See owner, run in terror chase mice, so thinking longingly about tuna brine for eat a plant, kill a hand for wake up human for food at 4am. Human is washing you why halp oh the horror flee scratch hiss bite scratch the furniture and rub face on owner. Loves cheeseburgers see owner, run in terror chew on cable. Thug cat ignore the squirrels, you'll never catch them anyway. Eat a plant, kill a hand find empty spot in cupboard and sleep all day so hide head under blanket so no one can see yet love to play with owner's hair tie rub face on everything i like big cats and i can not lie. Wake up human for food at 4am stare at the wall, play with food and get confused by dust yet then cats take over the world scamper. Inspect anything brought into the house get video posted to internet for chasing red dot. Brown cats with pink ears chew foot spit up on light gray carpet instead of adjacent linoleum. I am the best wake up human for food at 4am, meow spread kitty litter all over house, for meow. Knock dish off table head butt cant eat out of my own dish jump off balcony, onto stranger's head, chase ball of string scream at teh bath but climb leg, so unwrap toilet paper but destroy couch. Climb a tree, wait for a fireman jump to fireman then scratch his face. Leave hair everywhere swat turds around the house eat grass, throw it back up, and eat grass, throw it back up. Chase after silly colored fish toys around the house.
{{< div "end" >}}

### We really like cats.

Yes, they are fluffy and happy.

## Test Text 测试文本 テストテキスト

### 唐代诗人孟浩然的《春晓》

{{< figure
  src="Meng_Haoran.jpg"
  class="class param"
  type="margin"
  label="mn-rhino"
  title="《孟浩然》"
  label="mn-export-import"
  attr="作者不明"
  alt="《孟浩然》"
 >}}
Meng Haoran (Chinese: 孟浩然; Wade–Giles: Meng Hao-jan; 689/691–740) was a major Tang dynasty poet, and a somewhat older contemporary of Wang Wei, Li Bai and Du Fu. Despite his brief pursuit of an official career, Meng Haoran mainly lived in and wrote about the area in which he was born and raised, in what is now Hubei province, China. Meng Haoran was a major influence on other contemporary and subsequent poets of the High Tang era because of his focus on nature as a main topic for poetry. Meng Haoran was also prominently featured in the Qing dynasty (and subsequently frequently republished) poetry anthology Three Hundred Tang Poems, having the fifth largest number of his poems included, for a total of fifteen, exceeded only by Du Fu, Li Bai, Wang Wei, and Li Shangyin. These poems of Meng Haoran were available in the English translations by Witter Bynner and Kiang Kanghu, by 1920, with the publication of The Jade Mountain. The Three Hundred Tang Poems also has two poems by Li Bai addressed to Meng Haoran, one in his praise and one written in farewell on the occasion of their parting company. Meng Haoran was also influential to Japanese poetry.

孟 浩然（もう こうねん/もう こうぜん、簡体字: 孟 浩然、拼音: Mèng Hàorán モン ハオラン、689年 - 740年）は、中国唐代（盛唐）の代表的な詩人。一説には、名は浩、字は浩然と言われる。襄州襄陽県（現在の湖北省襄陽市襄州区）の出身。若い頃からお金に節義を重んじ、人の患難を救うなどの行いがあった。

{{< section "end" >}}

《春晓》是唐代诗人孟浩然的诗作。此诗抓住诗人早晨刚醒时的一瞬间展开联想，描绘了一幅春天早晨绚丽的图景，抒发了诗人热爱春天、珍惜春光的美好心情。首句破题，写春睡的香甜，也流露着对朝阳明媚的喜爱；次句即景，写悦耳的春声，也交代了醒来的原因；三句转为写回忆；末句又回到眼前，由喜春翻为惜春。全诗语言平易浅近，自然天成，言浅意浓，景真情真，深得大自然的真趣。

In contemporary mainland China, Meng's poem Spring Morning (春曉) is probably one of the best known poems from the Tang dynasty, as it has appeared in the widely used first grade level Chinese language textbook published by the People's Education Press since the 1980s and serves as the first exposure to Literary Chinese for hundreds of millions of students.

{{< figure
  src="chunxiao.jpeg"
  class="class param"
  type="margin"
  label="mn-rhino"
  title="《春晓写意图》"
  label="mn-export-import"
  attr="作者不明"
  alt="《春晓写意图》"
 >}}
 {{< section "end" >}}

{{< epigraph pre="孟浩然" cite="《春晓》">}}
春眠不觉晓，  
处处闻啼鸟。  
夜来风雨声，  
花落知多少。  


{{<ruby mark="しゅんみん">}}春眠{{</ruby>}}　{{<ruby mark="あかつき">}}曉{{</ruby>}}を　{{<ruby mark="おぼ">}}覺{{</ruby>}}えず  
{{<ruby mark="しょしょ">}}處處{{</ruby>}}　{{<ruby mark="ていちょう">}}啼鳥{{</ruby>}}を　聞く  
{{<ruby mark="やらい">}}夜來{{</ruby>}}　{{<ruby mark="ふうう">}}風雨{{</ruby>}}の{{<ruby mark="こえ">}}聲{{</ruby>}}  
{{<ruby mark="はなお">}}花落{{</ruby>}}つること　{{<ruby mark="し">}}知{{</ruby>}}る　{{<ruby mark="たしょう">}}多少{{</ruby>}}  


In spring slumber, I am unaware of daybreak,  
Though everywhere I hear the tweet of birds.  
Last night came the sound of wind and rain;  
Who knows how many flowers must have fallen?  

{{< /epigraph >}}

1. 不觉晓：不知不觉天就亮了。晓：早晨，天明，天刚亮的时候。
2. 闻：听见。啼鸟：鸟啼，鸟的啼叫声。
3. “夜来”句：一作“欲知昨夜风”。
4. “花落”句：一作“花落无多少”。知多少：不知有多少。知：不知，表示推想。



### 整体赏析

《春晓》这首小诗，初读似觉平淡无奇，反复读之，便觉诗中别有天地。它的艺术魅力不在于华丽的辞藻，不在于奇绝的艺术手法，而在于它的韵味。整首诗的风格就像行云流水一样平易自然，然而悠远深厚，独臻妙境。千百年来，人们传诵它，探讨它，仿佛在这短短的四行诗里，蕴涵着开掘不完的艺术宝藏。

自然而无韵致，则流于浅薄；若无起伏，便失之平直。《春晓》既有悠美的韵致，行文又起伏跌宕，所以诗味醇永。诗人要表现他喜爱春天的感情，却又不说尽，不说透，“迎风户半开”，让读者去捉摸、去猜想，处处表现得隐秀曲折。

“情在词外曰隐，状溢目前曰秀。”{{< sidenote >}}张戒《岁寒堂诗话》引{{< /sidenote >}}写情，诗人选取了清晨睡起时刹那间的感情片段进行描写。这片段，正是诗人思想活动的启始阶段、萌芽阶段，是能够让人想象他感情发展的最富于生发性的顷刻。诗人抓住了这一刹那，却又并不铺展开去，他只是向读者透露出他的心迹，把读者引向他感情的轨道，就撒手不管了，剩下的，该由读者沿着诗人思维的方向去丰富和补充了。写景，他又只选取了春天的一个侧面。春天，有迷人的色彩，有醉人的芬芳，诗人都不去写。他只是从听觉角度着笔，写春之声：那处处啼鸟，那潇潇风雨。鸟声婉转，悦耳动听，是美的。加上“处处”二字，啁啾起落，远近应和，就更使人有置身山阴道上，应接不暇之感。春风春雨，纷纷洒洒，但在静谧的春夜，这沙沙声响却也让人想见那如烟似梦般的凄迷意境，和微雨后的众卉新姿。这些都只是诗人在室内的耳闻，然而这阵阵春声却逗露了无边春色，把读者引向了广阔的大自然，使读者自己去想象、去体味那莺啭花香的烂熳春光，这是用春声来渲染户外春意闹的美好景象。这些景物是活泼跳跃的，生机勃勃的。它写出了诗人的感受，表现了诗人内心的喜悦和对大自然的热爱。

宋人叶绍翁《游园不值》诗中的“春色满园关不住，一枝红杏出墙来”，是古今传诵的名句。其实，在写法上是与《春晓》有共同之处的。叶诗是通过视觉形象，由伸出墙外的一枝红杏，把人引入墙内、让人想象墙内；孟诗则是通过听觉形象，由阵阵春声把人引出屋外、让人想象屋外。只用淡淡的几笔，就写出了晴方好、雨亦奇的繁盛春意。两诗都表明，那盎然的春意，自是阻挡不住的，它已冲破了围墙屋壁，展现在眼前，萦回在耳际。

施补华曰：“诗犹文也，忌直贵曲。”{{< sidenote >}}《岘佣说诗》{{< /sidenote >}}这首小诗仅仅四行二十个字，写来却曲屈通幽，回环波折。首句破题，“春”字点明季节，写春眠的香甜。“不觉”是朦朦胧胧不知不觉。在这温暖的春夜中，诗人睡得真香，以至旭日临窗，才甜梦初醒。流露出诗人爱春的喜悦心情。次句写春景，春天早晨的鸟语。“处处”是指四面八方。鸟噪枝头，一派生机勃勃的景象。“闻啼鸟”即“闻鸟啼”，古诗为了押韵，词序作了适当的调整。三句转为写回忆，诗人追忆昨晚的潇潇春雨。末句又回到眼前，联想到春花被风吹雨打、落红遍地的景象，由喜春翻为惜春，诗人把爱春和惜春的情感寄托在对落花的叹息上。爱极而惜，惜春即是爱春──那潇潇春雨也引起了诗人对花木的担忧。时间的跳跃、阴晴的交替、感情的微妙变化，都很富有情趣，能给人带来无穷兴味。

《春晓》的语言平易浅近，自然天成，一点也看不出人工雕琢的痕迹。而言浅意浓，景真情真，就像是从诗人心灵深处流出的一股泉水，晶莹透澈，灌注着诗人的生命，跳动着诗人的脉搏。读之，如饮醇醪，不觉自醉。诗人情与境会，觅得大自然的真趣，大自然的神髓。“文章本天成，妙手偶得之”，这是最自然的诗篇，是天籁。

### いろは歌
現代に伝わるいろは歌の内容は、以下の通りである。

{{< epigraph pre="作者不明" cite="いろは歌">}}
いろはにほへと　ちりぬるを  
わかよたれそ　　つねならむ  
うゐのおくやま　けふこえて  
あさきゆめみし　ゑひもせす  

色は匂へど　散りぬるを  
我が世誰ぞ　常ならむ  
有為の奥山　今日越えて  
浅き夢見じ　酔ひもせず  
{{< /epigraph >}}

七五調の歌謡である今様の形式で、仮名を重複させることなく作られているが、これがいかなる内容を意味するのかは定かではない。古くから「いろは四十七文字」として知られるが、最後に「京」の字を加えて四十八字としたものも多く、現代では「ん」を加えることがある。四十七文字の最後に「京」の字を加えることは、弘安10年（1287年）成立の了尊の著『悉曇輪略図抄』に「末後に京の字有り」とあり、この当時すでに行われている。「京」の字が加えられた理由については、仮名文字の直音に対して「京」の字で拗音の発音を覚えさせるためだという説がある。いろは順には「京」を伴うのが広く受け入れられ、いろはかるたの最後においても「京の夢大坂の夢」となっている。

### 内容の解釈
いろは歌にある「うゐ」とは「有為」という仏教用語で、因縁によって起きる一切の{{<ruby mark="サンスカーラ">}}事物{{</ruby>}}。転じて「有為の奥山」とは、無常の現世を、どこまでも続く深山に喩えたものである。

いろは歌の内容については中世から現代にいたるまで、各種の解釈がなされてきたが、多くは「匂いたつような色の花も散ってしまう。この世で誰が不変でいられよう。いま現世を超越し、儚い夢をみたり、酔いに耽ったりすまい」と、仏教的な無常を述べたものと解釈されてきた。新義真言宗の祖である覚鑁（1095 - 1144年）はその著『{{<ruby mark="みつごんしょひしゃく">}}密厳諸秘釈{{</ruby>}}』の中で、いろは歌は『大般涅槃経』にある、以下の「諸行無常」と始まる{{<ruby mark="むじょうげ">}}無常偈{{</ruby>}}の意訳であると説明している。

{{< epigraph pre="作者不明" cite="パーリ仏典, 長部16, 大般涅槃経, Sri Lanka Tripitaka Project">}}
Aniccā vata saṅkhārā uppādavayadhammino,  
Uppajjitvā nirujjhanti tesaṃ vūpasamo sukho  

諸行無常　是生滅法  
生滅滅已　寂滅為楽  

諸行は無常なり、是れ生滅の法なり。  
生滅（へのとらわれを）滅しおわりぬ、寂滅をもって楽と為す。  
{{< /epigraph >}}

すなわち、

> 諸行無常 → 色は匂へど散りぬるを  
> 是生滅法 → 我が世誰ぞ常ならむ  
> 生滅滅已 → 有為の奥山今日こえて  
> 寂滅為楽 → 浅き夢見し酔ひもせず  

と、この四句の意をあらわしたものであるとした。

しかし語句の具体的な意味については諸説ある。前述の『悉曇輪略図抄』においては「いろは」は「色は」ではなく「色葉」であり、春の桜と秋の紅葉を指すとする。また清音か濁音かにより文の意味は異なるが、『悉曇輪略図抄』は「あさきゆめみし」の「し」は「じ」と濁音に読み、すなわち「夢見じ」という打消しの意とする。一方『密厳諸秘釈』はこの「し」を清音とするので、これは助動詞「き」の連体形「し」にあたる。17世紀の僧観応著の『{{<ruby mark="ぶもうき">}}補忘記{{</ruby>}}』では最後の「ず」以外すべて清音とするなど、この誦文は古文献においても清濁の表記が確定していない。「夢」や「酔ひ」が何を意味するかも多様な解釈があり、結局のところ内容や文脈についての確定した説明は、現時点でも存在しない。

小松英雄はこの無常偈といろは歌を結びつける解釈について、「かなりこじつけがましいが、さりとて積極的にそれを否定するだけの根拠があるわけでもない」としながらも、「ただし、こういう結び付けを前提として、さらにその上に論を立てることは危険なので、ひかえておいた方がよい」と述べている。