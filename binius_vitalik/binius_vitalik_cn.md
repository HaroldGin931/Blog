# Binius: highly efficient proofs over binary fields

这篇文章主要面向大致熟悉 2019 年密码学的读者，尤其是 SNARK 和 STARK。如果你不是，我建议你先阅读这些文章。特别感谢 Justin Drake、Jim Posen、Benjamin Diamond 和 Radi Cojbasic 的反馈和评论。

> This post is primarily intended for readers roughly familiar with 2019-era cryptography, especially SNARKs and STARKs. If you are not, I recommend reading those articles first. Special thanks to Justin Drake, Jim Posen, Benjamin Diamond and Radi Cojbasic for feedback and review.

在过去的两年中，STARK已经成为一种关键且不可替代的技术，可以有效地生成对**复杂语句的易验证加密证明**（例如，证明以太坊区块是有效的）。其中一个关键原因是域的大小：为了确保安全性，基于椭圆曲线的 SNARK 需要工作在 256 位整数上，而 STARK 可以使用更小的域，并获得更高的效率：首先是 Goldilocks field（64 位整数），然后是 Mersenne31 和 BabyBear（均为 31 位）。由于这些效率的提高，使用Goldilocks的Plonky2在证明多种计算方面比其前辈快数百倍。

> Over the past two years, [STARKs](https://vitalik.eth.limo/general/2018/07/21/starks_part_3.html) have become a crucial and irreplaceable technology for efficiently making [easy-to-verify cryptographic proofs of very complicated statements](https://vitalik.eth.limo/general/2021/01/26/snarks.html) (eg. proving that an Ethereum block is valid). A key reason why is *small field sizes*: whereas elliptic curve-based SNARKs require you to work over 256-bit integers in order to be secure enough, STARKs let you use much smaller field sizes, which are more efficient: first [the Goldilocks field](https://polygon.technology/blog/plonky2-a-deep-dive) (64-bit integers), and then [Mersenne31 and BabyBear](https://blog.icme.io/small-fields-for-zero-knowledge/) (both 31-bit). Thanks to these efficiency gains, Plonky2, which uses Goldilocks, is [hundreds of times faster](https://polygon.technology/blog/introducing-plonky2) at proving many kinds of computation than its predecessors.

自然而然的：在这种域越来越小的趋势下，我们能否通过直接操作 0 和 1 来构建一个更快的证明系统？这正是 Binius 试图做的事情，Binius 使用了许多数学技巧，使其与三年前的 SNARK 和 STARK 截然不同。这篇文章介绍了为什么小域（Small fields）使证明生成更有效率，为什么二进制字段具有独特的强大功能，以及 Binius 用来使二进制字段上的证明如此有效的技巧。

> A natural question to ask is: can we take this trend to its logical conclusion, building proof systems that run even faster by operating directly over zeroes and ones? This is exactly what [Binius](https://eprint.iacr.org/2023/1784.pdf) is trying to do, using a number of mathematical tricks that make it *very* different from the [SNARKs](https://vitalik.eth.limo/general/2019/09/22/plonk.html) and [STARKs](https://vitalik.eth.limo/general/2018/07/21/starks_part_3.html) of three years ago. This post goes through the reasons why small fields make proof generation more efficient, why binary fields are uniquely powerful, and the tricks that Binius uses to make proofs over binary fields work so effectively.

![Overview](https://vitalik.eth.limo/images/binius/binius.drawio.png)

## Table of contents
- [ ] 生成目录 `Markdown All in One: Create Table of Contents`

## Recap: finite fields

加密证明系统的关键任务之一是对大量数据进行操作，同时让生成的结果足够的小（存储意义上的足够小）。如果把一个大程序的语句“压缩”成一个包含几个数字的数学方程式，但这些数字和原来的程序一样大，毫无疑问，这样的操作是没有意义的。

> One of the key tasks of a cryptographic proving system is to operate over huge amounts of data, while keeping the numbers small. If you can compress a statement about a large program into a mathematical equation involving a few numbers, but those numbers are as big as the original program, you have not gained anything.

在进行复杂算术运算的同时确保数字维持在一个很小的范围内，密码学家通常使用模运算。简单来说， 我们选择一些素数$p$。$\%$ 运算符表示“取余数”：$15 \% 7=1, 53 \% 10 = 3$，依此类推（请注意，取模的结果始终是非负的，例如$-1 \% 10 = 9$）。

> To do complicated arithmetic while keeping numbers small, cryptographers generally use **modular arithmetic**. We pick some prime "modulus" $p$. The $\%$ operator means "take the remainder of" $15 \% 7=1, 53 \% 10 = 3$, etc (note that the answer is always non-negative, so for example $-1 \% 10 = 9$).

为了在保持小数字的同时进行复杂的算术运算，密码学家通常使用模算术（取模运算）。我们选择一些素数p。% 运算符表示“取余数”：15 % 7=1,53 % 10=3，依此类推（请注意，取模的结果始终是非负的，例如 −1 % 10=9）。

![Overview](https://vitalik.eth.limo/images/binius/clock.png)

*时钟就是一个模运算的例子，例如，9：00 之后的四个小时是什么时间？但在实际应用中，我们不只进行模加减运算，我们还可以进行进行乘法、除法和指数运算。*

> You've probably already seen modular arithmetic, in the context of adding and subtracting time (eg. what time is four hours after 9:00?). But here, we don't just add and subtract modulo some number, we also multiply, divide and take exponents.

我们重新定义如下的计算：

> We redefine:

$$
\begin{align*}
x + y =>& (x + y) \% p \\
x \times y =>& (x \times y) \% p \\
x^y =>& (x^y) \% p \\
x - y =>& (x - y) \% p \\
x/y =>& (x \times y^{p-2}) \% p
\end{align*}
$$

上面的规则都是自洽的， 例如，我们取 $p=7$，则：
- $5 + 3 = 1$ (because $8 \% 7 = 1$)
- $1 - 3 = 5$ (because $-2 \% 7 = 5$)
- $2 \times 5 = 3$ (because $10 \% 7 = 3$)
- $3/5 = 2$ (because $(3 \times 5^5) \% 7 = 9375 \%7 = 2$)

这种结构的一个更一般的术语是有限域。有限域是一种数学结构，它遵循上述的算术法则，但是可能的计算结果是有限的，因此每一种可能的输出都可以用固定的长度来表示。

> A more general term for this kind of structure is a **finite field**. A [finite field](https://en.wikipedia.org/wiki/Finite_field) is a mathematical structure that obeys the usual laws of arithmetic, but where there's a limited number of possible values, and so each value can be represented in a fixed size.
>
> 补充：有限域是一种数学结构，其包含了有限个元素，域中的元素能够进行加减乘除的运算，且计算后的结果仍是属于这个有限域，因此有限域中的每个元素都可以用固定长度表示。有限域最常见的例子是 modulus $p$ 为素数。我们依然用 modulus = 7 举例，有限域 $F_7$ 包含的元素有 $\{0, 1, 2, 3, 4, 5, 6\}$, 这个域中的每一个元素都是整数，在这个域中任取两个元素进行计算后的结果仍然属于这个域。

模算术（或素数域）是最常见的有限域类型，但也有另一种类型：扩域。您之前可能已经看过一个扩域：复数。我们“想象”一个新元素，我们用 $i$ 表示 ，并声明它满足 $i^2=−1$。然后，您可以使用任意的实数和 $i$ 做线性组合 ，并用它做数学运算： $(3i+2)*(2i+4)=6i^2+12i+4i+8=16i+2$ 。我们同样可以对素数域进行扩域。当我们开始处理更小的字段时，素数域的扩域对于保持安全性变得越来越重要，而 Binius 使用的二进制域及其扩域具有非常好的实用性。

> Modular arithmetic (or **prime fields**) is the most common type of finite field, but there is also another type: **extension fields**. You've probably already seen an extension field before: the complex numbers. We "imagine" a new element, which we label $i$, and declare that it satisfies $i^2=−1$. You can then take any combination of regular numbers and $i$, and do math with it: $(3i+2)*(2i+4)=6i^2+12i+4i+8=16i+2$ . We can similarly take extensions of prime fields. As we start working over fields that are smaller, extensions of prime fields become increasingly important for preserving security, and binary fields (which Binius uses) depend on extensions entirely to have practical utility.

## Recap: arithmetization 

TODO:
- [ ] 

> The way that SNARKs and STARKs prove things about computer programs is through **arithmetization**: you convert a statement about a program that you want to prove, into a mathematical equation involving polynomials. A valid solution to the equation corresponds to a valid execution of the program.

> To give a simple example, suppose that I computed the 100'th Fibonacci number, and I want to prove to you what it is. I create a polynomial $F$ that encodes Fibonacci numbers: so $F(0)=F(1)=1,F(2)=2,F(3)=3,F(4)=5$, and so on for 100 steps. The condition that I need to prove is that $F(x+2)=F(x)+F(x+1)$ across the range $x=\{0,1 \ldots 98\}$. I can convince you of this by giving you the quotient:

$$
H(x)=\frac{F(x+2)-F(x+1)-F(x)}{Z(x)}
$$

> Where $Z(x)=(x-0)*(x-1)*\ldots*(x-98)$. If I can provide valid $F$ and $H$ that satisfy this equation, then $F$ must satisfy $F(x+2)-F(x+1)-F(x)$ across that range. If I additionally verify that $F$ satisfies $F(0)=F(1)=1$ , then $F(100)$ must actually be the 100th Fibonacci number.

> If you want to prove something more complicated, then you replace the "simple" relation $F(x+2)=F(x)+F(x+1)$ with a more complicated equation, which basically says "$F(x+1)$ is the output of initializing a virtual machine with the state $F(x)$, and running one computational step". You can also replace the number 100 with a bigger number, eg. 100000000, to accommodate more steps.

> All SNARKs and STARKs are based on this idea of using a simple equation over polynomials (or sometimes vectors and matrices) to represent a large number of relationships between individual values. Not all involve checking equivalence between adjacent computational steps in the same way as above: [PLONK](https://vitalik.eth.limo/general/2019/09/22/plonk.html) does not, for example, and neither does R1CS. But many of the most efficient ones do, because enforcing the same check (or the same few checks) many times makes it easier to minimize overhead.

## Plonky2: from 256-bit SNARKs and STARKs to 64-bit... only STARKs

五年前，ZK证明系统被分为有两种类型: (基于椭圆曲线的) SNARK 和(基于 hash 的) STARK。准确地说，STARK 是 SNARK 的一种，但在实践中，通常用“ SNARK” 来指基于椭圆曲线的证明系统，用“ STARK”来指基于 hash 的证明系统。SNARK 生成的 proof size 很小，因此其可以被快速验证，并轻松的放到链上。STARK 的 proof size 很大，但是 STARK 不需要 trusted setups ，并具有抗量子的特性。

> Five years ago, a reasonable summary of the different types of zero knowledge proof was as follows. There are two types of proofs: (elliptic-curve-based) SNARKs and (hash-based) STARKs. Technically, STARKs are a type of SNARK, but in practice it's common to use "SNARK" to refer to only the elliptic-curve-based variety, and "STARK" to refer to hash-based constructions. SNARKs are small, and so you can verify them very quickly and fit them onchain easily. STARKs are big, but they don't require [trusted setups](https://vitalik.eth.limo/general/2022/03/14/trustedsetup.html), and they are quantum-resistant.

![Overview](https://vitalik.eth.limo/images/binius/starks.png)

*STARK 的工作原理是将数据当作多项式处理，计算该多项式在大量点上的评估，并使用这些数据的 Merkle 根作为“多项式承诺”*

> STARKs work by treating the data as a polynomial, computing evaluations of that polynomial across a large number of points, and using the Merkle root of that extended data as the "polynomial commitment"

这里一个关键的历史是，基于椭圆曲线的 SNARK 首先被广泛使用：直到 2018 年左右，FRI 出现，STARK 才变得足够高效，而那时 Zcash 已经运行了一年多。基于椭圆曲线的 SNARK 有一个关键限制：如果要使用 SNARK，则这些方程中的算术必须使用椭圆曲线上的点数来模一个整数
*(FIXME: 这句话和本人目前理解的椭圆曲线上的运算不太一致， 需要重新翻译)* 
来完成。这是一个很大的数字，通常接近 $2^{256}$ ：例如，对于 BN128 曲线，它是 $21888242871839275222246405745257275088548364400416034343698204186575808495617$。但被算数化的程序中往往用到较小数字的频率更多：考虑一个“真实”的程序，它使用的大部分东西都是 counter、for 循环中的索引、程序中的位置、表示 True 或 False 的 bit，以及其他几乎总是只有几位长度的东西。

> A key bit of history here is that elliptic curve-based SNARKs came into widespread use first: it took until roughly 2018 for STARKs to become efficient enough to use, thanks to [FRI](https://eccc.weizmann.ac.il/report/2017/134/), and by then [Zcash](https://z.cash/) had already been running for over a year. Elliptic curve-based SNARKs have a key limitation: if you want to use elliptic curve-based SNARKs, then the arithmetic in these equations must be done with integers modulo the number of points on the elliptic curve. This is a big number, usually near $2^{256}$: for example, for the bn128 curve, it's $21888242871839275222246405745257275088548364400416034343698204186575808495617$. But the actual computation is using small numbers: if you think about a "real" program in your favorite language, most of the stuff it's working with is counters, indices in for loops, positions in the program, individual bits representing True or False, and other things that will almost always be only a few digits long.

即使您的“原始”数据由“小”数字组成，证明过程也需要计算商、扩域、随机线性组合和其他数据转换，这会导致相同或更多数量的对象，平均而言，这些对象与域的全尺寸一样大。这是导致低效的关键，即：要证明 $n$ 个小值的计算，你必须对 n 个大得多的值进行更多的计算。起初，STARK 继承了使用 SNARK 的 256 位 Field 的习惯，因此也遭受了同样的低效现象。

> Even if your "original" data is made up of "small" numbers, the proving process requires computing quotients, extensions, random linear combinations, and other transformations of the data, which lead to an equal or larger number of objects that are, on average, as large as the full size of your field. This creates a key inefficiency: to prove a computation over $n$ small values, you have to do even more computation over $n$ much bigger values. At first, STARKs inherited the habit of using 256-bit fields from SNARKs, and so suffered the same inefficiency.

![](https://vitalik.eth.limo/images/binius/rs_example.png)

一些多项式求值的 Reed-Solomon 扩展。即使原始值很小，额外的值也会放大到字段的完整大小（在本例中为 $2^{31}-1$ ）。

> A Reed-Solomon extension of some polynomial evaluations. Even though the original values are small, the extra values all blow up to the full size of the field (in this case $2^{31} - 1$).

2022 年，Plonky2 发布。Plonky2 的主要创新是将算术模数计算为较小的素数： $2^{64}-2^{32}+1=18446744069414584321$  。现在，每次加法或乘法都可以在 CPU 上只需几条指令即可完成，并且将所有数据哈希在一起的速度比以前快 4 倍。但这有一个问题：这种方法是 STARK 独有的。如果您尝试使用具有如此小尺寸的椭圆曲线的 SNARK，则椭圆曲线会变得不安全。

> In 2022, Plonky2 was released. Plonky2's main innovation was doing arithmetic modulo a smaller prime: $2^{64}-2^{32}+1=18446744069414584321$. Now, each addition or multiplication can always be done in just a few instructions on a CPU, and hashing all of the data together is 4x faster than before. But this comes with a catch: this approach is STARK-only. If you try to use a SNARK, with an elliptic curve of such a small size, the elliptic curve becomes insecure.

> To continue to be safe, Plonky2 also needed to introduce *extension fields*. A key technique in checking arithmetic equations is "sampling at a random point": if you want to check if $H(x) * Z(x)$ actually equals $F(x+2) - F(x+1) - F(x)$, you can pick some random coordinate $r$, provide *polynomial commitment opening proofs* proving $H(r), Z(r), F(r), F(r+1)$ and $F(r+2)$, and then actually check if $H(r) * Z(r)$ equals $F(r+2) - F(r+1) - F(r)$. If the attacker can guess the coordinate ahead of time, the attacker can trick the proof system - hence why it must be random. But this also means that the coordinate must be sampled from a set large enough that the attacker cannot guess it by random chance. If the modulus is near $2^{256}$, this is clearly the case. But with a modulus of $2^{64} - 2^{32} + 1$, we're not quite there, and if we drop to $2^{31} - 1$, it's *definitely* not the case. Trying to fake a proof two billion times until one gets lucky is absolutely within the range of an attacker's capabilities.

>  To stop this, we sample $r$ from an extension field. For example, you can define $y$ where $y^3=5$, and take combinations of 1, $y$ and $y^2$. This increases the total number of coordinates back up to roughly $2^93$. The bulk of the polynomials computed by the prover don't go into this extension field; they just use integers modulo $2^{31} - 1$, and so you still get all the efficiencies from using the small field. But the random point check, and the FRI computation, does dive into this larger field, in order to get the needed security.

## From small primes to binary

计算机通过将较大的数字表示为 0 和 1 的序列来进行算术运算，并在这些位之上构建“电路”来计算加法和乘法等内容。计算机特别针对 16 位、32 位和 64 位整数进行计算进行了优化。模量像 $2^{64} - 2^{32} + 1$ 和 $2^{31} - 1$ 之所以选择它们，不仅是因为它们符合这些 Bound，还因为它们与这些 Bound 很好地对齐：你可以做乘法模 $2^{64} - 2^{32} + 1$ 通过执行常规的 32 位乘法，并在几个地方按位移位和复制输出；[这篇文章](https://xn--2-umb.com/22/goldilocks/)很好地解释了一些技巧。

> Computers do arithmetic by representing larger numbers as sequences of zeroes and ones, and building "circuits" on top of those bits to compute things like addition and multiplication. Computers are particularly optimized for doing computation with 16-bit, 32-bit and 64-bit integers. Moduluses like $2^{64} - 2^{32} + 1$ and $2^{31} - 1$ are chosen not just because they fit within those bounds, but also because they *align well* with those bounds: you can do multiplication modulo $2^{64} - 2^{32} + 1$ by doing regular 32-bit multiplication, and shift and copy the outputs bitwise in a few places; [this article](https://xn--2-umb.com/22/goldilocks/) explains some of the tricks well.
>
> TODO:
> - [ ] 计算机 word 对齐

> What would be even better, however, is doing computation in binary directly. What if addition could be "just" XOR, with no need to worry about "carrying" the overflow from adding 1 + 1 in one bit position to the next bit position? What if multiplication could be more parallelizable in the same way? And these advantages would all come on top of being able to represent True/False values with just one bit.

> Capturing these advantages of doing binary computation directly is exactly what Binius is trying to do. A table from the [Binius team's zkSummit presentation](https://docs.google.com/presentation/d/1WuTiof1BiaL6vB50CSeb-hvi5H4j_oqUt19-sZTQEB4/edit#slide=id.g2c9c013854e_0_95) shows the efficiency gains:

![](https://vitalik.eth.limo/images/binius/zksummit_slides.png)

> Despite being roughly the same "size", a 32-bit binary field operation takes 5x less computational resources than an operation over the 31-bit Mersenne field.

## From univariate polynomials to hypercubes

假设我们被这个推理所说服，并希望用 0 和 1 做所有事情。我们如何实际承诺一个表示十亿位的多项式？

> Suppose that we are convinced by this reasoning, and want to do everything over bits (zeroes and ones). How do we actually commit to a polynomial representing a billion bits?

我们要面对两个问题：
1. 如果用多项式（polynomial）去表示很多值，那么这些值在多项式的评估（evaluations）时是可访问的：比如前文提到的斐波那契例子$F(0),\:F(1)\:...\:F(100)，而在更大规模的计算中，$F(x)$ 的索引会达到数百万。更大的索引需要我们用更大的域来表示。
2. 证明我们在Merkle树中承诺的任何值（就像所有的STARKs所做的那样）需要对它进行Reed-Solomon编码：将 $n$ 个值扩展到 $8n$ 个值，利用这种冗余来防止恶意的证明者通过伪造计算中间的一个值来进行欺诈。这也需要有一个足够大的域：为了将一百万的值扩展到八百万，你需要有八百万不同的点来评估多项式。

> Here, we face two practical problems:
> 1. For a polynomial to represent a lot of values, those values need to be accessible at evaluations of the polynomial: in our Fibonacci example above, $F(0),\:F(1)\:...\:F(100)$, and in a bigger computation, the indices would go into the millions. And the field that we use needs to contain numbers going up to that size.
> 2. Proving anything about a value that we're committing to in a Merkle tree (as all STARKs do) requires Reed-Solomon encoding it: extending $n$ values into eg. $8n$ values, using the redundancy to prevent a malicious prover from cheating by faking one value in the middle of the computation. This also requires having a large enough field: to extend a million values to 8 million, you need 8 million different points at which to evaluate the polynomial.

Binius 通过两种不同的方式来表示同一组数据，用以分别解决上述的两个问题。首先，基于椭圆曲线的 SNARKs ，2019年的 STARKs，Plonky2 和其他的证明系统通常处理单变量多项式 $F(x)$ 。Binus
则从 Spartan 协议中获取灵感，使用多变量多项式：$F(x_1,x_2\ldots x_k)$ 。事实上，我们用一个 Hypercube 来表示整个 computational trace ，其中每一个的 $x_i$ 的取值要么是 0 ，要么是 1 。句一个例子，如果我们想要表示一个斐波那契数列，同时我们用一个足够大的域去表示这些数字，我们把前 16 个数字进行可视化后得到

> A key idea in Binius is solving these two problems separately, and doing so by representing the same data in two different ways. First, the polynomial itself. Elliptic curve-based SNARKs, 2019-era STARKs, Plonky2 and other systems generally deal with polynomials over *one* variable: $F(x)$ . Binius, on the other hand, takes inspiration from the [Spartan](https://eprint.iacr.org/2019/550.pdf) protocol, and works with *multivariate* polynomials: $F(x_1,x_2\ldots x_k)$. In fact, we represent the entire computational trace on the "hypercube" of evaluations where each $x_i$ is either 0 or 1. For example, if we wanted to represent a sequence of Fibonacci numbers, and we were still using a field large enough to represent them, we might visualize the first sixteen of them as being something like this:

![](https://vitalik.eth.limo/images/binius/hypercube.png)

在这个例子中，“$F(0, 0, 0, 0)$ 为 1，$F(1, 0, 0, 0)$ 也为 1，$F(0, 1, 0, 0)$ 为 2，依此类推，直到 $F(1, 1, 1, 1) = 987$” 表示了一个超立方体中的各个点的取值。对于这样一组取值，存在唯一一个多线性多项式（每个变量的阶为1），可以生成这些值。因此，我们可以用这组取值来代表该多项式；从而无需计算系数。

> That is, $F(0, 0, 0, 0)$ would be 1, $F(1, 0, 0, 0)$ would also be 1, $F(0, 1, 0, 0)$ would be 2, and so forth, up until we get to $F(1, 1, 1, 1) = 987$. Given such a hypercube of evaluations, there is exactly one multilinear (degree-1 in each variable) polynomial that produces those evaluations. So we can think of that set of evaluations as representing the polynomial; we never actually need to bother computing the coefficients.
>
> 补充：这里也可以理解成一个 key-value pair（键值对），key 是这个 Hypercube 上的所有点，value 是斐波那契数列中的数字，并且如果读者去从右到左去阅读这些点，你会发现这些点和二进制表示的整数是一致的。
> | hypercube 上的点 | 用十进制整数表示的 Key/Index | Value |
> | ---- | ---- | ---- |
> |(0, 0, 0, 0)|0|1|
> |(1, 0, 0, 0)|1|1|
> |(0, 1, 0, 0)|2|2|
> |(1, 1, 0, 0)|3|3|
> |(0, 0, 1, 0)|4|5|
> |(1, 0, 1, 0)|5|8|
> |(0, 1, 1, 0)|6|13|
> |(1, 1, 1, 0)|7|21|
> |(0, 0, 0, 1)|8|34|
> |(1, 0, 0, 1)|9|55|
> |(0, 1, 0, 1)|10|89|
> |(1, 1, 0, 1)|11|144|
> |(0, 0, 1, 1)|12|233|
> |(1, 0, 1, 1)|13|377|
> |(0, 1, 1, 1)|14|610|
> |(1, 1, 1, 1)|15|987|

这个只是一个简化的例子：在实践中，使用 Hypercube 的真正目的是让我们能够处理单个比特。用“Binius-native” 方式来计算斐波那契数的话，会使用更高维的立方体，例如用每组16个比特来存储一个数字。

FIXME: 这需要一些巧妙的方法来实现这些bits的整数加法 ，

但使用Binius来实现这一点并不太困难。

> This example is of course just for illustration: in practice, the whole point of going to a hypercube is to let us work with individual bits. The "Binius-native" way to count Fibonacci numbers would be to use a higher-dimensional cube, using each set of eg. 16 bits to store a number. This requires some cleverness to implement integer addition on top of the bits, but with Binius it's not too difficult.


现在，我们来讨论纠删码。STARKs 的工作方式是：你取 $n$ 个值，通过 Reed-Solomon 编码将它们扩展到更多的值（通常是 $8n$，一般在 $2n$ 到 $32n$ 之间），然后从扩展的数据中随机选择一些 Merkle 分支并进行某种检查。一个超立方体在每个维度的长度为 2。因此，直接扩展它是不切实际的：从 16 个值中抽样 Merkle 分支的“空间”不足。那么我们该怎么办呢？我们假装超立方体是一个方块！

> Now, we get to the erasure coding. The way STARKs work is: you take $n$ values, Reed-Solomon extend them to a larger number of values (often $8n$, usually between $2n$ and $32n$), and then randomly select some Merkle branches from the extension and perform some kind of check on them. A hypercube has length 2 in each dimension. Hence, it's not practical to extend it directly: there's not enough "space" to sample Merkle branches from 16 values. So what do we do instead? We pretend the hypercube is a square!

## Simple Binius - an example

*See* *[here](https://github.com/ethereum/research/blob/master/binius/simple_binius.py) for a python implementation of this protocol.*

让我们用一个新的例子，为了方便起见，我们使用整数域（在实际的 Binius 实现中使用的是二进制域）。首先，我们获取要承诺的 Hypercube，并将其编码为方块：

> Let's go through an example, using regular integers as our field for convenience (in a real implementation this will be binary field elements). First, we take the hypercube we want to commit to, and encode it as a square:

![](https://vitalik.eth.limo/images/binius/basicbinius1.drawio.png)

现在我们使用里德所罗门编码去扩展这个方块，我们把每一行看成一个 3 阶多项式，并将行中的数字视为多项式在 $x = {0, 1, 2, 3}$ 上的取值，然后取这个多项式在 $x = {4, 5, 6, 7}$ 上的取值。

> Now, we Reed-Solomon extend the square. That is, we treat each row as being a degree-3 polynomial evaluated at $x = {0, 1, 2, 3}$, and evaluate the same polynomial at $x = {4, 5, 6, 7}$:
>
> 补充：以图中的第一行为例，让 $x = {0, 1, 2, 3}$ 作为输入，第一行中的四个值 $y = {3, 1, 4, 1}$ 作为输出。于是我们有点对 $(x=0, y=3), (x=1, y=1), (x=2, y=4), (x=1, y=1)$ ，计算其拉格朗日多项式，得到 $\frac{(x-1)(x-2)(x-3)}{-2} + \frac{x(x-2)(x-3)}{2} + 2x(x-1)(x-3) + \frac{x(x-1)(x-2)}{6}$ ，然后带入 $x = {4, 5, 6, 7}$ 到这个多项式中，得到 $y={-19, -67, -154, -291}$ 即图中右侧方块的第一行

![](https://vitalik.eth.limo/images/binius/basicbinius.drawio.png)

注意到数字的增长是非常快的，这也是我们总是在实际应用中采用有限域的原因。比如，我们采用 11 做为模，那么第一行的扩展会是 $[3, 10, 0, 6]$ 。

> Notice that the numbers blow up quickly! This is why in a real implementation, we always use a finite field for this, instead of regular integers: if we used integers modulo 11, for example, the extension of the first row would just be $[3, 10, 0, 6]$.

你可以使用该链接里的 [代码](https://github.com/ethereum/research/blob/master/binius/utils.py#L123) 来进行验证。

> If you want to play around with extending and verify the numbers here for yourself, you can use [my simple Reed-Solomon extension code here](https://github.com/ethereum/research/blob/master/binius/utils.py#L123).

接下来，从列的方向上来处理扩展后的方块，并且生成一个默克尔树，这个树的根节点就是我们的承诺

> Next, we treat this extension as columns, and make a Merkle tree of the columns. The root of the Merkle tree is our commitment.

![](https://vitalik.eth.limo/images/binius/binius_merkletree.drawio.png)

现在，我们假设 P 想要去证明这个多项式在某一个随机点上的取值 $r = (r_0, r_1, r_2, r_3)$，在  Binius 中，有一个细微的不同导致了其比其他的多项式承诺稍微弱一点，即 P 不应该在承诺默克尔根之前就知道或有能力猜到这个随机值 $r$ 。换句话说 $r$ 应该是一个依赖默克尔根的伪随机数。这使得该方案对
FIXME: "database lookup"
（举例：你给了我默克尔树的根，现在向我证明 $P(0, 0, 1, 0)$ ）毫无用处。但通常来说，我们使用的证明系统们通常不需要
FIXME: "database lookup"
这一特性，它们通常只需要检查多项式在一个随机点上的取值就好了，因此这个限制不会影响到我们要做的事情

> Now, let's suppose that the prover wants to prove an evaluation of this polynomial at some point $r = (r_0, r_1, r_2, r_3)$. There is one nuance in Binius that makes it somewhat weaker than other polynomial commitment schemes: the prover should not know, or be able to guess, $s$, until after they committed to the Merkle root (in other words, $r$ should be a pseudo-random value that depends on the Merkle root). This makes the scheme useless for "database lookup" (eg. "ok you gave me the Merkle root, now prove to me $P(0, 0, 1, 0)$ !"). But the actual zero-knowledge proof protocols that we use generally don't need "database lookup"; they simply need to check the polynomial at a random evaluation point. Hence, this restriction is okay for our purposes.

假设我们选取随机点 $r=\{1,2,3,4\}$ (多项式在该点的取值为 -137，可以用[这里的代码](https://github.com/ethereum/research/blob/master/binius/utils.py#L100)进行确认)。现在让我们来看一下 Proof 的生成过程，我们把 $r$ 分解成两个部份：第一个部份是 $\{1,2\}$ 表示对一个行中的列元素做线性组合，第二个部份 $\{3,4\}$ 表示对行做线性组合，

> Suppose we pick $r=\{1,2,3,4\}$ (the polynomial, at this point, evaluates to −137; you can confirm it [with this code](https://github.com/ethereum/research/blob/master/binius/utils.py#L100)). Now, we get into the process of actually making the proof. We split up $r$ into two parts: the first part $\{1,2\}$ representing a linear combination of *columns within a row*, and the second part $\{3,4\}$ representing a linear combination *of rows*. 
>
> 补充：这里的多项式指的是用本小节用的 Hypercube 上的点和其对应值计算出来的 MLE ，因为这个 Hypercube 是4维的，所以上面有16个点，对应着一个 4 变量多项式，$P(x_1, x_2, x_3, x_4)$

我们对列的计算其张量积

> We compute a "tensor product", both for the column part:

$$
\bigotimes_{i=0}^1(1-r_i,r_i)
$$

对于行的部份

> And for the row part:

$$
\bigotimes_{i=2}^3(1-r_i,r_i)
$$

张量积意味着：取每一个集合中的元素相乘，并把所有的结果给写出来

> What this means is: a list of all possible products of one value from each set.

，在行的情形中，我们有：

> In the row case, we get:

$$
[(1-r_2)\times (1-r_3),r_2\times (1-r_3),(1-r_2)\times r_3,r_2\times r_3]
$$

> 补充：先不要去管左边的 $\bigotimes$ 符号，把目光放在 $(1-r_i,r_i)$ 上，代入 $i$ 所有可能的取值 $i = 2, 3$ ，我们得到两个数对 $(1 - r_2, r_2),(1 - r_3, r_3)$ ，然后我们取第一个数对中的元素和第二个数对中的元素，并把它们乘起来，有：
>
> $(1 - r_2) \times (1 - r_3)$
> 
> $r_2 \times (1 - r_3)$
> 
> $(1 - r_2) \times r_3$
> 
> $r_2 \times r_3$
>
> 把上面得出的四个结果写到一个列表里面，我们就得到了上面的式子 

使用  $r={1,2,3,4}$  ，取 $r_2 = 3$ and $r_3 = 4$ ：

> Using $r={1,2,3,4}$ (so $r_2 = 3$ and $r_3 = 4$):

$$
[(1-3)\times (1-4),3\times (1-4),(1-3)\times 4,3\times 4]=[6,-9,-8,12]
$$

现在 通过对现有的行计算其线性组合，我们得到了一个新行 $t'$ ：

> Now, we compute a new "row" $t'$, by taking this linear combination of the existing rows. That is, we take:

$$
\begin{align*}
[&3,1,4,1] \times 6 +\\
[&5,9,2,6] \times (-9) +\\
[&5,3,5,8] \times (-8) +\\
[&9,7,9,3] \times 12 =\\
[&41,-15,74,-76]
\end{align*}
$$

你现在可以把这个操作堪称部份求值，如果我们去计算完整的张量积，你就会得到 $P(1,2,3,4)=-137$ ，在这里，我们将仅使用一半坐标的偏张量乘积相乘，并将 N 个值的方阵规约为一行 （ $\sqrt{N}$ 个值）。如果你把此行提供给其他人，他们可以用另一半的求值坐标的张量积来完成剩下的计算。

> You can view what's going on here as a partial evaluation. If we were to multiply the full tensor product $\bigotimes_{i=0}^3(1-r_i,r_i)$ by the full vector of all values, you would get the evaluation $P(1,2,3,4)=-137$ . Here we're multiplying a *partial* tensor product that only uses half the evaluation coordinates, and we're reducing a grid of $N$ values to a row of $\sqrt{N}$ values. If you give this row to someone else, they can use the tensor product of the other half of the evaluation coordinates to complete the rest of the computation.

P 向 V 提供刚刚计算得出的新行 $t'$ ，同时提供一些随机抽样列的 Merkle 证明。其空间复杂度为 $O(\sqrt{N})$ 。在本例中，我们让 P 只提供最后一列；而在实际应用中，P 需要提供给 V 更多的列来达到更好的安全性。

> The prover provides the verifier with this new row, $t'$, as well as the Merkle proofs of some randomly sampled columns. This is $O(\sqrt{N})$ data. In our illustrative example, we'll have the prover provide just the last column; in real life, the prover would need to provide a few dozen columns to achieve adequate security.
> 
> TODO:
> - [ ] 实际应用中要提供多少列才能达到安全，如果未提供足量的列会导致什么样的安全问题？

现在，我们利用 Reed-Solomon 的线性特性，其关键是：对 ”Reed-Solomon 扩展做线性组合“的结果等于“对线性组合做Reed-Solomon 扩展”，这种运算顺序的非相关性往往在两个运算都是线性运算时出现。

> Now, we take advantage of the linearity of Reed-Solomon codes. The key property that we use is: **taking a linear combination of a Reed-Solomon extension gives the same result as a Reed-Solomon extension of a linear combination**. This kind of "order independence" often happens when you have two operations that are both linear.
>
> 补充：即 $f(g(x)) = g(f(x))$

V 正是这样做的。他们计算了 $t'$ 的扩展，并且用同样的系数对 P 提供的列计算线性组合，并验证这两个过程是否给出相同的答案。

> The verifier does exactly this. They compute the extension of $t'$, and they compute the same linear combination of columns that the prover computed before (but only to the columns provided by the prover), and verify that these two procedures give the same answer.
>
> 补充：即我们之前计算出来的 $[6,-9,-8,12]$

![](https://vitalik.eth.limo/images/binius/basicbinius2.drawio.png)

在本例中，计算 $t'$ 的扩展，和对图中标记的竖列做线性组合，两者给出了相同的答案：-10746。这证明默克尔的根是「善意」构建的 ( 或者至少「足够接近」)，而且它是匹配 t 的：至少绝大多数列是相互兼容的。

> In this case, extending $t'$, and computing the same linear combination $([6,-9,-8,12])$ of the column, both give the same answer: $-10746$ . This proves that the Merkle root was constructed "in good faith" (or it at least "close enough"), and it matches $t'$ : at least the great majority of the columns are compatible with each other and with $t'$.
>
> TODO:
> - [ ] 这里和默克尔树的关联还是不够明显

到这里 V 还需要再检查一点东西：检查多项式 在 $r = \{r0, \ldots, r3\}$ 的取值。到目前为止，验证者的所有步骤实际上都没有依赖于证明者声称的值。我们是这样检查的。我们对 $t'$ 用前文提到的"column part"张量积的计算结果进行线性组合：

> But the verifier still needs to check one more thing: actually check the evaluation of the polynomial at $\{r_0..r_3\}$ . So far, none of the verifier's steps actually depended on the value that the prover claimed. So here is how we do that check. We take the tensor product of what we labelled as the "column part" of the evaluation point:

$$
\bigotimes_{i=0}^1(1-r_i,r_i)
$$

在我们的例子中，我们取 r 的前半部份，即 $\{1, 2\}$ ，有：

> In our example, where $r=\{1,2,3,4\}$ (so the half that chooses the column is $\{1, 2\}$ ), this equals:

$$
[(1-1)\times (1-2),1\times (1-2),(1-1)\times 2,1\times 2]=[0,-1,0,2]
$$

然后我们用刚才计算出来的结果计算 $t'$ 的线性组合
> So now we take this linear combination of $t'$ :

$$
0\times 41+(-1)\times (-15)+0\times 74+2\times (-76)=-137
$$

这和你直接把 $r=\{1,2,3,4\}$ 带入到多项式后计算出来的结果是一致的。

> Which exactly equals the answer you get if you evaluate the polynomial directly.
>
> TODO:
> - [ ] “选择 r 中的一半进行线性组合”，找一下这个方法在论文中的解释和证明。

以上内容是一个 简化 Binius 协议的完整描述。目前我们能发现一些有趣的优点：例如，由于数据被分成行和列，因此你只需要一个一半大小的域。但是用二进制域进行计算的好处不止于此。为此，我们需要完整的 Binius 协议。但首先，让我们更深入地了解二进制域。

> The above is pretty close to a complete description of the "simple" Binius protocol. This already has some interesting advantages: for example, because the data is split into rows and columns, you only need a field half the size. But this doesn't come close to realizing the full benefits of doing computation in binary. For this, we will need the full Binius protocol. But first, let's get a deeper understanding of binary fields.

## Binary fields

最小的有限域是模为 2 的有限域，用 $F_2$ 表示，这是在 $F_2$ 上的加法和乘法表：

> The smallest possible field is arithmetic modulo 2, which is so small that we can write out its addition and multiplication tables:

|**+**|**0**|**1**|     |**\***|**0**|**1**|
|---- |---- |---- |---- |----  |---- |---- |
|**0**|0    |1    |     |**0** |0    |0    |
|**1**|1    |0    |     |**1** |0    |1    |

通过扩展，我们能够得到一个更大的二进制域：我们对 $F_2$ （模为 2）进行扩域，定义 $x$ ,  其是方程 $x^2=x+1$ 的根，我们可以得到如下的加法乘法表

> We can make larger binary fields by taking extensions: if we start with $F_2$ (integers modulo 2) and then define $x$ where $x^2=x+1$, we get the following addition and multiplication tables:

|**+**  |**0**|**1**|**x**|**x+1**| |**\*** |**0**|**1**|**x**|**x+1**|
|----   |---- |---- |---- |----   |-|----   |---- |---- |---- |----   |
|**0**  |0    |1    |x    |x+1    | |**0**  |0    |0    |0    |0      |
|**1**  |1    |0    |x+1  |x      | |**1**  |0    |1    |x    |x+1    |
|**x**  |x    |x+1  |0    |1      | |**x**  |0    |x    |x+1  |1      |
|**x+1**|x+1  |x    |1    |0      | |**x+1**|0    |x+1  |1    |x      |

通过反复使用上述的技巧，我们可以把一个二进制域扩展到任意大的域。与实数域扩展到负数域不同，在引入虚数 $i$ 之后，你便不能再添加新的元素到复数域中（当然四元数是一个复数的扩域，不过其不满足交换率）。在有限域的的情形中，你总是能进行扩域的操作，每次扩域时，我们使用如下定义的元素。

> It turns out that we can expand the binary field to arbitrarily large sizes by repeating this construction. Unlike with complex numbers over reals, where you can add *one* new element $i$, but you can't add any more ([quaternions](https://en.wikipedia.org/wiki/Quaternion) do exist, but they're mathematically weird, eg. $ab\neq ba$ ), with finite fields you can keep adding new extensions forever. Specifically, we define elements as follows:

- $x_0$ satisfies $x_0^2=x_0+1$
- $x_1$ satisfies $x_1^2=x_0x_1+1$
- $x_2$ satisfies $x_2^2=x_1x_2+1$
- $x_3$ satisfies $x_3^2=x_3x_2+1$

> 补充： 
> - $x_0^2=x_0+1$ 的解在 $F_2$ 上不存在，通过把这个解添加到 $F_2$ 中，该方程变得有解，域则因为添加操作而变成了一个新的更大的域，此时一次扩域完成。
> - - [ ] $x_1, x_2, \ldots$ 像上面这样定义的原因
> - Vitalik 这里指的是，复数域是最大的代数域，你无法再通过代数扩张的方式得到一个比复数域还要大的域，且四元数不在我们讨论的范围内
> - 实数域，复数域的元素有无穷多个，是无限域
> - 扩域：通过对原有域中引入一个新的元素，并将这个元素和原域中的所有元素做线性组合，所得出的全部结果就是这个扩域后的全部域元素，
> - 以 $F_2$ 到 $F_{2^2}$ 的扩域为例，当扩域只引入了一个元素时，你可以把原域的元素铺成一个横轴和一个纵轴，如下表左侧，然后对纵轴上的每个数字都乘上被引入的新元素（如右侧）得到的值再和横轴的数字做加法，即可得到 $F_{2^2}$ 中的全部元素
> 
> > | $F_2$ | $0$     | $1$     |   | $F_{2^2}$      | $0$                | $1$                |
> > | ---   |---      |---      |---|---             |---                 |---                 |
> > | $0$   | $(0,0)$ | $(0,1)$ |==>| $0 \times x_0$ | $0 \times x_0 + 0$ | $0 \times x_0 + 1$ |
> > | $1$   | $(1,0)$ | $(1,1)$ |   | $1 \times x_0$ | $1 \times x_0 + 0$ | $1 \times x_0 + 1$ |

以此类推，这个扩域方式通常被叫做塔结构，因为每一次扩域就像是给这个塔加了新的一层一半，这当然不是唯一的二进制扩域形式，但是塔结构有着自己独到的优点，Binius 正是利用到了这些点

> And so on. This is often called the **tower construction**, because of how each successive extension can be viewed as adding a new layer to a tower. This is not the only way to construct binary fields of arbitary size, but it has some unique advantages that Binius takes advantage of.

我们可以用 bit 来表示这些数字 $1100101010001111$ 。第一位表示

> We can represent these numbers as a list of bits, eg. $1100101010001111$. The first bit represents multiples of 1, the second bit represents multiples of $x_0$ , then subsequent bits represent multiples of: $x_1, x_1 \times x_0, x_2, x_2 \times x_0$ , and so forth. This encoding is nice because you can decompose it:

$$
\begin{align*}
1100101010001111=11001010+10001111 \times x_3 =1100+1010  \times x_2+1000 \times x_3+1111 \times x_2x_3 \\
=11+10 \times x_2+10 \times x_2x_1+10 \times x_3+11 \times x_2x_3+11 \times x_1x_2x_3\\
=1+x_0+x_2+x_2x_1+x_3+x_2x_3+x_0x_2x_3+x_1x_2x_3+x_0x_1x_2x_3
\end{align*}
$$

This is a relatively uncommon notation, but I like representing binary field elements as integers, taking the bit representation where more-significant bits are to the right. That is, $1=1, x_0=01=2,1+x_0=11=3,1+x_0+x_2=11001000=19$ , and so forth. $1100101010001111$ is, in this representation, $61779$.

Addition in binary fields is just XOR (and, incidentally, so is subtraction); note that this implies that $x+x=0$ for any $x$ . To multiply two elements $x \times y$, there's a pretty simple recursive algorithm: split each number into two halves:

$$
x=L_x+R_x \times x_k~y=L_y+R_y\times x_k
$$

Then, split up the multiplication:

$$
x\times y=(L_x\times L_y)+(L_x\times R_y)\times x_k+(R_x\times L_y)\times x_k+(R_x\times R_y)\times x_k^2
$$

The last piece is the only slightly tricky one, because you have to apply the reduction rule, and replace $R_x*R_y*x_k^2$ with $R_x*R_y*(x_{k-1}*x_k+1)$ . There are more efficient ways to do multiplication, analogues of the [Karatsuba algorithm](https://en.wikipedia.org/wiki/Karatsuba_algorithm) and [fast Fourier transforms](https://vitalik.eth.limo/general/2019/05/12/fft.html), but I will leave it as an exercise to the interested reader to figure those out.

Division in binary fields is done by combining multiplication and inversion: $\frac35=3 \times \frac15$ . The "simple but slow" way to do inversion is an application of [generalized Fermat's little theorem](https://planetmath.org/fermatslittletheorem): $\frac1x=x^{2^{2^k}-2}$ for any $k$ where $2^{2^k}>x$ . In this case, $\frac15=5^{14}=14$ , and so $\frac35=3*14=9$ . There is also a more complicated but more efficient inversion algorithm, which you can find [here](https://ieeexplore.ieee.org/document/612935). You can use [the code here](https://github.com/ethereum/research/blob/master/binius/binary_fields.py) to play around with binary field addition, multiplication and division yourself.

![img](https://vitalik.eth.limo/images/binius/additiontable.png) ![img](https://vitalik.eth.limo/images/binius/multiplicationtable.png)

*Left: addition table for four-bit binary field elements (ie. elements made up only of combinations of $1\text{,}x_0\text{,}x_1\text{ and }x_0x_1$). Right: multiplication table for four-bit binary field elements.*

The beautiful thing about this type of binary field is that it combines some of the best parts of "regular" integers and modular arithmetic. Like regular integers, binary field elements are unbounded: you can keep extending as far as you want. But like modular arithmetic, if you do operations over values within a certain size limit, all of your answers also stay within the same bound. For example, if you take successive powers of $42$, you get:

$1,42,199,215,245,249,180,91, \ldots$

And after $255$ steps, you get right back to $42^{255}=1$. And like *both* regular integers and modular arithmetic, they obey the usual laws of mathematics: $a \times b=b \times a,a \times (b+c)=a \times b+a \times c$, and even some strange new laws, eg. $a^2+b^2=(a+b)^2$ (the usual $2ab$ term is missing, because in a binary field, $1+1=0$).

And finally, binary fields work conveniently with bits: if you do math with numbers that fit into $2^k$ bits, then all of your outputs will also fit into $2^k$ bits. This avoids awkwardness like eg. with Ethereum's [EIP-4844](https://www.eip4844.com/), where the individual "chunks" of a blob have to be numbers modulo `52435875175126190479447740508185965837690552500527637822603658699938581184513`, and so encoding binary data involves throwing away a bit of space and doing extra checks at the application layer to make sure that each element is storing a value less than  $2^{248}$ . It also means that binary field arithmetic is *super* fast on computers - both CPUs, and theoretically optimal FPGA and ASIC designs.

This all means that we can do things like the Reed-Solomon encoding that we did above, in a way that completely avoids integers "blowing up" like we saw in our example, and in a way that is extremely "native" to the kind of calculation that computers are good at. The "splitting" property of binary fields - how we were able to do $1100101010001111=11001010+10001111*x_3$ , and then keep splitting as little or as much as we wanted, is also crucial for enabling a lot of flexibility.

## Full Binius
*See* *[here](https://github.com/ethereum/research/blob/master/binius/packed_binius.py) for a python implementation of this protocol.*

Now, we can get to "full Binius", which adjusts "simple Binius" to (i) work over binary fields, and (ii) let us commit to individual bits. This protocol is tricky to understand, because it keeps going back and forth between different ways of looking at a matrix of bits; it certainly took me longer to understand than it usually takes me to understand a cryptographic protocol. But once you understand binary fields, the good news is that there isn't any "harder math" that Binius depends on. This is not [elliptic curve pairings](https://vitalik.eth.limo/general/2017/01/14/exploring_ecp.html), where there are deeper and deeper rabbit holes of algebraic geometry to go down; here, binary fields are all you need.

Let's look again at the full diagram:

![img](https://vitalik.eth.limo/images/binius/binius.drawio.png)

By now, you should be familiar with most of the components. The idea of "flattening" a hypercube into a grid, the idea of computing a row combination and a column combination as tensor products of the evaluation point, and the idea of checking equivalence between "Reed-Solomon extending then computing the row combination", and "computing the row combination then Reed-Solomon extending", were all in simple Binius.

What's new in "full Binius"? Basically three things:

- The individual values in the hypercube, and in the square, have to be bits (0 or 1)
- The extension process extends bits into more bits, by grouping bits into columns and temporarily pretending that they are larger field elements
- After the row combination step, there's an element-wise "decompose into bits" step, which converts the extension back into bits

We will go through both in turn. First, the new extension procedure. A Reed-Solomon code has the fundamental limitation that if you are extending $n$ values to $k \times n$ values, you need to be working in a field that has $k \times n$ different values that you can use as coordinates. With $F_2$ (aka, bits), you cannot do that. And so what we do is, we "pack" adjacent $F_2$ elements together into larger values. In the example here, we're packing two bits at a time into elements in $\{0,1,2,3\}$ , because our extension only has four evaluation points and so that's enough for us. In a "real" proof, we would probably back 16 bits at a time together. We then do the Reed-Solomon code over these packed values, and unpack them again into bits.

![img](https://vitalik.eth.limo/images/binius/basicbinius3.drawio.png)

Now, the row combination. To make "evaluate at a random point" checks cryptographically secure, we need that point to be sampled from a pretty large space, much larger than the hypercube itself. Hence, while the points *within* the hypercube are bits, evaluations *outside* the hypercube will be much larger. In our example above, the "row combination" ends up being [11,4,6,1].

This presents a problem: we know how to combine pairs of *bits* into a larger value, and then do a Reed-Solomon extension on that, but how do you do the same to pairs of much larger values?

The trick in Binius is to do it bitwise: we look at the individual bits of each value (eg. for what we labeled as "11", that's $[1,1,0,1]$ ), and then we extend *row-wise*. That is, we perform the extension procedure on the 1 row of each element, then on the $x_0$ row, then on the " $x_1$ " row, then on the $x_0 \times x_1$ row, and so forth (well, in our toy example we stop there, but in a real implementation we would go up to 128 rows (the last one being $x_6 \times \ldots \times x_0$ )).

Recapping:

- We take the bits in the hypercube, and convert them into a grid
- Then, we treat adjacent groups of bits *on each row* as larger field elements, and do arithmetic on them to Reed-Solomon extend the rows
- Then, we take a row combination of each *column* of bits, and get a (for squares larger than 4x4, much smaller) column of bits for each row as the output
- Then, we look at the output as a matrix, and treat the bits of *that* as rows again

Why does this work? In "normal" math, the ability to (often) do linear operations in either order and get the same result stops working if you start slicing a number up by digits. For example, if I start with the number 345, and I multiply it by 8 and then by 3, I get 8280, and if do those two operations in reverse, I also do 8280. But if I insert a "split by digit" operation in between the two steps, it breaks down: if you do 8x then 3x, you get:

$$
345\xrightarrow{\times8}2760\to[2,7,6,0]\xrightarrow{\times3}[6,21,18,0]
$$

But if you do 3x then 8x, you get:

$$
345\xrightarrow{\times3}1035\to[1,0,3,5]\xrightarrow{\times8}[8,0,24,40]
$$

But in binary fields built with the tower construction, this kind of thing *does* work. The reason why has to do with their separability: if you multiply a big value by a small value, what happens in each segment, stays in each segment. If we multiply $1100101010001111$ by $11$, that's the same as first decomposing $1100101010001111$ into $11+10 \times x_2+10 \times x_2x_1+10 \times x_3+11 \times x_2x_3+11 \times x_1x_2x_3$ , and then multiplying each component by $11$ separately.

## Putting it all together

Generally, zero knowledge proof systems work by making statements about polynomials that simultaneously represent statements about the underlying evaluations: just like we saw in the Fibonacci example, $F(X+2)-F(X+1)-F(X)=Z(X) \times H(X)$ simultaneously checks all steps of the Fibonacci computation. We check statements about polynomials by proving evaluations at a random point: given a commitment to $F$, you might randomly choose eg. 1892470, demand proofs of evaluations of $F$ , $Z$ and $H$ at that point (and $H$ at adjacent points), check those proofs, and then check if $F(1892472)-F(1892471)-F(1892470)=Z(1892470) \times H(1892470)$ . This check at a random point stands in for checking the whole polynomial: if the polynomial equation *doesn't* match, the chance that it matches at a specific random coordinate is tiny.

In practice, a major source of inefficiency comes from the fact that in real programs, most of the numbers we are working with are tiny: indices in for loops, True/False values, counters, and similar things. But when we "extend" the data using Reed-Solomon encoding to give it the redundancy needed to make Merkle proof-based checks safe, most of the "extra" values end up taking up the full size of a field, even if the original values are small.

To get around this, we want to make the field as small as possible. Plonky2 brought us down from 256-bit numbers to 64-bit numbers, and then Plonky3 went further to 31 bits. But even this is sub-optimal. With binary fields, we can work over *individual bits*. This makes the encoding "dense": if your actual underlying data has `n` bits, then your encoding will have `n` bits, and the extension will have `8 * n` bits, with no extra overhead.

Now, let's look at the diagram a third time:

![img](https://vitalik.eth.limo/images/binius/binius.drawio.png)

In Binius, we are committing to a *multilinear polynomial*: a hypercube $P(x_0,x_1\ldots x_k)$, where the individual evaluations $P(0,0\ldots0),P(0,0\ldots1)\text{ up to }P(1,1,\ldots1)$ are holding the data that we care about. To prove an evaluation at a point, we "re-interpret" the same data as a square. We then extend each *row*, using Reed-Solomon encoding over *groups* of bits, to give the data the redundancy needed for random Merkle branch queries to be secure. We then compute a random linear combination of rows, with coefficients designed so that the new combined row actually holds the evaluation that we care about. Both this newly-created row (which get re-interpreted as 128 rows of bits), and a few randomly-selected columns with Merkle branches, get passed to the verifier. This is $O(\sqrt{N})$ data: the new row has $O(\sqrt{N})$ size, and each of the (constant number of) columns that get passed has $O(\sqrt{N})$ size.

The verifier then does a "row combination of the extension" (or rather, a few columns of the extension), and an "extension of the row combination", and verifies that the two match. They then compute a *column* combination, and check that it returns the value that the prover is claiming. And there's our proof system (or rather, the *polynomial commitment scheme*, which is the key building block of a proof system).

## What did we *not* cover?
- **Efficient algorithms to extend the rows**, which are needed to actually make the computational efficiency of the verifier $O(\sqrt{N})$ . With naive Lagrange interpolation, we can only get $O(N^{\frac23})$ . For this, we use Fast Fourier transforms over binary fields, described [here](https://vitalik.eth.limo/general/2019/05/12/fft.html) (though the exact implementation will be different, because this post uses a less efficient construction not based on recursive extension).
- **Arithmetization**. Univariate polynomials are convenient because you can do things like $F(X+2)-F(X+1)-F(X)=Z(X) \times H(X)$ to relate adjacent steps in the computation. In a hypercube, the interpretation of "the next step" is not nearly as clean as " $X+1$ ". You *can* do $X \times k$ and jump around powers of $k$ , but this jumping around behavior would sacrifice many of the key advantages of Binius. The [Binius paper](https://eprint.iacr.org/2023/1784.pdf) introduces solutions to this (eg. see Section 4.3), but this is a "deep rabbit hole" in its own right.
- **How to actually safely do specific-value checks**. The Fibonacci example required checking key boundary conditions: $F(0)=F(1)=1$ , and the value of $F(100)$ . But with "raw" Binius, checking at pre-known evaluation points is insecure. There are fairly simple ways to convert a known-evaluation check into an unknown-evaluation check, using what are called sum-check protocols; but we did not get into those here.
- **Lookup protocols**, [another technology](https://zkresear.ch/t/lookup-singularity/65) which has been recently gaining usage as a way to make ultra-efficient proving systems. Binius can be combined with lookup protocols for many applications.
- **Going beyond square-root verification time**. Square root is expensive: a Binius proof of $x^{32}$ bits is about 11 MB long. You can remedy this using some other proof system to make a "proof of a Binius proof", thus gaining both Binius's efficiency in proving the main statement *and* a small proof size. Another option is the much more complicated [FRI-Binius](https://eprint.iacr.org/2024/504) protocol, which creates a poly-logarithmic-sized proof (like [regular FRI](https://vitalik.eth.limo/general/2017/11/22/starks_part_2.html)).
- **How Binius affects what counts as "SNARK-friendly"**. The basic summary is that, if you use Binius, you no longer need to care much about making computation "arithmetic-friendly": "regular" hashes are no longer more efficient than traditional arithmetic hashes, multiplication modulo $2^{32}$ or modulo $2^{256}$ is no longer a big headache compared to multiplication modulo $p$ , and so forth. But this is a complicated topic; lots of things change when everything is done in binary.

I expect many more improvements in binary-field-based proving techniques in the months ahead.