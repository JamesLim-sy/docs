.. _cn_paddle_nn_functional_loss_cosine_embedding_loss:

cosine_embedding_loss
-------------------------------

.. py:function:: paddle.nn.functional.cosine_embedding_loss(input1, input2, label, margin=0, reduction='mean', name=None)

该函数计算输入input1, input2和label之间的 `CosineEmbedding` 损失

如果label=1，则该损失函数的数学计算公式如下：

    .. math::
        Out = 1 - cos(input1, input2)

如果label=-1，则该损失函数的数学计算公式如下：

    .. math::
        Out = max(0, cos(input1, input2)) - margin

其中cos计算公式如下：

    .. math::
        cos(x1, x2) = \frac{x1 \cdot{} x2}{\Vert x1 \Vert_2 * \Vert x2 \Vert_2}

参数
:::::::::
    - **input1** (Tensor): - 输入的Tensor，维度是[N, M], 其中N是batch size，可为0，M是数组长度。数据类型为：float32、float64。
    - **input2** (Tensor): - 输入的Tensor，维度是[N, M], 其中N是batch size，可为0，M是数组长度。数据类型为：float32、float64。
    - **label** (Tensor): - 标签，维度是[N]，N是数组长度，数据类型为：float32、float64、int32、int64。
    - **margin** (float, 可选): - 可以设置的范围为[-1, 1]，建议设置的范围为[0, 0.5]。其默认为 `0` 。数据类型为float。
    - **reduction** (string, 可选): - 指定应用于输出结果的计算方式，可选值有: ``'none'``, ``'mean'``, ``'sum'`` 。默认为 ``'mean'``，计算 `CosineEmbeddingLoss` 的均值；设置为 ``'sum'`` 时，计算 `CosineEmbeddingLoss` 的总和；设置为 ``'none'`` 时，则返回 `CosineEmbeddingLoss`。数据类型为string。
    - **name** (str，可选): - 操作的名称(可选，默认值为 None）。更多信息请参见 :ref:`api_guide_Name`。

返回
:::::::::
``Tensor``，输入 ``input1`` 、 ``input2`` 和标签 ``label`` 间的 `CosineEmbeddingLoss` 损失。如果 `reduction` 是 ``'none'``, 则输出Loss的维度为 [N], 与输入 ``input1`` 和 ``input2`` 相同。如果 `reduction` 是 ``'mean'`` 或 ``'sum'``, 则输出Loss的维度为 [1]。


代码示例
:::::::::
COPY-FROM: paddle.nn.functional.cosine_embedding_loss:code-example1

