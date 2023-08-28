# How to write an MHub-IO Module

All operations, e.g., converting data from one format into another, organizing files, resample data, running an AI pipeline, etc. are separated in MHub-IO Modules and sequentially executed at run-time. Although the tasks various modules perform on any data can be anything, all modules share a similar data interface for incoming and generated data and can be configured through a single configuration file. The config file not only defines any customizable parameters for all modules in a central place it also describes the entire module chain, making the MHub-IO pipeline understandable and, implied by the high degree of flexibility offered by MHub-IO, simple to adapt.

When writing a new MHub-IO, a few design guidelines should be followed.

## Module class

Start with defining a class for your module. All MHub-IO modules inherit from the Module base class.

```python
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

Now that we have fetched input data, we need to know where to put any generated output data. MHub-IO will keep the connection in the background. Again, instead of dealing with the underlying references or paths, we simply provide an `@IO.Output` decorator to define the data we generate. The decorator works similarly to the `@IO.Input` decorator introduced earlier. This time, we set the optional data attribute to the name of the input data. Thereby, we set the reference, saying “out_data is produced based on in_data,' whatever in_data and out_data are. We also add the parameter out_data of type InstanceData to the signature of the `task()` method. Inside the `task()` method, we can then use `out_data.abspath` to access the path that MHub-IO generated for us to put the generated data. MHub-IO would perform automated checks in the background if the file were created.

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
      print('stor results in ' + out_data.abspath)
```