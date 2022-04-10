<a name="content">目录</a>

[文献调研：RNA editing](#title)
- [背景知识](#background)
- [*Sci.Rep*: Symmetrical RNA Editing Events in the Mitochondria of Salvia miltiorrhiza](#sci-rep)
	- [REDItools](#reditools)
- [*Nature*: Dynamic landscape and regulation of RNA editing in mammals](#nature)
	- [科普：mmPCR-seq](#mmpcr-seq)
	- [基于GATK4的分析流程](#workflow-base-on-gatk4)
- [*Nat.Meth*: Genome sequence–independent identification of RNA editing sites](#nat-meth)
	- [科普：互信息](#what-is-mi)
	- [GIREMI](#giremi)
		- [原理](#giremi-principle)
		- [用法](#giremi-usage)




<h1 name="title">文献调研：RNA editing</h1>

<a name="background"><h2>背景知识 [<sup>目录</sup>](#content)</h2></a>

**RNA editing**: an important post-transcriptional mechanism that alters primary RNAs through the **insertion/deletion** or **modification of specific nucleotides**

脱氨基作用 (deamination)： 如 C->U，或腺苷脱氨酶 (adenosine deaminase (ADAR) family)作用于dsRNA导致 A->I

影响：

- **UTRs**：altered expression, preventing efficient ribosome binding or recognition by small regulatory RNAs
- **coding protein regions**：**amino acid replacements** with variable functional consequences
- In addition： influence the **activity of ncRNAs** such as siRNAs, miRNAs and potentially of piwiRNAs by affecting base-pairing interactions within RNA secondary structures

鉴定原理：

很简单，即将转录本与其对应的基因组序列进行比较，但是要在整个基因组范围内实现准确鉴定仍然充满挑战：**在存在测序错误与mapping不准确的干扰下，怎样从基因组范围内的SNPs中鉴定出真正的RNA editing位点？**

解决方法之一：use DNA-Seq data from single individuals, annotations in dbSNPs and several stringent filters

<a name="sci-rep"><h2>*Sci.Rep*: Symmetrical RNA Editing Events in the Mitochondria of Salvia miltiorrhiza [<sup>目录</sup>](#content)</h2></a>

<p align="center"><img src=./picture/RNA-editing-sci-rep-pipeline.png width=500 /></p>

**参数优化**：maping时bowtie选用的mismatch参数的大小？
- 小的mismatch：低估了RNA-editing位点数
> 1. 许多基因的RNA editing位点十分靠近
> 2. 平均100bp的reads有3.4个位点

- 大的mismatch：高假阳性

**解决策略一**："assembly-base"，用trinity进行拼接`‘-SS_lib_type FR’ `

**解决策略二**："mapping-based"，尝试不同的mismatch值，2~10，最终选择最佳值7

<a name="reditools"><h3>REDItools [<sup>目录</sup>](#content)</h3></a>

包含三个主要脚本，用于处理来自同一样本/个体的DNA-seq和RNA-seq数据

- **REDItoolDnaRNA.py**：检测候选的RNA editing位点，通过比较pre-aligned RNA-Seq 和 DNA-Seq reads（BAM format）获得

实现步骤：

> 1\. 对RNA-seq数据，逐一扫描基因组位点并返回一个表格，表格中包含
> 
> - coverage depth
> - the mean quality score
> - the observed base distribution
> - the strand if available
> - the list of observed substitutions as well as the frequency of variation
> 
> 2\. 若提供了DNA-seq数据，获得与步骤1中相似的表格数据，用于之后除去潜在的SNPs
> 
> 3\. 对一些位点按照一定规则进行过滤： read coverage, base quality score, mapping quality, bases supporting the variation, type of substitution and frequency
> 
> 4\. 去除一些位点位于：
>
> - homopolymeric regions of predefined length
> - intronic sequences surrounding known splice sites
> - invariant RNA-Seq positions
> - sites not supported by DNA-Seq
> - positions near read ends

- **REDItoolKnown.py**：explore the RNA editing potential of RNA-Seq experiments by looking at known events only

- **REDItoolDenovo.py**：不需要重测序数据，只利用RNA-seq数据进行RNA editiong的denovo检测

<a name="nature"><h2>*Nature*: Dynamic landscape and regulation of RNA editing in mammals [<sup>目录</sup>](#content)</h2></a>

研究成果：
- dynamic spatiotemporal patterns
- new regulators of RNA editing
- discovered through an **extensive profiling of A-to-I RNA editing** from the Genotype-Tissue Expression (**GTEx**) project and in hundreds of other primate and mouse samples

结论：
- 与重复coding区域相比，非重复coding区域的RNA editing level在不同组织之间变化比较大
- ADAR1主要编辑repetitive coding sites，ADAR2主要编辑non-repetitive coding sites，而ADAR3能抑制RNA editing

<a name="mmpcr-seq"><h3>科普：mmPCR-seq [<sup>目录</sup>](#content)</h3></a>

a targeted RNA sequencing method that couples **microfluidics-based multiplex PCR** and **deep sequencing**

常规RNA-seq存在的问题：
> - the large dynamic range of RNA expression, which leads to inaccurate quantification of allelic ratios for genes with low-to-moderate expression levels
> - 即RNA丰度差异较大，对于中低丰度的RNA的定量不准

mmPCR-seq优点：
> - uniformly and simultaneously amplify up to 960 loci in 48 samples independently of their gene expression levels and to accurately and cost-effectively measure allelic ratios even for low-quantity or low-quality RNA samples
> 
> - 即成比例扩增RNA片段，而不影响基因表达水平的相对定量，同时能提高对低丰度RNA的灵敏度

<p align="center"><img src=./picture/RNA-editing-nature-mmPCR-seq.png width=600 /></p>

这个测序技术的关键在于进行类似454测序中用到的乳化PCR，即让每个RNA片段处于一个独立的PCR反应环境中

<a name="workflow-base-on-gatk4"><h3>基于GATK4的分析流程 [<sup>目录</sup>](#content)</h3></a>

**1\. Mapping of RNA-seq and mmPCR–seq reads**

**Mapping**：用BWA将RNA-seq reads mapping到reference genome和已知的剪接区域附近的exonic sequences

**过滤**：过滤mapping结果，保留高质量的mapping结果（uniquely mapped且q > 10），并用samtools rmdup过滤PCR重复

**重比对与重校正**：用GATK中的 IndelRealigner 和 TableRecalibration 对保留下来的高质量的Unique reads进行局部重比对（local realignment）和碱基值重校正（base score recalibration），

**2\. Identification of editing sites from RNA-seq data**

**variant calling**：用GATK中的UnifiedGenotyper来call variants，与普通的variant calling不同，这里采用了比较宽松的选项：stand_call_conf 0, stand_emit_conf 0, and output mode EMIT_VARIANTS_ONLY

**remove all known SNPs**：利用dbSNP数据，过滤已知的SNPs

**remove false positive variant calls**：过滤因技术操作原因导致的variant calling中的假阳性结果
- required a variant call quality q > 20
- 若variants落在read的头6个碱基里，过滤掉
- 除去落在重复区域的variants
- 过滤intron中离剪接位点4bp范围内的variants

<a name="nat-meth"><h2>*Nat.Meth*: Genome sequence–independent identification of RNA editing sites [<sup>目录</sup>](#content)</h2></a>

当前RNA editing位点鉴别存在的挑战：
- 需要来自同一样本的genome sequence data 来过滤SNPs的影响
- 即使提供了genome sequence data，但是由于测序覆盖度（sequencing coverage）不一致等原因，使得仍然无法完全去除SNPs的干扰

> 其他不需要genome sequence data的方法：use multiple RNA-seq data sets to increase the confidence of finding individual sites
>
> 存在的问题：this precludes analysis of single data sets and may miss unique changes

开发的新工具：[**GIREMI**](https://www.ibp.ucla.edu/research/xiao/GIREMI.html)

<p align="center"><img src="https://www.ibp.ucla.edu/research/xiao/GIREMI_files/CoverArt.GIREMI.NMETH-BC22074E.png" width=300 /></p>

优点：不需要genome sequence即可进行RNA editing位点的准确鉴定，即使RNA-seq dataset只有较低的测序深度

<a name="what-is-mi"><h3>科普：互信息 [<sup>目录</sup>](#content)</h3></a>

- 首先要知道什么是信息熵

<p align="center"><img src=./picture/RNA-editing-nat-meth-introduction-of-mi-formula-information-entropy.png width=300 /></p>

一条信息的信息量与其不确定性有着直接的关系。信息熵是消除不确定性所需信息量的度量，所以可以认为，信息量就等于不确定性的多少。

当存在n种可能的选择，如果每种选择等可能，即为1/n，则此时要做出选择没有任何其他可参考的信息，几乎就是瞎猜，则此时不确定性最大，则此时的信息熵也是最大，为log(n)，所以理论上：

<p align="center"><strong>max H(X) = log(n)</strong></p>

- 接着来讨论什么是互信息

互信息，Mutual Information，缩写为MI，表示两个变量X与Y是否有关系，以及关系的强弱。

<p align="center"><img src=./picture/RNA-editing-nat-meth-introduction-of-mi-formula.png width=400 /></p>

<p align="center"><img src=./picture/RNA-editing-nat-meth-introduction-of-mi-formula-derivation.png width=800 /></p>

可以看出，I(X,Y)可以解释为由X引入而使Y的不确定度减小的量，这个减小的量为H(Y|X)，，称为条件熵

<p align="center"><img src=./picture/RNA-editing-nat-meth-introduction-of-mi-formula-conditional-entropy.png width=400 /></p>

性质：
> - 如果X，Y关系越密切，I(X,Y)就越大
>
> - I(X,Y)最大的取值是H(Y)，此时H(Y|X)为0，意义为X和Y完全相关，在X确定的情况下Y是个定值，没有出现其他不确定情况的概率，所以为H(Y|X)为0
>
> - I(X,Y)取0时，代表X与Y独立，此时H(Y)=H(Y|X)，意义为X的出现不影响Y

互信息的物理解释：

<p align="center"><img src=./picture/RNA-editing-nat-meth-introduction-of-mi-graphic-explain.jpg width=600 /></p>

<a name="giremi"><h3>GIREMI [<sup>目录</sup>](#content)</h3></a>

<a name="giremi-principle"><h4>原理 [<sup>目录</sup>](#content)</h4></a>

鉴别RNA-editing/SNP的原理：

<table>
<tr>
	<td><img src=./picture/RNA-editing-nat-meth-GIREMI-principle.png width=500 /></td>
	<td><img src=./picture/RNA-editing-nat-meth-mi-compare.png width=500 /></td>
</tr>
</table>

> A pair of SNPs in the same read (or read pair, in paired-end sequencing) maintains the same haplotype in the RNA as in reference genomic DNA
>
> In contrast, a SNP and an RNA editing site exhibit variable allelic linkage because RNA editing occurs post-transcriptionally to either allele randomly
>
> the allelic linkage for a pair of RNA editing sites may also appear random

1\. Mapping of RNA-seq reads

特殊的mapping策略：enable unbiased mapping of alternative RNA alleles corresponding to RNA editing or expressed SNPs
> - 用bowtie和blat去mapping参考基因组，用bowtie去mapping转录组
> - 将三种方法的mapping结果merge在一起，形成union
> - 对union的结果进行过滤，将满足以下要求的mapped reads保留下来：<br>
>   1. 最多允许有n<sub>1</sub>个mismatches的情况下，reads有唯一的map
>   2. 最多允许有n<sub>2</sub>个mismatches的情况下(n<sub>2</sub>\>n<sub>1</sub>)，不mapping到其他位置上
>
>  n<sub>2</sub>和n<sub>1</sub>一般分别设置为reads长度的5%和12%

2\. 识别与过滤mismatches
> - pile up
> - remove duplicate reads, except the one with the highest-quality score at the mismatch position
> - 保留满足以下条件的mismatch位点：
>   1. coverage ≥5
>   2. 差异位点至少出现在3条reads中

3\. GIREMI：用机器学习方法来进行MI（Mutual information，互信息）的统计推断，并依此来预测RNA editing sites

Input
> - list of SNVs (mismatches)
> - known SNPs

Output
> - predicted RNA editing sites
> - their editing levels

4\. 计算SNVs和RNA editing位点的MI

> 1\. 从RNA-seq数据中提取出至少含有两个SNPs的reads
>
> 2\. 用genome-dependent的方法，从RNA-seq数据中预测得到RNA editing位点
> 
> 3\. 用最大似然法，计算P(S<sub>i</sub>)，P(S<sub>j</sub>) 和 P(S<sub>i</sub>,S<sub>j</sub>)，然后就是MI：
>
> <img src=./picture/RNA-editing-nat-meth-mi-formula-pair-site.png width=300 />
> <br>
> <img src=./picture/RNA-editing-nat-meth-mi-formula-per-site.png width=150 />

<a name="giremi-usage"><h4>用法 [<sup>目录</sup>](#content)</h4></a>

该工具底层依赖的工具：

- HTSlib：这是用于对SAM/BAM文件进行读写操作的库

	若未安装好该库，请安装，使用conda可以实现一步式安装`conda install htslib`；若安装成功，请将库文件所在的路径添加进 `$LD_LIBRARY_PATH` 环境变量
	
	查看 `$LD_LIBRARY_PATH` 环境变量
	
	```
	$ echo $LD_LIBRARY_PATH
	
	/home/miniconda2/lib/
	```
	
	修改 `$LD_LIBRARY_PATH` 环境变量
	
	```
	$ export LD_LIBRARY_PATH=/Path/To/lib:$LD_LIBRARY_PATH
	```
	
	由于版本更新的原因，可能出现GIREMI内部默认使用的htslib和当前新系统中安装的htslib不一致的问题，具体体现在执行 `perl giremi.pl` 时，出现以下报错信息：
	
	`./giremi: error while loading shared libraries: libhts.so.1: cannot open shared object file: No such file or directory`
	
	解决方法：找到报错信息中指明的对应的htslib库文件，将当前库文件的文件名，比如是 **libhts.so.1.8** 改为 **libhts.so.1** ，简单粗暴 ╮(～▽～)╭
	
	最后执行 `perl giremi.pl` ，查看配置是否成功，若输出帮助文档信息，则说明配置成功
	
- samtools：用于构建参考基因组的faidx索引

	在运行GIREMI前，请提前用 `samtools faidx`命令构建好参考基因组的faidx索引，而且要保证参考基因组的fasta文件与faidx文件要位于同一文件夹下
	
- R：用于GLM（广义线性模型）的训练与预测

Usage：

```
$ giremi [options] in1.bam [in2.bam [...]]
```

重要参数：

> -  -f, --fasta-ref FILE &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; reference genome sequence file in fasta format 
> - -m, --min INT &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; minimal number of total reads covering candidate editing sites  [default: 5]
> - -p, --paired-end INT &nbsp;&nbsp; 1:paired-end RNA-Seq reads; 0:single-end [default: 1]

输出结果的格式说明：

<p align="center"><img src=./picture/RNA-editing-GIREMI-output-format.png width=800 /></p>

参考资料：

(1) Ernesto Picardi, Graziano Pesole; REDItools: high-throughput RNA editing detection made easy.[J]. Bioinformatics, 2013, 29:1813–1814.

(2) Wu B, Chen H, Shao J, et al. Identification of Symmetrical RNA Editing Events in the Mitochondria of Salvia miltiorrhiza by Strand-specific RNA Sequencing.[J]. Scientific Reports, 2017, 7:42250.

(3) Tan M H, Li Q, Shanmugam R, et al. Dynamic landscape and regulation of RNA editing in mammals[J]. Nature, 2017, 550(7675):249-254.

(4) Rui Z, Xin L, Ramaswami G, et al. Quantifying RNA allelic ratios by microfluidic multiplex PCR and sequencing[J]. Nature Methods, 2014, 11(1):51.

(5) Zhang Q, Xiao X. Genome Sequence-Independent Identification of RNA Editing Sites[J]. Nature Methods, 2015, 12(4):347.

(6) [CSDN博客：互信息（Mutual Information）的介绍](https://blog.csdn.net/lk7688535/article/details/52529610)

(7) [GIREMI官方主页](https://www.ibp.ucla.edu/research/xiao/GIREMI.html)

