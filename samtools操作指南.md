<a name="content">目录</a>

[samtools 操作指南](#title)

- [1. 提取比对质量高的reads](#high-quality)
- [2. 按染色体分割bam/sam文件](#split-chrom)
- [3. 提取未比对的reads](#get-unmap)
- [4. 滤除未比对的reads](#filt-unmap)
- [5. 过滤PCR重复](#filt-pcr)
- [6. 提取左右端测序数据比对到不同染色体的PE reads](#diff-map)
- [7. 根据比对结果来统计测序深度和覆盖度](#depth-coverage)
	- [7.1. 计算每条染色体平均的测序深度和覆盖度](#chrom-depth-coverage)
	- [7.2. 以WIG文件输出测序深度](#ouput-wig)
- [8. 提取multiple mapping的reads](#multiple-mapping)
- [9. samtools自带统计命令](#statistic-command)

<a name="title"><h1>samtools 操作指南</h1></a>

---

<p align="right">以下内容整理自【直播我的基因组】系列文章</p>

对sam文件的操作是基于对sam文件格式的理解：

![](/picture/sam-format.PNG "SAM-format")


<a name="high-quality"><h3>1. 提取比对质量高的reads [<sup>目录</sup>](#content)</h3></a>

```
$ samtools view -q <int> -O bam -o sample1.highQual.bam [sample1.sam|sample1.bam]
```
> `-q`设置 MAPQ (比对质量) 的阈值，只保留高于阈值的高质量部分

用MAPQ也可以用于分析 unique mapping 和 mutiple mapping [<sup>1</sup>](#1):
> 通过 **BWA** 比对得到的sam文件的第五列MAPQ值，直接通过它可以区分unique mapping和mutiple mapping的情况。在使用bwa这个软件来把测序数据比对到参考基因组的时候如果没有加上-a这个参数，那么输出的sam文件里面，bwa会对每一个有multiple mapping情况的reads的MAPQ值设置为0，所以提取unique mapping的reads是非常容易的：
> `samtools view -q 1 -O bam -o sample1.highQual.bam [sample1.sam|sample1.bam]`

<a name="split-chrom"><h3>2. 按染色体分割bam/sam文件  [<sup>目录</sup>](#content)</h3></a> 

```
for chrom in `seq 1 22` X Y MT;
do
samtools view -bh input.bam chr${chrom} | samtools sort -@ 8 -O bam -o input.chr${chrom}.bam;
samtools index input.chr${chrom}.bam;
done
```
> 根据region部分参数来指定染色体

这个功能也可以用现成的工具**bamtools**实现

```
$ bamtools split -in file.bam -reference 
```
<a name="get-unmap"><h3>3. 提取未比对的reads  [<sup>目录</sup>](#content)</h3></a>

```
$ samtools view -f4 sample.bam > sample.unmapped.sam

# 或者
$ samtools view sample.bam |perl -alne '{print if $F[2] eq "*" or $F[5] eq "*" }' > sample.unmapped.sam
```

虽然上面两个方法得到的结果是一模一样的，但是这个perl脚本运行速度远远比不上上面的samtools自带的参数。

参数说明（<font color="red">小写的 f 是提取，大写的 F 是过滤</font>）
> -f INT   only include reads with all  of the FLAGs in INT present [0]
> -F INT   only include reads with none of the FLAGS in INT present [0]

因为我们测序数据的双端的，那么sam文件的第3列是reads1的比对情况，第6列是reads2的比对情况。所以未比对成功的测序数据可以分成3类，仅reads1，仅reads2，和两端reads都没有比对成功。
也可以用下面的代码分步提取这3类未比对成功的reads:

```
# 分三步分别提取未比对的reads
samtools view -u  -f 4 -F264 alignments.bam  > unmap.tmps1.bam # 仅reads1
samtools view -u -f 8 -F 260 alignments.bam  > unmap.tmps2.bam # 仅reads2
samtools view -u -f 12 -F 256 alignments.bam > unmap.tmps3.bam # 两端reads均未比对成功

# 合并三类未必对的reads
samtools merge -u - tmps[123].bam | samtools sort -n - unmapped

# 将未必对的reads导出到fastq文件中，使用bedtools中的bamToFastq
bamToFastq -bam unmapped.bam -fq1 unmapped_reads1.fastq -fq2 unmapped_reads2.fastq
```
> FLAG值所表示的意思可以在Picard的 [**Explain SAM flags**](http://broadinstitute.github.io/picard/explain-flags.html)中检索

bamtools也可以完成这个任务：

```
$ bamtools -split -in my.bam -mapped
```
> 注意：它只考虑了PE reads**均未比对成功**的情况。

<a name="filt-unmap"><h3>4. 滤除未比对的reads [<sup>目录</sup>](#content)</h3></a>

```
$ samtools view -h -F4 input.bam
```

<a name="filt-pcr"><h3>5. 过滤PCR重复 [<sup>目录</sup>](#content)</h3></a>

```
$ samtools rmdup input.sorted.bam output.rmdup.bam
```

<a name="diff-map"><h3>6. 提取左右端测序数据比对到不同染色体的PE reads  [<sup>目录</sup>](#content)</h3></a>

sam文件的第3，7列指明了该reads比对到哪条染色体，以及该reads的配对reads比对到了哪条染色体(如果比对到同一条染色体，那么第7列是=符号)。所以我们只需要写脚本来提取即可
```
$ samtools view input.bam|perl -alne '{print if $F[6] ne "="}' 
```

<a name="depth-coverage"><h3>7. 根据比对结果来统计测序深度和覆盖度  [<sup>目录</sup>](#content)</h3></a>

这个统计主要依赖于samtools的depth功能，或者说mpileup功能，输入文件都是sort好bam格式的比对文件。<font color="red">事实上，其实depth功能调用的就是mpileup的函数。但是mpileup可以设置一系列的过滤参数。而depth命令是纯天然的，所以mpileup的结果一定会小于depth的测序深度。</font>

对mpileup，可以不选择-u -f 参数指定参考基因组，因为我们只需要测序深度情况，还有，可以指定-q 1 来过滤掉多比对情况。还可以用 -Q 来过滤低质量的碱基(base pair)，用 -A 来过滤无法定位的reads。

```
$ samtools mpileup -A input.bam | head
```

<a name="chrom-depth-coverage"><h4>7.1. 计算每条染色体平均的测序深度和覆盖度</h4></a>

可以写脚本来计算每条染色体平均的测序深度和各个染色体的覆盖度：

```
nohup time samtools mpileup input.bam | perl -alne '{$pos{$F[0]}++;$depth{$F[0]}+=$F[3]} END{foreach sort keys %pos {print "$_\t$pos{$_}\t$depth{$_}"}}' 1>coverage_depth.txt 2>coverage_depth.log &
```

以上可以得到每条染色体所有位点覆盖度和测序深度的总和，将它们处于染色体的长度就得到了每条染色体平均的测序深度和各个染色体的覆盖度，其中**chromosome的长度**在bam文件里面可以看到，用`samtools view -H input.bam` 即可。

<a name="output-wig"><h4>7.2. 以WIG文件输出测序深度  [<sup>目录</sup>](#content)</h4></a>

```
samtools depth SRR1042593.sorted.bam | perl -ne 'BEGIN{ print "track type=print wiggle_0 name=SRR1042593 description=SRR1042593\n"}; ($c, $start, $depth) = split; if ($c ne $lastC) { print "variableStep chrom=$c span=10\n"; };$lastC=$c; next unless $. % 10 ==0;print "$start\t$depth\n" unless $depth<3' > SRR1042593.wig

samtools depth SRR1042594.sorted.bam | perl -ne 'BEGIN{ print "track type=print wiggle_0 name=SRR1042594 description=SRR1042594\n"}; ($c, $start, $depth) = split; if ($c ne $lastC) { print "variableStep chrom=$c span=10\n"; };$lastC=$c; next unless $. % 10 ==0;print "$start\t$depth\n" unless $depth<3' > SRR1042594.wig
```

> `$.` 上次阅读的文件的当前输入行号

<a name="multiple-mapping"><h3>8. 提取multiple mapping的reads  [<sup>目录</sup>](#content)</h3></a>

BWA对multiple mapping的处理策略：

> 从中选择一个最好的，如果有两个或以上最好的比对位置，则从中随机选择一个

在sam文件的第五列是MAPQ值，对某些软件，比如BWA，直接通过它就可以区分unique mapping和mutiple mapping的情况，就是判断这条reads是否只比对到参考基因组的唯一的位点。所以对于这种情况下的sam文件，提取unique mapping非常简单，用samtools view -q 1 （过滤掉MAPQ值低于1的情况）

<p align="center"><img src=./picture/samtools-note-multiple-mapping.jpg width=500 /></p>

<a name="statistic-command"><h3>9. samtools自带统计命令  [<sup>目录</sup>](#content)</h3></a>

- `samtool idxstats`

```
samtools idxstats in.sam|in.bam|in.cram
```
检索和打印与输入文件相对应的index file里的统计信息，在使用idxstats之前需要用`samtools index`对输入的bam文件建索引

输出形式如下：

```
#reference sequence name	sequence length	mapped reads	unmapped reads
chr10   130694993       2371861 5312
chr11   122082543       2270187 6192
chr12   120129022       2264479 5378
chr13   120421639       2217186 5182
chr14   124902244       2247612 5225
chr15   104043685       1839099 4404
chr16   98207768        1701380 4694
chr17   94987271        1723899 5425
chr18   90702639        1606990 3973
chr19   61431566        1082557 2808
chr1    195471971       3459553 8376
```

- `samtools flagstat`

```
samtools flagstat <in.bam> |<in.sam> | <in.cram>
```

统计输入文件的相关数据并将这些数据输出至屏幕显示。每一项统计数据都由两部分组成，分别是QC pass和QC failed，表示通过QC的reads数据量和未通过QC的reads数量。以“PASS + FAILED”格式显示

输出形式如下：

```
85207970 + 0 in total (QC-passed reads + QC-failed reads)
5350668 + 0 secondary
0 + 0 supplementary
0 + 0 duplicates
73549948 + 0 mapped (86.32% : N/A)
79857302 + 0 paired in sequencing
39928651 + 0 read1
39928651 + 0 read2
62977510 + 0 properly paired (78.86% : N/A)
63902424 + 0 with itself and mate mapped
4296856 + 0 singletons (5.38% : N/A)
173078 + 0 with mate mapped to a different chr
127302 + 0 with mate mapped to a different chr (mapQ>=5)
```


---

参考资料：

(1) <a id=1>[【直播】我的基因组（十七）:初步分析一下multiple mapping 的情况](https://mp.weixin.qq.com/s?__biz=MzAxMDkxODM1Ng==&mid=2247483789&idx=1&sn=db86148a1b7e97818ddfe05fbfdfd8ab&chksm=9b484136ac3fc820023e53654f375912df4d800fbe2faa5ecd162c2002b48c2486061a404f17&scene=21#wechat_redirect)</a>

(2) <a id=2>[【直播】我的基因组（十四）:bam文件给按照染色体给分割成小文件 ](https://mp.weixin.qq.com/s?__biz=MzAxMDkxODM1Ng==&mid=2247483779&idx=1&sn=1ca34ed7b5ef9a3cf1835aa59d8501cc&chksm=9b484138ac3fc82e5de532dd118cec8e17ae2279eff3c3ee638ad0e5a307999dba1c289a8981&scene=21#wechat_redirect)</a>

(3) <a id=3>[【直播】我的基因组（十五）:提取未比对的测序数据 ](https://mp.weixin.qq.com/s?__biz=MzAxMDkxODM1Ng==&mid=2247483782&idx=1&sn=a2c7bac48e03195b646ae34f3b82ac29&chksm=9b48413dac3fc82b61f9868c4385609af3eba9bf68dc026ab78a3f14980ce750cb35623195ce&scene=21#wechat_redirect)</a>

(4) <a id=4>[【直播】我的基因组（十六）:提取左右端测序数据比对到不同染色体的PE reads](https://mp.weixin.qq.com/s?__biz=MzAxMDkxODM1Ng==&mid=2247483786&idx=1&sn=8792e5ab7ae32c72e594f93775322ce8&chksm=9b484131ac3fc827577c7b1286d3871d5d601e921be606e8d284b8508dd59254cad40c0e5b49&scene=21#wechat_redirect)</a>

(5) [【直播】我的基因组23：对比对结果文件进行过滤 ](https://mp.weixin.qq.com/s?__biz=MzAxMDkxODM1Ng==&mid=2247483810&idx=1&sn=90bc7ba1cd00350a6ed8285618d4f9ae&chksm=9b484119ac3fc80fce7b52d98209603fec8340bf5b77fad92edc97347cdffc3d9d844d9c7a13&scene=21#wechat_redirect)

(6) <a id=5>[【直播】我的基因组19：根据比对结果来统计测序深度和覆盖度](https://mp.weixin.qq.com/s?__biz=MzAxMDkxODM1Ng==&mid=2247483801&idx=1&sn=f7a68e5be71b66faae9b6b40a3d13df7&chksm=9b484122ac3fc8347d764e9bd5f423183d0bc9c361bbc0105dc6c5add336b02b0eafc471202d&scene=21#wechat_redirect)</a>

(7) <a id=6> [使用CEAS软件来对CHIP-seq的peaks进行](http://www.bio-info-trainee.com/1779.html)</a>
