.. _user_guide_model_save_load:

#############
模型存储与载入
#############

一、存储载入体系简介
##################

飞桨框架2.x对模型与参数的存储与载入相关接口进行了梳理，根据接口使用的场景与模式，分为三套体系，分别是：

1.1 动态图存储载入体系
--------------------

为提升框架使用体验，飞桨框架2.0将主推动态图模式，动态图模式下的存储载入接口包括：

- paddle.save
- paddle.load
- paddle.jit.save
- paddle.jit.load

本文主要介绍飞桨框架2.0动态图存储载入体系，各接口关系如下图所示：

.. image:: https://github.com/PaddlePaddle/FluidDoc/blob/develop/doc/paddle/guides/images/save_2.0.png?raw=true
.. image:: https://github.com/PaddlePaddle/FluidDoc/blob/develop/doc/paddle/guides/images/load_2.0.png?raw=true

1.2 静态图存储载入体系（飞桨框架1.x）
----------------------------

静态图存储载入相关接口为飞桨框架1.x版本的主要使用接口，出于兼容性的目的，这些接口仍然可以在飞桨框架2.x使用，但不再推荐。相关接口包括：

- paddle.io.save
- paddle.io.load
- paddle.io.save_inference_model
- paddle.io.load_inference_model
- paddle.io.load_program_state
- paddle.io.set_program_state

由于飞桨框架2.0不再主推静态图模式，故本文不对以上主要用于飞桨框架1.x的相关接口展开介绍，如有需要，可以阅读对应API文档。

1.3 高阶API存储载入体系
---------------------

- paddle.Model.fit (训练接口，同时带有参数存储的功能)
- paddle.Model.save
- paddle.Model.load

飞桨框架2.0高阶API存储载入接口体系清晰，表意直观，若有需要，建议直接阅读相关API文档，此处不再赘述。

.. note::
    本教程着重介绍飞桨框架2.x的各个存储载入接口的关系及各种使用场景，不对接口参数进行详细介绍，如果需要了解具体接口参数的含义，请直接阅读对应API文档。


二、参数存储载入（训练调优）
#######################

若仅需要存储/载入模型的参数，可以使用 ``paddle.save/load`` 结合Layer和Optimizer的state_dict达成目的，此处state_dict是对象的持久参数的载体，dict的key为参数名，value为参数真实的numpy array值。

2.1 参数存储
------------

参数存储时，先获取目标对象（Layer或者Optimzier）的state_dict，然后将state_dict存储至磁盘，示例如下:

.. code-block:: python

    import numpy as np
    import paddle
    import paddle.nn as nn
    import paddle.optimizer as opt

    BATCH_SIZE = 16
    BATCH_NUM = 4
    EPOCH_NUM = 4

    IMAGE_SIZE = 784
    CLASS_NUM = 10

    # define a random dataset
    class RandomDataset(paddle.io.Dataset):
        def __init__(self, num_samples):
            self.num_samples = num_samples

        def __getitem__(self, idx):
            image = np.random.random([IMAGE_SIZE]).astype('float32')
            label = np.random.randint(0, CLASS_NUM - 1, (1, )).astype('int64')
            return image, label

        def __len__(self):
            return self.num_samples

    class LinearNet(nn.Layer):
        def __init__(self):
            super(LinearNet, self).__init__()
            self._linear = nn.Linear(IMAGE_SIZE, CLASS_NUM)

        def forward(self, x):
            return self._linear(x)

    def train(layer, loader, loss_fn, opt):
        for epoch_id in range(EPOCH_NUM):
            for batch_id, (image, label) in enumerate(loader()):
                out = layer(image)
                loss = loss_fn(out, label)
                loss.backward()
                opt.step()
                opt.clear_grad()
                print("Epoch {} batch {}: loss = {}".format(
                    epoch_id, batch_id, np.mean(loss.numpy())))

    # enable dygraph mode
    place = paddle.CPUPlace()
    paddle.disable_static(place) 

    # create network
    layer = LinearNet()
    loss_fn = nn.CrossEntropyLoss()
    adam = opt.Adam(learning_rate=0.001, parameters=layer.parameters())

    # create data loader
    dataset = RandomDataset(BATCH_NUM * BATCH_SIZE)
    loader = paddle.io.DataLoader(dataset,
        places=place,
        batch_size=BATCH_SIZE,
        shuffle=True,
        drop_last=True,
        num_workers=2)

    # train
    train(layer, loader, loss_fn, adam)

    # save
    model_path = "linear_net"

    param_state_dict = layer.state_dict()
    paddle.save(param_state_dict, model_path)

    opt_state_dict = adam.state_dict()
    paddle.save(opt_state_dict, model_path)


2.2 参数载入
------------

参数载入时，先从磁盘载入保存的state_dict，然后通过set_state_dict方法配置到目标对象中，示例如下：

.. code-block:: python

    import numpy as np
    import paddle
    import paddle.nn as nn
    import paddle.optimizer as opt

    BATCH_SIZE = 16
    BATCH_NUM = 4
    EPOCH_NUM = 4

    IMAGE_SIZE = 784
    CLASS_NUM = 10

    # define a random dataset
    class RandomDataset(paddle.io.Dataset):
        def __init__(self, num_samples):
            self.num_samples = num_samples

        def __getitem__(self, idx):
            image = np.random.random([IMAGE_SIZE]).astype('float32')
            label = np.random.randint(0, CLASS_NUM - 1, (1, )).astype('int64')
            return image, label

        def __len__(self):
            return self.num_samples

    class LinearNet(nn.Layer):
        def __init__(self):
            super(LinearNet, self).__init__()
            self._linear = nn.Linear(IMAGE_SIZE, CLASS_NUM)

        def forward(self, x):
            return self._linear(x)

    def train(layer, loader, loss_fn, opt):
        for epoch_id in range(EPOCH_NUM):
            for batch_id, (image, label) in enumerate(loader()):
                out = layer(image)
                loss = loss_fn(out, label)
                loss.backward()
                opt.step()
                opt.clear_grad()
                print("Epoch {} batch {}: loss = {}".format(
                    epoch_id, batch_id, np.mean(loss.numpy())))

    # enable dygraph mode
    place = paddle.CPUPlace()
    paddle.disable_static(place) 

    # create network
    layer = LinearNet()
    loss_fn = nn.CrossEntropyLoss()
    adam = opt.Adam(learning_rate=0.001, parameters=layer.parameters())

    # create data loader
    dataset = RandomDataset(BATCH_NUM * BATCH_SIZE)
    loader = paddle.io.DataLoader(dataset,
        places=place,
        batch_size=BATCH_SIZE,
        shuffle=True,
        drop_last=True,
        num_workers=2)

    # load
    model_path = "linear_net"
    param_state_dict, opt_state_dict = paddle.load(model_path)

    layer.set_state_dict(param_state_dict)
    adam.set_state_dict(opt_state_dict)

    # train
    train(layer, loader, loss_fn, adam)

.. note::
     ``paddle.load`` 接口可能仍会改动，后续可能改为仅返回一个单独的dict。

三、模型&参数存储载入（训练部署）
############################

若要同时存储/载入模型结构和参数，可以使用 ``paddle.jit.save/load`` 实现。

3.1 模型&参数存储
----------------

同时存储模型和参数，需要结合动静转换功能使用。有以下三项注意点：

(1) Layer对象的forward方法需要经由 ``paddle.jit.to_static`` 装饰

经过 ``paddle.jit.to_static`` 装饰forward方法后，相应Layer在执行时，会先生成描述模型的Program，然后通过执行Program获取计算结果，示例如下：

.. code-block:: python

    import paddle
    import paddle.nn as nn

    IMAGE_SIZE = 784
    CLASS_NUM = 10

    class LinearNet(nn.Layer):
        def __init__(self):
            super(LinearNet, self).__init__()
            self._linear = nn.Linear(IMAGE_SIZE, CLASS_NUM)

        @paddle.jit.to_static
        def forward(self, x):
            return self._linear(x)

若最终需要生成的描述模型的Program支持动态输入，可以同时指明模型的 ``InputSepc`` ，示例如下：

.. code-block:: python

    import paddle
    import paddle.nn as nn
    from paddle.static import InputSpec

    IMAGE_SIZE = 784
    CLASS_NUM = 10

    class LinearNet(nn.Layer):
        def __init__(self):
            super(LinearNet, self).__init__()
            self._linear = nn.Linear(IMAGE_SIZE, CLASS_NUM)

        @paddle.jit.to_static(input_spec=[InputSpec(shape=[None, 784], dtype='float32')])
        def forward(self, x):
            return self._linear(x)


(2) 请确保Layer.forward方法中仅实现预测功能，避免将训练所需的loss计算逻辑写入forward方法

Layer更准确的语义是描述一个具有预测功能的模型对象，接收输入的样本数据，输出预测的结果，而loss计算是仅属于模型训练中的概念。将loss计算的实现放到Layer.forward方法中，会使Layer在不同场景下概念有所差别，并且增大Layer使用的复杂性，这不是良好的编码行为，同时也会在最终保存预测模型时引入剪枝的复杂性，因此建议保持Layer实现的简洁性，下面通过两个示例对比说明：

错误示例如下：

.. code-block:: python

    import paddle
    import paddle.nn as nn

    IMAGE_SIZE = 784
    CLASS_NUM = 10

    class LinearNet(nn.Layer):
        def __init__(self):
            super(LinearNet, self).__init__()
            self._linear = nn.Linear(IMAGE_SIZE, CLASS_NUM)

        @paddle.jit.to_static
        def forward(self, x, label=None):
            out = self._linear(x)
            if label:
                loss = nn.functional.cross_entropy(out, label)
                avg_loss = nn.functional.mean(loss)
                return out, avg_loss
            else:
                return out
            

正确示例如下：

.. code-block:: python

    import paddle
    import paddle.nn as nn

    IMAGE_SIZE = 784
    CLASS_NUM = 10

    class LinearNet(nn.Layer):
        def __init__(self):
            super(LinearNet, self).__init__()
            self._linear = nn.Linear(IMAGE_SIZE, CLASS_NUM)

        @paddle.jit.to_static
        def forward(self, x):
            return self._linear(x)


(3) 使用 ``paddle.jit.save`` 存储模型和参数

直接将目标Layer传入 ``paddle.jit.save`` 存储即可，完整示例如下：

.. code-block:: python

    import numpy as np
    import paddle
    import paddle.nn as nn
    import paddle.optimizer as opt

    BATCH_SIZE = 16
    BATCH_NUM = 4
    EPOCH_NUM = 4

    IMAGE_SIZE = 784
    CLASS_NUM = 10

    # define a random dataset
    class RandomDataset(paddle.io.Dataset):
        def __init__(self, num_samples):
            self.num_samples = num_samples

        def __getitem__(self, idx):
            image = np.random.random([IMAGE_SIZE]).astype('float32')
            label = np.random.randint(0, CLASS_NUM - 1, (1, )).astype('int64')
            return image, label

        def __len__(self):
            return self.num_samples

    class LinearNet(nn.Layer):
        def __init__(self):
            super(LinearNet, self).__init__()
            self._linear = nn.Linear(IMAGE_SIZE, CLASS_NUM)

        @paddle.jit.to_static
        def forward(self, x):
            return self._linear(x)

    def train(layer, loader, loss_fn, opt):
        for epoch_id in range(EPOCH_NUM):
            for batch_id, (image, label) in enumerate(loader()):
                out = layer(image)
                loss = loss_fn(out, label)
                loss.backward()
                opt.step()
                opt.clear_grad()
                print("Epoch {} batch {}: loss = {}".format(
                    epoch_id, batch_id, np.mean(loss.numpy())))

    # enable dygraph mode
    place = paddle.CPUPlace()
    paddle.disable_static(place) 

    # 1. train & save model.

    # create network
    layer = LinearNet()
    loss_fn = nn.CrossEntropyLoss()
    adam = opt.Adam(learning_rate=0.001, parameters=layer.parameters())

    # create data loader
    dataset = RandomDataset(BATCH_NUM * BATCH_SIZE)
    loader = paddle.io.DataLoader(dataset,
        places=place,
        batch_size=BATCH_SIZE,
        shuffle=True,
        drop_last=True,
        num_workers=2)

    # train
    train(layer, loader, loss_fn, adam)

    # save
    model_path = "linear.example.model"
    paddle.jit.save(layer, model_path)


.. note::
    后续仍会优化此处的使用方式，支持不装饰 ``to_static`` 也能够通过 ``paddle.jit.save`` 直接存储模型和参数。


3.2 模型&参数载入
----------------

载入模型参数，使用 ``paddle.jit.load`` 载入即可，载入后得到的是一个Layer的派生类对象 ``TranslatedLayer`` ， ``TranslatedLayer`` 具有Layer具有的通用特征，支持切换 ``train`` 或者 ``eval`` 模式，可以进行模型调优或者预测。

载入模型及参数，示例如下：

.. code-block:: python

    import numpy as np
    import paddle
    import paddle.nn as nn
    import paddle.optimizer as opt

    # enable dygraph mode
    place = paddle.CPUPlace()
    paddle.disable_static(place) 

    BATCH_SIZE = 16
    BATCH_NUM = 4
    EPOCH_NUM = 4

    IMAGE_SIZE = 784
    CLASS_NUM = 10

    # load
    model_path = "linear.example.model"
    loaded_layer = paddle.jit.load(model_path)

载入模型及参数后进行预测，示例如下（接前述示例）：

.. code-block:: python

    # inference
    loaded_layer.eval()
    x = paddle.randn([1, IMAGE_SIZE], 'float32')
    pred = loaded_layer(x)

载入模型及参数后进行调优，示例如下（接前述示例）：

.. code-block:: python

    # define a random dataset
    class RandomDataset(paddle.io.Dataset):
        def __init__(self, num_samples):
            self.num_samples = num_samples

        def __getitem__(self, idx):
            image = np.random.random([IMAGE_SIZE]).astype('float32')
            label = np.random.randint(0, CLASS_NUM - 1, (1, )).astype('int64')
            return image, label

        def __len__(self):
            return self.num_samples

    def train(layer, loader, loss_fn, opt):
        for epoch_id in range(EPOCH_NUM):
            for batch_id, (image, label) in enumerate(loader()):
                out = layer(image)
                loss = loss_fn(out, label)
                loss.backward()
                opt.step()
                opt.clear_grad()
                print("Epoch {} batch {}: loss = {}".format(
                    epoch_id, batch_id, np.mean(loss.numpy())))

    # fine-tune
    loaded_layer.train()
    dataset = RandomDataset(BATCH_NUM * BATCH_SIZE)
    loader = paddle.io.DataLoader(dataset,
        places=place,
        batch_size=BATCH_SIZE,
        shuffle=True,
        drop_last=True,
        num_workers=2)
    loss_fn = nn.CrossEntropyLoss()
    adam = opt.Adam(learning_rate=0.001, parameters=loaded_layer.parameters())
    train(loaded_layer, loader, loss_fn, adam)


此外， ``paddle.jit.save`` 同时保存了模型和参数，如果您只需要从存储结果中载入模型的参数，可以使用 ``paddle.load`` 接口载入，返回所存储模型的state_dict，示例如下：

.. code-block:: python

    import paddle
    import paddle.nn as nn

    IMAGE_SIZE = 784
    CLASS_NUM = 10

    class LinearNet(nn.Layer):
        def __init__(self):
            super(LinearNet, self).__init__()
            self._linear = nn.Linear(IMAGE_SIZE, CLASS_NUM)

        @paddle.jit.to_static
        def forward(self, x):
            return self._linear(x)

    # enable dygraph mode
    paddle.disable_static() 

    # create network
    layer = LinearNet()

    # load
    model_path = "linear.example.model"
    state_dict, _ = paddle.load(model_path)

    # inference
    layer.set_state_dict(state_dict, use_structured_name=False)
    layer.eval()
    x = paddle.randn([1, IMAGE_SIZE], 'float32')
    pred = layer(x)


四、旧存储格式兼容载入
###################

如果您是从飞桨框架1.x切换到2.x，曾经使用飞桨框架1.x的接口存储模型或者参数，飞桨框架2.x也对这种情况进行了兼容性支持，包括以下几种情况。

4.1 从 ``paddle.io.save_inference_model`` 存储结果中载入模型&参数
------------------------------------------------------------------

曾用接口名为 ``paddle.fluid.io.save_inference_model`` 。

(1) 同时载入模型和参数

使用 ``paddle.jit.load`` 配合 ``paddle.SaveLoadConfig`` 载入模型和参数。

模型准备及训练示例，该示例为后续所有示例的前序逻辑：

.. code-block:: python

    import numpy as np
    import paddle
    import paddle.fluid as fluid
    import paddle.nn as nn
    import paddle.optimizer as opt

    BATCH_SIZE = 16
    BATCH_NUM = 4
    EPOCH_NUM = 4

    IMAGE_SIZE = 784
    CLASS_NUM = 10

    # define a random dataset
    class RandomDataset(paddle.io.Dataset):
        def __init__(self, num_samples):
            self.num_samples = num_samples

        def __getitem__(self, idx):
            image = np.random.random([IMAGE_SIZE]).astype('float32')
            label = np.random.randint(0, CLASS_NUM - 1, (1, )).astype('int64')
            return image, label

        def __len__(self):
            return self.num_samples

    image = fluid.data(name='image', shape=[None, 784], dtype='float32')
    label = fluid.data(name='label', shape=[None, 1], dtype='int64')
    pred = fluid.layers.fc(input=image, size=10, act='softmax')
    loss = fluid.layers.cross_entropy(input=pred, label=label)
    avg_loss = fluid.layers.mean(loss)

    optimizer = fluid.optimizer.SGD(learning_rate=0.001)
    optimizer.minimize(avg_loss)

    place = fluid.CPUPlace()
    exe = fluid.Executor(place)
    exe.run(fluid.default_startup_program())

    # create data loader
    dataset = RandomDataset(BATCH_NUM * BATCH_SIZE)
    loader = paddle.io.DataLoader(dataset,
        feed_list=[image, label],
        places=place,
        batch_size=BATCH_SIZE, 
        shuffle=True,
        drop_last=True,
        num_workers=2)

    # train model
    for data in loader():
        exe.run(
            fluid.default_main_program(),
            feed=data, 
            fetch_list=[avg_loss])
    

如果您是按照 ``paddle.fluid.io.save_inference_model`` 的默认格式存储的，可以按照如下方式载入（接前述示例）：

.. code-block:: python

    # save default
    model_path = "fc.example.model"
    fluid.io.save_inference_model(
        model_path, ["image"], [pred], exe)

    # enable dygraph mode
    paddle.disable_static(place)

    # load
    fc = paddle.jit.load(model_path)

    # inference
    fc.eval()
    x = paddle.randn([1, IMAGE_SIZE], 'float32')
    pred = fc(x)

如果您指定了存储的模型文件名，可以按照以下方式载入（接前述示例）：

.. code-block:: python

    # save with model_filename
    model_path = "fc.example.model.with_model_filename"
    fluid.io.save_inference_model(
        model_path, ["image"], [pred], exe, model_filename="__simplenet__")

    # enable dygraph mode
    paddle.disable_static(place)

    # load
    config = paddle.SaveLoadConfig()
    config.model_filename = "__simplenet__"
    fc = paddle.jit.load(model_path, config=config)

    # inference
    fc.eval()
    x = paddle.randn([1, IMAGE_SIZE], 'float32')
    pred = fc(x)

如果您指定了存储的参数文件名，可以按照以下方式载入（接前述示例）：

.. code-block:: python

    # save with params_filename
    model_path = "fc.example.model.with_params_filename"
    fluid.io.save_inference_model(
        model_path, ["image"], [pred], exe, params_filename="__params__")

    # enable dygraph mode
    paddle.disable_static(place)

    # load
    config = paddle.SaveLoadConfig()
    config.params_filename = "__params__"
    fc = paddle.jit.load(model_path, config=config)

    # inference
    fc.eval()
    x = paddle.randn([1, IMAGE_SIZE], 'float32')
    pred = fc(x)

(2) 仅载入参数

如果您仅需要从 ``paddle.fluid.io.save_inference_model`` 的存储结果中载入参数，以state_dict的形式配置到已有代码的模型中，可以使用 ``paddle.load`` 配合 ``paddle.SaveLoadConfig`` 载入。

如果您是按照 ``paddle.fluid.io.save_inference_model`` 的默认格式存储的，可以按照如下方式载入（接前述示例）：

.. code-block:: python

    model_path = "fc.example.model"

    load_param_dict, _ = paddle.load(model_path)

如果您指定了存储的模型文件名，可以按照以下方式载入（接前述示例）：

.. code-block:: python

    model_path = "fc.example.model.with_model_filename"

    config = paddle.SaveLoadConfig()
    config.model_filename = "__simplenet__"
    load_param_dict, _ = paddle.load(model_path, config)

如果您指定了存储的参数文件名，可以按照以下方式载入（接前述示例）：

.. code-block:: python

    model_path = "fc.example.model.with_params_filename"

    config = paddle.SaveLoadConfig()
    config.params_filename = "__params__"
    load_param_dict, _ = paddle.load(model_path, config)

.. note::
    一般预测模型不会存储优化器Optimizer的参数，因此此处载入的仅包括模型本身的参数。

.. note::
    由于 ``structured_name`` 是动态图下独有的变量命名方式，因此从静态图存储结果载入的state_dict在配置到动态图的Layer中时，需要配置 ``Layer.set_state_dict(use_structured_name=False)`` 。

4.2 从 ``paddle.io.save`` 存储结果中载入参数
----------------------------------------------

曾用接口名为 ``paddle.fluid.save`` 。

 ``paddle.fluid.save`` 的存储格式与2.x动态图接口 ``paddle.save`` 存储格式是类似的，同样存储了dict格式的参数，因此可以直接使用 ``paddle.load`` 载入state_dict，示例如下（接前述示例）：

.. code-block:: python

    # save by fluid.save
    model_path = "fc.example.model.save"
    program = fluid.default_main_program()
    fluid.save(program, model_path)

    # enable dygraph mode
    paddle.disable_static(place)

    load_param_dict, _ = paddle.load(model_path)


.. note::
    由于 ``paddle.fluid.save`` 接口原先在静态图模式下的定位是存储训练时参数，或者说存储Checkpoint，故尽管其同时存储了模型结构，目前也暂不支持从 ``paddle.fluid.save`` 的存储结果中同时载入模型和参数，后续如有需求再考虑支持。


4.3 从 ``paddle.io.save_params/save_persistables`` 存储结果中载入参数
-----------------------------------------------------------------------

.. note::
    以下方式仅为暂时解决方案，后续计划会在 ``paddle.load`` 接口支持此功能。

曾用接口名为 ``paddle.fluid.io.save_params/save_persistables`` 。

此处可以使用 ``paddle.io.load_program_state`` 接口从以上两个接口的存储结果中载入state_dict，并用于动态图Layer的配置，示例如下（接前述示例）：

.. code-block:: python

    # save by fluid.io.save_params
    model_path = "fc.example.model.save_params"
    fluid.io.save_params(exe, model_path)

    # load 
    state_dict = paddle.io.load_program_state(model_path)
