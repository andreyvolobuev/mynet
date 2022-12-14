# Mynet
![Mynet fancy logo](images/fancy-image.jpg "Mynet logo")

Education purposes library that will help you understand how neural networks work under the hood.

It is and always will be 100% dependency free, pure Python from scratch. All major components of neural networks (e.g. auto differentiation, loss functions, activation functions, optimizers) are implemented 😌 If you wish to add a new component – PR's are welcome 🙏🏻 If you found a bug in the code or consider Mynet to work incorrectly - please open an issue 👍

Absolutely not suitable for production as it's slow as hell and definitely won't be able to run a model playing StarCraft realtime. Instead it can run a relatively simple model and show you what is going on under the hood of ANY major deep learning framework as all of them are basically doing the same thing. That is the point.


In the simplest usecase scenario what you do is import mynet, declare a model and stuff it with some layers along with an opimizer, instantiate it and you are ready to go

```
import mynet


class Model(mynet.Model):
    def __init__(self):
        self.l1 = mynet.Layer(1, 2, activation=mynet.relu)
        self.l2 = mynet.Layer(2, 1)
        self.optim = mynet.GradientDecent(self.parameters())


if __name__ == '__main__':
    model = Model()
    
    dataset = get_dataset()  # get your dataset somehow
    
    for n in range(N_EPOCH):
        X, y = get_random_batch(dataset)  # get a batch of samples from your dataset
        model.optim.zero_grad()
        out = model.forward(X)
        loss = mynet.mse_loss(y, out)
        loss.backward()
        model.optim.step()
```

However this same behaviour can be achieved with [pytorch](https://github.com/pytorch/pytorch) and it's going to be faster and much more reliable. What you probably really want is to understand what is going on under the hood of this thing. I'm going to break it down.


## Instantiating a model

When instantiating a model you add mynet.Layer instances to it. Let's take a look at what's inside of [mynet/layer.py](https://github.com/andreyvolobuev/mynet/blob/master/mynet/layer.py)  

```
class Layer:
    def __init__(self, n_inputs=None, n_outputs=None, neurons=None, activation=None):
        self.neurons = neurons or [Neuron(n_inputs) for n in range(n_outputs)]
        self.activation = activation
```

So a layer is just a list of Neuron instances and an activation function. We'll get back to the activation function later, but for now let's look inside [mynet/neuron.py](https://github.com/andreyvolobuev/mynet/blob/master/mynet/neuron.py) to see what is a Neuron  

```
class Neuron:
    def __init__(self, n_inputs=None, weights=None, bias=None):
        self.weights = weights or [Value(random.uniform(-1, 1)) for i in range(n_inputs)]
        self.bias = bias or Value(0)
```
Every Neuron is represented by weights and a bias. The weights is a vector (or list) of Values initialized with a random sample from a uniform distribution and the bias is a scalar Value initialized with just 0. So if we go up for a bit and look at the Layer once again, we'll see that the number of inputs to the Layer corresponds to the number of inputs to every Neuron in that Layer which in turn corresponds to the the number of weights that each Neuron has.   
Also the number of outputs of a Layer corresponds to the number of Neurons in that Layer.

So in our simple example from the top a Model that has two Layers gets created:
1. the first Layer that has two Neurons, each Neuron with only one weight initialized randomly and one bias initialized with 0 (as every single Neuron ALWAYS has one single bias)
2. The second Layer has only one Neuron but with two randomly initialized weights and 0-initialized bias. 


## Forward pass

Now talk about Model. In [mynet/model.py](https://github.com/andreyvolobuev/mynet/blob/master/mynet/model.py) we see that Model is just a base class that needs to be inherited from by user's own custom models. Every model needs to implement a forward method. It can be redefined but we'll just have a look at the default implementation:
```
def forward(self, X):
    for layer in self._layers():
        X = layer.forward(X)
    return X
```
It takes user's input X (obtained from a dataset or whereever else) and then sequentially calls forward method of every Layer instance inside the model. Please note that output of the first Layer is then treated as an input for the second Layer and so on.

Inside Layer's forward method the inputs are put through the forward method of every Neuron of the Layer and the result then gets `activated` with the Layer's activation function (we'll get back to it later, I promise). See [mynet/layer.py](https://github.com/andreyvolobuev/mynet/blob/master/mynet/layer.py)  
```
def forward(self, X):
    return [self._activate([n.forward(x) for n in self.neurons]) for x in X]

def _activate(self, n):
    return self.activation(n) if self.activation else n
```

Finally inside [mynet/neuron.py](https://github.com/andreyvolobuev/mynet/blob/master/mynet/neuron.py) the Neuron's forward method does the actual math that we need for out Neural Net to make predictions:

```
def forward(self, X):
    ...
    result = self.bias
    for w, x in zip(self.weights, X):
        result += w * x
    return result
```
It just multiplies each weight of the Neuron with each X value that user provided, sums the products together and adds the bias. That's it. This value is returned back to the Layer's forward method and then passed to the Layer's activation function. 

What is an activation function? Well, it's just a functions that makes our output to be non-linear. The math that a Neuron does with the data is `x*w + b` and if you recall from your high-school that's a formula for a straight line. However our real-life scenario functions (like the one that given a board state returns next best move) are rarely linear. So in order for neural net to be able to fit complicated squiggles we need to make Neuron outputs to be non-linear too. And that is what we need an activation function for.

One of the most powerful hence popular activation functions is ReLU (**Re**ctified **L**inear **U**nit). What it does is just output whatever maximum of two values: zero and Neurons actual output. See the implementation in [mynet/activation.py](https://github.com/andreyvolobuev/mynet/blob/master/mynet/activation.py)
```
def relu(X):
    return [maximum(x, 0) for x in X]
```

After we `activate` our Neuron with ReLU it's graph is no longer a straight line:  

![ReLU](images/relu.png "ReLU")

The `activated` result is then returned to the Model's forward method and is either passed through another Layer's forward method or returned to the user as Model's output result.  


## Calculating the loss

Let's make up some imaginary numbers and pretend we just did a forward pass. As mentioned earlier, out simple Model has two layers. First Layer has two Neurons with one weight and one bias each. Let's call them **w1**, **w2**, **b1** and **b2**. Second Layer has one Neuron with two weights and one bias. They are called **w3**, **w4** and **b3**.
```
w1 = Value(-0.11)
w2 = Value(0.21)
w3 = Value(0.89)
w4 = Value(-0.75)
b1 = Value(0)
b2 = Value(0)
b3 = Value(0)
```
*Don't forget that this are randomly initialized values, except for biases that are usually initialized at 0*

Out imaginary input (usualy denoted with **X**) will be 2.0. It means nothing but we can pretend that there's a point behind. What if we say that the X means the number of candies that we gave to our daughter and we expect our Model to tell us how many candies will she share with her brother (why not?)

When we do the forward pass through the first Layer it's going to output something like this:

First Neuron of the first Layer will output:
> X * w1 + b1 == 2.0 * -0.11 + 0 == -0.22  
> Let's call it **x1**

Second Neuron of the first Layer will output
> X * w2 + b2 == 2.0 * 0.21 + 0 == 0.42  
> Let's call it **x2**

Now we need to `activate` **x1** and **x2** with relu.
> max(0, **x1**) == max(0, -0.22) == 0  
> Let's call it **a1** (like **activated1**)  
>  
> max(0, **x2**) == max(0, 0.42) == 0.42  
> Let's call it **a2** (like **activated2**)

The second Layer comes into play. The input for this Layer will be not the original value **X** (2.0), but the activated output from the first Layer **a1** and **a2** (0 and 0.42). Recall that it has only one Neuron but with two weights so it will multiply each weight with the corresponding input, sum the products together and add the bias.

> a1 * w3 + a2 * w4 + b3 == 0 * 0.89 + 0.42 * -0.75 + 0 == 0 + -0.315 + 0 == -0.315  

This is our Model's final **output** (or prediction). But is it good or is it bad? If it is bad then how bad is it? Is there a way to tell how wrong our Model actually is?

Well, there is a way. We did actually gave our daughter two candies and she shared one with her brother so the output of our Model should have been 1 instead of -0.315. We can calculate squared difference of the actually observed value (usually denoted with **y**) and the predicted value (-0.315). It's called `Sum of the squared error` and it's implemented inside [mynet/loss.py](https://github.com/andreyvolobuev/mynet/blob/master/mynet/loss.py).

```
def sse_loss(targets, predictions):
    sum_sq_err = 0
    for target, pred in zip(targets, predictions):
        for t, p in zip(target, pred):
            sum_sq_err += (t - p) ** 2
    return sum_sq_err
```

So in out case `the squared error` or `the loss` will be as follows:
> (**y** - **output**)**2 == (1 - -0.315)**2 == **1.73**  



## Understanding the loss

Here starts the most interesting part. When we know the loss (**1.73**), how can we use that value in order to make our Model to give better predictions? If you recall from a high-school class again, the graph of a square function (and our `Sum of the squared error` is indeed a square function) is a porabola. Look at the picture:  
![Derivative on porabola](images/derivative.jpg "Derivative on porabola")


The parabola has it's minima - it's the point of smallest value of the function. And as it's a graph of a loss function, we need to find it's minima to minimize the loss (that's what we want, don't we?). Hope you're following. 

In order to find the porabola's minima, we need to take the functions derivative in the current point with respect to the models parameters (weights and biases of all of the Models layers: w1, w2, w3, w4, b1, b2 and b3). The current point (value of the loss function) is **1.73** . 

So say it simple, if we find the derivative of the `Sum of the squared error` function in the current point (current value of the loss function) with respect to the Model's parameters it will give us a hint of how big or how small we have to modify the parameters and in which direction (plus or minus) in order for those parameters to produce the desired output which will result in the minimal value of the loss function (ideally 0).


How to take derivative of the loss function with respect to the models parameters? 

Our loss function is: `(y - PREDICTED_VALUE)**2`
There are no Model's parameters in this equasion yet. Where are they? They are actually *inside* of the PREDICTED_VALUE, so let's break it down a bit:

> (y - (a1 * w3 + a2 * w4 + b3))**2

Great! Here we get **w3**, **w4** and **b3**. But these are not all of the parameters. Let's get the rest. If you recall, **a1** is an activated output from the first Neuron of the first Layer, and **a2** is an activated output from the second Neuron of the first Layer, so let's substitute those values in the equasion:

> (y - (max(0, **x1**) * w3 + max(0, **x2**) * w4 + b3))**2

**x1** and **x2** are the outputs from the first Neuron of the first Layer and from the second Neuron of the first Layer respectively. Let's substitute them with the equasion inside of the Neurons:

> (y - (max(0, X * w1 + b1) * w3 + max(0, X * w2 + b2) * w4 + b3))**2

There it is. We now have all of our Model parameters inside of an equasion. We just have to take the derivate of this one with respect to the parameters and we'll be able to modify them in order to minimize the function, which in this case happens to be a loss function, which is exactly what we want to minimize (I hope I have already stressed this enough times).

If this equasion AND ESPECIALLY IT'S DERIVATIVE freaks you out - you're not alone. Luckily here's when our old friend calculus comes into play. It tells us that according to `the chain rule` we don't need to take a derivative of this monster. What we can do instead is to break the equasion down into small pieces and take derivatives of them and them multiply those derivatives together.

![Math operations tree](images/loss_decomposed.png "Math operations tree")


This thing that looks like a tree is actually a DAG (directed acyclic graph). By just looking at it we can see how our loss got created as a result of small math operations (addition, subtraction, division, power, maximum). We easily can start from the top of the DAG (which is our loss) and slowly take the derivative of each math operation that lead to it. The chain rule says that we have to multiply each parent node derivative with it's child node's derivative. 

Let's actually do that:

1. First we have an operation of taking `the error to the power of 2`. It is achieved by moving the power to the front of the equation and subtracting the power by 1.
> f'(squared_loss) = 2 * (y - PREDICTED_VALUE) == 2 * (1 - -0.315) = 2 * 1.315 = 2.63  


2. Next there's subtraction of the **predicted value** (-0.315) from the **observed value** (1). As the observed value has no term for the predicted value we treat it as a constant and the derivative of a constant is always 0. The predicted value is treated as a single variable and it's derivative is 1. Lastly we multiply the derivative (-1) by previous operation's derivative and get -2.63
> f'(loss) = CHILD_DY * (0 - PREDICTED_VALUE) = 2.63 * (0 - 1) = -2.63  


3. Derivative of addition is almost identical to the derivative of subtraction. We treat terms addends that have no term for b3 as constants and treat b3 itself as 1. Lastly we multiply by previous derivative:
> f'(b3) = CHILD_DY * (a1*w3 + a2*w4 + b3) = -2.63 * (0 + 0 + 1) = -2.63  
> NOTE: At this point we already know the derivative of the first parameter of our model: **b3**, and it's value is **-2.63**  

4. Addition again: product of a1 and w3 is added to product of a2 and w4. If we solve for the derivative with respect to a1*w3 then we treat it as 1 and a2*w4 is treated as 0 and vise versa. The result is again multiplied by the previous derivative:
> f'(a1*w3) = CHILD_DY * (a1*w3 + a2*w4) = -2.63 * (1 + 0) = -2.63  
> f'(a2*w4) = CHILD_DY * (a1*w3 + a2*w4) = -2.63 * (0 + 1) = -2.63  


5. When solving for derivative of multiplication of a1 and w2 with respect to a1 we just treat a1 as 1 and the other variable is unchanged. *Do I have to again write down that we multiply the result by the previous derivative?*
> f'(a1) = CHILD_DY * 1 * w3 = -2.63 * 0.89 = -2.3407  


6. Multiplication again, just the derivative with respect to w3 this time. We treat w3 as 1 and the other variable is unchanged. Multiply the result derivative by the previous derivative again...
> f'(w3) = CHILD_DY * 1 * a1 = -2.63 * 0 = 0  
> Hurray! We've got the derivative of the second parameter of out Model: **w3**'s deirivative is **0**  


7. Derivative of ReLU activation function is either 0 or 1 (multiplied by the previous derivative)
> f'(x1) = CHILD_DY * max(0, x1) = -2.3407 * 0 = 0  


8. We've done addition before: just treat our target variable as 1 and the other one as 0 (don't forget to multiply by you know what...)
> f'(b1) = CHILD_DY * (0 + 1) = CHILD_DY * 1 = 0 * 1 = 0  
> One more parameter's derivative has just been found: **b1**'s derivative is **0**  


9. Multiplication again. We're solving for derivative with respect to w1, so w1 becomes 1 and the other variable remains unchanged. The result is multiplied by child's derivative:
> f'(w1) = CHILD_DY * (X * w1) = 0 * X = 0 * 2 = 0  
> We have just found derivative of **w1** which is **0**. 
> Also note that we've got to the bottom of one branch of the DAG. So now we go up to the step #4 and start solving another branch.


10. Here starts another branch of derivative solving. Please consider previous step to be #4. So we do the same thing as in the step #5 but with respect to a2 instead of a1.
> f'(a2) = CHILD_DY * 1 * w4 = -2.63 * -0.75 = 1.9725  


11. Same as step #6 but with respect to w4  
> f'(w4) = CHILD_DY * 1 * a2 = -2.63 * 0.42 = -1.1046  
> Look at you! We've solved one more: derivative of **w4** is **-1.1046**  


12. Derivative of ReLU activation function is either 0 or 1 (multiplied by the previous derivative)  
> f'(x2) = CHILD_DY * max(0, x2) = 1.9725 * 1 = 1.9725  


13. Same as step #8 but with respect to b2  
> f'(b2) = CHILD_DY * (0 + 1) = 1.9725 * 1 = 1.9725  
> Note: Derivative of **b2** is **1.9725**  


14. Same as step #9 but with respect to w2
> f'(w2) = CHILD_DY * X * w2 = CHILD_DY * X = 1.9725 * 2 = 3.945  
> Note: Derivative of **w2** is **3.945**  


So for now we have a derivative for each value in our Model's parameters:
```
derivative of w1 is 0
derivative of w2 is 3.945
derivative of w3 is 0
derivative of w4 is -1.1046
derivative of b1 is 0
derivative of b2 is 1.9725
derivative of b3 is -2.63
```

Let's test our calculations with [pytorch](https://github.com/pytorch/pytorch):
![Torch same results](images/torch_compare_results.png "Torch shows same results")



## Automatic differentiation 

That was a lot of math though relatively simple. Is there a way to automate that? Indeed there is. If only we had a data structure that would "remember" it's parents and the math operation that produced it, then we could use exactly the same math from the above to calculate the *gradient* (it's just a fancy name for derivative, but all of the deep learning frameworks use it so we'll do too).

Such a data structure is called simply a Value and is declared inside of [mynet/value.py](https://github.com/andreyvolobuev/mynet/blob/master/mynet/value.py). Let's have a look at it:

```
class Value:
    def __init__(self, data=None, parents=None, grad_fn=None):
        self.data = data
        self.parents = parents
        self.grad_fn = grad_fn
        self.grad = None
```

Every instance of Value should maintain:
1. data – the actual value of a Value (unavoidable pun)
2. list of parent Values that produced it
3. function that given the previous Value's derivative solves for the derivative of the current Value
4. derivative (or gradient) of the Value represented as another Value class instance


When a math operation is performed on a Value it will return a new Value with it's parents set the values that produces it along with the function that will be called durring the process of calculating the derivatives.

Let's look how is it implemented on example of multiplication:

```
def __mul__(self, other):
    def MulBack(grad):
        self.grad += grad * other
        other.grad += self * grad
    return Value(data=self.data*other.data, parents=[self, other], grad_fn=MulBack)
```

Let's create an example to understand it better:

```
from mynet import Value

x = Value(5)
y = x * 2

print(y)
Value(10)

print(y.parents)
[5, 2]

print(y.grad_fn)
<function Value.__mul__.<locals>.MulBack at 0x7f9376a63c10>

print(y.grad is None)
True
```

We created a Value(5) and multiplied it by 2. The result was another Value(10). It's parents are 5 and 2. It's grad function is MulBack. It's gradient (or derivative) is None because we have to actually call the MulBack to calculate the derivative and we haven't done that just yet. Why not? Because we have to calculate the derivative going from the top of our DAG (recall my ungly sketch from the top that looks like a tree). And we have to call those functions in reverse order with the last operation being the first who's grad_fn will be called. 

So when we already know that we are at the top and there will be no more operations we can traverse the tree of parents of the topmost Value and call the grad_fn of each one of them respectively. The MulBack grad_fn will accept the previous operation's derivative (recall when we did the calculations by hand I wrote like dozen times that we have to mutiply current derivative with the previous derivative), will calculate the derivative of the current Value and will multiply that by the previous derivative. 

Let's look at the upper example again. When solving for derivative with respect to a value we treat the respected value as 1 and the other value remains unchanged. 
```
f(x) = 5 * 2
f'(x) = 1 * 2 = 2
```

In order to traverse the tree of Value parents we'll have to call backward on that Value. Let's have a look at the implementation:

```
def backward(self):
    self.grad = Value(1)
    for value in reversed(self.deepwalk()):
        value.grad_fn(value.grad)
```

Everything is pretty straight-forward. We set the derivative of the initial Value to 1 (as if we set it to 0 it will get multiplied by the parent's gradient and that will become 0 too, so it needs to be 1) and then respectively call grad_fn on every parent of the Values in reversed order. That's it. How do we get the parent's? There are many ways to implement that algorithm. Here's just one of them:

```
def deepwalk(self):
    path, seen = [], set()
    def _deepwalk(value):
        value.grad = value.grad or Value(0)
        if value.grad_fn and value not in seen:
            seen.add(value)
            for parent in value.parents:
                _deepwalk(parent)
            path.append(value)
    _deepwalk(self)
    return path
```


And the last but not least: all of the math operations that you can imagine can be represented as addition, multiplication and power so we only have to create 3 grad functions and the rest comes for free...

```
def __mul__(self, other):
    def MulBack(grad):
        self.grad += grad * other
        other.grad += self * grad
    return Value(data=self.data*other.data, parents=[self, other], grad_fn=MulBack)

def __add__(self, other):
    def AddBack(grad):
        self.grad += grad
        other.grad += grad
    return Value(data=self.data+other.data, parents=[self, other], grad_fn=AddBack)

def __pow__(self, other):
    def PowBack(grad):
        self.grad += grad * other * self ** (other - 1)
        other.grad += grad * self ** other
    return Value(data=self.data**other.data, parents=[self, other], grad_fn=PowBack)
```

This is not entirely true because for example to subtract a value we have to multiply the subtrahend by -1 first and only then do summation with minuend. So it's two operations (multiplication and addition) instead of one. But for simplicity sake we'll leave it like this.


So can we actually do the same math as we did by hand (and then repeated in pytorch) with Mynet? Yes, indeed we can and we'll get exactly the same result:
![Mynet backward](images/mynet_backward.png "Mynet backward")


## Optimizing the parameters
So now, when we've calculated derivative for every parameter of our neural net, how can we use those derivatives to improve the performance of our model?

There's an algorithm called Gradient Decent. Here it some pseudo-code for it:
```
model = Model(parameters=random)
dataset = get_dataset()
X, y = test_train_split(dataset)

for n in range(N_EPOCH):
    out = model.forward(X)
    loss = calculate_loss(y, out)
    loss.backward()

    for parameter in model.parameters:
        parameter = parameter - learning_rate * parameter.derivative
        parameter.derivative = 0
```

As you can see we've already done everything up to the last 3 lines. We just need a way to subtract paraters derivative * learning_rate from each corresponding parameter. We'll get to the learning rate in a bit. Now let's focus on how to get all of the model parameters at one place. We start with [mynet/model.py](https://github.com/andreyvolobuev/mynet/blob/master/mynet/model.py). Model class actually has a method called parameters:  

```
def parameters(self):
    return [p for l in self._layers() for p in l.parameters()]
```

All it does is just returns a list of whatever is returned by parameters() method of each Layer of the model. So let's look inside of[mynet/layer.py](https://github.com/andreyvolobuev/mynet/blob/master/mynet/layer.py) to see what it is:  

```
def parameters(self):
    return [p for n in self.neurons for p in n.parameters()]
```

Unsurprisingly the Layer just does a proxy-call of parameter method of each neuron inside of it. Let's then look inside of [mynet/neuron.py](https://github.com/andreyvolobuev/mynet/blob/master/mynet/neuron.py) for the implementation:

```
def parameters(self):
    return self.weights + [self.bias]
```

A-ha!  
So when we call the parameters method of a Model it returns a list of weights and a bias of each Neuron inside of every Layer of that Model. The weights and biases are instances of Value so when we do the forward pass and then calculating the loss, the loss Value will keep track of all of its parent Values (including the model's parameters). So when we call the backward method on the loss - every parameter of the model will have it's gradient (or derivative) set automatically. We are just left to optimize the Values and for this theres another optimizer class defined in [mynet/optim.py](https://github.com/andreyvolobuev/mynet/blob/master/mynet/optim.py)

```
class GradientDecent(Optim):
    def __init__(self, parameters, lr=1):
        self.parameters = parameters
        self.lr = lr

    def step(self):
        for parameter in self.parameters:
            parameter.data -= self.lr * parameter.grad.data 
```

So in the initializer GradientDecent optimizer shall take a list of parameters to optimize and a learning_rate. The learning rate is called a hyperparameter. It's because it's the job of neural net operator to set it. Optimizer itself won't optimize it. The idea behing learning rate is that it sets the step size for the optimizer. If the step size is too large then the optimizer will quickly jump over the minima of the loss function (and as you recall the minima of the loss functions is where we want to eventually come, not jump it over). In the another case if the step size is too small then the optimizer will take much longer time to eventually get to the loss minima or it even might stuck on what is called a `local minima` (which is not the actual minima of the functon). So we need to set it carefully and if the things go not how we expect - then we just do it all over again.

In the step method of the optimizer we just iterate over the parameters and subtract derivative of the parameter multiplied by the learning rate from the corresponding parameter data. As you may have noticed the GradientDecent is a subclass of Optim class. The Optim has only one method: zero_grad which iterates over the optim parameters and set's their derivatives to None. We need to do that because the gradients will accumulate with each forward pass and if we skip zeroing the grad we'll end up with incorrect steps that the optimizer shall take.

```
class Optim:
    def zero_grad(self):
        for parameter in self.parameters:
            parameter.grad = None
```

Now let's try to transform our pseudo-code into real code and see how fast the Gradient Decent agorithm will converge (if ever?) to optimal values that will output 1 to the input of 2 (which means our daughter will share 1 candy with her brother IF given two candies herself)

![Convergence](images/convergence.png "Convergence")

It converges after 21 iteration and will always output 1 in case it's input is 2 and this is exactly what we wanted to achive. *Please note that I had to uglyfy the code a bit just because I wanted it to fit on one screen* 😂

That is actually all there is to machine learning, deep learning and neural nets. They are just fancy names for bunch of static value multiplicatons and taking of derivatives. There's hardly any "soul" inside of those equasions and formulas. There's no need to be afraid that these numbers will eventually conquer the world and enslave the mankind. And it kind of feels good.

Questions, Issues, Comments, etc. are welcome!

avvolob@gmail.com