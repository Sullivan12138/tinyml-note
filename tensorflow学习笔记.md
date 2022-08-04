tensorflow1.x中使用session的方式训练模型，从tf1.15开始，支持keras训练，tf1.15是tensorflow1.x的最后一个版本，从tf2.x开始就抛弃了session，所以tensorflow1.15是唯一既支持session又支持keras的版本。
tensorflow1.x是不支持lstm模型转化为tflite的。只有从tf2.x开始才支持。
一般情况下，只要能支持转化为tflite，并且tflite模型的预测正确，就能在mcu上部署（前提是我们要在github上下载最新的arduino上的tflite库），并且在mcu上的预测也就正确，mcu上的输出是和服务器上tflite模型的输出是一样的。
（我遇到一个例外，当时用tf1.x的keras训练出lstm模型，然后用tf2.x转化为了tflite，tflite的输出很正常，但是在mcu上的输出错误很多，可能是tf1.x的keras训练出的lstm模型还是不对吧，必须用tf2.x的keras去训练）
tflite模型如何进行预测，以下面代码为例：
```
q_interpreter = tf.lite.Interpreter('multi_tempter.tflite')
q_interpreter.allocate_tensors()
input_details = q_interpreter.get_input_details()[0]
output_details = q_interpreter.get_output_details()[0]
# print(input_details)
# print(output_details)
input_scale, input_zero_point = input_details['quantization']
output_scale, output_zero_point = output_details['quantization']
print(input_zero_point)
print(output_zero_point)
for i, (x, y) in enumerate(test_dataset.skip(1000).take(1)):
    xx = x / input_scale + input_zero_point
    xx = tf.cast(xx, input_details['dtype'])
    x_np = np.array(xx)
    print(x_np.shape)
    q_interpreter.set_tensor(input_details["index"], x_np[0].reshape([1,120,3]))
    q_interpreter.invoke()
    output = q_interpreter.get_tensor(output_details["index"])[0]
    print(output)
    output = tf.cast(output, tf.float32)
    output2 = (output - output_zero_point) * output_scale
```
output2即为预测输出。主要步骤就是从input_details和output_details中获取输入和输出量化参数，然后把模型输入向量手动量化了喂给模型，得到的输出再手动反量化一下。
LSTM模型在转化为tflite的时候，正常都会报错，大意是维度是动态的不能变成静态的，解决方法如下：
```
run_model = tf.function(lambda x: model(x))
step = 6
concrete_func = run_model.get_concrete_function(tf.TensorSpec([BATCH_SIZE, int(HISTORY_SIZE/step), INPUT_SIZE], model.inputs[0].dtype))
MODEL_DIR="saved_model"
model.save(MODEL_DIR,save_format="tf",signatures=concrete_func)
converter=tf.lite.TFLiteConverter.from_saved_model(MODEL_DIR)
```
怀疑是要指定输入的大小。总之就是这样指定一下输入的确定大小，然后重新保存一遍模型，保存的时候，加个concrete_func，然后再从这个保存的模型中加载converter就行了。
另外在训练lstm的时候指定参数unroll=True也能解决这个问题，但是结果是得到的tflite文件太大，并不能满足MCU上模型的要求。
在量化为全8bit的时候需要指定representative_dataset，例如下面所示：
```
def representative_dataset():
  for data, label in train_dataset.take(10000):
    yield [tf.dtypes.cast(data, tf.float32)]
```
正常我们取训练数据集的一个子集就行了，需要注意的是，取的子集大小不能太小。之前我的train_dataset的batch_size是256，所以这里take了100个，但是后面batch_size改为1以后，忘记把take的数量修改，导致tflite模型的输出几乎是一条直线，原理我猜是representative_dataset太小，导致tflite对模型的输入的范围估计错误。把take的数量修改为10000后问题就解决了。
## tensorflow中的模型保存与加载方式
#### tensorflow1.x中session训练的模型
保存：
例子：
```
saver = tf.train.Saver()
with tf.Session() as sess:
    saver.save(sess, "model_dir/model.ckpt")
```
保存的model_dir文件夹下有以下几种文件：
- checkpoint: 记录了最近 5 次（创建 tf.train.Saver 类对象时，参数 max_to_keep 的默认值）训练所保存的模型文件的一个列表，该文件可以通过普通文本编辑器打开查看；
- .meta文件: 保存了模型的图（网络结构）；
- .index文件: 描述variable中key和value的对应关系
- .data-00000-of-00001 包含训练变量的文件
另外，在model和.meta、.index、.data之间还有个数字，表示保存这个文件时训练的epoch个数
加载：
```
saver = tf.train.Saver()
with tf.Session() as sess:
    saver.restore(sess, 'model.dir/model.ckpt')
```
#### keras保存的模型
保存：
```
tf.saved_model.save(model, 'model_dir)
# 或者
model.save('model.h5')
```
加载：
```
model = tf.keras.models.load_model('model_dir')
# 或者
model = tf.keras.models.load_model('model.h5')
```
## 将tensorflow模型转化为tflite模型
#### 从saved_model形式转化：
```
converter = tf.lite.TFLiteConverter.from_saved_model(saved_model_dir) # path to the SavedModel directory
tflite_model = converter.convert()

# Save the model.
with open('model.tflite', 'wb') as f:
  f.write(tflite_model)
```
#### 从h5文件转化：
```
converter = tf.lite.TFLiteConverter.from_keras_model('model.h5')
...
```
#### 从pb文件转化：
```
tf.compat.v1.lite.TFLiteConverter.from_frozen_graph(graph_def_file, input_arrays, output_arrays, input_shapes=None)
# 其中，input_arrays表示输入的tensor，output_arrays表示输出的tensor，都是必须指定的
```
#### 从session转化：
```
With tf.Session() as sess:
    # 训练过程省略
    tf.compat.v1.lite.TFLiteConverter.from_session(sess, input_tensors, output_tensors)
```
#### 从checkpoint文件转化：
首先把checkpoint文件转化为pb文件
```
import tensorflow as tf

meta_path = 'model.ckpt-22480.meta' # Your .meta file
output_node_names = ['output:0']    # Output nodes

with tf.Session() as sess:
    # Restore the graph
    saver = tf.train.import_meta_graph(meta_path)

    # Load weights
    saver.restore(sess,tf.train.latest_checkpoint('path/of/your/.meta/file'))
    init=tf.global_variables_initializer()
    sess.run(init)
    # Freeze the graph
    frozen_graph_def = tf.graph_util.convert_variables_to_constants(
        sess,
        sess.graph_def,
        output_node_names)

    # Save the frozen graph
    with open('output_graph.pb', 'wb') as f:
      f.write(frozen_graph_def.SerializeToString())
```
必须要知道原模型输出节点的名字。查看输出节点名字可以用Netron输入原模型查看，或者把所有节点作为输出节点：
```
output_node_names = [n.name for n in tf.get_default_graph().as_graph_def().node]
```
注意这句话必须在convert_variable_to_constants之前写
转化成pb文件以后继续按上面的方式来转化。