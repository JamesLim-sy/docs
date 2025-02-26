.. _cn_api_fluid_layers_case:

case
-------------------------------


.. py:function:: paddle.static.nn.case(pred_fn_pairs, default=None, name=None)


该OP的运行方式类似于python的if-elif-elif-else。

参数
::::::::::::

    - **pred_fn_pairs** (list|tuple) - 一个list或者tuple，元素是二元组(pred, fn)。其中 ``pred`` 是形状为[1]的布尔型 Tensor，``fn`` 是一个可调用对象。所有的可调用对象都返回相同结构的Tensor。
    - **default** (callable，可选) - 可调用对象，返回一个或多个张量。
    - **name** (str，可选) – 具体用法请参见 :ref:`api_guide_Name` ，一般无需设置，默认值：None。

返回
::::::::::::
Tensor|list(Tensor)

- 如果 ``pred_fn_pairs`` 中存在pred是True的元组(pred, fn)，则返回第一个为True的pred的元组中fn的返回结果；如果 ``pred_fn_pairs`` 中不存在pred为True的元组(pred, fn) 且 ``default`` 不是None，则返回调用 ``default`` 的返回结果；
- 如果 ``pred_fn_pairs`` 中不存在pred为True的元组(pred, fn) 且 ``default`` 是None，则返回 ``pred_fn_pairs`` 中最后一个pred的返回结果。


代码示例
::::::::::::

.. code-block:: python

    import paddle

    paddle.enable_static()

    def fn_1():
        return paddle.full(shape=[1, 2], dtype='float32', fill_value=1)

    def fn_2():
        return paddle.full(shape=[2, 2], dtype='int32', fill_value=2)

    def fn_3():
        return paddle.full(shape=[3], dtype='int32', fill_value=3)

    main_program = paddle.static.default_startup_program()
    startup_program = paddle.static.default_main_program()

    with paddle.static.program_guard(main_program, startup_program):
        x = paddle.full(shape=[1], dtype='float32', fill_value=0.3)
        y = paddle.full(shape=[1], dtype='float32', fill_value=0.1)
        z = paddle.full(shape=[1], dtype='float32', fill_value=0.2)

        pred_1 = paddle.less_than(z, x)  # true: 0.2 < 0.3
        pred_2 = paddle.less_than(x, y)  # false: 0.3 < 0.1
        pred_3 = paddle.equal(x, y)      # false: 0.3 == 0.1

        # Call fn_1 because pred_1 is True
        out_1 = paddle.static.nn.case(
            pred_fn_pairs=[(pred_1, fn_1), (pred_2, fn_2)], default=fn_3)

        # Argument default is None and no pred in pred_fn_pairs is True. fn_3 will be called.
        # because fn_3 is the last callable in pred_fn_pairs.
        out_2 = paddle.static.nn.case(pred_fn_pairs=[(pred_2, fn_2), (pred_3, fn_3)])

        exe = paddle.static.Executor(paddle.CPUPlace())
        res_1, res_2 = exe.run(main_program, fetch_list=[out_1, out_2])
        print(res_1)  # [[1. 1.]]
        print(res_2)  # [3 3 3]
