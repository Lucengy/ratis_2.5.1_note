## 1. PeerConfiguration

持有两个相同类型的map，一个是表示Leader/Candidate/Follower，另外一个是用来表示Follower

* Map<RaftPeerId, RaftPeer> peers;
* Map<RaftPeerId, RaftPeer> listeners

稍微需要在意的点是peers和listeners是没有overlap的

这里有几个稍稍晦涩的方法

1. getPeerMap(RaftPeerRole)

   ```java
     private Map<RaftPeerId, RaftPeer> getPeerMap(RaftPeerRole r) {
       if (r == RaftPeerRole.FOLLOWER) {
         return peers;
       } else if (r == RaftPeerRole.LISTENER) {
         return listeners;
       } else {
         throw new IllegalArgumentException("Unexpected RaftPeerRole " + r);
       }
     }
   ```

   目前来看，peers中的RaftPeer的Role类型只能是Follower，

2. getPeer(RaftPeerId, RaftPeerRole...)

   ```java
     RaftPeer getPeer(RaftPeerId id, RaftPeerRole... roles) {
       if (roles == null || roles.length == 0) {
         return peers.get(id);
       }
       for(RaftPeerRole r : roles) {
         final RaftPeer peer = getPeerMap(r).get(id);
         if (peer != null) {
           return peer;
         }
       }
       return null;
     }
   ```

   如果单看这个方法，是有点晦涩的，形参怎么是变长参数RaftPeerRole，稍稍琢磨一下，caller在调用这个方法时，并不知道RaftPeer此时的Role类型，那么可以传入Follower和Listener来test对应的RaftPeer是否在Conf中；也可以在已知RaftPeer的Role类型时，传入对应的Follower或Listener来test对应的RaftPeer是否在Conf中

3. contains(RaftPeerId) && contains(RaftPeerId, RaftPeerRole)

   ```java
     boolean contains(RaftPeerId id) {
       return contains(id, RaftPeerRole.FOLLOWER);
     }
   
     boolean contains(RaftPeerId id, RaftPeerRole r) {
       return getPeerMap(r).containsKey(id);
     }
   ```

   这里需要注意的是contains(RaftPeerId)方法test的是形参中的RaftPeerId是否在RaftPeerRole==Followers的peers中

4. size()

   ```java
     int size() {
       return peers.size();
     }
   ```

   返回的也是RaftPeerRole==Followers的peers的size

5. hasMajority(Collection\<RaftPeerId>, RaftPeerId) && majorityRejectVotes(Collection\<RaftPeerId>)

   ```java
     boolean hasMajority(Collection<RaftPeerId> others, RaftPeerId selfId) {
       Preconditions.assertTrue(!others.contains(selfId));
       int num = 0;
       if (contains(selfId)) {
         num++;
       }
       for (RaftPeerId other : others) {
         if (contains(other)) {
           num++;
         }
       }
       return num > size() / 2;
     }
   
     boolean majorityRejectVotes(Collection<RaftPeerId> rejected) {
       int num = size();
       for (RaftPeerId other : rejected) {
         if (contains(other)) {
           num --;
         }
       }
       return num <= size() / 2;
     }
   ```

   这个没什么好说的，还是强调一下size()方法，返回的是RaftPeerRole==Follower的peers的size，contains(RaftPeerId)test的也是RaftPeerRole==Follower的peers

## 2. RaftConfigurationImpl

持有两个PeerConfiguration对象，一个用来表示currentConf，一个用来表示oldConf，这里需要注意注释中表述的，只有当且仅当conf处在transitional的状态时，oldConf才不为null。同时，这个类时imuutable，即所有的field都被final修饰，class被final修饰。

```java
/**
 * The configuration of the raft cluster.
 * <p>
 * The configuration is stable if there is no on-going peer change. Otherwise,
 * the configuration is transitional, i.e. in the middle of a peer change.
 * <p>
 * The objects of this class are immutable.
 */
final class RaftConfigurationImpl implements RaftConfiguration
```

除去两个conf对象，RaftConfigurationImpl还有一个logEntryIndex实例变量，用来表示curConf在logEntry中的偏移量

* final PeerConfiguration oldConf
* final PeerConfiguration conf
* final long logEntryIndex

需要提一下isTransitional()和isStable()方法，入类描述中所说，当且仅当oldConf为null时，此对象为stable状态，否咋额为transitional状态

```java
  boolean isTransitional() {
    return oldConf != null;
  }

  /** Is this configuration stable, i.e. no on-going peer change? */
  boolean isStable() {
    return oldConf == null;
  }
```

## 3. ConfigurationManager

主要封装了一个map，key为logIndex，value为RaftConfigurationImpl对象，用来记录产生的所有RaftConfigurationImpl对象及其在log entry中的index，同时记录了initial的conf和current的conf

* RaftConfigurationImpl initialConf
* RaftConfigurationImpl currentConf
* NavigableMap<Long, RaftConfigurationImpl> configurations

