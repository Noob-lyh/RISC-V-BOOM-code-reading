# types

定义Boom模块的基本类型BoomModule类和Boom模块的基本端口类型BoomBundle类。

两者都从rocket已实现的类型继承，并且混入了Boom自己定义的HasBoomCoreParameters特质（见parameters.scala）。

```scala
//******************************************************************************
// Copyright (c) 2018 - 2018, The Regents of the University of California (Regents).
// All Rights Reserved. See LICENSE and LICENSE.SiFive for license details.
//------------------------------------------------------------------------------

package boom.commo
import chisel3._
import freechips.rocketchip.config.Parameters


/**
 * BOOM module that is used to add parameters to the module
 */
  abstract class BoomModule(implicit p: Parameters) extends freechips.rocketchip.tile.CoreModule
    with HasBoomCoreParameters


/**
 * BOOM bundle used to add parameters to the object/class/trait/etc
 */
  class BoomBundle(implicit val p: Parameters) extends freechips.rocketchip.util.ParameterizedBundle
    with HasBoomCoreParameters
```