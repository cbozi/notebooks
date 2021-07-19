expirationTime计算

expirationTime计算的代码位于`packages/react-reconciler/src/ReactFiberExpirationTime.js`
```

export const NoWork = 0;
// TODO: Think of a better name for Never. The key difference with Idle is that
// Never work can be committed in an inconsistent state without tearing the UI.
// The main example is offscreen content, like a hidden subtree. So one possible
// name is Offscreen. However, it also includes dehydrated Suspense boundaries,
// which are inconsistent in the sense that they haven't finished yet, but
// aren't visibly inconsistent because the server rendered HTML matches what the
// hydrated tree would look like.
export const Never = 1;
// Idle is slightly higher priority than Never. It must completely finish in
// order to be consistent.
export const Idle = 2;
// Continuous Hydration is slightly higher than Idle and is used to increase
// priority of hover targets.
export const ContinuousHydration = 3;
export const Sync = MAX_SIGNED_31_BIT_INT;
export const Batched = Sync - 1;

const UNIT_SIZE = 10;
const MAGIC_NUMBER_OFFSET = Batched - 1;

// 1 unit of expiration time represents 10ms.
export function msToExpirationTime(ms: number): ExpirationTime {
  // Always subtract from the offset so that we don't clash with the magic number for NoWork.
  return MAGIC_NUMBER_OFFSET - ((ms / UNIT_SIZE) | 0);
}

export function expirationTimeToMs(expirationTime: ExpirationTime): number {
  return (MAGIC_NUMBER_OFFSET - expirationTime) * UNIT_SIZE;
}

function ceiling(num: number, precision: number): number {
  return (((num / precision) | 0) + 1) * precision;
}

function computeExpirationBucket(
  currentTime,
  expirationInMs,
  bucketSizeMs,
): ExpirationTime {
  return (
    MAGIC_NUMBER_OFFSET -
    ceiling(
      MAGIC_NUMBER_OFFSET - currentTime + expirationInMs / UNIT_SIZE,
      bucketSizeMs / UNIT_SIZE,
    )
  );
}
```

`ExpirationTime`这个类型是可以与时间（ms)进行换算的，计算公式为
`MAGIC_NUMBER_OFFSET - ((ms / UNIT_SIZE) | 0);`。其中的`MAGIC_NUMBER_OFFSET`为最大的31位有符号整数-2。`UNIT_SIZE`为10，表示最小单位是10ms。

`computeAsyncExpiration`,`computeInteractiveExpiration`等函数会调用`computeExpirationBucket`计算expirationTime，使用不同的`expirationInMs`,`buketSizeMs`参数。以`computeAsyncExpiration`为例，

```
export const LOW_PRIORITY_EXPIRATION = 5000;
export const LOW_PRIORITY_BATCH_SIZE = 250;

export function computeAsyncExpiration(
  currentTime: ExpirationTime,
): ExpirationTime {
  return computeExpirationBucket(
    currentTime,
    LOW_PRIORITY_EXPIRATION,
    LOW_PRIORITY_BATCH_SIZE,
  );
}
```
公式为`（1073741823 - 2） - ceil(1073741823 - 2 - currentTime + 5000 / 10, 250 / 10)` 

意为：当前时间的5000ms之后超时，且无视250ms内的误差。`expirationInMs`越大，代表这个任务不急着完成，优先级越低。

为什么要用`MAGIC_NUMBER_OFFSET`减去当前时间呢，因为原来的计算方式是`expirationTime`越小，优先级越高。后来开发者改变了计算方法，使`expirationTime`越大，优先级越高，这样更符合直觉。`MAGIC_NUMBER_OFFSET`原本为2，加上这么一个数字是为了使计算得到的`expirationTime`不与`Nowork`和`Sync`重复。
