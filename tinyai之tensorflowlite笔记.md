### 对于tf1.x中使用session训练的模型

##### 直接从sess转化
使用tf.lite.TFLiteConverter.from_session()。例子：
```
import tensorflow as tf

img = tf.placeholder(name="img", dtype=tf.float32, shape=(1, 64, 64, 3))
var = tf.get_variable("weights", dtype=tf.float32, shape=(1, 64, 64, 3))
val = img + var
out = tf.identity(val, name="out")

with tf.Session() as sess:
  sess.run(tf.global_variables_initializer())
  converter = tf.lite.TFLiteConverter.from_session(sess, [img], [out])
  tflite_model = converter.convert()
  open("converted_model.tflite", "wb").write(tflite_model)
```
其中指定的tf.tensor必须是确定shape的

##### 对于checkpoint文件
先转化成pb文件。如何转化？

##### 对于pb文件

### lstm模型如何转化为tflite
使用keras，并把unroll设置为true
LSTM的几个参数：
首先LSTM的结构类似HMM，有隐状态和输出的观测，一般来说keras.layers.LSTM不给return_sequences，模型就会输出最后一个时间步骤的观测h，return_sequences = True，就会把每一个时间
步骤的观测输出出来，return_state 这个参数的作用就是输出最后一个时间步骤的观测h和cell状态c:
```
out,h,c = LSTM(units=128, return_sequences=True,implementation = 2,recurrent_activation = 'hard_sigmoid',use_bias = False,return_state = True)(LSTM_input)
```
同样LSTM层在输入的时候不仅可以输入观测，还可以输入隐状态的初始化：
```
x, h_state, c_state = LSTM(128, return_sequences=True,return_state=True)(x, initial_state=in_state)
```
如果推断的时候没有给初始化的状态，LSTM将在每一个batch里面对初始状态进行初始化。在进行模型实时化改造的时候，每一次输入的数据仅仅是当前时间帧的数据，不同时间帧算是不同batch了，因此我们必须把上一个时间帧的状态输出，传给当前的初始隐状态。如果对tensorflow来说的话，相当于一部分数据在计算图之外被处理了。为了避免这个麻烦可以使用stateful参数，这个参数为True时，上一个batch的隐状态将被保留到下一个batch，下面的对比了是否使用stateful时的计算结果，可以发现stateful LSTM即使推断了两次，这两次的数据在不同batch里面，但是计算结果和将两次数据放在一个batch里面推断的结果一致。

注意在同一batch里面不同的sample是并行计算的，他们之间的状态不会传递，传递只会在不同batch对应的同一个sample间进行。

最后说一下unroll参数，这个参数如果为True，必须要求输入模型的batchsize不能为None，在这个情况下，tensorflow将会将原本使用符号计算的LSTM展开成计算图，可以在时间帧短的情况下达成加速，只不过会消耗更多内存。

lstm的输入一般是这样的形式：(batch_size, time_steps, input_size)，
- batch_size：表示batch的大小，即一个batch中序列的个数，
- time_steps：表示序列本身的长度，如在Char RNN中，长度为10的句子对应的time_steps就等于10。
- input_size：表示输入数据单个序列单个时间维度上固有的长度。
例如，温度预测模型中，batch_size为256，time_steps为120，Input_size为3
最后一步的隐状态，它的形状为(batch_size, cell.state_size), state_size就是LSTM的unit数目
tf.keras.Input指定shape时不包含batch_size，batch_size可以在预测时指定，或者也可以指定batch_shape，这个是包含batch_size的shape

1:
for i in range(10): #training
    model.fit(trainX, trainY, epochs=1, batch_size=batch_size, verbose=0, 
              shuffle=False)
    model.reset_states()

2:
model.fit(trainX, trainY, epochs=10, batch_size=batch_size, verbose=0, 
          shuffle=False)

1和2基本一致，但是1允许你在每个epoch之间做点事，比如model.reset_states()，其实2也可以做到，通过在fit里面使用回调函数，例如on_epoch_begin等
如果我们设置了stateful为true,
