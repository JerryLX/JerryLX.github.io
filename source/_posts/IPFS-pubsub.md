---
title: IPFS-pubsub
date: 2021-01-31 14:56:31
categories:
- ipfs
---


## 节点打分
```
Score(p) = TopicCap(Σtᵢ*(w₁(tᵢ)*P₁(tᵢ) + w₂(tᵢ)*P₂(tᵢ) + w₃(tᵢ)*P₃(tᵢ) + w₃b(tᵢ)*P₃b(tᵢ) + w₄(tᵢ)*P₄(tᵢ))) + w₅*P₅ + w₆*P₆ + w₇*P₇
```
其中，`tᵢ`是topic的权重。

参数定义如下:
- `P₁`: **Time in Mesh** for a topic. **在网格中的存在时间。** This is the time a peer has been in the mesh, capped to a small value and mixed with a small positive weight. This is intended to boost peers already in the mesh so that they are not prematurely pruned because of oversubscription.
- `P₂`: **First Message Deliveries** for a topic. **首发消息传递次数。** This is the number of messages first delivered by the peer in the topic, mixed with a positive weight. This is intended to reward peers who first forward a valid message.
- `P₃`: **Mesh Message Delivery Rate** for a topic. **对于不发送数据节点的惩罚因子。** This parameter is a threshold for the expected message delivery rate within the mesh in the topic. If the number of deliveries is above the threshold, then the value is 0. If the number is below the threshold, then the value of the parameter is the square of the deficit. This is intended to penalize peers in the mesh who are not delivering the expected number of messages so that they can be removed from the mesh. The parameter is mixed with a negative weight.
- `P₃b`: **Mesh Message Delivery Failures** for a topic. **发送失败次数**。This is a sticky parameter that counts the number of mesh message delivery failures. Whenever a peer is pruned with a negative score, the parameter is augmented by the rate deficit at the time of prune. This is intended to keep history of prunes so that a peer that was pruned because of underdelivery cannot quickly get re-grafted into the mesh. The parameter is mixed with negative weight.
- `P₄`: **Invalid Messages** for a topic. **无效消息数量。** This is the number of invalid messages delivered in the topic.
  This is intended to penalize peers who transmit invalid messages, according to application-specific
  validation rules. It is mixed with a negative weight.
- `P₅`: **Application-Specific** score. This is the score component assigned to the peer by the application
  itself, using application-specific rules. The weight is positive, but the parameter itself has an
  arbitrary real value, so that the application can signal misbehaviour with a negative score or gate
  peers before an application-specific handshake is completed.
- `P₆`: **IP Colocation Factor**. 防止女巫攻击的相同ip因子。This parameter is a threshold for the number of peers using the same IP address. If the number of peers in the same IP exceeds the threshold, then the value is the square of the surplus, otherwise it is 0. This is intended to make it difficult to carry out sybil attacks by using a small number of IPs. The parameter is mixed with a negative weight.
- `P₇`: **Behavioural Penalty**. **特殊事件的惩罚因子。** This parameter captures penalties applied for misbehaviour. The parameter has an associated (decaying) counter, which is explicitly incremented by the router on specific events. The value of the parameter is the square of the counter and is mixed with a negative weight.


## 分数阈值

- `0`: 最基本的阈值，如果分数低于0，那么对等方将会被移出网格。需要注意的是，即便移出了网格它的分数记录也会在一段时间内被保留，这样也可以防止它短时间内再次加入。
- `gossipThreshold`: 如果低于该阈值，那么不会向该对等方发送闲话，同时收到来自该对等方的闲话也应直接丢弃而不会处理。该值为负数，因为在网格外的对等方大多都分数不高，如果大于0 则能发送闲话的对等方将非常少，不利于维护更新。
- `publishThreshold`: 如果低于该阈值，那么就不能收到来自主题的消息，该值且应低于闲话阈值。
- `graylistThreshold`: 如果低于该阈值，那么将被拉黑，来自它的控制消息也将被忽略（也就是说加入请求也会被忽略），该值应低于发布阈值。
- `acceptPXThreshold`: 如果低于该阈值，那么就不会接受对等交换该阈值为正。
- `opportunisticGraftThreshold`: 网格中所有对等方的分数中位数。如果中位数低于阈值，那么说明后面有很多表现不那么好的对等方，这时我们就会对网格进行修整，引入一些分数高于此阈值的新对等点，并对分数低的对等点进行剪切，此阈值也为正。