# 使用预训练的词向量完成文本分类任务

**作者**: [fiyen](https://github.com/fiyen)<br>
**日期**: 2021.10<br>
**摘要**: 本示例教程将会演示如何使用飞桨内置的Imdb数据集，并使用预训练词向量进行文本分类。

## 一、环境设置
本教程基于Paddle 2.2.0-rc0 编写，如果你的环境不是本版本，请先参考官网[安装](https://www.paddlepaddle.org.cn/install/quick) Paddle 2.2.0-rc0。


```python
import paddle
from paddle.io import Dataset
import numpy as np
import paddle.text as text
import random

print(paddle.__version__)
```

    2.2.0-rc0


## 二、数据载入

在这个示例中，将使用 Paddle 2.2.0-rc0 完成针对 Imdb 数据集（电影评论情感二分类数据集）的分类训练和测试。Imdb 将直接调用自 Paddle 2.2.0-rc0，同时，
利用预训练的词向量（[GloVe embedding](http://nlp.stanford.edu/projects/glove/)）完成任务。


```python
print('自然语言相关数据集：', paddle.text.__all__)
```

    自然语言相关数据集： ['Conll05st', 'Imdb', 'Imikolov', 'Movielens', 'UCIHousing', 'WMT14', 'WMT16']



由于 Paddle 2.2.0-rc0 提供了经过处理的Imdb数据集，可以方便地调用所需要的数据实例，省去了数据预处理的麻烦。目前， Paddle 2.2.0-rc0 以及内置的高质量
数据集包括 Conll05st、Imdb、Imikolov、Movielens、HCIHousing、WMT14 和 WMT16 等，未来还将提供更多常用数据集的调用接口。

以下定义了调用 imdb 训练集合测试集的方法。其中，cutoff 定义了构建词典的截止大小，即数据集中出现频率在 cutoff 以下的不予考虑；mode 定义了返回的数据用于何种用途（test: 测试集，train: 训练集）。

### 2.1 定义数据集


```python
imdb_train = text.Imdb(mode='train', cutoff=150)
imdb_test = text.Imdb(mode='test', cutoff=150)
```

调用 Imdb 得到的是经过编码的数据集，每个 term 对应一个唯一 id，映射关系可以通过 imdb_train.word_idx 查看。将每一个样本即一条电影评论，表示成 id 序列。可以检查一下以上生成的数据内容：


```python
print("训练集样本数量: %d; 测试集样本数量: %d" % (len(imdb_train), len(imdb_test)))
print(f"样本标签: {set(imdb_train.labels)}")
print(f"样本字典: {list(imdb_train.word_idx.items())[:10]}")
print(f"单个样本: {imdb_train.docs[0]}")
print(f"最小样本长度: {min([len(x) for x in imdb_train.docs])};最大样本长度: {max([len(x) for x in imdb_train.docs])}")
```

    训练集样本数量: 25000; 测试集样本数量: 25000
    样本标签: {0, 1}
    样本字典: [(b'the', 0), (b'and', 1), (b'a', 2), (b'of', 3), (b'to', 4), (b'is', 5), (b'in', 6), (b'it', 7), (b'i', 8), (b'this', 9)]
    单个样本: [5146, 43, 71, 6, 1092, 14, 0, 878, 130, 151, 5146, 18, 281, 747, 0, 5146, 3, 5146, 2165, 37, 5146, 46, 5, 71, 4089, 377, 162, 46, 5, 32, 1287, 300, 35, 203, 2136, 565, 14, 2, 253, 26, 146, 61, 372, 1, 615, 5146, 5, 30, 0, 50, 3290, 6, 2148, 14, 0, 5146, 11, 17, 451, 24, 4, 127, 10, 0, 878, 130, 43, 2, 50, 5146, 751, 5146, 5, 2, 221, 3727, 6, 9, 1167, 373, 9, 5, 5146, 7, 5, 1343, 13, 2, 5146, 1, 250, 7, 98, 4270, 56, 2316, 0, 928, 11, 11, 9, 16, 5, 5146, 5146, 6, 50, 69, 27, 280, 27, 108, 1045, 0, 2633, 4177, 3180, 17, 1675, 1, 2571]
    最小样本长度: 10;最大样本长度: 2469


对于训练集，将数据的顺序打乱，以优化将要进行的分类模型训练的效果。


```python
shuffle_index = list(range(len(imdb_train)))
random.shuffle(shuffle_index)
train_x = [imdb_train.docs[i] for i in shuffle_index]
train_y = [imdb_train.labels[i] for i in shuffle_index]

test_x = imdb_test.docs
test_y = imdb_test.labels
```

从样本长度上可以看到，每个样本的长度是不相同的。然而，在模型的训练过程中，需要保证每个样本的长度相同，以便于构造矩阵进行批量运算。
因此，需要先对所有样本进行填充或截断，使样本的长度一致。


```python
def vectorizer(input, label=None, length=2000):
    if label is not None:
        for x, y in zip(input, label):
            yield np.array((x + [0]*length)[:length]).astype('int64'), np.array([y]).astype('int64')
    else:
        for x in input:
            yield np.array((x + [0]*length)[:length]).astype('int64')
```

### 2.2 载入预训练向量
以下给出的文件较小，可以直接完全载入内存。对于大型的预训练向量，无法一次载入内存的，可以采用分批载入，并行处理的方式进行匹配。
此外，AIStudio 中提供了 glove.6B 数据集挂载，用户可在 AIStudio 中直接载入数据集并解压。


```python
# 下载并解压预训练向量
!wget http://nlp.stanford.edu/data/glove.6B.zip
!unzip -q glove.6B.zip
```


```python
glove_path = "./glove.6B.100d.txt"
embeddings = {}
```

观察上述GloVe预训练向量文件一行的数据：


```python
# 使用utf8编码解码
with open(glove_path, encoding='utf-8') as gf:
    line = gf.readline()
    print("GloVe单行数据：'%s'" % line)
```

    GloVe单行数据：'the -0.038194 -0.24487 0.72812 -0.39961 0.083172 0.043953 -0.39141 0.3344 -0.57545 0.087459 0.28787 -0.06731 0.30906 -0.26384 -0.13231 -0.20757 0.33395 -0.33848 -0.31743 -0.48336 0.1464 -0.37304 0.34577 0.052041 0.44946 -0.46971 0.02628 -0.54155 -0.15518 -0.14107 -0.039722 0.28277 0.14393 0.23464 -0.31021 0.086173 0.20397 0.52624 0.17164 -0.082378 -0.71787 -0.41531 0.20335 -0.12763 0.41367 0.55187 0.57908 -0.33477 -0.36559 -0.54857 -0.062892 0.26584 0.30205 0.99775 -0.80481 -3.0243 0.01254 -0.36942 2.2167 0.72201 -0.24978 0.92136 0.034514 0.46745 1.1079 -0.19358 -0.074575 0.23353 -0.052062 -0.22044 0.057162 -0.15806 -0.30798 -0.41625 0.37972 0.15006 -0.53212 -0.2055 -1.2526 0.071624 0.70565 0.49744 -0.42063 0.26148 -1.538 -0.30223 -0.073438 -0.28312 0.37104 -0.25217 0.016215 -0.017099 -0.38984 0.87424 -0.72569 -0.51058 -0.52028 -0.1459 0.8278 0.27062
    '


可以看到，每一行都以单词开头，其后接上该单词的向量值，各个值之间用空格隔开。基于此，可以用如下方法得到所有词向量的字典。


```python
with open(glove_path, encoding='utf-8') as gf:
    for glove in gf:
        word, embedding = glove.split(maxsplit=1)
        embedding = [float(s) for s in embedding.split(' ')]
        embeddings[word] = embedding
print("预训练词向量总数：%d" % len(embeddings))
print(f"单词'the'的向量是：{embeddings['the']}")
```

    预训练词向量总数：400000
    单词'the'的向量是：[-0.038194, -0.24487, 0.72812, -0.39961, 0.083172, 0.043953, -0.39141, 0.3344, -0.57545, 0.087459, 0.28787, -0.06731, 0.30906, -0.26384, -0.13231, -0.20757, 0.33395, -0.33848, -0.31743, -0.48336, 0.1464, -0.37304, 0.34577, 0.052041, 0.44946, -0.46971, 0.02628, -0.54155, -0.15518, -0.14107, -0.039722, 0.28277, 0.14393, 0.23464, -0.31021, 0.086173, 0.20397, 0.52624, 0.17164, -0.082378, -0.71787, -0.41531, 0.20335, -0.12763, 0.41367, 0.55187, 0.57908, -0.33477, -0.36559, -0.54857, -0.062892, 0.26584, 0.30205, 0.99775, -0.80481, -3.0243, 0.01254, -0.36942, 2.2167, 0.72201, -0.24978, 0.92136, 0.034514, 0.46745, 1.1079, -0.19358, -0.074575, 0.23353, -0.052062, -0.22044, 0.057162, -0.15806, -0.30798, -0.41625, 0.37972, 0.15006, -0.53212, -0.2055, -1.2526, 0.071624, 0.70565, 0.49744, -0.42063, 0.26148, -1.538, -0.30223, -0.073438, -0.28312, 0.37104, -0.25217, 0.016215, -0.017099, -0.38984, 0.87424, -0.72569, -0.51058, -0.52028, -0.1459, 0.8278, 0.27062]


### 3.3 给数据集的词表匹配词向量
接下来，提取数据集的词表，需要注意的是，词表中的词编码的先后顺序是按照词出现的频率排列的，频率越高的词编码值越小。


```python
word_idx = imdb_train.word_idx
vocab = [w for w in word_idx.keys()]
print(f"词表的前5个单词：{vocab[:5]}")
print(f"词表的后5个单词：{vocab[-5:]}")
```

    词表的前5个单词：[b'the', b'and', b'a', b'of', b'to']
    词表的后5个单词：[b'troubles', b'virtual', b'warriors', b'widely', '<unk>']


观察词表的后5个单词，发现最后一个词是"\<unk\>"，这个符号代表所有词表以外的词。另外，对于形式b'the'，是字符串'the'
的二进制编码形式，使用中注意使用b'the'.decode()来进行转换（'\<unk\>'并没有进行二进制编码，注意区分）。
接下来，给词表中的每个词匹配对应的词向量。预训练词向量可能没有覆盖数据集词表中的所有词，对于没有的词，设该词的词
向量为零向量。


```python
# 定义词向量的维度，注意与预训练词向量保持一致
dim = 100

vocab_embeddings = np.zeros((len(vocab), dim))
for ind, word in enumerate(vocab):
    if word != '<unk>':
        word = word.decode()
    embedding = embeddings.get(word, np.zeros((dim,)))
    vocab_embeddings[ind, :] = embedding
```

## 四、组网

###  4.1 构建基于预训练向量的Embedding
对于预训练向量的Embedding，一般期望它的参数不再变动，所以要设置trainable=False。如果希望在此基础上训练参数，则需要
设置trainable=True。


```python
pretrained_attr = paddle.ParamAttr(name='embedding',
                                   initializer=paddle.nn.initializer.Assign(vocab_embeddings),
                                   trainable=False)
embedding_layer = paddle.nn.Embedding(num_embeddings=len(vocab),
                                      embedding_dim=dim,
                                      padding_idx=word_idx['<unk>'],
                                      weight_attr=pretrained_attr)
```

### 4.2 构建分类器
这里，构建简单的基于一维卷积的分类模型，其结构为：Embedding->Conv1D->Pool1D->Linear。在定义Linear时，由于需要知
道输入向量的维度，可以按照公式[官方文档](https://www.paddlepaddle.org.cn/documentation/docs/zh/2.0-beta/api/paddle/nn/layer/conv/Conv2d_cn.html)
来进行计算。这里给出计算的函数如下：


```python
def cal_output_shape(input_shape, out_channels, kernel_size, stride, padding=0, dilation=1):
    return out_channels, int((input_shape + 2*padding - (dilation*(kernel_size - 1) + 1)) / stride) + 1


# 定义每个样本的长度
length = 2000

# 定义卷积层参数
kernel_size = 5
out_channels = 10
stride = 2
padding = 0

output_shape = cal_output_shape(length, out_channels, kernel_size, stride, padding)
output_shape = cal_output_shape(output_shape[1], output_shape[0], 2, 2, 0)
sim_model = paddle.nn.Sequential(embedding_layer,
                             paddle.nn.Conv1D(in_channels=dim, out_channels=out_channels, kernel_size=kernel_size,
                                              stride=stride, padding=padding, data_format='NLC', bias_attr=True),
                             paddle.nn.ReLU(),
                             paddle.nn.MaxPool1D(kernel_size=2, stride=2),
                             paddle.nn.Flatten(),
                             paddle.nn.Linear(in_features=np.prod(output_shape), out_features=2, bias_attr=True),
                             paddle.nn.Softmax())

paddle.summary(sim_model, input_size=(-1, length), dtypes='int64')
```

    ---------------------------------------------------------------------------
     Layer (type)       Input Shape          Output Shape         Param #    
    ===========================================================================
      Embedding-1       [[1, 2000]]         [1, 2000, 100]        514,700    
       Conv1D-1       [[1, 2000, 100]]       [1, 998, 10]          5,010     
        ReLU-1         [[1, 998, 10]]        [1, 998, 10]            0       
      MaxPool1D-1      [[1, 998, 10]]        [1, 998, 5]             0       
       Flatten-1       [[1, 998, 5]]          [1, 4990]              0       
       Linear-1         [[1, 4990]]             [1, 2]             9,982     
       Softmax-1          [[1, 2]]              [1, 2]               0       
    ===========================================================================
    Total params: 529,692
    Trainable params: 14,992
    Non-trainable params: 514,700
    ---------------------------------------------------------------------------
    Input size (MB): 0.01
    Forward/backward pass size (MB): 1.75
    Params size (MB): 2.02
    Estimated Total Size (MB): 3.78
    ---------------------------------------------------------------------------






    {'total_params': 529692, 'trainable_params': 14992}



### 4.3 读取数据，进行训练
可以利用飞桨2.0的io.Dataset模块来构建一个数据的读取器，方便地将数据进行分批训练。


```python
class DataReader(Dataset):
    def __init__(self, input, label, length):
        self.data = list(vectorizer(input, label, length=length))

    def __getitem__(self, idx):
        return self.data[idx]

    def __len__(self):
        return len(self.data)
        

# 定义输入格式
input_form = paddle.static.InputSpec(shape=[None, length], dtype='int64', name='input')
label_form = paddle.static.InputSpec(shape=[None, 1], dtype='int64', name='label')

model = paddle.Model(sim_model, input_form, label_form)
model.prepare(optimizer=paddle.optimizer.Adam(learning_rate=0.001, parameters=model.parameters()),
              loss=paddle.nn.loss.CrossEntropyLoss(),
              metrics=paddle.metric.Accuracy())

# 分割训练集和验证集
eval_length = int(len(train_x) * 1/4)
model.fit(train_data=DataReader(train_x[:-eval_length], train_y[:-eval_length], length),
          eval_data=DataReader(train_x[-eval_length:], train_y[-eval_length:], length),
          batch_size=32, epochs=10, verbose=1)
```

    The loss value printed in the log is the current step, and the metric is the average value of previous steps.
    Epoch 1/10
    step 586/586 [==============================] - loss: 0.5608 - acc: 0.7736 - 5ms/step        
    Eval begin...
    step 196/196 [==============================] - loss: 0.4902 - acc: 0.8000 - 4ms/step         
    Eval samples: 6250
    Epoch 2/10
    step 586/586 [==============================] - loss: 0.4298 - acc: 0.8138 - 5ms/step        
    Eval begin...
    step 196/196 [==============================] - loss: 0.4801 - acc: 0.8142 - 4ms/step         
    Eval samples: 6250
    Epoch 3/10
    step 586/586 [==============================] - loss: 0.4947 - acc: 0.8298 - 6ms/step        
    Eval begin...
    step 196/196 [==============================] - loss: 0.4568 - acc: 0.8230 - 4ms/step         
    Eval samples: 6250
    Epoch 4/10
    step 586/586 [==============================] - loss: 0.4202 - acc: 0.8455 - 5ms/step        
    Eval begin...
    step 196/196 [==============================] - loss: 0.4503 - acc: 0.8266 - 4ms/step         
    Eval samples: 6250
    Epoch 5/10
    step 586/586 [==============================] - loss: 0.4847 - acc: 0.8564 - 5ms/step        
    Eval begin...
    step 196/196 [==============================] - loss: 0.4647 - acc: 0.8280 - 4ms/step         
    Eval samples: 6250
    Epoch 6/10
    step 586/586 [==============================] - loss: 0.4952 - acc: 0.8667 - 5ms/step        
    Eval begin...
    step 196/196 [==============================] - loss: 0.4855 - acc: 0.8272 - 4ms/step         
    Eval samples: 6250
    Epoch 7/10
    step 586/586 [==============================] - loss: 0.4016 - acc: 0.8704 - 5ms/step        
    Eval begin...
    step 196/196 [==============================] - loss: 0.4764 - acc: 0.8248 - 4ms/step         
    Eval samples: 6250
    Epoch 8/10
    step 586/586 [==============================] - loss: 0.4262 - acc: 0.8807 - 5ms/step        
    Eval begin...
    step 196/196 [==============================] - loss: 0.4970 - acc: 0.8104 - 4ms/step         
    Eval samples: 6250
    Epoch 9/10
    step 586/586 [==============================] - loss: 0.3585 - acc: 0.8862 - 6ms/step        
    Eval begin...
    step 196/196 [==============================] - loss: 0.4614 - acc: 0.8272 - 4ms/step         
    Eval samples: 6250
    Epoch 10/10
    step 586/586 [==============================] - loss: 0.3333 - acc: 0.8935 - 5ms/step        
    Eval begin...
    step 196/196 [==============================] - loss: 0.4986 - acc: 0.8272 - 4ms/step         
    Eval samples: 6250


## 五、评估效果并用模型预测


```python
# 评估
model.evaluate(eval_data=DataReader(test_x, test_y, length), batch_size=32, verbose=1)

# 预测
true_y = test_y[100:105] + test_y[-110:-105]
pred_y = model.predict(DataReader(test_x[100:105] + test_x[-110:-105], None, length), batch_size=1)
test_x_doc = test_x[100:105] + test_x[-110:-105]

# 标签编码转文字
label_id2text = {0: 'positive', 1: 'negative'}

for index, y in enumerate(pred_y[0]):
    print("原文本：%s" % ' '.join([vocab[i].decode() for i in test_x_doc[index] if i < len(vocab) - 1]))
    print("预测的标签是：%s, 实际标签是：%s" % (label_id2text[np.argmax(y)], label_id2text[true_y[index]]))
```

    Eval begin...
    step 782/782 [==============================] - loss: 0.4462 - acc: 0.8262 - 4ms/step        
    Eval samples: 25000
    Predict begin...
    step 10/10 [==============================] - 4ms/step        
    Predict samples: 10
    原文本：albert and tom are brilliant as sir and his of course the play is brilliant to begin with and nothing can compare with the and of theatre and i think you listen better in theatre but on the screen we become more intimate were more than we are in the theatre we witness subtle changes in expression we see better as well as listen both the play and the movie are moving intelligent the story of the company of historical context of the two main characters and of the parallel characters in itself if you cannot get to see it in a theatre i dont imagine its produced much these days then please do yourself a favor and get the video
    预测的标签是：positive, 实际标签是：positive
    原文本：this film has its and may some folks who frankly need a good the head but the film is top notch in every way engaging poignant relevant naturally is larger than life makes an ideal i thought the performances to be terribly strong in both leads and character provides plenty of dark humor the period is well captured the supporting cast well chosen this is to be seen and like a fine i only wish it were out on dvd
    预测的标签是：positive, 实际标签是：positive
    原文本：this is a movie that deserves another you havent seen it for a while or a first you were too young when it came out 1983 based on a play by the same name it is the story of an older actor who heads a company in england during world war ii it deals with his stress of trying to perform a shakespeare each night while facing problems such as theaters and a company made up of older or physically young able ones being taken for military service it also deals with his relationship with various members of his company especially with his so far it all sounds rather dull but nothing could be further from the truth while tragic overall the story is told with a lot of humor and emotions run high throughout the two male leads both received oscar for best actor and so i strongly recommend this movie to anyone who enjoys human drama shakespeare or who has ever worked in any the make up another of the movie that will be fascinating to most viewers
    预测的标签是：positive, 实际标签是：positive
    原文本：sir has played over tonight he cant remember his opening at the eyes reflect the kings madness his him the is an air of desperation about both these great actor knowing his powers are major wife aware of his into madness and knowing he is to do more than ease his passing the is really a love story between the the years they have become on one another to the extent that neither can a future without the other set during the second world concerns the of a frankly second rate company an equal number of has and led by sir a theatrical knight of what might be called the old part he is playing he stage and out over the his audience into inside most of the time deep beneath the he still remains an occasional of his earlier is to catch a glimpse of this that his audiences hope for mr very cleverly on the to the point of when you are ready to his performance as mere and he will produce a moment of subtlety and that makes you realise that a great actor is playing a great actor the same goes for mr easy to write off his br of norman as an exercise in we have a middle aged rather than camp theatrical his way through the company of the girls and loving the wicked in the were and i strongly suspect still are many men just like norman in the kind and more about the plays than many of the run with wisdom and believe the vast majority of them would with laughter at mr portrait i saw the on the london stage the norman was rather more than in the was played by the great mr jones to huge from the was a memorable performance that mr him rather to an also ran as opposed to an actor on level idea that sir and norman might be almost without each other went right out of the window norman was reduced to being his im not sure was what made for breathtaking theatre and the balance in the to the relationship both men have come a long way since their early appearances in the british new wave pictures when they became the of the vaguely class and ashamed of it the british cinema virtually committed in the 1970s they on the theatre apart from a few roles to keep the wolf from the the of more in the bright br the their with energy and talent to the world at large were still not a big movie but is a great one
    预测的标签是：positive, 实际标签是：positive
    原文本：anyone who fine acting and dialogue will br this film taken from taking sides its a funnybr br and ultimately of a relationship between br very types albert is as the br actor who barely the world war br around him so intent is he on the of his br company and his own psychological and emotional br tom is as norman the of the br whose apparent turns out to be anything but br really a must see
    预测的标签是：positive, 实际标签是：positive
    原文本：well i guess i know the answer to that question for the money we have been so with cat in the hat advertising and that we almost believe there has to be something good about this movie i admit i thought the trailers looked bad but i still had to give it a chance well i should have went with my it was a complete piece hollywood trash once again that the average person can be into believing anything they say is good must be good aside from the insulting fact that the film is only about 80 minutes long it obviously started with a eaten script its full of failed attempts at senseless humor and awful it jumps all over the universe with no nor direction this is then with yes ill say it bad acting i couldnt help but feel like i was watching coffee talk on every time mike myers opened his mouth was the cat intended to be a middle aged jewish woman and were no prize either but mr myers should disappear under a rock somewhere until hes ready to make another austin powers movie f no stars 0 on a scale of 110 save your money
    预测的标签是：negative, 实际标签是：negative
    原文本：when my own child is me to leave the opening show of this film i know it is bad i wanted to my eyes out i wanted to reach through the screen and slap mike myers for the last of dignity he had this is one of the few films in my life i have watched and immediately wished to if only it were possible the other films being 2 and fast and both which are better than this crap in the br i may drink myself to sleep tonight in a attempt to forget i ever witnessed this on the good br to mike myers i say stick with austin or even world just because it worked for jim carrey doesnt mean is a success for all br
    预测的标签是：negative, 实际标签是：negative
    原文本：holy what a piece of this movie is i didnt how these filmmakers could take a word book and turn it into a movie i guess they didnt know either i dont remember any or in the book do youbr br they took this all times childrens classic added some and sexual and it into a joke this should give you a good idea of what these hollywood producers think like i have to say visually it was interesting but the brilliant visual story is ruined by toilet humor if you even think that kind of thing is funny i dont want the kids that i know to think it isbr br dont take your kids to see dont rent the dvd i hope the ghost of doctor ghost comes and the people that made this movie
    预测的标签是：negative, 实际标签是：negative
    原文本：i was so looking forward to seeing this when it was in it turned out to be the the biggest let down a far cry from the world of dr it was and i dont think dr would have the stole christmas was much better i understand it had some subtle adult jokes in it but my children have yet to catch on whereas the cat in the hat they caught a lot more than i would have up with dr it really bothered me to see how this timeless classic got on the big screen lets see what they do with a hope this one does dr some justice
    预测的标签是：positive, 实际标签是：negative
    原文本：ive seen some bad things in my time a half dead trying to get out of high a head on between two cars a thousand on a kitchen floor human beings living like br but never in my life have i seen anything as bad as the cat in the br this film is worse than 911 worse than hitler worse than the worse than people who put in br it is the most disturbing film of all time br i used to think it was a joke some elaborate joke and that mike myers was maybe a high drug who lost a bet or br i
    预测的标签是：negative, 实际标签是：negative

