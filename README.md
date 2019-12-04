# machine-learning-remote-ue4
Machine Learning plugin for the Unreal Engine, encapsulating calls to remote python servers running Tensorflow.

Should have the same api as tensorflow-ue4, but with freedom to run a host server on platform of choice (e.g. remote win/linux/mac instances).

## Machine Learning Plugin Variants

Want to run tensorflow or pytorch on a remote python server?

- https://github.com/getnamo/machine-learning-remote-ue4

Want to use tensorflow python api with a python instance embedded in your unreal engine project? 

- https://github.com/getnamo/tensorflow-ue4

Want native tensorflow inference? (WIP)

- https://github.com/getnamo/tensorflow-native-ue4

## Quick Install & Setup

1. Install https://github.com/getnamo/ml-remote-server on your target backend (can be a local folder)
2. Download [Latest Release](https://github.com/getnamo/machine-learning-remote-ue4/releases)
3. Create new or choose project.
4. Browse to your project folder (typically found at Documents/Unreal Project/{Your Project Root})
5. Copy Plugins folder into your Project root.
6. Plugin should be now ready to use. Remember to startup your [server](https://github.com/getnamo/ml-remote-server) when using this plugin. 

## How to use

### Blueprint API

Add a ```MachineLearningRemote``` component to an actor of choice

![](https://i.imgur.com/Mx3gNAi.png)

Change server endpoint and default script (see https://github.com/getnamo/machine-learning-remote-ue4#pythonapi) to fit your use case

![](https://i.imgur.com/R3YVPtm.png)

In your script the ```on_setup``` and if ```self.should_train_on_start``` is true ```on_begin_training``` gets called. When your script has trained or it is otherwise ready, you can send inputs to it using ```SendSIOJsonInput``` or other variants (string/raw).

![](https://i.imgur.com/WjmFLAu.png)

Your inputs will process and any value you return will be processed in the callback and returned in ```ResultData``` as *USIOJsonValue* in your latent callback.

#### Other input variants

See https://github.com/getnamo/machine-learning-remote-ue4/blob/master/Source/MachineLearningRemote/Public/MachineLearningRemoteComponent.h for all variants

#### Custom Function

Change the ```FunctionName``` parameter in the ```SendSIOJsonInput``` to call a different function name in your script. This name will be used verbatim.

### Python API
These scripts should be placed in your https://github.com/getnamo/ml-remote-server ```scripts``` folder. If a matching script is defined in your ```MachineLearningRemote```->```DefaultScript``` property it should load on connect.

Keep in mind that ```tensorflow``` is optional and used as an illustrative example of ML, you can use any other valid python library e.g. pytorch instead without issue.

See https://github.com/getnamo/ml-remote-server/tree/master/scripts for additional examples.

#### empty_example

Bare bones API example.

```python
import tensorflow as tf
from mlpluginapi import MLPluginAPI

class ExampleAPI(MLPluginAPI):

	#optional api: setup your model for training
	def on_setup(self):
		pass
		
	#optional api: parse input object and return a result object, which will be converted to json for UE4
	def on_json_input(self, input):
		result = {}
		return result

	#optional api: start training your network
	def on_begin_training(self):
		pass


#NOTE: this is a module function, not a class function. Change your CLASSNAME to reflect your class
#required function to get our api
def get_api():
	#return CLASSNAME.getInstance()
	return ExampleAPI.get_instance()
```
#### add_example

Super basic example showing how to add using the tensorflow library.

```python
import tensorflow as tf
from mlpluginapi import MLPluginAPI

class ExampleAPI(MLPluginAPI):

	#expected optional api: setup your model for training
	def on_setup(self):
		self.sess = tf.InteractiveSession()
		#self.graph = tf.get_default_graph()

		self.a = tf.placeholder(tf.float32)
		self.b = tf.placeholder(tf.float32)

		#operation
		self.c = self.a + self.b

		print('setup complete')
		pass
		
	#expected optional api: parse input object and return a result object, which will be converted to json for UE4
	def on_json_input(self, jsonInput):
		
		print(jsonInput)

		feed_dict = {self.a: jsonInput['a'], self.b: jsonInput['b']}

		rawResult = self.sess.run(self.c, feed_dict)

		print('raw result: ' + str(rawResult))

		return {'c':rawResult.tolist()}

	#custom function to change the op
	def change_operation(self, type):
		if(type == '+'):
			self.c = self.a + self.b

		elif(type == '-'):
			self.c = self.a - self.b
		print('operation changed to ' + type)


	#expected optional api: start training your network
	def on_begin_training(self):
		pass
    
#NOTE: this is a module function, not a class function. Change your CLASSNAME to reflect your class
#required function to get our api
def get_api():
	#return CLASSNAME.get_instance()
	return ExampleAPI.get_instance()
```

#### mnist_simple

One of the most basic ML examples using tensorflow to train a softmax mnist recognizer.

```python
#Converted to ue4 use from: https://www.tensorflow.org/get_started/mnist/beginners
#mnist_softmax.py: https://github.com/tensorflow/tensorflow/blob/r1.1/tensorflow/examples/tutorials/mnist/mnist_softmax.py

# Import data
from tensorflow.examples.tutorials.mnist import input_data

import tensorflow as tf
import unreal_engine as ue
from mlpluginapi import MLPluginAPI

import operator

class MnistSimple(MLPluginAPI):
	
	#expected api: storedModel and session, json inputs
	def on_json_input(self, jsonInput):
		#expect an image struct in json format
		pixelarray = jsonInput['pixels']
		ue.log('image len: ' + str(len(pixelarray)))

		#embedd the input image pixels as 'x'
		feed_dict = {self.model['x']: [pixelarray]}

		result = self.sess.run(self.model['y'], feed_dict)

		#convert our raw result to a prediction
		index, value = max(enumerate(result[0]), key=operator.itemgetter(1))

		ue.log('max: ' + str(value) + 'at: ' + str(index))

		#set the prediction result in our json
		jsonInput['prediction'] = index

		return jsonInput

	#expected api: no params forwarded for training? TBC
	def on_begin_training(self):

		ue.log("starting mnist simple training")

		self.scripts_path = ue.get_content_dir() + "Scripts"
		self.data_dir = self.scripts_path + '/dataset/mnist'

		mnist = input_data.read_data_sets(self.data_dir)

		# Create the model
		x = tf.placeholder(tf.float32, [None, 784])
		W = tf.Variable(tf.zeros([784, 10]))
		b = tf.Variable(tf.zeros([10]))
		y = tf.matmul(x, W) + b

		# Define loss and optimizer
		y_ = tf.placeholder(tf.int64, [None])

		# The raw formulation of cross-entropy,
		#
		#   tf.reduce_mean(-tf.reduce_sum(y_ * tf.log(tf.nn.softmax(y)),
		#                                 reduction_indices=[1]))
		#
		# can be numerically unstable.
		#
		# So here we use tf.losses.sparse_softmax_cross_entropy on the raw
		# outputs of 'y', and then average across the batch.
		cross_entropy = tf.losses.sparse_softmax_cross_entropy(labels=y_, logits=y)
		train_step = tf.train.GradientDescentOptimizer(0.5).minimize(cross_entropy)

		#update session for this thread
		self.sess = tf.InteractiveSession()
		tf.global_variables_initializer().run()

		# Train
		for i in range(1000):
			batch_xs, batch_ys = mnist.train.next_batch(100)
			self.sess.run(train_step, feed_dict={x: batch_xs, y_: batch_ys})
			if i % 100 == 0:
				ue.log(i)
				if(self.should_stop):
					ue.log('early break')
					break 

		# Test trained model
		correct_prediction = tf.equal(tf.argmax(y, 1), y_)
		accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32))
		finalAccuracy = self.sess.run(accuracy, feed_dict={x: mnist.test.images,
										  y_: mnist.test.labels})
		ue.log('final training accuracy: ' + str(finalAccuracy))
		
		#return trained model
		self.model = {'x':x, 'y':y, 'W':W,'b':b}

		#store optional summary information
		self.summary = {'x':str(x), 'y':str(y), 'W':str(W), 'b':str(b)}

		self.stored['summary'] = self.summary
		return self.stored

#required function to get our api
def get_api():
	#return CLASSNAME.get_instance()
	return MnistSimple.get_instance()
```
