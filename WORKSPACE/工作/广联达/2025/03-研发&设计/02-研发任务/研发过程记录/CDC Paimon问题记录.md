~~~ref
startPersistence含义太模糊
starMode需要再优化一下，难以理解
当前catalog有地方直接强制转了FileStoreTable，这里会不会有问题？  PaimonSinkContext.java line:262 271
paimon的bucket如何设置？在cdc里面有什么影响
paimon有专门的obsFileIO，可以考虑使用，或许兼容性更好
PaimonChangeConsumer:singleConsumer方法歧义太大
touchBucket是什么作用
~~~

