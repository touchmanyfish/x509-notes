# Max Path Building Attempts

在构建证书过程中，对某个证书进行了一次“快速验证+验签”组合操作叫做一次“attempts”。

Max Path Building Attempts用于限制attempts的次数。

## “边构建边验证” vs “先构建在验证”

“边构建边验证”其实是将一定会进行的 **“证书验签操作”** 提前，顺便还产生了 **提前减枝** 的好处。