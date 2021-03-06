---
layout: article
title: Summary：Keras 常见问题总结
date: 2020-01-27 00:10:00 +0800
tags: [Summary, Keras, Deep Learning]
categories: blog
pageview: true
key: Summary-of-keras-on-GNN
---

------

## 一、模型输入输出

### 1.1. keras使用稀疏矩阵输入进行训练

稀疏矩阵一般用于表示那些数值为0的元素数目远大于非零元素数目的矩阵。图数据集一般都以稀疏矩阵表示其邻接矩阵。一般用普通的ndarray存储稀疏矩阵会造成很大的内存浪费，在python中可以使用scipy的sparse模块构建稀疏矩阵。

在用keras搭建神经网络的时候，使用稀疏矩阵作为神经网络的输入时，需要做一些处理才能使用sparse格式的数据。

#### 方法一：使用keras函数式API中的sparse参数实现

keras的Sequential顺序模型是不支持稀疏输入的，如果非要用Sequential模型，可以参考方法二。在使用函数式API模型时，Input层初始化时有一个sparse参数，用来指明要创建的占位符是否是稀疏的，如图：

<img src="http://ww1.sinaimg.cn/large/005NduT8ly1gbawzwpj7ej30os0mywgi.jpg" alt="9adca6237dc9db8d1125ee984073e82.png" style="zoom:50%;" />

使用的过程中，设置sparse参数为True即可：

```python
G = Input(batch_shape=(None, None), name='A', sparse=True)
```

**注意：**这么使用有一个问题，就是**指定的batch_size无效**，不管设置多大的batch_size，训练的时候都是按照batch_size为1来进行。

#### 方法二：使用生成器方法实现

参考链接：

- [https://www.jianshu.com/p/a7dadd842f78](https://www.jianshu.com/p/a7dadd842f78)
- [https://stackoverflow.com/questions/37609892/keras-sparse-matrix-issue](https://stackoverflow.com/questions/37609892/keras-sparse-matrix-issue)



### 1.2. 模型多输入多输出

当输入输出不止一个时，可用列表，元组或字典形式存放多变量，多个变量的样本数要一致；

当存储形式为字典时，key为输出该变量的layer的names;

```python
# 单输入多输出
# 字典
self.vae.fit(x=x_train,
             y={'output': x_train,
                'z_vars': np.ones(shape=(x_train.shape[0], 2 * self.latent_dim))},
             shuffle=True,
             nb_epoch=self.nb_epoch,
             batch_size=self.batch_size)

# 多输入多输出
# 列表+字典
model.fit(x=[feature, adj],
          y={'adj_rec': adj_label.toarray().flatten()[np.newaxis, :],
             'Z_vars': np.zeros(shape=(1, adj_norm.shape[1], 32))},
          batch_size=1, epochs=1, shuffle=False, verbose=0)

# 指定验证集
modelll.fit(x=[x_train, x_train], 
            y=y_train, 
            validation_data=([x_val, x_val], y_val), 
            epochs=10, batch_size=64)
```

常见报错：

```
ValueError: Error when checking model target: the list of Numpy arrays that you are passing to your model is not the size the model expected.
```

```
Traceback (most recent call last):
  File "/.../test.py", line 102, in <module>
    model.fit(set, epochs=epochs, steps_per_epoch=steps)
  File "/usr/local/lib/python3.6/site-packages/tensorflow/python/keras/engine/training.py", line 1363, in fit
    validation_steps=validation_steps)
  File "/usr/local/lib/python3.6/site-packages/tensorflow/python/keras/engine/training_arrays.py", line 187, in fit_loop
    if issparse is not None and issparse(ins[i]) and not K.is_sparse(feed[i]):
IndexError: list index out of range
```



## 二、验证集设置

### 2.1. 模型多输入时验证集设置

没找到多输入时model.fit中设置validation_data的例子，而且用validation_split也一样是错的，但是验证集又是必须用的，所以就把多输入改成单输入了，其实就是在输入之前先拼接，输入之后再拆分，费点功夫而已，单输入时validation_data是没问题的。



### 2.2. 模型训练时验证集设置

参考链接：

- [https://www.jianshu.com/p/0c7af5fbcf72](https://www.jianshu.com/p/0c7af5fbcf72)
- [Test data being used for validation data](https://github.com/fchollet/keras/issues/1753)

首先Keras的**fit函数中，传入的validation data并不用于更新权重**，只是用是来检测loss和accuracy等指标的。但是！作者说了，即使模型没有直接在validation data上训练，这也会导致信息泄露，模型会对validation data逐渐熟悉。所以这里我简单总结一下比较方便的data split方法。

1. 用sklearn的`train_test_split`来把数据分割为training data和test data.
2. 用keras的模型fit时，不要使用`validation_data`这个参数（因为我们也没有准备validatoin data），而是直接使用`validation_split`这个参数，把training data中的一部分用来作为validation data就行了。
3. 上面两步的目的是用来调参的，必须在validation data上进行验证，输出loss，观察变化。
4. 调参：更改layer，unit，加dropout，使用L2正则化，添加新feature等等
5. 等调参结束后，拿着我们满意的参数，再一次在整个training data上进行训练，这一次就不用`validation_split`了。因为我们已经调好了参数，不需要观察输出的loss。
6. 训练完之后，用`model.evaluate()`在test data上进行预测。



### 2.3. 验证集参数设置

参考链接：https://blog.csdn.net/ygfrancois/java/article/details/84942803

验证会进行在当前epoch结束后进行，

- validation_steps：设置了验证使用的validation data steps数量(batch数量)，

  应该不超过 `TotalvalidationSamples / ValidationBatchSize`

  如validation batch size(没必要和train batch相等)=64，validation_steps=100，则会从validation data中取6400个数据用于验证(如果一次step后validation data set剩下的data足够下一次step，会继续从剩下的data set中选取，如果不够会重新循环)。

  > 建议使用整个验证集用于验证，否则每次验证的结果没有太大可比性(使用的验证数据不同)，这种情况可以跳过validation_steps参数， 默认使用所有data来做validation。 如果验证数据集很大，全部使用验证会很耗时，可以设置固定大小的validation_steps=10，则在10个验证batch后，计算损失平均值给出结果。



## 三、损失函数（loss）

### 3.1. 自定义loss的调用

keras.losses函数有一个get(identifier)方法。其中需要注意以下一点：

如果identifier是可调用的一个函数名，也就是一个自定义的损失函数，这个损失函数返回值是一个张量。这样就轻而易举的实现了自定义损失函数。除了使用str和dict类型的identifier，我们也可以直接使用keras.losses包下面的损失函数。

loss函数的输入为`(y_true, y_pred)`，所以自定义的loss函数的输入也要定义为`(y_true, y_pred)`（即使用不到）；

```python
def kl_loss(self, y_true, y_pred):
    z_mean, z_log_var = y_pred[:, :self.latent_dim], y_pred[:, self.latent_dim:]
    return - 0.5 * K.mean(K.sum(1 + z_log_var - K.square(z_mean) - K.exp(z_log_var), axis=-1))

losses = {
            'output': 'mse',         # 通用loss
            'z_vars': self.kl_loss	 # 自定义loss
        }
loss_weights = {
        		'output': 1.0,
        		'z_vars': 1.0
        	   }
```



## 四、评价指标（Metrics）

在model.compile()函数中，optimizer和loss都是单数形式，只有metrics是复数形式。因为一个模型只能指明一个optimizer和loss，却可以指明多个metrics。metrics也是三者中处理逻辑最为复杂的一个。

在keras最核心的地方keras.engine.train.py中有如下处理metrics的函数。这个函数其实就做了两件事：

- 根据输入的metric找到具体的metric对应的函数
- 计算metric张量

在寻找metric对应函数时，有两种步骤：

- 使用字符串形式指明准确率和交叉熵
- 使用keras.metrics.py中的函数

无论怎么使用metric，最终都会变成metrics包下面的函数。当使用字符串形式指明accuracy和crossentropy时，keras会非常智能地确定应该使用metrics包下面的哪个函数。因为metrics包下的那些metric函数有不同的使用场景，例如：

- 有的处理的是one-hot形式的y_input(数据的类别)，有的处理的是非one-hot形式的y_input
- 有的处理的是二分类问题的metric，有的处理的是多分类问题的metric

当使用字符串“accuracy”和“crossentropy”指明metric时，keras会根据损失函数、输出层的shape来确定具体应该使用哪个metric函数。在任何情况下，直接使用metrics下面的函数名是总不会出错的。

keras.metrics.py文件中也有一个get(identifier)函数用于获取metric函数。

- 如果identifier是字符串或者字典，那么会根据identifier反序列化出一个metric函数。
- 如果identifier本身就是一个函数名，那么就直接返回这个函数名。这种方式就为自定义metric提供了巨大便利。

```python
# 为某一输出指定计算多个metrics
metrics = {
        	'adj_rec': [binary_accuracy, precision, recall, f1_score]
    	  }

vgae.compile(optimizer=adam, loss=losses, loss_weights=loss_weights, metrics=metrics)
```





## 五、模型保存与加载

### 5.1. 模型保存

#### 5.1.1. 完整保存模型

使用`model.save()`完整地保存整个模型，将Keras模型和权重保存在一个`HDF5`文件中，该文件将包含：

- 模型的结构，允许重新创建模型
- 模型的权重
- 训练配置项（损失函数，优化器）
- 优化器状态，允许准确地从你上次结束的地方继续训练。

```python
model.save(fname)
```

#### 5.1.2. 保存模型结构

使用`to_json()`方法或者`to_yaml()`将模型结构保存到`json`文件或者`yaml`文件。

```python
# 保存模型的结构
json_string = model.to_json()		# 方式1
open('model_architecture_1.json', 'w').write(json_string)
yaml_string = model.to_yaml()		# 方式2
open('model_arthitecture_2.yaml', 'w').write(yaml_string)
```

#### 5.1.3. 只保存模型权重

只保留模型的权重可以通过`save_weights()`方法实现，也可以通过检查点`checkpoint`的设置实现。

- 通过save_weights方法实现

    ```python
    # 保存模型的权重
    model.save_weights('my_model_weights.h5')
    ```

- 通过设置检查点实现

  通过创建[`ModelCheckpoint`](https://keras.io/zh/callbacks/#modelcheckpoint)类的实例来设置检查点，代码如下：

  ```python
  from keras.callbacks import ModelCheckpoint
  
  model = Sequential()
  model.add(Dense(10, input_dim=784, kernel_initializer='uniform'))
  model.add(Activation('softmax'))
  model.compile(loss='categorical_crossentropy', optimizer='rmsprop')
  
  # 如果验证损失下降， 那么在每个训练轮之后保存模型。
  checkpointer = ModelCheckpoint(filepath='/tmp/weights.hdf5', monitor='val_loss', verbose=1, save_best_only=True)
  model.fit(x_train, y_train, batch_size=128, epochs=20, verbose=0, validation_data=(X_test, Y_test), callbacks=[checkpointer])
  ```

#### 5.1.4. 保存模型图

- 通过`model.summary()`打印模型的概述信息

  <img src="http://ww1.sinaimg.cn/large/005NduT8ly1gepgwg7zrtj30d50auaa3.jpg" alt="7a58f2cd3d92ed3b40c32c51bcd991a.png" style="zoom: 80%;" />

- 通过`plot_model()`方法保存模型的基本结构图

  ```python
  from keras.utils import plot_model
  
  # 保存模型的基本结构图
  plot_model(model, 'model_plot.png')
  ```

  

### 5.2. 模型加载

#### 5.2.1. 加载整个模型

加载整个模型，对应着5.1.1.中的保存整个模型，我们从保存模型信息的文件中获取模型的结构、模型的权重和优化器的配置信息，则可以使用数据继续训练模型。这里使用`load_model()`来加载整个模型。

```python
# 加载整个模型
model = load_model('my_model.h5')
```

#### 5.2.2. 只加载模型结构

```python
# 从 JSON 重建模型：
from keras.models import model_from_json
open('model_architecture_1.json', 'w').read(json_string)
model = model_from_json(json_string)

# 从 YAML 重建模型：
from keras.models import model_from_yaml
open('model_architecture_2.yaml', 'w').read(yaml_string)
model = model_from_yaml(yaml_string)
```

#### 5.2.3. 加载模型权重

假设你有用于实例化模型的代码，则可以将保存的权重加载到具有相同结构的模型中：

```python
model.load_weights('my_model_weights.h5')
```

如果你需要将权重加载到不同的结构（有一些共同层）的模型中，例如微调或迁移学习，则可以按层的名字来加载权重：

```python
model.load_weights('my_model_weights.h5', by_name=True)
```

例子：

```python
"""
假设原始模型如下所示：
    model = Sequential()
    model.add(Dense(2, input_dim=3, name='dense_1'))
    model.add(Dense(3, name='dense_2'))
    ...
    model.save_weights(fname)
"""

# 新模型
model = Sequential()
model.add(Dense(2, input_dim=3, name='dense_1'))  # 将被加载
model.add(Dense(10, name='new_dense'))  # 将不被加载

# 从第一个模型加载权重；只会影响第一层，dense_1
model.load_weights(fname, by_name=True)
```

#### 5.2.4. 处理已保存模型中的自定义层（或其他自定义对象）

如果要加载的模型包含**自定义层**或**其他自定义类或函数**，则可以通过 `custom_objects` 参数将它们传递给加载机制：

```python
# 假设你的模型包含一个 AttentionLayer 类的实例
model = load_model('my_model.h5', custom_objects={'AttentionLayer': AttentionLayer})
```

或者，你可以使用[自定义对象作用域](https://keras.io/utils/#customobjectscope)：

```python
from keras.utils import CustomObjectScope

with CustomObjectScope({'AttentionLayer': AttentionLayer}):
    model = load_model('my_model.h5')
```

自定义对象的处理与 `load_model`, `model_from_json`, `model_from_yaml` 的工作方式相同：

```python
from keras.models import model_from_json
model = model_from_json(json_string, custom_objects={'AttentionLayer': AttentionLayer})
```



## Reference

- [Keras中文文档](https://keras.io/zh/getting-started/faq/#_6)