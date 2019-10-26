# README

## How to run
python jddc_eval.py ./data/dev_question.txt ./out/test.txt

## How to run bert
```
--task_name=jd --vocab_file="./JDAI-BERT/vocab.txt" --bert_config_file="./JDAI-BERT/bert_config.json" --output_dir="./out" --do_eval=False --do_predict=True --data_dir=“./JDAI-BERT”
```

### build requirment
```
pipreqs --encoding=utf-8 ./
```

## score in Online Judge
1. 0.0618: 10per bert 无停用词 k = 30 中心  只输入q 不替换
1. 0.058596: 10per elmo 无停用词 k = 30 中心
2. 0.058389: 10per tfidf 无停用词 k = 30 中心
1. 0.056447: 1per elmo 无停用词 k = 30 中心
1. 0.05353: 10per tfidf 无停用词 k = 20 中心
2. 0.050728: 10per tfidf 无停用词 k = 15 中心
2. 0.035051：100per tfidf 无停用词 k = 10 中心
1. 0.026642：0.1per elmo k = 30 bert
1. 0.021707: 10per tfidf 都用停用词 k = 15 中心


## 目录结构：
- data
- code
- resource

Just put following dir in /code:
/ELMo
/DAM
/JDAI-BERT
/JDAI-WORD-EMBEDDIN1G

ELMo:https://github.com/HIT-SCIR/ELMoForManyLangs  
DAM:https://github.com/baidu/Dialogue/tree/master/DAM  
JDAI-BERT， JDAI-WORD-EMBEDDIN1G:https://github.com/jd-aig/nlp_baai  
SMN_Pytorch：https://github.com/MaoGWLeon/SMN_Pytorch


## 数据EDA以及预处理
### EDA
1. 句子长短分布
2. 对话轮数分布
3. 异常数据分析
3. 常见对话及其所占比例

### 预处理
1. 将不同订单号和链接用同一特殊字段代替，去掉表情
2. 将连续的多个Q、多个A的句子合并成一句
3. 去掉空白的、只有符号或表情的句子
4. 去掉长度超过100的句子

## 检索式模型
The Dialog Systems Technology Challenges 7 (DSTC7)定义了5个子问题：

No | Subtask Description
-|-
1 | Select the next response from a set of 100 choices that
contains 1 correct response
2 | Select the next response from a set of 120,000 choices
3 | Select the next response(s) from a set of 100 choices
that contains between 1 and 5 correct responses
4 | Select the next response or NONE from a set of 100
choices that contains 0 or 1 correct response
5 | Select the next response from a set of 100 choices, given
access to an external dataset

对于本次大赛的检索式的对话模型，我们重点关注问题1和2——如何从大量对话数据中选取top k的少量候选数据？以及如何使用更加精准的重排（rerank）模型，从候选数据中选取最为匹配的答案。
因此我们的检索式模型分为两大部分——粗筛模块和重排模块。
而因为模型面临的场景是多轮会话，对于每个模块需要考虑是否引入历史信息。

### 粗筛模块
1. 根据用户的问题与语料库中的问题，计算语句基于词或字的embedding
2. 将词或字的embedding通过加权得到句子的embedding
3. 将问题的sentence embedding和语料库中的，计算余弦相似度
4. 选取top k的相似问题，将k个问题和答案都传入重排模块

#### 是否使用对话历史
经过反复验证，结论是不使用：
1. 句向量的表示能力有局限性。随着句子长度的增加，未必能表示出整个句子的关键信息
2. 在当前问题的基础上附加过长的历史信息会导致词向量中当前问题的信息保留较少，检索时重心偏于历史信息的表示，而忽略了当前问题
3. 无法解决对话中“话题漂移”的现象，即之前的历史并不能代表当前问题的语境。此时的检索相当于引入了错误的信息。

#### 粗筛embedding
我们尝试了以下各种embedding：
1. bow
2. tfidf
3. lsi
4. lda
5. 电商领域基于skip-gram的字embedding
6. 基于上下文的elmo词embedding
7. 基于上下文的bert字embedding

优缺点概述：
1. tfidf是一个有稳定表现的无参数向量化表示方法，有着较好的表示效果。
2. lsi和lda的表示效果受到主题数量选取的影响很大，而且随着数据量的指数增大，表示效果不能提升。
3. 基于skip-gram的字embedding，表示效果优于tfidf。虽然没有包含上下文信息，但是考虑到模型的应用场景为电商领域，语言的多义性会较少，而且运行速度快，计算字向量时查表即可。
4. elmo得益于其对上下文语义的理解，生成的词向量能够包含上下文的语义，效果好于tfidf和skip-gram。但是缺点也同样是因为对于不同上下文的同一个词都有不同的向量表示，随着语料集的指数扩大，语料集生成词向量的过程需要非常耗时。
5. bert的优缺点和elmo类似，虽然能够建模上下文信息，但是所有向量需要长时间生成，大量空间储存。
综上所述，我们最后选择了电商领域基于skip-gram的字embedding。

### 重排模块
1. 将候选的top k个答案传入重拍模型
2. 重拍模型输出一个最优的答案

#### 是否使用对话历史
使用：
通过引入神经网络模型，模型对对话历史的建模能力大大提升，从而历史中的信息可以有效和正确地被利用。

#### 重排模型的选取
我们尝试了以下重排模型：
1. 中心选取
3. SMN
2. bert

优缺点概述：
1. 中心选取的方法是假设top k答案大多数都是比较匹配的回答，在top k中选取和所有答案相似度最高的、共性最强的
2. SMN虽然能够对历史信息进行建模，但是模型设计以现在的眼光来看比较粗糙，提取能力不强
3. bert作为当下最流行的模型，通过海量预训练数据，有着强大的语言理解能力。通过全量对话数据的fine tuning，能够有效结合历史信息和当前问题。

## score in test set
在整个实验过程中，我们依次确定了一下参数：
1. 是否使用停用词 —— 不使用
2. 选择什么样的unsupervised reranker —— 中心
3. topk的k应该设置成多大 —— 30
4. lsi和lda的embedding —— 被elmo完爆
5. 是否输入全部数据，或者只输入q —— 在所有模型上，只输入q都有提升
5. 是否在embedding时进行特殊字符的替换 —— 不替换
6. bert embedding ——
7. bert reranker ——
8. 对于bert的k ——



Note：
1. ~~bert答案不稳定~~
2. elmo的答案也不稳定
3. 可能是因为batch的padding


Note：下面的表格没有按照上述顺序。

### task dialog
k=30 中心 只输入q  

train size | embedding | step | no task | task
-|-|-|-|-
1per | tfidf | - | 0.014776057987866967 | 0.010788855903886355
10per | tfidf | - |  |

### bert reranker
k=30 中心 只输入q  

train size | embedding | step | ur score | bert score
-|-|-|-|-
1per | bert | 63000 | 0.014442738292227185 | 0.014048221282771974
1per | bert | 171000 |  |

### bert embedding
k=30 中心 只输入q  

train size | embedding | reranker | score
-|-|-|-
0.1per | bert | ur | 0.011230678772122138
1per | bert | ur | 0.013131209502587263
10per | bert | ur | 0.015230553124618745

### elmo
k=30 中心 只输入q  

train size | embedding | reranker | score
-|-|-|-
0.1per | elmo | ur | 0.010975129839875232
1per | elmo | ur | 0.013982037996709435
5per | elmo | ur | 0.01268805555856596

### 是否输入全部数据
使用不替换数据 k=30 中心  

train size | embedding | 是否全量数据 | score
-|-|-|-
0.01per | tdidf | 只有q | 0.014383755741864323
0.01per | tdidf | 全部数据 | 0.012997054348273201
0.01per | elmo | 只有q | 0.015390332452465709
0.01per | elmo | 全部数据 | 0.010437204998100533
1per | tdidf | 只有q | 0.013483501888183026
1per | tdidf | 全部数据 | 0.012848696840693473

### unsupervised reranker
k = 30  

train size | embedding | 选取方式 | score
-|-|-|-
1per | tfidf | 第一个 | 0.004963330927578965
10per | tfidf | 第一个 | 0.005813867175527974
1per | tfidf | 中心 | 0.013600463289462657
10per | tfidf | 中心 | 0.01278348452600705
0.01per | elmo | 第一个 | 0.006319689484710318
0.01per | elmo | 中心 | 0.013490131245278697

### 停用词调参
#### 总结
事实证明，tfidf不需要使用停用词。其他模型应该也不需要。

#### 1per
 train size | 停用词 | k | 选取方式 | score
-|-|-|-|-
1per | 全不使用停用词 | 10 | 中心 | 0.008554900692186014
1per | TFIDF使用 | 10 | 中心 | 0.007695293483604558
1per | 检索时使用 | 10 | 中心 | 0.007417600955727696
1per | 聚类时使用 | 10 | 中心 | 0.008527690838414773
1per | 都使用 | 10 | 中心 | 0.006643151597518594

 train size | 停用词 | k | 选取方式 | score
-|-|-|-|-
1per | 全不使用停用词 | 15 | 中心 | 0.010930076940092955
1per | TFIDF使用 | 15 | 中心 |
1per | 检索时使用 | 15 | 中心 |
1per | 聚类时使用 | 15 | 中心 |
1per | 都使用 | 15 | 中心 |

 train size | 停用词 | k | 选取方式 | score
-|-|-|-|-
1per | 全不使用停用词 | 20 | 中心 | 0.01490869784672498
1per | TFIDF使用 | 20 | 中心 | 0.01346708503468178
1per | 检索时使用 | 20 | 中心 | 0.00850552305526961
1per | 聚类时使用 | 20 | 中心 | 0.007632402631464895
1per | 都使用 | 20 | 中心 | 0.006743577323141914

 train size | 停用词 | k | 选取方式 | score
-|-|-|-|-
1per | 全不使用停用词 | 25 | 中心 | 0.011331763821217396
1per | TFIDF使用 | 25 | 中心 |
1per | 检索时使用 | 25 | 中心 |
1per | 聚类时使用 | 25 | 中心 |
1per | 都使用 | 25 | 中心 |

### k和停用词调参
#### 总结
tfidf确实不需要停用词。不过对于k来说，随着数据集的增大，有必要相应增大。

#### 10per
 train size | 停用词 | k | 选取方式 | score
-|-|-|-|-
10per | 全不使用停用词 | 10 | 中心 | 0.008675446566447679
10per | 都使用 | 10 | 中心 |
10per | 全不使用停用词 | 15 | 中心 | 0.010576244561356318
10per | TFIDF使用 | 15 | 中心 |
10per | 检索时使用 | 15 | 中心 |
10per | 聚类时使用 | 15 | 中心 |
10per | 都使用 | 15 | 中心 |
10per | 全不使用停用词 | 20 | 中心 | 0.010226410024601663
10per | 都使用 | 10 | 中心 |  
10per | 全不使用停用词 | 25 | 中心 | 0.012067605975253125
10per | 都使用 | 10 | 中心 |  
10per | 全不使用停用词 | 30 | 中心 | 0.01278348452600705 （虽然分数没有下降，但人眼可见的质量下降）
10per | 都使用 | 10 | 中心 |  
10per | 全不使用停用词 | 50 | 中心 | 0.01442813116502754 （开始出现大量通用性高的重复回答）
10per | 都使用 | 10 | 中心 |  

### 主题模型topic_num调参
#### lsi调参
k = 30  

 train size | model | topic_num | score
-|-|-|-
1per | lsi | 20 | 0.01071816672917079
1per | lsi | 30 | 0.01104208774490005
1per | lsi | 40 | 0.013137852396458436
1per | lsi | 60 | 0.012325831746581317
1per | lsi | 80 | 0.011505039619989454
1per | lsi | 120 | 0.011522613952786432
10per | lsi | 20 | 0.010151781262281053
10per | lsi | 40 | 0.008701707439616796
10per | lsi | 60 | 0.011182269704425877

k = 20  

 train size | model | topic_num | score
-|-|-|-
10per | lsi | 40 | 0.008171042618110087
10per | lsi | 60 | 0.010761877752782888


#### lda调参
k = 20  

 train size | model | topic_num | score
-|-|-|-
10per | lda | 40 | 0.008900965387194669
10per | lda | 60 | 0.007588860032794996


k = 30  

 train size | model | topic_num | score
-|-|-|-
10per | lda | 40 | 0.008900965387194669
10per | lda | 60 | 0.01256071632922341
10per | lda | 80 | 0.006515710589744413
