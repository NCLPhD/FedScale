

# Building your own FL algorithm with FedScale

This tutorial will show you how to implement your own FL algorithm with FedScale
through giving some common examples including optimization algorithms and client selection algorithms.

Please check the [setup](./tutorial.md) guide and [dataset](./dataset/Femnist_stats.ipynb) tutorial to prepare your FL experiments.
Here we focus on the API you need to implement the FL algorithms.
More existing FL algorithms can be found in `./examples`.

## Algorithm API
As mentioned in [tutorial](https://www.tensorflow.org/federated/tutorials/building_your_own_federated_learning_algorithm),
federated algorithms have 4 main components in most cases:

- A server-to-client broadcast step.
- A local client update step.
- A client-to-server upload step.
- A server update step.


To modify each of these steps for your FL algorithm, you can customize your server and client respectively.
Here we give several examples to explain the necessary components in FedScale.


## Aggregation Algorithm

FedScale uses Federated averaging as the default aggregation algorithm.
[FedAvg](https://arxiv.org/pdf/1602.05629.pdf) is a communication efficient algorithm.
In FedAvg, clients keep their data locally for privacy protection; a central parameter server is used to communicate between clients.
Each participant locally performs E epochs of stochastic gradient descent (SGD) during every round.
The participants then communicate their model updates to a central server, where they are averaged.

The aggregation algorithm in FedScale is mainly reflected in two code segment.

1. **Client updates**: Function `training_handler` in `fedscale/core/executor.py` will be called to initiate client training.
The following code segment from `fedscale/core/client.py` shows how the client trains the model and updates the gradient (FedProx).


```
class Client(object):
   """Basic client component in Federated Learning"""
   def __init__(self, conf):
       self.optimizer = ClientOptimizer()
       ...
      
   def train(self, client_data, model, conf):
       # Prepare for training
       ...
       # Conduct local training
       while completed_steps < conf.local_steps:
           try:
               for data_pair in client_data:
                   # Forward Pass
                   ...
                  
                   optimizer.zero_grad()
                   loss.backward()
                   optimizer.step()
                   self.optimizer.update_client_weight(conf, model, global_model if global_model is not None else None  )

                   completed_steps += 1
                   if completed_steps == conf.local_steps:
                       break
                
       # Collect training results
       return results
```
```
class ClientOptimizer(object):
   def __init__(self, sample_seed):
       pass
  
   def update_client_weight(self, conf, model, global_model = None):
       if conf.gradient_policy == 'fed-prox':
           for idx, param in enumerate(model.parameters()):
               param.data += conf.learning_rate * conf.proxy_mu * (param.data - global_model[idx])

```


2. **Server aggregates**: Function `round_weight_handler` in `fedscale/core/aggregator.py` will be called to do the aggregation at the end of each round.
In the function `round_weight_handler`, you can customize your aggregator optimizer in `fedscale/core/optimizer.py`.
The following code segment shows how FedYogi and FedAvg aggregate the participant gradients.


```
class ServerOptimizer(object):

   def __init__(self, mode, args, device, sample_seed=233):
       self.mode = mode
       if mode == 'fed-yogi':
           from utils.yogi import YoGi
           self.gradient_controller = YoGi(eta=args.yogi_eta, tau=args.yogi_tau, beta=args.yogi_beta, beta2=args.yogi_beta2)
      
       ...
      
   def update_round_gradient(self, last_model, current_model, target_model):
      
       if self.mode == 'fed-yogi':
           """
           "Adaptive Federated Optimizations",
           Sashank J. Reddi, Zachary Charles, Manzil Zaheer, Zachary Garrett, Keith Rush, Jakub Konecný, Sanjiv Kumar, H. Brendan McMahan,
           ICLR 2021.
           """
           last_model = [x.to(device=self.device) for x in last_model]
           current_model = [x.to(device=self.device) for x in current_model]

           diff_weight = self.gradient_controller.update([pb-pa for pa, pb in zip(last_model, current_model)])

           for idx, param in enumerate(target_model.parameters()):
               param.data = last_model[idx] + diff_weight[idx]

       elif self.mode == 'fed-avg':
           # The default optimizer, FedAvg, has been applied in aggregator.py on the fly
           pass
```

## Client Selection

FedScale uses random selection among all available clients by default.
However, you can customize the client selector by modifying the `client_manager` in `fedscale/core/aggregator.py`,
which is defined in `fedscale/core/client_manager.py`.


Upon every device checking in or reporting results, FedScale aggregator calls `client_manager.registerClient(...)` or `client_manager.registerScore(...)`
to record the necessary client information that could help you with the selection decision.
At the beginning of the round, FedScale aggregator calls `client_manager.resampleClients(...)` to select the training participants.



For example, [Oort](https://www.usenix.org/conference/osdi21/presentation/lai) is a client selector
that considers both statistical and system utility to improve the model time-to-accuracy performance.
You can find more details of Oort implementation in `thirdparty/oort/oort.py` and `fedscale/core/client_manager.py`.





## Other examples

You can find more FL algorithm examples in `examples/`, which includes differential privacy, malicious scenario, [HeteroFL](https://openreview.net/forum?id=TNkPBBYFkXg) and so on.
By customizing the `fedscale/core/aggregator.py`, `fedscale/core/executor.py` and `fedscale/core/client.py`, you have more freedom to explore your own FL algorithms!






