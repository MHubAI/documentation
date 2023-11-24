# How to write an MHub-IO Module

All operations, e.g., converting data from one format into another, organizing files, resample data, running an AI pipeline, etc. are separated in MHub-IO Modules and sequentially executed at run-time. Although the tasks various modules perform on any data can be anything, all modules share a similar data interface for incoming and generated data and can be configured through a single configuration file. The config file not only defines any customizable parameters for all modules in a central place it also describes the entire module chain, making the MHub-IO pipeline understandable and, implied by the high degree of flexibility offered by MHub-IO, simple to adapt.

When writing a new MHub-IO, a few design guidelines should be followed.

## Module class

Start with defining a class for your module. All MHub-IO modules inherit from the Module base class.

```python
from mhubio.core import Module

class MyModule(Module):
   pass
```

By inheriting from Module, the MyModule class now takes a Config instance as the only argument at instantiation. The parent class Module has `execute()` and `task()` methods. As the name implies, the first executes the module and must not be overridden. The `task()` method needs to be overridden with the custom implementation, the 'task' of the module.

```python
class MyModule(Module):
   def task(self):
      pass
```

## Configurable parameters

Next, think about any parameter that needs to be customizable. For example, a module might have a fast mode, which will be used by default but can be disabled by the user. The type of such a parameter is a boolean, as this is either true or false. We add the parameter to the module and expose it using the `@IO.Config` decorator.

```python
from mhubio.core import IO

@IO.Config('fast', bool, True, the='option to enable fast mode')
class MyModule(Module):
   fast: bool

   def task(self):
      print('fast mode is ' + 'enabled' if self.fast else 'disabled')
```

The default value of `MyModule(config).fast` will be `true`. However, it can be overridden using the configuration file (config.yml):

```yaml
modules:
   MyModule:
      fast: false
```

In edge cases, the parameter can also be overridden in python (e.g., in a custom run.py script)

```python
from mhubio.core import Config

config = Config('config.yml')

m = MyModule(config)
m.fast = false
m.execute()
```

## The task method

The `task()` method will be executed on each instance. MHub-IO handles the instance management in the background and will automatically pass one instance at a time to our task method. The `@IO.Instance` decorator signalizes that the decorated method can run on a single instance.

```python
@IO.Config('fast', bool, True, the='option to enable fast mode')
class MyModule(Module):
   fast: bool

   @IO.Instance
   def task(self, instance: Instance):
      print('called on instance: ', str(instance))

```

Now it is time to implement the task of the module. A task can operate on a single instance or on multiple instances. An instance is a single sample, e.g. a patient's CT scan, that holds all data relevant to that instance. Data always has a file type, e.g. NRRD, NIFTI, DICOM, and a variable amount of key-value pairs, the metadata. When working with data on MHub-IO, it is important to understand that the framework handles all file-level organization. Instead of passing references and assembling paths using `os.path.join(..)`, any data can be fetched from an instance using a semantic description of their file type and a list of required metadata. An example for an NRRD file that has the metadata attribute mod (for modality) set to CT in string format: “nrrd:mod=ct'.

## Defining the input data of the module

We first define what we need to add input data to our task. Let's assume we need a single input, a nifti file, and as in the example before should be a CT scan. We then add a `@IO.Input` decorator to the task method that will fetch the data automatically and provide it as an input parameter to the decorated function. The `@IO.Input` takes the parameter name as the first argument. In the example below, this is set to 'in_data', so we add a parameter `in_data` of type `InstanceData` to the signature of the decorated task method. The second argument is used to describe the data type of the input data using a string representation, starting with the file type, followed by any number of meta-key-value pairs separated by a colon. We describe what the input data is in the last argument choosing a structure that reads like *this input data is the ...*.

In this example, the input data is of type NIFTI and has the modality set to ct which is encoded as `nifti:mod=ct`.

```python
@IO.Config('fast', bool, True, the='option to enable fast mode')
class MyModule(Module):
   fast: bool

   @IO.Instance
   @IO.Input('in_data', 'nifti:mod=ct', the='chest ct scan')
   def task(self, instance: Instance, in_data: InstanceData):
      print('input data: ' + in_data.abspath)
```

## Defining the output data of the module

Now that we have fetched input data, we need to know where to put any generated output data. MHub-IO will keep the connection in the background. Again, instead of dealing with the underlying references or paths, we simply provide an `@IO.Output` decorator to define the data we generate. The decorator works similarly to the `@IO.Input` decorator introduced earlier. This time, we set the optional data attribute to the name of the input data. Thereby, we set the reference, saying *'out_data is produced based on in_data,'* whatever in_data and out_data are. We also add the parameter `out_data` of type `InstanceData` to the signature of the `task()` method. Inside the `task()` method, we can then use `out_data.abspath` to access the path that MHub-IO generated for us to put the generated data. MHub-IO would perform automated checks in the background if the file were created.

```python
@IO.Config('fast', bool, True, the='option to enable fast mode')
class MyModule(Module):
   fast: bool

   @IO.Instance
   @IO.Input('in_data', 'nifti:mod=ct', the='chest ct scan')
   @IO.Output('out_data', 'nifti:mod=ct:processed=true', data='in_data', 
the='processed ct scan')
   def task(self, instance: Instance, in_data: InstanceData, out_data: InstanceData):
      print('input data: ' + in_data.abspath)
      
      # do sth with the input data
      print('stor results in ' + out_data.abspath)
```

## Configurable Input Data

In our previous example, we showed how to define an input using the `@IO.Input` decorator. This develops a static query of the input data that a module asks for, nifti data that has the modality CT in the above example. However, it may be that your module accepts nrrd files or works with both CT and MR data because you handle both modalities internally. If that is the case, you may want to allow some control over your model's input to the configuration file. Even if you only accept a specific file type and modality, the files may contain additional custom metadata that users may want to filter on. Therefore, it is a good practice to expose the query via the configuration file, but define a reasonable default value. You can (and should) always check the actual file type or metadata within your module implementation for the data that your task implementation supports anyway. Note that we only mention inputs. This is because while your module can work with different input data, it will always produce consistent output data. If you have different configurations in which your model can generate data, we recommend you implement them in separate modules.

We provide a special decorator for this use case. The `@ IO.ConfigInput` decorator takes four arguments. First, you specify the `name` of the input. The same name is used to override the semantic data query from the configuration file. As a second argument you specify a `default` semantic query, which should cover the most common use cases. In our example, we used a default value that covers all of the above options, namely nifti and nrrd files, and ct and mr modality simultaneously. The third argument `class_attribute`, which defaults to `false`, can be set to `true` if you want to expose the configurable parameter as a class attribute to allow programmatic setting of the parameter, for example in custom run.py scripts, but you probably won't need this option and we don't recommend it unless you have a good reason. The fourth argument is `the` description of the configurable parameter. If you want to use the configurable input data in your `IO.Input` decorator, just specify the same name and they will be linked automatically. You must remove the dtype argument here, as it is automatically derived from the default value or the value specified in the configuration file.

```python
@IO.Config('fast', bool, True, the='option to enable fast mode')
@IO.ConfigInput('in_data', 'nifti|nrrd:mod=ct|mr', the='chest ct scan')
class MyModule(Module):
   fast: bool

   @IO.Instance
   @IO.Input('in_data', the='chest ct scan')
   @IO.Output('out_data', 'nifti:mod=ct:processed=true', data='in_data', 
the='processed ct scan')
   def task(self, instance: Instance, in_data: InstanceData, out_data: InstanceData):
      print('input data: ' + in_data.abspath)
      
      # do sth with the input data
      print('store results in ' + out_data.abspath)
```

## Handling Logical Output Data

Unlike segmentation models, classification or prediction models typically don't produce a file-based output, but a single prediction number or vector. Technically, this is also true for segmentation models, however, due to their pre- and post-processing chain, they usually generate an image as output and provide metadata such as orientation and position in 3D image space for their numerical output. Similarly, some prediction and classification models can produce a file-based report, such as a JSON, CSV, or TXT file. You can use the `@IO.Output` decorator to export these files. However, if your model predicts a one-dimensional vector, we provide a different set of decorators to describe and store this data. This is of interest to you even if your model generates its own file. This is because we store this data internally in a structured format and provide a flexible export module to create [JSON reports](mhubio_modules.md#reportexporter), for example.

### Defining a Value Output

This method is used to define a single value output, e.g. a risk prediction value.

First, define a class that inherits from `ValueOutput` to describe your predicted value. In this example, we define a value output for a model that predicts the Agatston score.

```python
from mhubio.core import ValueOutput

class AgatstonScore(ValueOutput):
   pass
```

We now need to describe the name and type of this value output and give a label, description, and optional meta data. For these, we provide a set of decorators. The `@ValueOutput.Name(str: name)` decorator is used to set an unique name. This name will then be used to reference your value in the Module class and to include the value in a custom report. The `@ValueOutput.Type(type: type)` decorator is used to set the type of the value. The type can be any python type, e.g. `int`, `float`, `str`, `bool`. The `@ValueOutput.Label(str: label)` decorator is used to set a human-readable name. The `@ValueOutput.Description(str: description)` decorator is used to set long-form a human-readable description for the value. You can also specify metadata as known from file based  inputs and outputs using the optional `@ValueOutput.Meta(dict: meta)` decorator. The meta data can be used to store additional information about the value, e.g. the unit of the value.

```python
from mhubio.core import Meta

@ValueOutput.Name('ags')
@ValueOutput.Meta(Meta(key="value"))
@ValueOutput.Label('AgatstonScore')
@ValueOutput.Type(int)
@ValueOutput.Description('Prediction of the agatson score.')
class AgatstonScore(ValueOutput):
   pass
```

The class serves only as a wrapper for our value. All functions are inherited from the base class `ValueOutput` and you shouldn't add any additional implementation to the class.

Now to access the value in your module, use the `@IO.OutputData` decorator and specify the name of the value output, the class of the value output, optionally the data from which the value is derived if it can be associated with a single file similar to the `@IO.Output` decorator, and a description of the value output. The value output is passed to the task method as an additional parameter with the name you specified as the first argument of the decorator, and with a type of your value output class (`AgatstonScore` in this example`).

```python
@IO.Config('fast', bool, True, the='option to enable fast mode')
@IO.ConfigInput('in_data', 'nifti|nrrd:mod=ct|mr', the='chest ct scan')
class MyModule(Module):
   fast: bool

   @IO.Instance
   @IO.Input('in_data', the='chest ct scan')
   @IO.OutputData('ags', AgatstonScore, data='in_data', the='agatston score')
   def task(self, instance: Instance, in_data: InstanceData, ags: AgatstonScore):
      print('input data: ' + in_data.abspath)
      
      # set the value on your output value
      ags.value = 130
```

As shown in the example above, you simply set the `value` to the instance of your output value class, which is automatically created by the decorator `@IO.OutputData`.

### Defining a Vector Output

With class prediction models, you usually have a little more information. They are based on the classes that the model can predict, each of which deserves its own label and description. In addition, each class may have its own probability that you want to specify along with the final decision of the model (the predicted class). This type of vector output can be stored in a class output. In the following example, we define a class output for the risk category, which can be either low, medium or high.

First, define a class that inherits from `ClassOutput` to describe your predicted class. In this example, we define a class output for a model that predicts the risk category.
Similarly to a value output as introduced before, we specify a name, label and description (note, we use decorators from `@IO.ClassOutput` instead of `@IO.ValueOutput`).

```python
from mhubio.core import ClassOutput

@ClassOutput.Name('rc')
@ClassOutput.Label('RiskCategory')
@ClassOutput.Description('Prediction of the risk category.')
class RiskCategory(ClassOutput):
   pass
```

Now, we need to define the classes that the model can predict. For each class we can specify a `@ClassOutput.Class(classID: str | int, label: str, the: str)`. The decorator `@ClassOutput.Class` is the only decorator that can be used multiple times. You use one per output class you want to define. You can then later reference each class by its `ClassID`. The classID can be specified either numerically or as a string. Whenn choosing a string-based ClassID, the string should be as short as possible and contain only alphanumerical characters, understcores and dashes. Specify a human readable class name in `label: str` and as usual a human readable description of the class in the `the: str` argument.

```python

```python
@ClassOutput.Name('rc')
@ClassOutput.Label('RiskCategory')
@ClassOutput.Description('Prediction of the risk category.')
@ClassOutput.Class('low', 'Low', the='Class describing the lowest risk group.')
@ClassOutput.Class('moderate', 'Moderate', the='Moderate risk.')
@ClassOutput.Class('high', 'High', 'High risk.')
class RiskCategory(ClassOutput):
   pass
```

Extending our example from earlier, we can now provide our class output format similarly to the value output using an `@IO.OutputData` decorator on our Module's `task()` method.

```python
@IO.Config('fast', bool, True, the='option to enable fast mode')
@IO.ConfigInput('in_data', 'nifti|nrrd:mod=ct|mr', the='chest ct scan')
class MyModule(Module):
   fast: bool

   @IO.Instance
   @IO.Input('in_data', the='chest ct scan')
   @IO.OutputData('ags', AgatstonScore, data='in_data', the='agatston score')
   @IO.OutputData('rc', RiskCategory, data='in_data', the='risk category derived from the agatston score')
   def task(self, instance: Instance, in_data: InstanceData, ags: AgatstonScore, rc: RiskCategory):
      print('input data: ' + in_data.abspath)
      
      # set the value on your output value
      ags.value = 130

      # set the class output
      rc.value = 'moderate'   # use the classID here

      # optionally assign probabilities to each classses simultaneously
      rc.assign_probabilities({
         'low': 0.1,
         'moderate': 0.7,
         'high': 0.2
      })

      # you can also pass a list with probabilities in the same order 
      #  as the classes were specified in the decorators
      rc.assign_probabilities([0.1, 0.7, 0.2])

      # or assign the probability for each class separately via its classID
      rc['low'].probability = 0.1
      rc['moderate'].probability = 0.7
      rc['high'].probability = 0.2
```

We demonstrated how to set the prediction values either using the `assign_probabilities` method or by setting the probability for each class individually. You can access each class by class ID (e.g. `rc['moderate']` to access the moderate class). You can also pass a list instead of a dictionary to the `assign_probabilities` method and set the probability for each class in the order you specified the classes in the decorators.

## Handling Multiple and Dynamic Inputs and Outputs

In some cases we want to create a module that takes more than one input file or generates more than one output file. Simply use more than one `@IO.Input` or multiple `@IO.Output` decorators to explicitly specify each input and each output. This is all you need for most scenarios and should always be the preferred method.

However, sometimes this is not enough and we want to specify dynamic length inputs or outputs. MHub-IO is able to handle these cases as well. Note that this is an advanced use case. Therefore, carefully consider whether what you want to achieve cannot be accomplished by using multiple individual input or output decorators to optimize the readability of your module implementation.

### Dynamic Input [1:n]

Let's look at some examples where dynamic input is actually required. Take, for example, a module that takes in all dicom CT files and outputs the number of these files. While this has little practical significance, there are many scenarios where such an `n:1` operation is required, e.g. reporting the maximum HU values for multiple files.

```Python
@ValueOutput.Name('maxHU')
@ValueOutput.Label('Maximal HU value')
@ValueOutput.Type(int)
@ValueOutput.Description('The maximal HU value over all CT scans.')
class MaxHU(ValueOutput):
   pass

def getMaxHU(dicom_dir) -> int:
   # ... calculate and get the HU here
   return maxHU
   
class MyModule(Module):

   @IO.Instance
   @IO.Inputs('in_datas', 'dicom:mod=ct', the='all MHA files to be converted')
   @IO.OutputData('maxHU', MaxHU)
   def task(self, instance: Instance, in_datas: InstanceDataCollection, maxHU: MaxHU):
     
     # print the number of files matching dicom:mod=ct on the instance
     print(f"This instance has {len(in_datas)} dicom CT files.")

     # calculate the max HU value
     maxHU.value = max(getMaxHU(in_data.abspath) for in_data in in_datas)
```

Now, instead of the `@IO.Input` decorator we use the `@IO.Inputs` and instead of a single `InstanceData` we get a `InstanceDataCollection`. For a better readability, we choose `in_datas` (plural) as a name for our input variable. The `InstanceDataCollection` contains all `InstanceData` matching the query `dicom:mod=ct` but any other query can be used. The `InstanceDataCollection` is iteratable. To fetch a specific data or subset, use one of the following methods: `.get(index)`, `.first()` or `.filter('any:mod=ct')`.

### Multiple pairwise linked Inputs and Outputs [n:n]

Now for the next example, consider a module that takes all segmentation mha files `mha:mod=seg` and converts them into nifti files `nifti:mod=seg`. This is a `n:n` operation, where the number of input files is dynamic and the number of output files is dynamic. However, each input file corresponds with exqactly one output file.

```Python
class MyModule(Module):

   @IO.Instance
   @IO.Inputs('in_datas', 'mha:mod=seg', the='all MHA files to be converted')
   @IO.Outputs('out_datas', '[basename].nii.gz', 'nifti', data='in_datas', the='mha files converted to nifti')
   def task(self, instance: Instance, in_datas: InstanceDataCollection, out_datas: InstanceDataCollection):
     
      # loop over all in_data-out_data pairs
      for i, in_data in enumerated(in_datas):
         out_data = out_datas.get(i)
      
         # conversion 
         mha2nifti(in_data.abspath, out_data.abspath)

```

Now, instead of the `@IO.Input` and `@IO.Output` decorators we use the `@IO.Inputs` and `@IO.Outputs`decorators and instead of a single `InstanceData` we get a `InstanceDataCollection`.

Because we link the outputs with the inputs by using the input datas as reference data for the output datas (`data='in_datas'`), MHub-IO will automatically generate an `InstanceData` object for each input and will also take care of transferring any metadata information from each input data to it's linekd output data. This is why we can access for each input data a related output data via `out_datas.get(i)` in the body of our `task()` method.

Because this time we create multiple files, we cannot specify a static file name (e.g. converted.nii.gz), because they would overwrite each other. (An excemption is, if all input_data instances already have distinct bundles). We therefore can use placeholders (as described [here](https://github.com/MHubAI/documentation/blob/main/documentation/mhubio/mhubio_modules.md#dataorganizer)) in the file path which are replaced by properties from the linked input file automatically by MHub-IO. Alternatively, one can specify the `auto_increment=True` argument on the `@IO.Outputs` decorator which will automatically append a `_{n+1}` to the file name if the file already exists on disk or the filename does already exist in the instance (including non-confirmed files).

The described technique works similarly for logical output data via the `@IO.OutputDatas` decorator, including the linking to input files specified in the reference data argument `data='in_datas'`.

### Dynamic Output [1:n]

We can specify dynamic output by removing the `data` link in the `@IO.Outputs` decorator and manually create `InstanceData` objects. This is a highly advanced technique that usually shouldn't be used because it makes the the generated output of a Module ambiguous and prevents static interpretation.

```Python
class MyModule(Module):

   @IO.Instance
   @IO.Outputs('out_datas', 'random.txt', 'txt', the='generated output files')
   def task(self, instance: Instance, out_datas: InstanceDataCollection):
     
     # create files dynamically
      for i in range(0, random.randint(1, 5)):

         # specify datatype
         dtype = DataType.fromString(f'txt:round={i+1}')

         # instanciate instance data
         # link instance data to a bundle or an instance (which is the same as calling 
         #  instance.add(data) after the initialization but would prevent the auto_increment path resolution)
         data = InstanceData('outa.txt', dtype, instance=instance, auto_increment=True) 

         # generate file
         with open(data.abspath, 'w') as f: 
            f.write("Hello World")

         # append the generated instance data to the output collection to ensure 
         #  post-checks and data confirmation can be run.
         out1.add(data)
```

Dynamic [logical output](#handling-logical-output-data) works similarly and can be required in scenarios where a dynamic number of values are generated based on a static number of input data or input files.
Imagine, you trained a lung-nodule algorithm that takses a single CT file in nifti format and will return for each detected lung nodule some location parameters and a risk prediction.

```Python
@ValueOutput.Name('lnrisk')
@ValueOutput.Label('Lung Nodule Risk-Score.')
@ValueOutput.Type(int)
@ValueOutput.Description('The predicted risk score for a single lung nodule detected by the alggorithm.')
class LNRisk(ValueOutput):
   pass

def getLungNodulesRiskScores(dicom_dir) -> List[int]:
   # ... find lung nodules, and report back an array of risk scores
   return lst_scores
   
class MyModule(Module):

   @IO.Instance
   @IO.Input('in_data', 'dicom:mod=ct', the='chest CT image')
   @IO.OutputDatas('lnrisks', LNRisk)
   def task(self, instance: Instance, in_data: InstanceData, lnrisks: LNRisk):

      scores = getLungNodulesRiskScores(in_data.abspath)

      for nodule_i, score in enumerate(scores):

         # create value output instance and set the value (we can also modify the description)
         lnrisk = LNRisk()
         lnrisk.description += f" (for nodule {nodule_i})"
         lnrisk.value = score

         # add to collection
         lnrisks.add(lnrisk)

```

### Fully Dynamic Inputs and Outputs [n:m]

An arbitrary number of inputs and outputs of any kind (without relational linking between inputs and outputs) can be reched by an independend combination of the [1:m] and [n:1] scenarios, thus by using `@IO.InputDatas` and `@IO.OutputDatas` on the same Module without specifying a data reference `datas='...'` on the `@IO.OutputDatas` decorator, keeping the number of outputs independend from the number of input data.

## Console output and Logging

MHub controls what is displayed in the console output. By default, we display the progress of the MHub workflow and catch all log messages in log files, which are then attached to the affected instance and module (but you can also display the console output with the `--print` cli flag instead).

### Log messages

For logging messages, the `Module` class provides a `log(*args, level: str = 'NOTICE')` function. You can specify one of the following log levels:

- `NOTICE` is a general information message. It is used to inform the user about the progress of the module. This is the default log level and should be used for all general information messages.
- `WARNING` informs the user about a possible problem. The module will continue to run, but the user should heed the situation.
- `DEPRECATED` informs the user that a feature is deprecated and will be removed in the future. The module will continue to run, but the user should be aware that it is deprecated.
- `ERROR` informs the user that an error has occurred. The module may stop or produce incomplete output.
- `DEBUG` is used for debug messages to provide additional information for debugging.
- `EXTERNAL` is used by us for comsole output generated by an external processes that is started by the module.
- `CAPTURED` all console outputs that we capture, for example, with the `print()` instruction in the `task()` method of the module are logged under the captured log level.

```python
class MyModule(Module):
   fast: bool

   def task(self):

      # print statements are logged as CAPTURED. 
      # We recommend the use of self.log instead
      print('input data:', in_data.abspath)
      
      # In some of our examples you may find the old self.v method.
      # This is a legacy method and you should use the self.log method isntead.
      self.log('input data:', in_data.abspath)

      # if you want to log a message with a different log level, you can specify it as the level argument
      self.log('input data:', in_data.abspath, level='WARNING')
```

### Running a Subprocess from a Module

If you want to start a subprocess from your module, you must use the `self.subprocess()` method provided by the `Module` base class. This method takes the same arguments as the Python `subprocess.Popen()` method. The method catches all console output and logs it as `EXTERNAL` log messages.

```python
class MyModule(Module):
   fast: bool

   def task(self):

      # build command
      cmd = [
         'echo', 
         'Hello World!'
      ]

      # run the command as subprocess 
      # the console outpout (Hello World) will be logged as EXTERNAL
      self.subprocess(cmd, text=True) 
```
