# Tornasole Analysis
Tornasole is an upcoming AWS service designed to be a debugger for machine learning models. 
It lets you go beyond just looking at scalars like losses and accuracies during training and gives you 
full visibility into all tensors 'flowing through the graph' during training or inference.

Tornasole's analysis module helps you analyze tensors saved from machine learning jobs.
It allows you to run Rules on these tensors as well as anything else you might want to do with 
access to raw tensors such as inspection or visualization. It provides access to the tensors in the form of numpy arrays. 

## The Programming Model
The library is organized using the following constructs.

### Trial
Trial the construct which lets you query for tensors for a given Tornasole run, specified by the path in which Tornasole artifacts are being saved or were saved. 
You can pass a path which holds data for a past run (which has ended) as well as a path for a current run (to which tensors are being written).
Trial is capable of loading new tensors as and when they become available at the given location.     

There are two types of trials you can create: LocalTrial or S3Trial. 
We provide a wrapper method to create the appropriate trial. 

The parameters you have to provide are:
- `name`: name can be any string. It is to help you manage different trials. 
Make sure to give it a unique name to prevent confusion.
- `path`: path can be a local path or an S3  path of the form `s3://bucket/prefix`. This path should be where Tornasole hooks (TF or MXNet) save data to. 
You should see the directory `events` and the file `collections.ts` in this path.

##### Creating local trial
```
from tornasole.trials import create_trial
trial = create_trial(path='/home/ubuntu/tornasole_outputs/train', 
                     name='resnet_training_run')
```
##### Creating S3 trial
```
from tornasole.trials import create_trial
trial = create_trial(path='s3://tornasole-testing-bucket/outputs/resnet', 
                     name='resnet_training_run')
```
###### Restricting analysis to a range of steps
To any of these methods you can optionally pass `range_steps` to restrict your analysis to a certain range of steps.
Note that if you do so, Trial will not load data from other steps.

*Examples*
- `range_steps=(100, None)`: This will load all steps after 100
- `range_steps=(None, 100)`: This will load all steps before 100
- `range_steps=(100, 200)` : This will load steps between 100 and 200
- `range_steps=None`: This will load all steps  

```
lt = create_trial(path='ts_outputs/resnet', name='resnet_training',
                  range_steps=(100, 200))
```


### Mode
A machine learning job can be executing steps in multiple modes, such as training, evaluating, or predicting. 
Tornasole provides you the construct of a `mode` to keep data from these modes separate
and make it easy for analysis. To leverage this functionality you have to 
call the `set_mode` function of hook such as the following call `hook.set_mode(modes.TRAIN)`.
The different modes available are `modes.TRAIN`, `modes.EVAL` and `modes.PREDICT`. 

When you set a mode, steps in that mode have a sequence. We refer to these numbers 
as `mode_step`. Each `mode_step` has a global step number associated with it, which represents the
sequence of steps across all modes executed by the job.    

For example, your job executes 10 steps, out of which the first 4 are training steps, 5th is evaluation step, 6-9 are training steps, and 10th is evaluation step.
Please note that indexing starts from 0. 
In such a case, when you query for the global steps as below:
```
trial.available_steps()
``` 
you will see `[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]`.

If you query for training steps as below:
```
from tornasole_rules import modes
trial.available_steps(modes.TRAIN)
```
 you will see `[0, 1, 2, 3, 4, 5, 6, 7, 8]` because there were 8 training step. 
The training step with mode_step 4 here refers to the global step number 5. 
You can query this as follows:
```
trial.global_step(mode=modes.TRAIN, mode_step=4)
```

If you did not explicitly set a mode during the running of the job,
the steps are global steps, and are in the `modes.GLOBAL` mode. 
In such a case, `global_step` is the same as `mode_step` where mode is `modes.GLOBAL`.  

Below, we describe the above functions and others that the Trial API provides.

#### Trial API
Once you have a trial object you can do the following
**See names of all tensors available**
```
trial.tensors()
```
This returns tensors seen for any mode if mode was set during the machine learning job.

**See all steps seen by the Trial for a particular mode**

The returned list is the step number within that mode.
Each of these mode steps has a global step number associated with it.
The global step represents the sequence of steps across all modes executed by the job. 
```
from tornasole import modes
trial.available_steps(mode=modes.TRAIN)
```
**See all global steps seen by the Trial**

This is the list of steps across all modes. 
```
trial.available_steps()
```

**Get the mode and step number within mode for a given global step**
You can get the `mode` of `global_step` 100 as follows:
```
mode = trial.mode(global_step=100)
```
You can get the `mode_step` for `global_step` 100 as follows:
```
mode_step = trial.mode_step(global_step=100)
```
**Know the global step number for a given mode step**
```
from tornasole import modes
global_step_num = trial.global_step(modes.TRAIN, mode_step=10)
```
**See all modes for which the trial has data**
```
trial.modes()
```
**Access a particular tensor**
A tensor is identified by a string which represents its name.
```
trial.tensor('relu_activation:0')
```
**See the global steps for which tensor's value was saved** 
```
trial.tensor('relu_activation:0').steps()
```

**See the steps for a given mode when tensor's value was saved**
This returns the mode steps for those steps when this tensor's value was saved for this mode. 
```
from tornasole import modes
trial.tensor('relu_activation:0').steps(mode=modes.TRAIN)
```

**Get the value of the tensor at a global step**
This returns the tensor value as a numpy array for the 10th global step.
```
trial.tensor('relu_activation:0').value(10)
```
Please note that this can raise exceptions if the step is not available. 
Please see [this section](#when-a-tensor-is-not-available-during-rule-execution) for more details on the different exceptions that can be raised.
**Get the value of the tensor at a step number for a given mode**
This returns the tensor value as a numpy array for the 10th training step.
```
from tornasole import modes
trial.tensor('relu_activation:0').value(10, mode=modes.TRAIN)
```
Please note that this can raise exceptions if the step is not available. 
Please see [this section](#when-a-tensor-is-not-available-during-rule-execution) for more details on the different exceptions that can be raised.
**Get reduction value of a tensor at a step**

Tornasole provides a few reductions out of the box that you can query with the following API.
This below returns the mean of the absolute values at step 10.
```
trial.tensor('relu:0').reduction_value(10, 'mean', abs=True)
```
The different reductions you can query for are the same as what are allowed in [ReductionConfig](https://github.com/awslabs/tornasole_tf/blob/master/docs/api.md) when saving tensors.
This API thus allows you to access the reduction you might have saved instead of the full tensor.
If you had saved the full tensor, it will calculate the requested reduction now and cache it.

- `min`, `max`, `mean`, `prod`, `std`, `sum`, `variance` 
- `l1`, `l2` norms

Each of these can be retrieved for the absolute value of the tensor or the original tensor. 
Above was an example to get the mean of the absolute value of the tensor. 
`abs` can be set to `False` if you want to see the `mean` of the actual tensor.

*Note that if you had only saved a particular reduction, you will not be able 
to access the full tensor value or any other reduction during analysis. 
This also applies to the `abs` flag, meaning that if you had saved the
`mean` of `abs` values of the tensor you can not query for the non absolute values mean.
If you do so, Tornasole will return `None`.*

If you had saved the tensor without any reduction, then you can retrieve the actual tensor
as a numpy array and compute any function you might be interested in.

Please note that this can raise exceptions if the step is not available. 
Please see [this section](#when-a-tensor-is-not-available-during-rule-execution) for more details on the different exceptions that can be raised.
 
**Get names of tensors matching regex**

This method takes a regex pattern or a list of regex patterns. 
Each regex pattern is a python style regex pattern string.
```
trail.tensors_matching_regex(['relu_activation*'])
```
**List tensors in a collection**

This returns names of all tensors saved in a given collection.  
`gradients` below is the name of the collection we are interested in.
```
trial.tensors_in_collection('gradients')
```
**List collections**

Below returns all collections belonging to the trial as a dictionary. 
This dictionary is indexed by the name of the collection, and the value is the collection object.
```
trial.collections()
```
**Refresh or do not refresh tensors**

By default Tornasole refreshes tensors each time you try to query the tensor. 
It looks for whether this tensor is saved for new steps and if so fetches them.
If you know the saved data will not change (stopped the machine learning job), or
are not interested in the latest data, you can stop the refreshing of tensors as follows:

`no_refresh` takes a trial or a list of trials, which should not be refreshed.
Anything executed inside the with `no_refresh` block will not be refreshed.
```
from tornasole.analysis.utils import no_refresh
with no_refresh(trials):
    pass
``` 

Similarly if you want to refresh tensors only within a block, you can do:
```
from tornasole.analysis.utils import refresh
with refresh(trials):
    pass
```

#### When a tensor is not available
Tornasole is designed to be aware that tensors required to execute a rule may not be available at every step. 
Hence it raises a few exceptions which allow us to control what happens when a tensor is missing.
These are available in the `tornasole.exceptions` module. You can import them as follows:
```
from tornasole.exceptions import *
``` 
Here are the exceptions and their meanings:

- `TensorUnavailableForStep` : This means that the tensor requested is not available for the step. This might mean that 
this step might not be saved at all by the hook, or that this step might have saved some tensors but the requested
tensor is not part of them. Note that when you see this exception, it means that this tensor can never become available
for this step in the future.

- `TensorUnavailable` : This means that this tensor is not being saved or has not been saved by Tornasole. This means 
that this tensor will never be seen for any step in Tornasole.

- `StepUnavailable`: This means that the step was not saved and Tornasole has no data from the step.

- `StepNotYetAvailable`: This means that the step has not yet been seen by Tornasole. It may be available in the future if the training is still going on. 
Tornasole automatically loads new data as and when it becomes available. 

- `NoMoreData` : This will be raised when the training ends. Once you see this, you will know that there will be no more steps
and no more tensors saved. This is **WIP**. TornasoleRules can not currently do this. 
For now, we expect you to stop the rule execution after the job is finished yourself.

### Rules
Rules are the medium by which Tornasole executes a certain piece of code regularly on different steps of the jobs.
A rule is assigned to a trial and can be invoked at each new step of the trial. 
It can also access other trials for its execution.  
You can evaluate a rule using tensors from the current step or any step before the current step. 
Please ensure your logic respects these semantics, else you will get a `TensorUnavailableForStep` 
exception as the data would not yet be available.
 
#### Writing a rule
Writing a rule involves implementing the [Rule interface](../../tornasole/rules/rule.py).


##### Constructor
Creating a rule involves first inheriting from the base Rule class Tornasole provides.
For this rule here we do not need to look at any other trials, so we set `other_trials` to None.
```
from tornasole.rules import Rule

class VanishingGradientRule(Rule):
    def __init__(self, base_trial, threshold=0.0000001):
        super().__init__(base_trial, other_trials=None)
        self.threshold = threshold
```

Please note that apart from `base_trial` and `other_trials` (if required), we require all 
arguments of the rule constructor to take a string as value. This means if you want to pass
a list of strings, you might want to pass them as a comma separated string. This restriction is
being enforced so as to let you create and invoke rules from json using Sagemaker's APIs.  

##### RequiredTensors

Next you need to implement a method which lets Tornasole know what tensors you 
are interested in for invocation at a given step. This is the `required_tensors` method.
It should return a list of instances of the class `RequiredTensors`, one for each trial
that this rule needs tensors from. 

As mentioned above, for this rule we need tensors from only one trial.
Hence we pass a list with one instance of RequiredTensors. 
Before we look at how to define this method, let us look at the API for RequiredTensors

###### API
[RequiredTensors](../../tornasole/rules/rule.py) is a class which takes one argument to instantiate, namely
- `trial` : The trial whose tensors we need, to execute the rule.

```
req_tensors = RequiredTensors(self.base_trial)
```
To add a given tensor name as required, we do the following:
```
# the steps we need this tensor from to execute the rule at one step, step_num
steps = [step_num]

# need_tensor takes the name of the tensor (as string) and a list of steps (each represented by `int` step number)
req_tensors.need_tensor(tname, steps)

# you can also pass a tensorname regex to need_tensor as follows
# this will match the regex mentioned as tname and fetch all tensors matching this regex
# for each step in steps
req_tensors.need_tensor(tname, steps, should_match_regex=True)
``` 

###### Declare required tensors
Now the required tensors method looks as follows if we need all the gradients for this step `step_num`.
```
    def required_tensors(self, step_num):
        req_tensors = RequiredTensors(self.base_trial)
        for tname in self.base_trial.tensors_in_collection('gradients'):
            steps = [step_num]
            req_tensors.need(tname, steps)
        return [req_tensors]
``` 

This function will be used by the rule execution engine to fetch all the 
required tensors from local disk or S3 before it executes the rule. 
If you try to retrieve the value of a tensor which was not mentioned as part of `required_tensors`,
it might not be fetched from the trial path. 
In such a case you might see one of the exceptions 
`TensorUnavailableForStep` or `TensorUnavailable` or get the tensor object as None.
This is because the rule invoker executes the rule with `no_refresh` mode. Refer discussion above on this for more.
 
##### Function to invoke at a given step
In this function you can implement the core logic of what you want to do with these tensors.
It should return a boolean value `True` or `False`. 
This can be used to define actions that you might want to take based on the output of the rule.
A simplified version of the actual invoke function for `VanishingGradientRule` is below:
```
    def invoke_at_step(self, step):
        for tname in self.base_trial.tensors_in_collection('gradients'):
            t = self.base_trial.tensor(tname)
            abs_mean = t.reduction_value(s, 'mean', abs=True)
            if abs_mean < self.threshold:
                return True
            else:
                return False
```
#### Executing a rule
Now that you have written a rule, here's how you can execute it. We provide a function to invoke rules easily. 
Refer [tornasole/rules/rule_invoker.py](../../tornasole/rules/rule_invoker.py)
The invoke function has the following syntax. 
It takes a instance of a Rule and invokes it for a series of steps one after the other.
```
invoke(rule_obj, start_step=0, end_step=None)
```

An example of invoking the VanishingGradientRule we described above is 
[examples/analysis/scripts/check_grads.py](../../examples/analysis/scripts/check_grads.py).
```
trial_obj = create_trial(trial_dir)
vr = VanishingGradientRule(base_trial=trial_obj, threshold=0.0000001)
invoke(vr)
```

For first party Rules (see below) that we provide, and that you want to run with default arguments, 
you can run them as follows
```
python -m tornasole.rules.rule_invoker --trial-dir ~/ts_outputs/vanishing_gradients --rule-name VanishingGradient
``` 

#### First party rules
We provide a few rules which we built. These are supposed to be general purpose rules that you can use easily. 
We also hope these serve as examples for you to build your own rules.  

##### VanishingGradient
This rule helps you identify if you are running into a situation where your gradients vanish, i.e. have a 
really low or zero magnitude. 

Here's how you import and instantiate this rule. 
Note that it takes two parameters, `base_trial` the trial whose execution will invoke the rule, and a `threshold` which is
used to determine whether the gradient is `vanishing`. Gradients whose mean of absolute values are lower than this threshold
will return True when we invoke this rule.
```
from tornasole.rules.generic import VanishingGradient
r = VanishingGradient(base_trial, threshold=0.0000001)
```
##### ExplodingTensor
This rule helps you identify if you are running into a situation where any tensor has non finite values.
By default this rule treats `nan` and `infinity` as exploding.
If you want to check only for nan, pass the `only_nan` flag as True  
                        
Here's how you import and instantiate this rule. 
Note that it takes two parameters, `base_trial` the trial whose execution will invoke the rule and
`only_nan` which can be set to True if you only want to monitor for `nan` and not for `infinity`.

```
from tornasole.rules.generic import ExplodingTensor
r = ExplodingTensor(base_trial)
```

##### SimilarAcrossRules
This rule helps you compare tensors across runs. Note that this rule takes two trials as inputs. First trial is the `base_trial` whose
execution will invoke the rule, and the `other_trial` is what is used to compare this trial's tensors with. 
The third argument is a regex pattern which can be used to restrict this comparision to certain tensros.
It returns `True` if tensors are different at a given step between the two trials.
```
from tornasole.rules.generic import SimilarAcrossRuns
r = SimilarAcrossRuns(base_trial, other_trial, include=None)
```

##### WeightUpdateRatio
This rule helps you keep track of the ratio of the updates to weights during training. It takes as inputs three arguments.
First, is the `base_trial` as usual. Second and third are `large_threshold` and `small_threshold`. 
This returns True if the ratio of updates to weights is larger than `large_threshold` 
or when this ratio is smaller than `small_threshold`.  

It is a good sign for training when the updates are in a good scale
compared to the gradients. Very large updates can push weights away from optima, and very small updates mean slow convergence.
```
from tornasole.rules.generic import WeightUpdateRatio
wur = WeightUpdateRatio(base_trial, large_threshold, small_threshold)
```

##### AllZero
This rule helps to identify whether the tensors contain all zeros. It takes following arguments

- `base_trial`: The trial whose execution will invoke the rule. The rule will inspect the tensors gathered during this trial.
- `collection_names`: The list of collection names. The rule will inspect the tensors that belong to these collections.
- `tensor_regex`: The list of regex patterns. The rule will inspect the tensors that match the regex patterns specified in this list.

For this rule, users must specify either the `collection_names` or `tensor_regex` parameter. If both the parameters are specified the rule will inspect union on tensors.

```
from tornasole.rules.generic import AllZero
collections = ['weights', 'bias']
tensor_regex = ['input*']
allzero = AllZero(base_trial=trial_obj, collection_names=collections, tensor_regex=tensor_regex)
```

##### UnchangedTensor
This rule helps to identify whether a tensor is not changing across steps. 
This rule runs `numpy.allclose` method to check if the tensor is unchanged. 
It takes following arguments

- `base_trial`: The trial whose execution will invoke the rule. The rule will inspect the tensors gathered during this trial.
- `collection_names`: The list of collection names. The rule will inspect the tensors that belong to these collections.
- `tensor_regex`: The list of regex patterns. The rule will inspect the tensors that match the regex patterns specified in this list.
- `num_steps`: Number of steps across which we check if the tensor has changed. Note that this checks the last num_steps that are available. 
They need not be consecutive.
If num_steps is 2, at step `s` it does not necessarily check for s-1 and s. 
If s-1 is not available, it checks the last available step along with s. 
In that case it checks the last available step with the current step.
- `rtol`: The relative tolerance parameter, as a float, to be passed to numpy.allclose. 
- `atol`: The absolute tolerance parameter, as a float, to be passed to numpy.allclose
- `equal_nan`: Whether to compare NaN’s as equal. If True, NaN’s in a will be considered 
equal to NaN’s in b in the output array. This will be passed to numpy.allclose method
    
For this rule, users must specify either the `collection_names` or `tensor_regex` parameter. 
If both the parameters are specified the rule will inspect union on tensors.

```
from tornasole.rules.generic import UnchangedTensor
ut = UnchangedTensor(base_trial=trial_obj, tensor_regex=['.*'], num_steps=3)
```

## Examples
We have a few example scripts and notebooks to help you get started. Please go to the `examples` folder. 

We also have an end-to-end flow example from saving tensors to plotting using saved tensors for MXNet at 
`examples/mxnet/notebook/mnist`. 

## ContactUs
We would like to hear from you. If you have any question or feedback, 
please reach out to us tornasole-users@amazon.com

## License
This library is licensed under the Apache 2.0 License.